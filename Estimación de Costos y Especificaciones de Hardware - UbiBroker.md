# Estimación de Costos y Especificaciones de Hardware - UbiBroker

Este documento detalla las configuraciones de hardware recomendadas y los costos operativos estimados para los diferentes entornos de despliegue de la arquitectura de microservicios UbiBroker (Spring Boot, Golang, PostgreSQL, Redis, RabbitMQ).

---

## 1. Entornos de Demostración (Cloud - Microsoft Azure)

El objetivo de estos entornos es mantener un bajo costo operativo asegurando un rendimiento fluido durante las presentaciones a clientes.

### Opción A: IaaS - Máquina Virtual "All-in-One" (Recomendada para Demos)
Despliegue unificado utilizando `docker-compose`.

| Componente | Especificación Técnica | Costo Mensual (24/7) | Costo Optimizado (40h/mes) |
| :--- | :--- | :--- | :--- |
| **Computo (VM)** | `Standard_B4ms` (4 vCPUs, 16 GB RAM) | ~$121.00 USD | ~$6.50 USD |
| **Almacenamiento** | Disco Premium SSD de 64 GB | ~$10.00 USD | ~$10.00 USD |
| **Red** | IP Pública Estática + Ancho de banda | ~$5.00 USD | ~$5.00 USD |
| **Total Estimado** | | **~$136.00 USD** | **~$21.50 USD** |

> **Nota de Arquitectura:** El "Costo Optimizado" asume la configuración de Auto-Shutdown en Azure, encendiendo la VM únicamente durante el horario de demostraciones.

### Opción B: PaaS - Enfoque Cloud-Native
Despliegue utilizando servicios gestionados para mostrar una topología de nube avanzada.

| Componente | Especificación Técnica (Tier Básico) | Costo Mensual Estimado |
| :--- | :--- | :--- |
| **Contenedores** | Azure Container Apps (Serverless) | ~$40.00 USD |
| **Base de Datos** | PostgreSQL Flexible (`Burstable B1ms` - 1 vCPU, 2 GB RAM)| ~$15.00 USD |
| **Caché** | Azure Cache for Redis (Tier `C0` - 250 MB) | ~$16.00 USD |
| **Mensajería** | Contenedor RabbitMQ en ACA | ~$10.00 USD |
| **Total Estimado**| | **~$81.00 USD** |

---

## 2. Entorno de Producción (Local / On-Premises)

Para el entorno de producción local gestionado con **Docker Swarm**, los costos cambian de un modelo OPEX (suscripción mensual) a un modelo CAPEX (compra de hardware físico). 

La siguiente es la topología para un **Clúster de Alta Disponibilidad (HA)**.

### Nodos Worker (Capa de Aplicación y Orquestación)
Aloja la red de enrutamiento y los microservicios sin estado (Java, Go, API Gateway). Se requieren mínimo 2 servidores físicos para HA.

| Recurso | Configuración Recomendada (Por Servidor) | Justificación |
| :--- | :--- | :--- |
| **Procesador** | 8 a 16 Núcleos (Cores) | Manejo de alta concurrencia (hilos JVM de Spring Boot). |
| **Memoria RAM** | 32 GB  | Reserva de memoria holgada para el Garbage Collector de Java. |
| **Almacenamiento**| 500 GB SSD Enterprise | Sistema operativo y caché temporal de imágenes Docker. |

### Nodos de Datos (Capa de Persistencia)
Aloja los servicios con estado (PostgreSQL, Redis, RabbitMQ). Se requieren 2 servidores configurados en Activo/Pasivo o Clúster.

| Recurso | Configuración Recomendada (Por Servidor) | Justificación |
| :--- | :--- | :--- |
| **Procesador** | 8 Núcleos (Cores) | Consultas relacionales complejas y enrutamiento AMQP. |
| **Memoria RAM** | 32 GB a 64 GB RAM | Caché de índices de Postgres y persistencia en memoria pura de Redis. |
| **Almacenamiento**| 1 TB a 2 TB NVMe (RAID 1 o 10) | **Crítico:** Las bases de datos financieras exigen latencia de I/O en submilisegundos y tolerancia a fallos de disco. |

