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
