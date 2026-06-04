# Plan de Implementación y Hoja de Ruta (Roadmap) - Proyecto UbiBroker
**Estrategia de Desarrollo Acelerado mediante IA (Cursor / Google Antigravity)**

Este documento traza la ruta de ejecución para construir la arquitectura de microservicios (Java, Go, Postgres, Redis, RabbitMQ) utilizando agentes de inteligencia artificial para maximizar la productividad y reducir el *Time-to-Market*. 

Al delegar la creación de infraestructura y *boilerplate* a la IA, el equipo de arquitectura puede centrarse exclusivamente en la lógica de negocio y la resiliencia del sistema.

---

## 1. Hoja de Ruta de Alto Nivel (Roadmap)

El proyecto se divide en 4 grandes hitos (Milestones) secuenciales:

* **Hito 1: Cimientos y Mock (La Red de Seguridad):** Establecer la infraestructura base en Docker y emular el core bancario/bursátil para no depender de sistemas externos lentos durante el desarrollo.
* **Hito 2: Capa Anticorrupción (Velocidad):** Desarrollar el servicio en Golang con Redis para interceptar y acelerar las peticiones de lectura.
* **Hito 3: Cerebro Transaccional (Reglas de Negocio):** Implementar los microservicios en Java Spring Boot bajo los principios de *Domain-Driven Design* (Auth, Order, Client).
* **Hito 4: Asincronía y Despliegue (Robustez):** Conectar RabbitMQ para flujos de órdenes, configurar migraciones automatizadas y desplegar el clúster local con Docker Swarm.

---

## 2. Plan de Implementación Acelerado (Cronograma Estimado: 3 Semanas)

*Nota: Los tiempos asumen la asistencia continua de un agente de IA (como Antigravity IDE) para la generación de código, configuraciones YAML y pruebas unitarias.*

### Fase 1: Cimientos de Infraestructura y Orquestación
**Duración:** Días 1 - 3
**Objetivo:** Tener el entorno unificado corriendo en local.

| Tarea Técnica | Entregable | Aceleración por IA (Antigravity/Cursor) |
| :--- | :--- | :--- |
| Configuración de Orquestación | `docker-compose.yml` raíz funcional. | Generación automática de topología de redes, volúmenes y puertos. |
| Aprovisionamiento de Datos | Contenedores de Postgres, Redis, RabbitMQ. | Creación de scripts de inicialización de bases de datos y usuarios. |
| Creación del Virtual Broker | Servicio `virtual-broker-mock` en Java. | Generación de endpoints REST falsos y datos semilla (JSON) a partir de descripciones de negocio. |

### Fase 2: Construcción de la Capa Anticorrupción
**Duración:** Días 4 - 7
**Objetivo:** Interceptar tráfico y cachear respuestas en milisegundos.

| Tarea Técnica | Entregable | Aceleración por IA (Antigravity/Cursor) |
| :--- | :--- | :--- |
| Inicialización de Golang | `integration-service-go` estructurado. | Creación de módulos Go, configuración de Fiber/Gin y `Dockerfile` optimizado. |
| Integración con Redis | Patrón Cache-Aside implementado. | Autocompletado de la lógica de conexión, TTL y manejo de errores (Cache Miss/Hit). |
| Enrutamiento y Proxies | Peticiones fluyendo de Go hacia el Mock. | Generación de clientes HTTP concurrentes (Goroutines) para consumo de APIs. |

### Fase 3: Core de Negocio (Microservicios Spring Boot)
**Duración:** Días 8 - 14
**Objetivo:** Implementar la lógica transaccional y la seguridad.

| Tarea Técnica | Entregable | Aceleración por IA (Antigravity/Cursor) |
| :--- | :--- | :--- |
| Seguridad y Autenticación | `auth-service` con emisión de JWT. | Implementación rápida de Spring Security y filtros de validación de tokens. |
| Dominio de Órdenes | `order-service` con base de datos propia. | Creación de Entidades JPA, Repositorios, Controladores y migraciones Flyway/Liquibase. |
| Dominio de Clientes | `client-service` gestionando perfiles. | Mapeo de DTOs y validaciones de entrada basadas en reglas descritas en lenguaje natural. |

### Fase 4: Mensajería Asíncrona y Preparación a Producción
**Duración:** Días 15 - 21
**Objetivo:** Desacoplar servicios, afinar el rendimiento y preparar el clúster.

| Tarea Técnica | Entregable | Aceleración por IA (Antigravity/Cursor) |
| :--- | :--- | :--- |
| Eventos con RabbitMQ | Productores y Consumidores AMQP conectados. | Configuración de colas, *exchanges* y *bindings* a partir de descripciones de flujos (ej. "Orden Creada"). |
| Contención de Recursos | Límites de RAM implementados (`-Xmx`). | Cálculo automático de `cgroups` y parámetros de JVM para ajustarse al hardware objetivo. |
| Transición a Swarm | `docker-compose.prod.yml` listo. | Conversión de la topología local a directivas `deploy`, políticas de reinicio y *Zero Downtime*. |
| Observabilidad Base | Archivos de logs estandarizados. | Instrumentación automática del código con OpenTelemetry SDKs para Go y Java. |

---

## 3. Beneficios Directos del Desarrollo Agentic (Agent-First)

1.  **Reducción de Deuda Técnica:** Las herramientas como Antigravity garantizan que el código generado siga patrones de diseño modernos (Hexagonal, DDD) desde el día cero, evitando refactorizaciones costosas.
2.  **Resolución de Errores en Tiempo Real:** En lugar de depurar un error de red de Docker (ej. "Connection Refused") durante horas, el agente analiza los `docker logs` y corrige el orden de arranque de forma autónoma.
3.  **Foco en el Valor:** El 80% del tiempo del Arquitecto/Desarrollador se invierte en definir flujos de negocio y validar integraciones, mientras la IA asume el 20% operativo pesado (escribir `Dockerfiles`, mapear entidades a base de datos, configurar *middlewares*).