# Presupuesto de Memoria (RAM Mapping) - Servidor 32 GB

Este documento detalla la estrategia de asignación y limitación de memoria RAM para el despliegue de la arquitectura **UbiBroker** en un entorno de producción (nodo único o clúster base) con un límite físico de 32 GB de RAM, diseñado para soportar una concurrencia estimada de 100 usuarios simultáneos.

---

## 2.1 Distribución del Presupuesto (RAM Budget)

La siguiente tabla muestra la distribución estricta de la memoria para evitar el colapso del servidor por falta de recursos (Out Of Memory).

| Componente | Límite RAM Asignado | Porcentaje Aprox. | Justificación Arquitectónica |
| :--- | :--- | :--- | :--- |
| **Sistema Operativo + Docker** | 4.0 GB | 12.5% | Reserva inamovible para garantizar la estabilidad del kernel de Linux y los demonios de Swarm. |
| **PostgreSQL** | 6.0 GB | 18.7% | Asignación holgada para `shared_buffers` y manejo de >100 conexiones concurrentes sin tocar disco duro. |
| **Redis + RabbitMQ** | 2.0 GB | 6.2% | Redis consumirá <100 MB para caché transaccional. RabbitMQ utilizará el resto para el enrutamiento de eventos asíncronos. |
| **API Gateway / Nginx** | 2.0 GB | 6.2% | Manejo de conexiones entrantes, terminación SSL y enrutamiento hacia la red interna. |
| **Integration Service (Golang)**| 0.5 GB | 1.5% | Microservicio de alto rendimiento. En operación real consumirá entre 30-60 MB. Límite preventivo. |
| **Servicios Spring Boot (x5)** | 12.5 GB | 39.0% | *Auth, Order, Client, Document, Notif*. Límite duro de **2.5 GB por contenedor** para contener la JVM. |
| **Colchón de Seguridad** | 5.0 GB | 15.6% | Memoria libre no asignada para absorber picos inesperados de tráfico o procesos en segundo plano. |
| **TOTAL ASIGNADO** | **32.0 GB** | **100%** | |

---

## 2.2 Configuración de Contención (Límites Estrictos)

El presupuesto anterior es teórico hasta que se imponen límites a nivel de infraestructura. Java (Spring Boot), por su naturaleza, intentará consumir toda la RAM disponible si no se le restringe.

Para aplicar este presupuesto, se implementan dos capas de seguridad en el archivo `docker-compose.prod.yml`:

### Capa 1: Límite del Contenedor (cgroups de Linux)
Le indica a Docker Swarm el máximo absoluto permitido. Si el contenedor sobrepasa este límite, el **OOM Killer** de Linux lo detendrá inmediatamente (`Error 137`) para proteger al resto del servidor.

```yaml
    deploy:
      resources:
        limits:
          memory: 2560M # Límite duro de 2.5 GB
        reservations:
          memory: 1024M # 1.0 GB garantizados al inicializar

```
---

# Estrategia de Contención de Memoria y Resiliencia (UbiBroker)

Este documento detalla la configuración a nivel de aplicación (Capa 2) para el control de memoria de los microservicios Java y describe el comportamiento automatizado de la arquitectura ante desbordamientos (Plan de Contingencia).

---

## 2.3. Capa 2: Límite a Nivel de Aplicación (JVM Tuning)

Aunque el contenedor Docker tenga un límite duro de RAM asignado por el sistema operativo, la Máquina Virtual de Java (JVM) requiere instrucciones explícitas para no intentar reservar más memoria de la permitida. Si no se configura, la JVM asumirá que tiene acceso a toda la RAM del servidor físico.

Para garantizar que cada microservicio Spring Boot respete el presupuesto asignado y deje memoria libre para los procesos de red y el propio contenedor Linux, se implementa la variable de entorno `JAVA_OPTS` en el archivo de despliegue (`docker-compose.prod.yml`).

