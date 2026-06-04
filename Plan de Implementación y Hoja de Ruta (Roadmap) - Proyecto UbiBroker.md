# Plan de Implementación y Hoja de Ruta (Roadmap) - Proyecto UbiBroker
**Estrategia de Desarrollo Asistido por IA (6 Semanas)**

Este documento traza la ruta de ejecución para construir la arquitectura de microservicios (Java, Go, Postgres, Redis, RabbitMQ). Aunque el uso de agentes de inteligencia artificial (Cursor / Antigravity) acelera dramáticamente la escritura de código, este cronograma de 6 semanas asegura el tiempo necesario para la validación arquitectónica, pruebas de estrés y endurecimiento de seguridad (*Security Hardening*).

---

## 1. Hoja de Ruta de Alto Nivel (Milestones)

* **Fase 1 (Semana 1): Cimientos y Emulación.** Orquestación base y Virtual Broker Mock.
* **Fase 2 (Semana 2): Capa Anticorrupción.** Microservicio en Golang y caché en Redis.
* **Fase 3 (Semanas 3 y 4): Cerebro Transaccional.** Microservicios Spring Boot y reglas de negocio.
* **Fase 4 (Semanas 5 y 6): Asincronía, Observabilidad y Producción.** RabbitMQ, Stack PLG y clúster Swarm.

---

## 2. Cronograma Detallado de Implementación

### Fase 1: Cimientos de Infraestructura y Orquestación Base
**Duración:** Semana 1
**Objetivo:** Tener el entorno unificado corriendo en local y el sistema core emulado.

| Tarea Técnica | Entregable | Apoyo de IA (Antigravity/Cursor) |
| :--- | :--- | :--- |
| Orquestación Inicial | `docker-compose.yml` base funcional. | Generación automática de topología de redes, volúmenes y puertos seguros. |
| Aprovisionamiento de Datos | Contenedores de Postgres, Redis, RabbitMQ. | Creación de scripts SQL de inicialización (schemas, usuarios, roles). |
| Virtual Broker (Mock) | Servicio `virtual-broker-mock` operativo. | Generación de endpoints REST y JSON semilla basados en las especificaciones del core bancario. |
| Pruebas de Red | Comunicación entre contenedores verificada. | Diagnóstico de errores DNS internos de Docker. |

### Fase 2: Construcción de la Capa Anticorrupción (Velocidad)
**Duración:** Semana 2
**Objetivo:** Interceptar el tráfico de lectura y cachear respuestas en submilisegundos.

| Tarea Técnica | Entregable | Apoyo de IA (Antigravity/Cursor) |
| :--- | :--- | :--- |
| Estructura en Go | `integration-service-go` configurado. | Configuración de Fiber/Gin, *middlewares* y `Dockerfile` multi-etapa (imagen < 20MB). |
| Integración Redis | Patrón *Cache-Aside* implementado. | Autocompletado de lógica de conexión, TTL y control de fallos (Cache Miss/Hit). |
| Concurrencia | Enrutamiento HTTP hacia el Mock. | Generación de clientes HTTP seguros utilizando *Goroutines* y *Channels*. |
| Pruebas de Carga | Benchmarks del servicio Go. | Creación de scripts en K6 o JMeter para saturar el endpoint y validar la caché. |

### Fase 3: Core de Negocio (Domain-Driven Design en Java)
**Duración:** Semanas 3 y 4
**Objetivo:** Implementar la lógica transaccional, seguridad y persistencia relacional.

| Tarea Técnica | Entregable | Apoyo de IA (Antigravity/Cursor) |
| :--- | :--- | :--- |
| Identidad y Acceso | `auth-service` con emisión de JWT. | Implementación rápida de Spring Security y filtros de validación de tokens. |
| Dominio de Órdenes | `order-service` operativo. | Creación de Entidades JPA, Repositorios, Controladores y migraciones Flyway/Liquibase. |
| Dominios de Soporte | `client-service` y `document-service`. | Mapeo de DTOs y validaciones de entrada basadas en reglas descritas en lenguaje natural. |
| Contención de Memoria | JVM ajustada para alta concurrencia. | Cálculo de parámetros `-Xmx`, `-Xms` y asignación de `cgroups` de Docker. |

### Fase 4: Asincronía, Observabilidad y Despliegue en Clúster
**Duración:** Semanas 5 y 6
**Objetivo:** Desacoplar servicios, auditar la plataforma y lanzar a producción.

| Tarea Técnica | Entregable | Apoyo de IA (Antigravity/Cursor) |
| :--- | :--- | :--- |
| Eventos con RabbitMQ | Productores y Consumidores AMQP conectados. | Configuración de colas, *exchanges* y *bindings* para flujos asíncronos (ej. "Orden Ejecutada"). |
| Instrumentación OTel | OpenTelemetry integrado en Go y Java. | Inyección de librerías para trazabilidad distribuida sin alterar la lógica de negocio. |
| CI/CD Pipeline | GitHub Actions configurado. | Scripts para compilación automática, purga de imágenes antiguas y subida a registro (GHCR o ACR). |
| Transición a Swarm | `docker-compose.prod.yml` final. | Implementación de directivas `deploy`, *placement constraints* y despliegue *Zero Downtime*. |

---

## 3. Ventaja Estratégica del Plan a 6 Semanas

Distribuir el proyecto en 6 semanas utilizando herramientas de IA permite un equilibrio perfecto entre **velocidad de escritura** y **calidad de arquitectura**:
1. **Foco en el Diseño, no en la Sintaxis:** La IA asume el trabajo pesado de codificar los *boilerplates* y configuraciones YAML, permitiendo al equipo dedicar las semanas 3 y 4 puramente a refinar las reglas de negocio críticas.
2. **Espacio para el Caos (Chaos Engineering):** Las semanas 5 y 6 ofrecen un margen de tiempo vital para simular caídas de nodos, saturación de RAM (pruebas de OOMKilled) y validar que las políticas de *Self-Healing* (Autosaneamiento) de Swarm funcionen antes de recibir tráfico de usuarios reales.
