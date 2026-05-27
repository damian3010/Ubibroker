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

Caso 1: Registro por ID/RIF y Correo
Descripción: El cliente verifica si existe en la casa de bolsa para poder crear sus credenciales web.
```mermaid
sequenceDiagram
    autonumber
    actor Cliente
    participant ClientSrv as Client Service
    participant Go as Integration Srv
    participant VB as Virtual Broker REST
    participant Auth as Auth Service

    Cliente->>ClientSrv: POST /validate {rif, email}
    ClientSrv->>Go: Consulta existencia síncrona
    Go->>VB: GET /api/v1/broker/clients/check
    VB-->>Go: 200 OK (Existe: true, ID_VB: "VB-99")
    Go-->>ClientSrv: Retorna ID de Virtual Broker
    ClientSrv->>Auth: POST interno (Crear credenciales)
    Auth-->>Cliente: 201 Created (Registro Exitoso)
```
    
Caso 2: Recuperación de Usuario/Contraseña
Descripción: El cliente solicita restablecer su acceso mediante un código (OTP) o enlace enviado a su correo registrado.
```mermaid
sequenceDiagram
    autonumber
    actor Cliente
    participant Auth as Auth Service
    participant MQ as RabbitMQ
    participant Notif as Notification Srv

    Cliente->>Auth: POST /recover-password {email}
    Auth->>Auth: Genera Token temporal
    Auth->>MQ: Publica {action: "SEND_RECOVERY", email}
    Auth-->>Cliente: 202 Accepted (Revisa tu correo)
    MQ->>Notif: Consume mensaje
    Notif->>Notif: Envía correo electrónico (SMTP)
```
Caso 3: Consulta de Operaciones y Tableros (Posición Consolidada)
Descripción: El cliente entra al sistema y visualiza sus saldos, productos activos y transacciones pasadas de manera inmediata.
```mermaid
sequenceDiagram
    autonumber
    actor Cliente
    participant Go as Integration Srv (Golang)
    participant Redis as Redis Cache
    participant VB as Virtual Broker REST

    Cliente->>Go: GET /api/portfolio/consolidated
    Go->>Redis: Busca 'consolidated_user_123'
    alt Cache Hit (Datos en Memoria)
        Redis-->>Go: Retorna JSON (~10ms)
    else Cache Miss (Datos Vencidos/Vacíos)
        Go->>VB: GET /api/v1/broker/portfolio
        VB-->>Go: Retorna crudos (~2000ms)
        Go->>Redis: Guarda copia por 5 minutos
    end
    Go-->>Cliente: 200 OK (JSON formateado)
```
Caso 4: Creación de Órdenes y Reinversiones (Flujo Maker-Checker)
Descripción: El cliente crea una orden (o reinvierte un vencimiento). La orden queda en pausa hasta que un ejecutivo la aprueba en el Backoffice.
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
    Cliente->>Order: POST /api/orders {type, amount}
    Order->>Order: Guarda en DB (Estado: PENDIENTE)
    Order-->>Cliente: 201 Created

    Note over Cliente, Ejecutivo: ... Intervención Humana ...

    Note over Ejecutivo, Order: 2. Aprobación Ejecutiva (Backoffice)
    Ejecutivo->>Order: POST /admin/orders/{id}/approve
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
Caso 5: Visualización de Vencimientos en Calendario Interactivo
Descripción: El cliente revisa los vencimientos de sus productos en una vista de calendario sin que su dispositivo deba realizar cálculos complejos.
```mermaid
sequenceDiagram
    autonumber
    actor Cliente
    participant Go as Integration Srv
    participant VB as Virtual Broker REST

    Cliente->>Go: GET /api/portfolio/calendar?month=06
    Go->>VB: Pide lista plana de vencimientos
    VB-->>Go: Retorna array simple [{id, date, amount}]
    Go->>Go: Agrupa datos por día y formatea para FullCalendar
    Go-->>Cliente: 200 OK (JSON estructurado)
```
Caso 6: Actualización de Datos Personales
Descripción: El cliente modifica su dirección o teléfono desde UbiBroker y el sistema sincroniza la información con el core de la casa de bolsa.
```mermaid
sequenceDiagram
    autonumber
    actor Cliente
    participant ClientSrv as Client Service
    participant MQ as RabbitMQ
    participant Go as Integration Srv
    participant VB as Virtual Broker REST

    Cliente->>ClientSrv: PUT /api/clients/me {address}
    ClientSrv->>ClientSrv: Actualiza PostgreSQL local
    ClientSrv->>MQ: Publica {action: "UPDATE_VB_PROFILE"}
    ClientSrv-->>Cliente: 200 OK (Perfil Actualizado)
    MQ->>Go: Consume mensaje
    Go->>VB: PATCH /api/v1/broker/clients
    VB-->>Go: 200 OK
```
Caso 7: Generación de Estados de Cuenta (PDF)
Descripción: El cliente solicita su estado de cuenta mensual. El sistema lo genera de fondo para no bloquear la interfaz y le notifica cuando está listo.
```mermaid
sequenceDiagram
    autonumber
    actor Cliente
    participant DocSrv as Document Service (Spring)
    participant MQ as RabbitMQ
    participant S3 as MinIO (S3)
    participant Notif as Notification Srv

    Cliente->>DocSrv: POST /api/statements/request
    DocSrv->>MQ: Encola 'statement.generate.queue'
    DocSrv-->>Cliente: 202 Accepted (Procesando...)

    Note over DocSrv, Notif: Worker Asíncrono
    MQ->>DocSrv: Consume trabajo
    DocSrv->>DocSrv: Renderiza PDF en RAM
    DocSrv->>S3: Sube archivo .pdf
    S3-->>DocSrv: Retorna Presigned URL
    DocSrv->>MQ: Publica Evento 'statement.ready' {url}
    
    MQ->>Notif: Consume evento
    Notif-->>Cliente: Envía WebSocket Alert y Email
```