### Configuración en Docker Swarm:
```yaml
    environment:
      - JAVA_OPTS="-Xms1024m -Xmx1536m -XX:+UseG1GC"
```
### Justificación Técnica de los Parámetros:
* **-Xms1024m (Initial Heap Size):** Indica a la JVM que reserve inmediatamente 1 GB de RAM al arrancar. Esto evita que la aplicación pierda tiempo de CPU solicitando bloques de memoria al sistema operativo durante los primeros minutos de ejecución.

* **J-Xmx1536m (Maximum Heap Size):** Establece el límite absoluto para el almacenamiento interno de objetos en Java en 1.5 GB. Dado que el contenedor tiene un límite duro de 2.5 GB, esto garantiza que quede 1 GB libre para hilos nativos, buffers del sistema y el proceso base de Linux.

* **J-XX:+UseG1GC (Garbage-First Garbage Collector):** Activa el recolector de basura optimizado para entornos transaccionales. G1GC divide la memoria en regiones y limpia la basura en fracciones de milisegundos, evitando los bloqueos prolongados ("Stop-The-World pauses") que causarían lentitud en las operaciones de los ejecutivos.


# Plan de Contingencia: Resiliencia y Zero Downtime (UbiBroker)

Este documento describe el comportamiento automatizado de la arquitectura de UbiBroker ante fallos críticos de infraestructura o desbordamientos de memoria (Out-Of-Memory). El objetivo de este plan es garantizar la continuidad del negocio y mantener la experiencia del usuario intacta (Zero Downtime) durante incidentes aislados.

---

## 1. El Principio de Autosaneamiento (Self-Healing)

En una arquitectura de microservicios transaccionales diseñada para alta concurrencia (ej. 100 usuarios simultáneos), los fallos a nivel de contenedor no se tratan como emergencias manuales, sino como eventos esperados que el sistema debe resolver de forma autónoma.

La resiliencia del sistema se basa en no almacenar "estado" (sesiones de usuario, archivos temporales) dentro de los contenedores de procesamiento. Si un contenedor muere, no se pierde información crítica.

---

## 2. Flujo de Reacción ante Incidentes (Ej. Desbordamiento de RAM)

Cuando una anomalía provoca que un microservicio (como el `order-service` en Java) intente consumir más recursos de los que tiene asignados en su límite duro, la arquitectura ejecuta un protocolo de tres fases en cuestión de segundos:

### Fase A: Intervención del Sistema Operativo (Aislamiento)
Para evitar que un microservicio defectuoso consuma toda la memoria del servidor físico y provoque una caída general del sistema:
1. Las políticas de control de recursos del kernel de Linux (`cgroups`) detectan la infracción del límite de memoria asignado.
2. El sistema operativo invoca al **OOM Killer (Out-Of-Memory Killer)**.
3. Se envía una señal de terminación inmediata (`SIGKILL`) al proceso defectuoso. El contenedor es destruido al instante, registrando el evento con un **Error 137 (OOMKilled)** en los logs.

