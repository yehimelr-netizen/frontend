# Servidor REST — Sistema de Evaluación de Desempeño UNDA

Documentación técnica del backend ubicado en `server/`. El servidor es una **API REST** construida con **Express 5**, **Prisma ORM**, **MySQL** y autenticación mediante **JWT**.

---

## Tabla de Contenidos

1. [Stack tecnológico](#1-stack-tecnológico)
2. [Estructura de directorios](#2-estructura-de-directorios)
3. [Arquitectura general](#3-arquitectura-general)
4. [Arranque y ciclo de vida](#4-arranque-y-ciclo-de-vida)
5. [Endpoints de la API](#5-endpoints-de-la-api)
6. [Middlewares](#6-middlewares)
7. [Capa de servicios](#7-capa-de-servicios)
8. [Modelo de datos — Prisma / MySQL](#8-modelo-de-datos--prisma--mysql)
9. [Generación de PDF con Playwright](#9-generación-de-pdf-con-playwright)
10. [Servicio de email (Nodemailer / Gmail SMTP)](#10-servicio-de-email-nodemailer--gmail-smtp)
11. [Validador de evaluaciones](#11-validador-de-evaluaciones)
12. [Mapper de evaluaciones](#12-mapper-de-evaluaciones)
13. [Diagrama de flujo de una petición HTTP](#13-diagrama-de-flujo-de-una-petición-http)
14. [Diagrama de secuencia — Autenticación JWT](#14-diagrama-de-secuencia--autenticación-jwt)
15. [Diagrama de secuencia — Creación de evaluación](#15-diagrama-de-secuencia--creación-de-evaluación)
16. [Diagrama de secuencia — Generación y envío de PDF](#16-diagrama-de-secuencia--generación-y-envío-de-pdf)
17. [Diagrama de clases — Servicios y controladores](#17-diagrama-de-clases--servicios-y-controladores)
18. [Diagrama entidad-relación](#18-diagrama-entidad-relación)

---

## 1. Stack tecnológico

| Capa | Tecnología |
|---|---|
| Framework HTTP | Express 5 |
| ORM | Prisma 6 |
| Base de datos | MySQL |
| Autenticación | JWT (`jsonwebtoken`) + bcrypt |
| Validación | Zod v4 |
| Generación de PDF | Playwright (Chromium headless) |
| Email | Nodemailer (Gmail SMTP) |
| Logging | Pino + pino-http |
| Fechas | date-fns v4 |
| Runtime | Node.js + tsx (TypeScript) |
| Lenguaje | TypeScript 5.9 |

---

## 2. Estructura de directorios

```
server/
├── src/
│   ├── server.ts               # Entry point: conecta BD e inicia HTTP server
│   ├── app.ts                  # Instancia Express + CORS + rutas montadas
│   ├── logger.ts               # Instancia Pino
│   │
│   ├── routes/                 # Definición de rutas Express por recurso
│   │   ├── auth.route.ts
│   │   ├── empleado.route.ts
│   │   ├── evaluacion.route.ts
│   │   ├── itemEvaluacion.route.ts
│   │   ├── supervisor.route.ts
│   │   └── usuario.route.ts
│   │
│   ├── controllers/            # Capa HTTP: parseo de req/res, delegación al service
│   │   ├── auth.controllers.ts
│   │   ├── empleado.controller.ts
│   │   ├── evaluacion.controller.ts
│   │   ├── itemEvaluacion.controller.ts
│   │   ├── supervisor.controller.ts
│   │   └── usuario.controller.ts
│   │
│   ├── services/               # Lógica de negocio pura
│   │   ├── auth.service.ts
│   │   ├── email.service.ts
│   │   ├── empleado.service.ts
│   │   ├── evaluacion.service.ts
│   │   ├── itemEvaluacion.service.ts
│   │   ├── supervisor.service.ts
│   │   └── usuario.service.ts
│   │
│   ├── middlewares/
│   │   ├── auth.middleware.ts  # authenticate (JWT) + authorize (roles)
│   │   └── validateRequest.ts  # Zod schema validator para body/query/params
│   │
│   ├── schemas/                # Zod schemas compartidos (mismo contenido que web/schemas/)
│   │   ├── auth.schema.ts
│   │   ├── email.schema.ts
│   │   ├── empleado.schema.ts
│   │   ├── evaluacion.schema.ts
│   │   ├── evaluacionItemResultado.schema.ts
│   │   ├── itemEvaluacion.schema.ts
│   │   ├── pagination.schema.ts
│   │   ├── password.schema.ts
│   │   ├── supervisor.schema.ts
│   │   └── usuario.schema.ts
│   │
│   ├── prisma/
│   │   ├── schema.prisma       # Modelo de datos completo
│   │   ├── prismaClient.ts     # Singleton de PrismaClient
│   │   ├── seed.ts             # Datos iniciales
│   │   └── migrations/        # Historial de migraciones SQL
│   │
│   ├── mappers/
│   │   └── evaluacion.mapper.ts  # Prisma raw → EvaluacionResponseDTO
│   │
│   ├── validators/
│   │   └── evaluacion.validator.ts  # Validaciones de negocio de evaluaciones
│   │
│   ├── templates/              # Plantillas HTML para generación de PDF
│   │   ├── formato-evaluacion.html
│   │   ├── formato-solicitud-permiso.html
│   │   ├── formato-notificacion.html
│   │   └── reporte-empleados.html
│   │
│   ├── config/
│   │   └── pagination.ts       # DEFAULT_LIMIT, MAX_LIMIT
│   │
│   └── utils/
│       └── getErrorMessage.ts  # Extrae mensaje legible de cualquier error
│
├── prisma.config.ts
└── package.json
```

---

## 3. Arquitectura general

```mermaid
graph TD
    subgraph Cliente
        FE["Frontend Next.js\n:3000"]
    end

    subgraph "Express App (:5000)"
        MW_CORS[CORS Middleware]
        MW_JSON[express.json]
        MW_LOG[pino-http Logger]
        MW_AUTH[authenticate\nauthorize]
        MW_VAL[validateRequest\nZod]

        subgraph Routers
            R_AUTH[/api/auth]
            R_USR[/api/usuarios]
            R_EMP[/api/empleados]
            R_ITEM[/api/items-evaluaciones]
            R_EVAL[/api/evaluaciones]
            R_SUP[/api/supervisores]
        end

        subgraph Controllers
            C_AUTH[AuthController]
            C_USR[UsuarioController]
            C_EMP[EmpleadoController]
            C_ITEM[ItemEvaluacionController]
            C_EVAL[EvaluacionController]
            C_SUP[SupervisorController]
        end

        subgraph Services
            S_AUTH[AuthService]
            S_USR[UsuarioService]
            S_EMP[EmpleadoService]
            S_ITEM[ItemEvaluacionService]
            S_EVAL[EvaluacionService]
            S_SUP[SupervisorService]
            S_EMAIL[EmailService]
        end
    end

    subgraph Infraestructura
        DB[(MySQL)]
        PRISMA[PrismaClient]
        PDF[Playwright\nChromium]
        SMTP[Gmail SMTP]
    end

    FE -->|HTTP + JWT| MW_CORS --> MW_JSON --> MW_LOG
    MW_LOG --> Routers
    Routers --> MW_AUTH --> MW_VAL --> Controllers
    Controllers --> Services
    Services --> PRISMA --> DB
    Services --> PDF
    Services --> S_EMAIL --> SMTP
```

---

## 4. Arranque y ciclo de vida

```mermaid
flowchart TD
    A[Node.js ejecuta server.ts] --> B[prisma.$connect]
    B -->|error| Z[process.exit 1]
    B -->|éxito| C[app.listen PORT:5000]
    C --> D[Servidor listo]
    D --> E{Señal OS}
    E -->|SIGINT / SIGTERM| F[server.close]
    F --> G[prisma.$disconnect]
    G --> H[process.exit 0]
```

### Variables de entorno requeridas (`.env`)

| Variable | Descripción |
|---|---|
| `DATABASE_URL` | Cadena de conexión MySQL |
| `JWT_SECRET` | Secreto para firmar access tokens (15 min) |
| `JWT_REFRESH_SECRET` | Secreto para refresh tokens (7 días) |
| `BCRYPT_SALT_ROUNDS` | Rondas bcrypt (default: 10) |
| `SERVER_PORT` | Puerto del servidor (default: 5000) |
| `EMAIL_USER` | Cuenta Gmail remitente |
| `EMAIL_PASS` | App password de Gmail |
| `EMAIL_FROM_NAME` | Nombre visible del remitente |

---

## 5. Endpoints de la API

### Auth — `/api/auth`

| Método | Ruta | Descripción | Middlewares |
|---|---|---|---|
| POST | `/login` | Autenticar usuario → `{accessToken, refreshToken, user}` | validateRequest |
| POST | `/refresh` | Renovar access token usando refresh token | — |
| POST | `/forgot-password` | Enviar código de recuperación al email | validateRequest |
| POST | `/verify-reset-token` | Validar código de 6 dígitos | validateRequest |
| POST | `/reset-password` | Cambiar contraseña con código verificado | validateRequest |

### Usuarios — `/api/usuarios`

| Método | Ruta | Descripción |
|---|---|---|
| POST | `/` | Crear usuario (empleado interno o supervisor externo) |
| GET | `/` | Listar usuarios con filtros y paginación |
| GET | `/:id` | Obtener usuario por ID |
| PATCH | `/:id` | Actualizar parcialmente (email, password, rol) |
| DELETE | `/:id` | Eliminar usuario |

### Empleados — `/api/empleados`

| Método | Ruta | Descripción |
|---|---|---|
| POST | `/` | Crear empleado |
| GET | `/` | Listar con filtros (search, tipoPersonal, sexo, dependencia, cargo) |
| GET | `/report` | Generar PDF de reporte filtrado |
| GET | `/:id` | Obtener empleado por ID |
| PATCH | `/:id` | Actualizar parcialmente |
| DELETE | `/:id` | Eliminar empleado |
| POST | `/:id/solicitud-permiso` | Generar PDF de solicitud de permiso |
| POST | `/:id/notificacion` | Generar PDF de notificación |
| POST | `/:id/solicitud-permiso/send-email` | Generar PDF y enviar por email |
| POST | `/:id/notificacion/send-email` | Generar PDF de notificación y enviar por email |

### Evaluaciones — `/api/evaluaciones`

| Método | Ruta | Descripción |
|---|---|---|
| POST | `/` | Crear evaluación con ítems resultado |
| GET | `/` | Listar con filtros (periodos, IDs, rango, totalFinal) |
| GET | `/:id` | Obtener evaluación completa con ítems |
| PATCH | `/:id` | Actualizar evaluación y recalcular totales |
| DELETE | `/:id` | Eliminar evaluación |
| GET | `/:id/generate-pdf` | Generar y descargar PDF de la evaluación |
| POST | `/:id/send-email` | Generar PDF y enviar por email |

### Supervisores — `/api/supervisores`

| Método | Ruta | Descripción |
|---|---|---|
| GET | `/` | Listar supervisores con paginación y búsqueda |
| GET | `/:id` | Obtener supervisor por ID |
| PATCH | `/:id` | Actualizar datos del supervisor |

### Ítems de Evaluación — `/api/items-evaluaciones`

| Método | Ruta | Descripción |
|---|---|---|
| GET | `/` | Listar ítems con filtros (tipoItem, peso, obligatorio, esSistema) |
| GET | `/:id` | Obtener ítem por ID |

---

## 6. Middlewares

### `authenticate` — Verificación JWT

```mermaid
flowchart LR
    A[Request] --> B{Authorization header?}
    B -->|no| E1[401 No token provided]
    B -->|sí| C{jwt.verify JWT_SECRET}
    C -->|falla| E2[403 Invalid or expired token]
    C -->|éxito| D[req.user = payload]
    D --> F[next]
```

### `authorize(roles[])` — Control de acceso por rol

```mermaid
flowchart LR
    A[Request + req.user] --> B{req.user existe?}
    B -->|no| E1[401 Not authenticated]
    B -->|sí| C{rol en lista permitida?}
    C -->|no| E2[403 Not authorized]
    C -->|sí| D[next]
```

### `validateRequest(schema)` — Validación Zod

Valida de forma independiente `body`, `query` y/o `params` contra schemas Zod. Si falla alguno, responde `400` con `errors` detallados en formato `flatten()`. Si pasa, reemplaza `req.body/query/params` con los datos parseados y transformados.

---

## 7. Capa de servicios

### `AuthService`

| Método | Descripción |
|---|---|
| `login(email, password)` | Busca usuario, compara hash bcrypt, emite accessToken (15 min) y refreshToken (7 días) |
| `refresh(token)` | Verifica refreshToken con `JWT_REFRESH_SECRET`, emite nuevo accessToken |
| `requestPasswordReset(email)` | Genera código de 6 dígitos, lo persiste en BD con expiración de 15 min, envía email |
| `verifyResetToken(email, token)` | Valida código y expiración |
| `resetPassword(email, token, newPassword)` | Verifica token, hashea nueva contraseña, limpia campos reset en BD |

### `UsuarioService`

Crea usuarios con lógica **discriminada por rol**:
- `SUPERVISOR` → crea primero `SupervisorExterno`, luego `Usuario` vinculado.
- Otros roles → crea primero `Empleado`, luego `Usuario` vinculado.

Soporte completo CRUD con hash bcrypt automático en create y patch.

### `EmpleadoService`

- CRUD completo con filtros dinámicos (OR sobre nombres/apellidos/cédula).
- **Paginación dual**: soporta `page+limit` y `offset` directamente.
- Transacciones `$transaction` para `count + findMany` simultáneos.
- Generación de PDF con Playwright para: solicitud de permiso, notificación y reporte de empleados.

### `EvaluacionService`

El servicio más complejo, con transacciones Prisma para garantizar consistencia:

```mermaid
flowchart TD
    A[createEvaluacion] --> B[validarEvaluacionData\nverifica evaluado/evaluador/supervisor]
    B --> C[tx.evaluacion.create cabecera]
    C --> D[validarItemsResultado\nverifica rangos y calcula pesoXrango]
    D --> E[tx.evaluacionItemResultado.create para cada ítem]
    E --> F[Calcular totalModuloI + totalModuloII + totalFinal]
    F --> G[calcularRangoActuacion]
    G --> H[tx.evaluacion.update con totales]
    H --> I[mapEvaluacionToDTO]
    I --> J[Retornar EvaluacionResponseDTO]
```

### `SupervisorService`

CRUD básico con búsqueda full-text sobre múltiples campos (email, nombres, apellidos, cédula, cargo, dependencia). No permite eliminar supervisores que tengan cuenta de usuario asociada.

### `ItemEvaluacionService`

Solo lectura (GET all + GET by ID). Los ítems son configurados previamente en la BD (sistema) y no se crean desde la UI.

### `EmailService`

Singleton exportado. Usa Nodemailer con transporte Gmail SMTP. Expone:
- `sendEmail(to, subject, htmlBody)` — email sin adjunto
- `sendPdfEmail(to, subject, htmlBody, pdfBuffer, filename)` — email con PDF adjunto
- Generadores de templates HTML: `generatePasswordResetEmailBody`, `generateSolicitudPermisoEmailBody`, `generateNotificacionEmailBody`, `generateEvaluacionEmailBody`

---

## 8. Modelo de datos — Prisma / MySQL

```mermaid
erDiagram
    Usuario {
        Int id PK
        String email UK
        String passwordHash
        Rol rol
        String resetToken
        DateTime resetTokenExpiry
        DateTime createdAt
        DateTime updatedAt
    }

    Empleado {
        Int id PK
        String nombres
        String apellidos
        String cedula UK
        TipoPersonal tipoPersonal
        String sexo
        String telefono
        DateTime fechaIngreso
        String dependencia
        String cargo
        String situacionLaboral
        Int horasAcademicas
        Int horasAdministrativas
        String gradoImparte
        String secciones
        String especialidad
        String materias
        String periodoGrupo
        String observaciones
        Int usuarioId FK
        DateTime createdAt
        DateTime updatedAt
    }

    SupervisorExterno {
        Int id PK
        String nombres
        String apellidos
        String email UK
        String cedula UK
        String telefono
        String cargo
        String dependencia
        String especialidad
        String nivelJerarquico
        Int usuarioId FK
        DateTime createdAt
        DateTime updatedAt
    }

    Evaluacion {
        Int id PK
        Int idEvaluado FK
        Int idEvaluador FK
        Int idSupervisorExterno FK
        DateTime periodoDesde
        DateTime periodoHasta
        Int totalModuloI
        Int totalModuloII
        Int totalFinal
        String rangoActuacion
        String textoCursosRealizados
        String acotacionesSupervisor
        DateTime createdAt
        DateTime updatedAt
    }

    ItemEvaluacion {
        Int id PK
        TipoItem tipoItem
        String titulo
        String descripcion
        Boolean obligatorio
        Int peso
        Int rangoMinimo
        Int rangoMaximo
        Boolean esSistema
        Boolean activo
        Int orden
        DateTime createdAt
        DateTime updatedAt
    }

    EvaluacionItemResultado {
        Int id PK
        Int evaluacionId FK
        Int itemEvaluacionId FK
        Int rangoSeleccionado
        Int pesoXrango
        DateTime createdAt
        DateTime updatedAt
    }

    Usuario ||--o| Empleado : "tiene (interno)"
    Usuario ||--o| SupervisorExterno : "tiene (externo)"
    Empleado ||--o{ Evaluacion : "es evaluado en"
    Empleado ||--o{ Evaluacion : "es evaluador en"
    SupervisorExterno ||--o{ Evaluacion : "supervisa"
    Evaluacion ||--|{ EvaluacionItemResultado : "contiene"
    ItemEvaluacion ||--|{ EvaluacionItemResultado : "referenciado en"
```

### Enums del modelo

| Enum | Valores |
|---|---|
| `Rol` | `ADMINISTRATIVO`, `SUPERVISOR`, `DIRECTOR`, `COORDINADOR` |
| `TipoPersonal` | `ADMINISTRATIVO_DOCENTE`, `ADMINISTRATIVO`, `OBRERO`, `DOCENTE` |
| `TipoItem` | `ODI`, `COMPETENCIA` |

### Relaciones clave

- `Empleado ↔ Usuario`: relación 1-a-1 opcional. Al eliminar `Usuario` → `usuarioId` en Empleado se pone `NULL` (`onDelete: SetNull`).
- `Evaluacion` tiene dos FK a `Empleado` (evaluado y evaluador), identificadas por nombres de relación distintos (`"Evaluado"`, `"Evaluador"`).
- `EvaluacionItemResultado` tiene restricción única compuesta `(evaluacionId, itemEvaluacionId)` y se elimina en cascada si se elimina la `Evaluacion`.

---

## 9. Generación de PDF con Playwright

Tanto `EmpleadoService` como `EvaluacionService` usan Playwright (Chromium headless) para convertir HTML a PDF:

```mermaid
sequenceDiagram
    participant Controller
    participant Service
    participant FS as "fs (template HTML)"
    participant Playwright as "Playwright Chromium"

    Controller->>Service: generatePdf(id) / generateSolicitudPermiso(id, data)
    Service->>FS: readFileSync(templatePath)
    FS-->>Service: html string
    Service->>Service: Reemplazar {{variables}} en el HTML
    Service->>Playwright: chromium.launch({ headless: true })
    Playwright-->>Service: browser
    Service->>Playwright: browser.newPage()
    Service->>Playwright: page.setContent(html)
    Service->>Playwright: page.pdf({ format: "A4", ... })
    Playwright-->>Service: pdfBuffer (Uint8Array)
    Service->>Playwright: browser.close()
    Service-->>Controller: { buffer, filename }
    Controller-->>Client: res.type("pdf").send(buffer)
```

### Plantillas disponibles

| Archivo | Uso |
|---|---|
| `formato-evaluacion.html` | PDF de evaluación de desempeño completa |
| `formato-solicitud-permiso.html` | Documento de solicitud de días de permiso |
| `formato-notificacion.html` | Notificación individual del empleado |
| `reporte-empleados.html` | Tabla de empleados filtrada (landscape A4) |

---

## 10. Servicio de email (Nodemailer / Gmail SMTP)

```mermaid
sequenceDiagram
    participant Controller
    participant EmailService
    participant Gmail as "Gmail SMTP"
    participant Destinatario

    Controller->>EmailService: sendPdfEmail(to, subject, htmlBody, pdfBuffer, filename)
    EmailService->>Gmail: transporter.sendMail({ from, to, subject, html, attachments })
    Gmail-->>EmailService: { messageId }
    EmailService-->>Controller: void
    Gmail->>Destinatario: Email con PDF adjunto
```

El `EmailService` se instancia como **singleton** y verifica la conexión SMTP en el constructor. En caso de error de conexión, lo registra pero no lanza excepción (para no impedir el arranque del servidor).

---

## 11. Validador de evaluaciones

`validators/evaluacion.validator.ts` centraliza las reglas de negocio más complejas de la evaluación:

```mermaid
flowchart TD
    A[validarEvaluacionData] --> B{periodoDesde < periodoHasta?}
    B -->|no| ERR1[Error: periodo inválido]
    B -->|sí| C[Promise.all buscar evaluado + evaluador + supervisor]
    C --> D{Todos existen?}
    D -->|no| ERR2[Error: entidad no encontrada]
    D -->|sí| E{evaluado ≠ evaluador?}
    E -->|no| ERR3[Error: misma persona]
    E -->|sí| F{Rol evaluador: COORDINADOR o DIRECTOR?}
    F -->|no| ERR4[Error: rol inválido]
    F -->|sí| G{Rol supervisor: SUPERVISOR?}
    G -->|no| ERR5[Error: rol inválido]
    G -->|sí| H[return evaluado, evaluador, supervisor]

    I[validarItemsResultado] --> J[Buscar todos los ItemEvaluacion en BD]
    J --> K{Para cada item: rango en min..max?}
    K -->|no| ERR6[Error: rango fuera de límite]
    K -->|sí| L[pesoXrango = peso × rangoSeleccionado]
    L --> M[return resultadosValidados]
```

---

## 12. Mapper de evaluaciones

`mappers/evaluacion.mapper.ts` convierte el objeto crudo de Prisma al DTO de respuesta:

- Transforma fechas `Date` → `DD-MM-YYYY` (UTC-safe, sin desfase de zona horaria).
- Transforma timestamps `Date` → ISO string.
- Aplana la relación `itemsResultado → itemEvaluacion` al formato esperado por el cliente.

```mermaid
flowchart LR
    A["Prisma raw\n(periodoDesde: Date, ...)"] --> B["mapEvaluacionToDTO"]
    B --> C["EvaluacionResponseDTO\n(periodoDesde: 'DD-MM-YYYY', ...)"]
```

---

## 13. Diagrama de flujo de una petición HTTP

El siguiente diagrama muestra el pipeline completo para cualquier ruta protegida y con validación:

```mermaid
flowchart TD
    A[Cliente HTTP] --> B[CORS]
    B --> C[express.json]
    C --> D[pino-http logger]
    D --> E[Router match]
    E --> F[authenticate\njwt.verify]
    F -->|401/403| ERR1[Respuesta de error]
    F --> G[authorize roles\ncheck req.user.rol]
    G -->|403| ERR2[Respuesta de error]
    G --> H[validateRequest Zod\nbody / query / params]
    H -->|400| ERR3[Respuesta 400 con errores Zod]
    H --> I[Controller Handler]
    I --> J[Service method]
    J --> K[PrismaClient query]
    K --> L[(MySQL)]
    L --> K
    K --> J
    J --> I
    I --> M[res.json / res.send]
    M --> A
```

---

## 14. Diagrama de secuencia — Autenticación JWT

```mermaid
sequenceDiagram
    actor U as Cliente
    participant API as "POST /api/auth/login"
    participant AuthSvc as AuthService
    participant DB as "MySQL (Prisma)"
    participant SMTP as "Gmail SMTP"

    U->>API: { email, password }
    API->>AuthSvc: login(email, password)
    AuthSvc->>DB: usuario.findUnique({ email })
    DB-->>AuthSvc: { id, email, rol, passwordHash }
    AuthSvc->>AuthSvc: bcrypt.compare(password, hash)
    AuthSvc->>AuthSvc: jwt.sign({ id, rol }, JWT_SECRET, 15m)
    AuthSvc->>AuthSvc: jwt.sign({ id }, JWT_REFRESH_SECRET, 7d)
    AuthSvc-->>API: { accessToken, refreshToken, user }
    API-->>U: 200 { accessToken, refreshToken, user }

    Note over U,API: Renovación de token
    U->>API: POST /api/auth/refresh { refreshToken }
    API->>AuthSvc: refresh(refreshToken)
    AuthSvc->>AuthSvc: jwt.verify(refreshToken, JWT_REFRESH_SECRET)
    AuthSvc->>AuthSvc: jwt.sign({ id }, JWT_SECRET, 15m)
    AuthSvc-->>API: newAccessToken
    API-->>U: 200 { accessToken }
```

---

## 15. Diagrama de secuencia — Creación de evaluación

```mermaid
sequenceDiagram
    actor C as Cliente
    participant RT as "POST /api/evaluaciones"
    participant MW as validateRequest (Zod)
    participant EC as EvaluacionController
    participant ES as EvaluacionService
    participant VAL as evaluacion.validator
    participant TX as "Prisma $transaction"
    participant DB as MySQL

    C->>RT: { periodoDesde, periodoHasta, idEvaluado, idEvaluador, idSupervisorExterno, itemsResultado[] }
    RT->>MW: Valida body contra evaluacionCreateSchema
    MW-->>RT: 400 si error Zod
    MW->>EC: req.body validado
    EC->>ES: createEvaluacion(data)
    ES->>VAL: validarEvaluacionData(data)
    VAL->>DB: Promise.all [findUnique evaluado, evaluador, supervisor]
    DB-->>VAL: entidades encontradas
    VAL-->>ES: { evaluado, evaluador, supervisorExterno }
    ES->>TX: Inicia transacción
    TX->>DB: evaluacion.create (cabecera)
    DB-->>TX: evaluacion { id }
    TX->>VAL: validarItemsResultado(items, tx)
    VAL->>DB: itemEvaluacion.findMany({ id: in [...] })
    DB-->>VAL: items validados + pesoXrango calculado
    TX->>DB: Promise.all evaluacionItemResultado.create (×N items)
    TX->>TX: Sumar totalModuloI + totalModuloII → totalFinal
    TX->>TX: calcularRangoActuacion(totalFinal)
    TX->>DB: evaluacion.update totales + rangoActuacion
    DB-->>TX: evaluacion actualizada con include itemsResultado
    TX-->>ES: raw Prisma object
    ES->>ES: mapEvaluacionToDTO(raw)
    ES-->>EC: EvaluacionResponseDTO
    EC-->>C: 201 EvaluacionResponseDTO
```

---

## 16. Diagrama de secuencia — Generación y envío de PDF

```mermaid
sequenceDiagram
    actor C as Cliente
    participant API as "POST /api/empleados/:id/solicitud-permiso/send-email"
    participant EC as EmpleadoController
    participant ES as EmpleadoService
    participant PW as "Playwright Chromium"
    participant EM as EmailService
    participant SMTP as "Gmail SMTP"

    C->>API: { to, desde, hasta, dias, motivo, message }
    API->>EC: sendSolicitudPermisoEmail(req, res)
    EC->>ES: generateSolicitudPermiso(id, data)
    ES->>ES: readFileSync(template.html)
    ES->>ES: Reemplazar {{variables}} en HTML
    ES->>PW: chromium.launch + page.setContent(html)
    PW-->>ES: pdfBuffer
    EC->>ES: getById(id) → empleado
    EC->>EM: generateSolicitudPermisoEmailBody(nombre, dias, ...)
    EM-->>EC: htmlBody
    EC->>EM: sendPdfEmail(to, subject, htmlBody, pdfBuffer, filename)
    EM->>SMTP: transporter.sendMail({ ... attachments: [pdfBuffer] })
    SMTP-->>EM: { messageId }
    EM-->>EC: void
    EC-->>C: 200 { message, to, filename }
```

---

## 17. Diagrama de clases — Servicios y controladores

```mermaid
classDiagram
    class AuthController {
        +login(req, res)
        +refresh(req, res)
        +forgotPassword(req, res)
        +verifyResetToken(req, res)
        +resetPassword(req, res)
    }

    class AuthService {
        +login(email, password) LoginResult
        +refresh(token) string
        +requestPasswordReset(email) void
        +verifyResetToken(email, token) boolean
        +resetPassword(email, token, newPassword) void
        -generateResetToken() string
        -signToken(payload, expiresIn) string
        -comparePassword(password, hash) boolean
    }

    class EmpleadoController {
        +create(req, res)
        +getAll(req, res)
        +getById(req, res)
        +patch(req, res)
        +delete(req, res)
        +generateSolicitudPermiso(req, res)
        +generateNotificacion(req, res)
        +sendSolicitudPermisoEmail(req, res)
        +sendNotificacionEmail(req, res)
        +generateReport(req, res)
    }

    class EmpleadoService {
        +createEmpleado(data) Empleado
        +getAll(filters, pagination) PaginatedResult
        +getById(id, includeEvals?) Empleado
        +patchEmpleado(id, data) Empleado
        +deleteEmpleado(id) void
        +generateSolicitudPermiso(id, data) PdfResult
        +generateNotificacion(id) PdfResult
        +generateReport(filters, pagination) PdfResult
    }

    class EvaluacionController {
        +create(req, res)
        +getAll(req, res)
        +getById(req, res)
        +patch(req, res)
        +delete(req, res)
        +generatePdf(req, res)
        +sendEvaluacionEmail(req, res)
    }

    class EvaluacionService {
        +createEvaluacion(data) EvaluacionResponseDTO
        +getAll(filters, pagination) PaginatedResult
        +getById(id) Evaluacion
        +patchEvaluacion(id, data) EvaluacionResponseDTO
        +deleteEvaluacion(id) void
        +generatePdf(id) PdfResult
    }

    class UsuarioService {
        +createUsuario(data) Usuario
        +getAll(filters, pagination) PaginatedResult
        +getById(id) Usuario
        +patchUsuario(id, data) Usuario
        +deleteUsuario(id) void
    }

    class SupervisorService {
        +getAll(filters, pagination) PaginatedResult
        +getById(id) SupervisorExterno
        +patchById(id, data) SupervisorExterno
        +deleteById(id) void
    }

    class ItemEvaluacionService {
        +getAll(filters, pagination) PaginatedResult
        +getById(id) ItemEvaluacion
    }

    class EmailService {
        -transporter: Transporter
        +sendEmail(to, subject, html) void
        +sendPdfEmail(to, subject, html, buffer, filename) void
        +generatePasswordResetEmailBody(code, minutes) string
        +generateSolicitudPermisoEmailBody(...) string
        +generateNotificacionEmailBody(...) string
        +generateEvaluacionEmailBody(...) string
    }

    AuthController --> AuthService
    EmpleadoController --> EmpleadoService
    EmpleadoController --> EmailService
    EvaluacionController --> EvaluacionService
    EvaluacionController --> EmailService
    AuthService --> EmailService

    EmpleadoService --> PrismaClient
    EvaluacionService --> PrismaClient
    UsuarioService --> PrismaClient
    SupervisorService --> PrismaClient
    ItemEvaluacionService --> PrismaClient
    AuthService --> PrismaClient
```

---

## 18. Diagrama entidad-relación

Vista simplificada de cardinalidades:

```mermaid
graph LR
    U[Usuario] --"0..1"--> E[Empleado]
    U --"0..1"--> SE[SupervisorExterno]

    E --"evaluado en N"--> EV[Evaluacion]
    E --"evaluador de N"--> EV
    SE --"supervisa N"--> EV

    EV --"tiene N"--> EIR[EvaluacionItemResultado]
    IE[ItemEvaluacion] --"referenciado en N"--> EIR

    IE --tipoItem ODI--> M1[Módulo I\nODIs]
    IE --tipoItem COMPETENCIA--> M2[Módulo II\nCompetencias]

    M1 --> TF[totalModuloI\n+ totalModuloII\n= totalFinal]
    M2 --> TF
    TF --> RA[rangoActuacion\nMuy bajo → Excelente]
```

### Lógica de cálculo de rango de actuación (espejada del frontend)

```mermaid
graph LR
    A["totalFinal < 100"] --> Z1["Muy bajo"]
    B["100 – 124"] --> Z2["Deficiente"]
    C["125 – 149"] --> Z3["Regular"]
    D["150 – 374"] --> Z4["Bueno"]
    E["375 – 499"] --> Z5["Muy bueno"]
    F["≥ 500"] --> Z6["Excelente"]
```

---

> **Generado automáticamente** a partir del análisis del código fuente en `server/`.  
> Última actualización: junio 2026.