## 6. Diccionario de Endpoints REST (Contratos Base)

Esta tabla resume los endpoints expuestos a través del API Gateway, los cuales actúan como el contrato de comunicación entre el Frontend y los microservicios.

| Microservicio | Endpoint | Método | Descripción |
| :--- | :--- | :--- | :--- |
| **Auth** | `/api/auth/login` | `POST` | Autenticación y generación de JWT. |
| **Auth** | `/api/auth/recover-password` | `POST` | Dispara el flujo de recuperación de credenciales. |
| **Client** | `/api/clients/validate` | `POST` | Valida RIF/ID contra Virtual Broker. |
| **Client** | `/api/clients/register` | `POST` | Crea perfil local del cliente. |
| **Client** | `/api/clients/me` | `GET` | Consulta de perfil del cliente. |
| **Client** | `/api/clients/me` | `PUT` | Actualización de datos personales. |
| **Portfolio**| `/api/portfolio/consolidated` | `GET` | Posición global (saldos y totales vía Redis). |
| **Portfolio**| `/api/portfolio/products` | `GET` | Detalle de inversiones activas. |
| **Portfolio**| `/api/portfolio/calendar` | `GET` | Eventos de vencimiento formateados. |
| **Orders** | `/api/orders` | `POST` | Crear nueva orden o reinversión. |
| **Orders** | `/api/orders` | `GET` | Listar histórico de órdenes. |
| **Orders** | `/api/admin/orders` | `GET` | Listar pool de órdenes pendientes (Ejecutivos). |
| **Orders** | `/api/admin/orders/{id}/approve` | `POST` | Aprobación ejecutiva de transacción. |
| **Documents**| `/api/statements/request` | `POST` | Solicitar generación de estado de cuenta. |
| **Notif** | `/api/notifications` | `GET` | Obtener historial de alertas no leídas. |
| **Notif** | `/api/notifications/{id}/read` | `PATCH`| Marcar notificación como leída. |
| **Notif** | `wss://{host}/ws/notifications` | `WS` | Canal STOMP para alertas en tiempo real. |
WS wss://{host}/ws/notifications -> Canal STOMP persistente para alertas en tiempo real.