### Fase B: Recuperación Autónoma (Orquestación)
1. **Docker Swarm** monitorea constantemente el "estado deseado" de la infraestructura.
2. Al detectar que un contenedor ha sido destruido, evalúa la política de reinicio declarada en el despliegue:
   ```yaml
   restart_policy:
     condition: on-failure
     delay: 5s
     max_attempts: 3
3. El orquestador aprovisiona y despliega un nuevo contenedor limpio de forma automática. En el caso de Spring Boot, este proceso toma entre 15 y 20 segundos.
### Fase C: Aislamiento del Cliente (Zero Downtime)
El objetivo de negocio es que las operaciones de los ejecutivos no se vean interrumpidas:
1.Redundancia Activa: Gracias al despliegue en Alta Disponibilidad (replicas: 2), siempre hay al menos un contenedor de respaldo operando al 100% de su capacidad.
2.Balanceo de Carga Inteligente: Durante los segundos que toma reiniciar el contenedor caído, el Ingress Routing Mesh (la red de enrutamiento interno de Swarm) elimina la IP del nodo defectuoso.
3.Redirección Transparente: El 100% del tráfico de los usuarios se redirige instantáneamente hacia la réplica sobreviviente.


## 3. Infraestructura CI/CD y Registro de Imágenes (GitHub)

Costos asociados a la compilación y almacenamiento de las imágenes Docker (GitHub Actions + GitHub Container Registry).

| Plan | Costo Base | Incluye (Gratis mensual) | Costos por Excedente |
| :--- | :--- | :--- | :--- |
| **GitHub Free** | **$0.00 USD** | 500 MB Almacenamiento <br> 2,000 minutos CI/CD | $0.25 USD / GB extra <br> $0.008 USD / minuto extra |
| **GitHub Team** | **$4.00 USD / usuario** | 2 GB Almacenamiento <br> 3,000 minutos CI/CD | $0.25 USD / GB extra <br> $0.008 USD / minuto extra |

> **Estrategia de Optimización:** Para evitar sobrecostos en almacenamiento (debido al tamaño de las imágenes Java), se implementarán scripts de retención en GitHub Actions que eliminen versiones antiguas de las imágenes tras cada compilación exitosa, manteniendo solo las últimas 3 releases estables.

# Estimación de Costos - Azure Container Registry (ACR)

Este documento detalla los costos asociados con el almacenamiento y la distribución de nuestras imágenes Docker utilizando el registro privado nativo de Microsoft Azure. Es una alternativa robusta para centralizar las imágenes de nuestros microservicios de negocio.

---

## 4. Modelos de Precios de ACR (Tiers de Servicio)

Azure Container Registry se factura bajo una tarifa fija diaria basada en el "Tier" o nivel de servicio seleccionado. Cada nivel incluye una capacidad de almacenamiento base. Si se supera ese almacenamiento, se factura un costo incremental por gigabyte extra.

| Tier / Nivel | Tarifa Fija Diaria | Costo Mensual Aprox. | Almacenamiento Incluido | Operaciones Incluidas (Lectura/Escritura) |
| :--- | :--- | :--- | :--- | :--- |
| **Básico (Basic)** | ~$0.167 USD | **~$5.00 USD** | 10 GB | 10,000 Read / 2,000 Write (por día) |
| **Estándar (Standard)** | ~$0.667 USD | **~$20.00 USD** | 100 GB | 100,000 Read / 20,000 Write (por día) |
| **Premium** | ~$1.667 USD | **~$50.00 USD** | 500 GB | 500,000 Read / 50,000 Write (por día) |

### Costos por Excedentes (Pay-as-you-go)
* **Almacenamiento Adicional:** **$0.003 USD al día por GB** (aproximadamente **$0.10 USD por GB al mes**).
* **Ancho de banda (Egress Data):** La subida de imágenes a Azure es gratuita. La descarga de imágenes desde Azure hacia tus servidores locales (On-Premises) tiene un costo de ancho de banda de red de aproximadamente **$0.08 USD por GB** (después de los primeros 100 GB gratuitos al mes).

---

## 5. Recomendación para UbiBroker

### Para Demos y Pruebas iniciales: **Tier Básico**
* **Análisis de Capacidad:** Nuestras imágenes optimizadas ocupan aproximadamente:
  * Microservicios Java (Spring Boot) c/u: ~150 MB - 200 MB
  * Servicio de Integración (Golang): ~20 MB
  * Servicios de Infraestructura (Postgres, Redis, RabbitMQ): ~400 MB en total
* **Total del ecosistema por versión:** Menos de 1.5 GB.
* El **Tier Básico (10 GB)** es más que suficiente para almacenar hasta 5 o 6 versiones completas e históricas de toda nuestra arquitectura sin generar cargos por almacenamiento extra.

---

## 6. Estrategia de Conexión con nuestro Pipeline (GitHub Actions)

Al implementar ACR, el flujo de entrega continua se ejecuta de la siguiente manera sin necesidad de almacenar datos en GitHub:

1. **GitHub Actions** compila el código de Java o Go.
2. Construye la imagen Docker en los servidores de GitHub.
3. Hace un *Login* seguro en Azure usando una entidad de servicio (Service Principal).
4. Hace un `docker push` directamente a `ubibroker.azurecr.io`.
5. Durante el despliegue en producción (Docker Swarm local), nuestros servidores físicos se autentican contra Azure y descargan las imágenes actualizadas de forma segura.
