# Plan de Despliegue y CI/CD Controlado (UbiBroker)

Este documento define la estrategia de Integración Continua (CI) mediante GitHub Actions y el Despliegue Controlado (CD) en el clúster físico de Docker Swarm. La filosofía operativa es **Pull-Based**: GitHub compila y almacena los artefactos, pero el servidor de producción es quien decide cuándo descargarlos y aplicarlos mediante un script controlado.

---

## 1. Estrategia de Optimización (Monorepo Path Filtering)

Para evitar que un cambio en el `integration-service-go` desencadene la compilación innecesaria de los 5 microservicios de Java (consumiendo minutos y almacenamiento), implementaremos **Filtros de Ruta (Path Filtering)** en GitHub Actions.

Cada microservicio tendrá su propio archivo de *workflow* independiente en la carpeta `.github/workflows/`.

### Ejemplo: Pipeline exclusivo para el `order-service`
*(Archivo: `.github/workflows/build-order-service.yml`)*

```yaml
name: Build and Push - Order Service

on:
  push:
    branches:
      - main
    paths:
      - 'order-service/**' # LA MAGIA ESTÁ AQUÍ: Solo se ejecuta si hay cambios en esta carpeta
      - '.github/workflows/build-order-service.yml'

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Setup Java 17
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'temurin'

      # Opción A: Autenticación para GitHub Container Registry (GHCR)
      - name: Login to GitHub Container Registry
        uses: docker/login-action@v2
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      # Opción B: Autenticación para Azure Container Registry (ACR)
      # - name: Login to Azure Container Registry
      #   uses: docker/login-action@v2
      #   with:
      #     registry: ubibroker.azurecr.io
      #     username: ${{ secrets.ACR_USERNAME }}
      #     password: ${{ secrets.ACR_PASSWORD }}

      - name: Build and Push Docker Image
        uses: docker/build-push-action@v4
        with:
          context: ./order-service
          push: true
          tags: |
            ghcr.io/tu-organizacion/ubibroker-orders:latest
            ghcr.io/tu-organizacion/ubibroker-orders:${{ github.sha }}
```

Nota de Arquitectura: Al etiquetar con latest y con el hash del commit (${{ github.sha }}), mantenemos un historial rastreable. Si la versión latest falla en producción, podemos volver exactamente a la versión del commit anterior.

# 2. Gestión de Registros de Imágenes (Artifacts)

El registro de imágenes actúa como la "biblioteca" centralizada de nuestra arquitectura. Tras la fase de compilación en GitHub Actions, las imágenes Docker (nuestros artefactos) deben almacenarse en un entorno seguro, versionado y accesible para el clúster de producción.

La elección del registro depende directamente de la estrategia de gobernanza de datos y la ubicación de los servidores físicos.

---

## Estrategias de Almacenamiento

### A. GitHub Container Registry (GHCR)
Es la opción nativa si tu código fuente ya reside en GitHub. 
* **Ventaja:** Integración transparente, sin necesidad de autenticaciones externas complejas en el pipeline.
* **Modelo de acceso:** Las imágenes se identifican como `ghcr.io/tu-organizacion/ubibroker-servicio:etiqueta`.
* **Caso de uso:** Ideal para entornos de desarrollo, demos o equipos que buscan un *Time-to-Market* inmediato sin gestionar infraestructura de registros adicional.

### B. Azure Container Registry (ACR)
Es la opción de grado empresarial para una gestión centralizada y segura.
* **Ventaja:** Permite replicación geográfica, escaneo automático de vulnerabilidades y una gestión de permisos granular mediante Microsoft Entra ID (antes Azure AD).
* **Modelo de acceso:** Las imágenes se identifican como `ubibroker.azurecr.io/nombre-servicio:etiqueta`.
* **Caso de uso:** Altamente recomendado para producción en entornos financieros que exigen auditorías constantes sobre qué versiones de software están desplegadas en los servidores.

---

## Ciclo de Vida del Artefacto

Para mantener la eficiencia y los costos bajo control, implementamos una política de gestión de artefactos basada en tres pilares:

1. **Versionado Inmutable:** Cada imagen no solo lleva el tag `latest`, sino también el `SHA` (hash único) del commit que la generó. Esto permite un *rollback* (volver atrás) instantáneo a cualquier punto previo de la historia del proyecto.
2. **Promoción de Imágenes:** Las imágenes se etiquetan siguiendo un flujo lógico:
   * `build-id`: Etiquetada con el hash del commit para trazabilidad.
   * `latest`: Etiquetada como la versión candidata actual para el servidor de producción.
   * `release-x.x.x`: Etiquetada con versiones semánticas para marcar hitos de negocio significativos.
