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
```

## 5. Casos de Uso y Flujos de Datos (Sequence Diagrams)
Caso de Uso 1: Aprobación de Órdenes (Maker-Checker)
Descripción: Un cliente crea una orden, la cual queda en pausa hasta que un ejecutivo la aprueba en el Backoffice.
```mermaid
sequenceDiagram
    autonumber
    actor Cliente
    actor Ejecutivo
    participant Order as Order Service (Spring)
    participant MQ as RabbitMQ
    participant Go as Integration Service (Go)
    participant VB as Virtual Broker REST

    Note over Cliente, Order: 1. Creación por Cliente
    Cliente->>Order: POST /api/orders
    Order->>Order: Guarda en DB (Estado: PENDIENTE)
    Order-->>Cliente: 201 Created

    Note over Cliente, Ejecutivo: ... Intervención Humana ...

    Note over Ejecutivo, Order: 2. Aprobación Ejecutiva (Backoffice)
    Ejecutivo->>Order: POST /api/admin/orders/{id}/approve
    Order->>Order: Actualiza DB (Estado: ENVIANDO)
    Order->>MQ: Publica evento {action: "SEND_TO_VB"}
    Order-->>Ejecutivo: 200 OK

    Note over MQ, VB: 3. Inyección Asíncrona al Core
    MQ->>Go: Consume mensaje
    Go->>VB: POST /api/v1/broker/orders
    VB-->>Go: 200 OK (Aprobado)
    Go->>MQ: Publica evento {status: "COMPLETADA"}
    MQ->>Order: Actualiza estado final
```

Caso de Uso 2: Generación de Estados de Cuenta en PDF
Descripción: El sistema genera PDFs de forma asíncrona para no bloquear los hilos principales del servidor.
```mermaid
sequenceDiagram
    autonumber
    actor Cliente
    participant DocSrv as Document Service (Spring)
    participant MQ as RabbitMQ
    participant S3 as MinIO (S3)
    participant Internals as Golang & Client Srv
    participant Notif as Notification Srv

    Cliente->>DocSrv: POST /api/statements/request
    DocSrv->>MQ: Encola trabajo en 'statement.generate.queue'
    DocSrv-->>Cliente: 202 Accepted (Procesando...)

    Note over DocSrv, Internals: Worker Asíncrono
    MQ->>DocSrv: Consume trabajo
    DocSrv->>Internals: GET Datos Cliente + Movimientos VB
    Internals-->>DocSrv: Retorna JSON consolidado
    DocSrv->>DocSrv: Renderiza PDF en RAM
    
    DocSrv->>S3: Upload PDF File
    S3-->>DocSrv: Retorna Presigned URL
    DocSrv->>MQ: Publica Evento 'statement.ready' {url}
    
    MQ->>Notif: Consume evento
    Notif-->>Cliente: Envía WebSocket Alert y Email
```
---
## 6. Diccionario de Endpoints REST (Contratos Base)


6.1. Auth & Security (/api/auth)
POST /login -> Autenticación y generación de JWT.

POST /recover-password -> Dispara flujo de recuperación.

6.2. Client Management (/api/clients)
POST /validate -> Valida RIF/ID contra Virtual Broker.

POST /register -> Crea perfil local.

GET /me -> Consulta de perfil.

PUT /me -> Actualización de datos de contacto.

6.3. Integration / Dashboards (/api/portfolio) - Servidos por Golang/Redis
GET /consolidated -> Posición global (Saldos y Totales).

GET /products -> Detalle de inversiones activas.

GET /calendar?month=X&year=Y -> Eventos de vencimiento formateados.

6.4. Order Management (/api/orders)
POST / -> Crear nueva orden o reinversión (Cliente).

GET / -> Listar histórico (Cliente).

GET /admin?status=PENDIENTE -> Listar pool de órdenes (Ejecutivos).

POST /admin/{id}/approve -> Aprobar transacción (Ejecutivos).

6.5. Documents & Notifications
POST /api/statements/request -> Solicitar estado de cuenta.

GET /api/notifications -> Obtener historial de alertas no leídas.

PATCH /api/notifications/{id}/read -> Marcar alerta como leída.

WS wss://{host}/ws/notifications -> Canal STOMP persistente para alertas en tiempo real.
