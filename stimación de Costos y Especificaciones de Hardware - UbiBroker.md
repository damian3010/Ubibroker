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
| **Memoria RAM** | 32 GB a 64 GB RAM | Reserva de memoria holgada para el Garbage Collector de Java. |
| **Almacenamiento**| 500 GB SSD Enterprise | Sistema operativo y caché temporal de imágenes Docker. |

### Nodos de Datos (Capa de Persistencia)
Aloja los servicios con estado (PostgreSQL, Redis, RabbitMQ). Se requieren 2 servidores configurados en Activo/Pasivo o Clúster.

| Recurso | Configuración Recomendada (Por Servidor) | Justificación |
| :--- | :--- | :--- |
| **Procesador** | 8 Núcleos (Cores) | Consultas relacionales complejas y enrutamiento AMQP. |
| **Memoria RAM** | 32 GB a 64 GB RAM | Caché de índices de Postgres y persistencia en memoria pura de Redis. |
| **Almacenamiento**| 1 TB a 2 TB NVMe (RAID 1 o 10) | **Crítico:** Las bases de datos financieras exigen latencia de I/O en submilisegundos y tolerancia a fallos de disco. |

---

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