3. **Política de Purga (Retención):** Para evitar costos innecesarios de almacenamiento (ya sea en GHCR o ACR), el pipeline incluye un paso de "limpieza":
   * Solo se conservan las **últimas 3 versiones** etiquetadas como `latest`.
   * Todas las imágenes intermedias (hash del commit) sin uso durante más de 30 días se eliminan automáticamente.

---

## Seguridad en el Acceso (Pull-Based)

Para mantener la seguridad, el despliegue en el clúster de producción nunca expone credenciales en el archivo `docker-compose.prod.yml`.

* **Autenticación Temporal:** El script de despliegue (`deploy_ubibroker.sh`) realiza un `docker login` previo utilizando variables de entorno seguras.
* **Pull Privado:** Docker Swarm utiliza la bandera `--with-registry-auth`, que transfiere temporalmente las credenciales autenticadas del nodo *Manager* a los nodos *Workers* de forma cifrada, permitiendo que cualquier nodo físico del clúster pueda descargar la imagen privada sin que la contraseña se guarde permanentemente en disco.


# 3. Despliegue en Producción (El Script Pull-Based)

En arquitecturas empresariales, el despliegue no debe ser un proceso "automático" que reaccione a cualquier *commit*. Para UbiBroker, hemos optado por un modelo **Pull-Based** (basado en tracción): GitHub Actions prepara el artefacto, pero el servidor de producción es quien decide cuándo aplicar el cambio.

Este modelo garantiza una ventana de mantenimiento controlada y evita actualizaciones sorpresa en horarios no autorizados.

---

## El Proceso de Despliegue Controlado

El despliegue se ejecuta mediante un script de Bash (`deploy_ubibroker.sh`) diseñado para ser ejecutado por un administrador en el servidor de producción (Nodo Manager de Docker Swarm).

### Pasos Técnicos del Despliegue:

1. **Autenticación Segura:** El script se autentica contra el registro (GHCR o ACR) mediante variables de entorno seguras, garantizando que el servidor tenga permisos de lectura sobre las imágenes privadas.
2. **Pull Forzado:** Se descarga la etiqueta `latest` de cada microservicio. Esto asegura que, aunque la imagen ya exista localmente, el servidor fuerce la actualización a la versión recién compilada por GitHub Actions.
3. **Aplicación del Stack:** Se invoca a `docker stack deploy`. Gracias a la configuración de `update_config` en el archivo YAML, Docker Swarm ejecuta un despliegue **Rolling Update** (actualización gradual), reemplazando las réplicas antiguas por las nuevas sin interrumpir el servicio.
4. **Limpieza (Garbage Collection):** Se eliminan las imágenes huérfanas (`dangling images`) para liberar espacio en disco, manteniendo el servidor limpio y eficiente.

---

## Guion del Script de Despliegue (`deploy_ubibroker.sh`)

```bash
#!/bin/bash

# Salir si ocurre un error
set -e

echo "--- Iniciando Despliegue Controlado de UbiBroker ---"

# [1] Autenticación en el registro privado
echo "[1/4] Autenticando en el registro..."
echo $REGISTRY_PASSWORD | docker login $REGISTRY_URL -u $REGISTRY_USER --password-stdin

# [2] Descarga forzada de imágenes (Pull)
echo "[2/4] Descargando imágenes actualizadas..."
docker pull $REGISTRY_URL/ubibroker-orders:latest
docker pull $REGISTRY_URL/ubibroker-auth:latest
docker pull $REGISTRY_URL/ubibroker-integration-go:latest
# (Continuar con el resto de servicios)

# [3] Despliegue mediante Docker Swarm
# --with-registry-auth permite a los nodos workers acceder al registro privado
echo "[3/4] Desplegando en Swarm con estrategia Rolling Update..."
docker stack deploy --with-registry-auth -c docker-compose.prod.yml ubibroker

# [4] Optimización de recursos
echo "[4/4] Limpiando artefactos antiguos..."
docker image prune -a -f

echo "--- Despliegue finalizado exitosamente ---"
```
Ejecución Operativa
## Para aplicar una actualización, el administrador simplemente accede por SSH al servidor y ejecuta:
```bash
chmod +x deploy_ubibroker.sh
./deploy_ubibroker.sh
```
