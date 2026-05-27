# 🏛️ Documento de Arquitectura de Software (SAD) - UbiBroker

**Proyecto:** UbiBroker
**Fecha:** 27 de Mayo de 2026
**Versión:** 1.0.0

---

## 1. Resumen Ejecutivo
UbiBroker es una plataforma web transaccional diseñada para interactuar con el core de una casa de bolsa (Virtual Broker). La plataforma permite a clientes registrados consultar posiciones consolidadas, detalles de productos, calendarios de vencimientos y crear órdenes/reinversiones. Además, incluye un panel de Backoffice para que ejecutivos aprueben operaciones mediante un flujo "Maker-Checker". 

La arquitectura está basada en microservicios contenerizados (Docker), garantizando alta disponibilidad, separación de responsabilidades y tiempos de respuesta en milisegundos mediante el uso de cachés (Redis) y procesamiento asíncrono (RabbitMQ).

---

## 2. Stack Tecnológico
* **Backend Transaccional:** Spring Boot (Java)
* **Backend de Alto Rendimiento:** Golang
* **Bases de Datos Relacionales:** PostgreSQL (Patrón Database-per-Service)
* **Caché en Memoria:** Redis
* **Mensajería Asíncrona:** RabbitMQ
* **Almacenamiento de Archivos:** MinIO / Amazon S3
* **Observabilidad:** Stack PLG (Promtail, Loki, Tempo, Grafana) + OpenTelemetry

---

## 3. Catálogo de Microservicios

| Microservicio | Tecnología | Base de Datos | Responsabilidad Principal |
| :--- | :--- | :--- | :--- |
| **API Gateway** | Spring Cloud / Nginx | N/A | Único punto de entrada. Enrutamiento y validación inicial de JWT. |
| **Auth Service** | Spring Boot | PostgreSQL | Gestión de identidades, login, recuperación de contraseñas, JWT y OTP. |
| **Client Service** | Spring Boot | PostgreSQL | Mantiene el perfil local del usuario y valida RIF contra Virtual Broker. |
| **Integration Service** | Golang | Redis | Capa anticorrupción (BFF). Lectura masiva ultrarrápida del core de VB. |
| **Order Service** | Spring Boot | PostgreSQL | Motor transaccional. Gestiona el ciclo de vida de órdenes y aprobaciones. |
| **Document Service** | Spring Boot | MinIO (S3) | Worker asíncrono para generación y almacenamiento de PDFs (Estados de cuenta). |
| **Notification Service**| Spring Boot | PostgreSQL | Centro omnicanal. Envía WebSockets, Emails y guarda buzón de alertas. |

---

## 4. Diagrama de Arquitectura Global

```mermaid
graph TD
    %% Definición de estilos
    classDef spring fill:#6db33f,stroke:#fff,stroke-width:2px,color:#fff;
    classDef go fill:#00add8,stroke:#fff,stroke-width:2px,color:#fff;
    classDef db fill:#336791,stroke:#fff,stroke-width:2px,color:#fff;
    classDef broker fill:#ff6600,stroke:#fff,stroke-width:2px,color:#fff;
    classDef obs fill:#fca311,stroke:#fff,stroke-width:2px,color:#2b2b2b;
    classDef core fill:#2b2b2b,stroke:#fff,stroke-width:2px,color:#fff;

    Client[💻 Frontend Cliente / Admin] -->|HTTPS / REST| Gateway(API Gateway)
    Client -.->|WebSockets| Notif[Notification Service <br/> Spring Boot]

    subgraph Capa de Microservicios
        Gateway -->|/auth| Auth[Auth Service <br/> Spring Boot]
        Gateway -->|/clients| ClientSrv[Client Service <br/> Spring Boot]
        Gateway -->|/orders| Order[Order Service <br/> Spring Boot]
        Gateway -->|/docs| Doc[Document Service <br/> Spring Boot]
        Gateway -->|/notifications| Notif
    end

    subgraph Capa de Datos y Almacenamiento
        Auth --> AuthDB[(DB Auth)]
        ClientSrv --> ClientDB[(DB Client)]
        Order --> OrderDB[(DB Orders)]
        Doc --> S3[(MinIO / S3 Storage)]
        Notif --> NotifDB[(DB Notifs)]
    end

    Auth -.->|RabbitMQ| MQ((RabbitMQ))
    ClientSrv -.->|RabbitMQ| MQ
    Order -.->|RabbitMQ| MQ
    Doc -.->|RabbitMQ| MQ
    Notif -.->|Consume MQ| MQ

    subgraph Capa de Integración
        MQ -.->|Consume / Publica| GoSrv[Integration Service <br/> Golang]
        GoSrv <--> Redis[(Redis Cache)]
    end

    GoSrv ===>|REST| VB[Virtual Broker API Core]

    subgraph Observabilidad - Docker PLG
        Promtail(Agente Promtail / OTel) --> Loki[(Loki Logs)]
        Promtail --> Tempo[(Tempo Trazas)]
        Grafana[Grafana UI] --> Loki
        Grafana --> Tempo
        Gateway -.-> Promtail
        Auth -.-> Promtail
        GoSrv -.-> Promtail
    end

    class Auth,ClientSrv,Order,Doc,Notif spring;
    class GoSrv go;
    class AuthDB,ClientDB,OrderDB,NotifDB,S3,Redis db;
    class MQ broker;
    class Promtail,Loki,Tempo,Grafana obs;
    class VB core;
