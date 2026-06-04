# Cliente Web — Sistema de Evaluación de Desempeño UNDA

Documentación técnica del frontend ubicado en `web/`. El cliente es una aplicación **Next.js 15** (App Router) con **React 19**, **Mantine UI v8**, **TanStack Query v5** y autenticación vía **NextAuth v5**.

---

## Tabla de Contenidos

1. [Stack tecnológico](#1-stack-tecnológico)
2. [Estructura de directorios](#2-estructura-de-directorios)
3. [Arquitectura general](#3-arquitectura-general)
4. [Flujo de autenticación](#4-flujo-de-autenticación)
5. [Enrutamiento y páginas](#5-enrutamiento-y-páginas)
6. [Control de acceso por rol](#6-control-de-acceso-por-rol)
7. [Capa de datos — Hooks y API](#7-capa-de-datos--hooks-y-api)
8. [Componentes](#8-componentes)
9. [Schemas de validación (Zod)](#9-schemas-de-validación-zod)
10. [Helpers](#10-helpers)
11. [Diagrama de componentes por módulo](#11-diagrama-de-componentes-por-módulo)
12. [Diagrama de secuencia — Login](#12-diagrama-de-secuencia--login)
13. [Diagrama de secuencia — CRUD genérico](#13-diagrama-de-secuencia--crud-genérico)
14. [Diagrama de clases — DTOs / Schemas](#14-diagrama-de-clases--dtos--schemas)
15. [Diagrama de estados — modal CRUD](#15-diagrama-de-estados--modal-crud)

---

## 1. Stack tecnológico

| Capa | Tecnología |
|---|---|
| Framework | Next.js 15 (App Router) |
| UI | Mantine v8, Tabler Icons, mantine-datatable |
| Estado servidor | TanStack Query v5 |
| Formularios | React Hook Form v7 |
| Validación | Zod v4 |
| Autenticación | NextAuth v5 (Credentials provider, JWT) |
| HTTP client | Axios v1 |
| Fechas | dayjs, date-fns |
| Lenguaje | TypeScript 5.9 |
| Package manager | Yarn 4 (berry) |

---

## 2. Estructura de directorios

```
web/
├── app/                        # App Router de Next.js
│   ├── api/auth/[...nextauth]/ # Manejador de NextAuth
│   ├── auth/
│   │   ├── login/              # Página de inicio de sesión
│   │   └── forgot-password/    # Recuperación de contraseña
│   ├── dashboard/              # Área protegida principal
│   │   ├── layout.tsx          # Shell con sidebar y RoleGuard
│   │   ├── page.tsx            # Panel de control (tarjetas de módulos)
│   │   ├── empleados/page.tsx
│   │   ├── supervisores/page.tsx
│   │   ├── usuarios/page.tsx
│   │   └── evaluaciones/page.tsx
│   ├── providers/
│   │   └── QueryProvider.tsx   # TanStack QueryClientProvider
│   ├── unauthorized/           # Página de acceso denegado
│   ├── layout.tsx              # Root layout (MantineProvider, SessionProvider)
│   ├── page.tsx                # Raíz → redirige a /dashboard
│   └── middleware.ts           # Middleware Next.js
│
├── components/
│   ├── auth/
│   │   ├── LoginForm.tsx
│   │   ├── ForgotPasswordForm.tsx
│   │   └── RoleGuard.tsx
│   ├── common/
│   │   ├── CRUDTable.tsx           # Tabla genérica con filtros y paginación
│   │   ├── CRUDModal.tsx
│   │   ├── CRUDTableContainer.tsx
│   │   ├── DeleteDialogCRUD.tsx
│   │   ├── LoadingOverlayScreen.tsx
│   │   ├── NavbarContent.tsx
│   │   └── ColorSchemeToggle/
│   ├── empleados/
│   │   ├── EmpleadoForm.tsx
│   │   ├── EmpleadosCRUDTable.tsx
│   │   └── SolicitudPermisoForm.tsx
│   ├── supervisores/
│   │   ├── SupervisorForm.tsx
│   │   └── SupervisoresCRUDTable.tsx
│   ├── usuarios/
│   │   ├── UsuarioForm.tsx
│   │   └── UsuariosCRUDTable.tsx
│   ├── evaluaciones/
│   │   ├── EvaluacionForm.tsx
│   │   ├── EvaluacionesCRUDTable.tsx
│   │   ├── EmpleadoSelect.tsx
│   │   ├── EvaluadorSelect.tsx
│   │   └── SupervisorSelect.tsx
│   ├── items/
│   │   ├── ItemEvaluacionForm.tsx
│   │   └── ItemsEvaluacionTable.tsx
│   └── mail/
│       └── EnviarEmailForm.tsx
│
├── hooks/                      # Custom hooks — TanStack Query
│   ├── auth/usePasswordRecovery.ts
│   ├── empleado/useEmpleadoQueries.ts
│   ├── evaluacion/useEvaluacionQueries.ts
│   ├── evaluacionItemResultado/useEvaluacionItemResultadoQueries.ts
│   ├── item/useItemEvaluacionQueries.ts
│   ├── supervisor/useSupervisorQueries.ts
│   └── usuario/useUsuarioQueries.ts
│
├── schemas/                    # Zod schemas + tipos inferidos
│   ├── auth.schema.ts
│   ├── email.schema.ts
│   ├── empleado.schema.ts
│   ├── evaluacion.schema.ts
│   ├── evaluacionItemResultado.schema.ts
│   ├── itemEvaluacion.schema.ts
│   ├── pagination.schema.ts
│   ├── password.schema.ts
│   ├── supervisor.schema.ts
│   └── usuario.schema.ts
│
├── services/
│   └── api.ts                  # Instancia axios + interceptor de refresh token
│
├── helpers/
│   ├── badges.ts
│   ├── date-parse.ts
│   ├── dateHelpers.ts
│   ├── evaluacion.ts           # Cálculo de totales y rangos de actuación
│   └── selectsData.ts          # Datos estáticos para selects
│
├── types/
│   └── crud-tables.ts          # Props de las tablas CRUD tipadas
│
├── auth.ts                     # Configuración central de NextAuth
└── theme.ts                    # Tema Mantine personalizado
```

---

## 3. Arquitectura general

```mermaid
graph TD
    subgraph Browser
        A[Next.js App Router]
        B[SessionProvider]
        C[QueryProvider]
        D[MantineProvider]
        A --> B --> C --> D
    end

    subgraph Pages
        P1["/auth/login"]
        P2["/auth/forgot-password"]
        P3["/dashboard"]
        P4["/dashboard/empleados"]
        P5["/dashboard/supervisores"]
        P6["/dashboard/usuarios"]
        P7["/dashboard/evaluaciones"]
    end

    subgraph Auth
        NG[RoleGuard]
        NA[NextAuth JWT]
    end

    subgraph DataLayer
        HK[Custom Hooks\nuseXxxQueries]
        AX[axios instance\nservices/api.ts]
        TQ[TanStack Query Cache]
    end

    subgraph Backend
        API["REST API\nhttp://localhost:5000/api"]
    end

    D --> Pages
    Pages --> NG --> NA
    Pages --> HK
    HK --> TQ
    TQ --> AX
    AX --> API
```

---

## 4. Flujo de autenticación

La autenticación usa **NextAuth v5** con el provider `Credentials`. El backend entrega `accessToken` y `refreshToken` en el login; ambos se almacenan en el JWT de sesión del lado del servidor.

El cliente Axios (`services/api.ts`) incluye un interceptor de respuesta que detecta `401` y renueva el `accessToken` automáticamente usando el `refreshToken` guardado en `localStorage`.

```mermaid
sequenceDiagram
    actor Usuario
    participant LoginForm
    participant NextAuth
    participant Backend as "API :5000"
    participant ApiClient as "axios (api.ts)"

    Usuario->>LoginForm: Ingresa email + password
    LoginForm->>NextAuth: signIn("credentials", {email, password})
    NextAuth->>Backend: POST /api/auth/login
    Backend-->>NextAuth: { user, accessToken, refreshToken }
    NextAuth-->>LoginForm: result.url = "/dashboard"
    LoginForm->>LoginForm: router.push("/dashboard")

    Note over ApiClient,Backend: Uso posterior de la API
    ApiClient->>Backend: GET /api/empleados (Bearer accessToken)
    Backend-->>ApiClient: 401 Token expirado
    ApiClient->>Backend: POST /api/auth/refresh { refreshToken }
    Backend-->>ApiClient: { accessToken nuevo }
    ApiClient->>Backend: GET /api/empleados (nuevo token)
    Backend-->>ApiClient: 200 OK
```

### Recuperación de contraseña

El flujo de recuperación es un proceso de 3 pasos implementado en `ForgotPasswordForm.tsx` mediante el hook `usePasswordRecovery.ts`:

```mermaid
stateDiagram-v2
    [*] --> PedirEmail : usuario accede a /forgot-password
    PedirEmail --> VerificarCodigo : POST /auth/forgot-password → código enviado al email
    VerificarCodigo --> NuevaContraseña : POST /auth/verify-reset-token → código válido
    NuevaContraseña --> [*] : POST /auth/reset-password → redirige a login
    VerificarCodigo --> VerificarCodigo : código incorrecto / expirado
```

---

## 5. Enrutamiento y páginas

Todas las rutas bajo `/dashboard` están envueltas en `RoleGuard` a través del `layout.tsx` del segmento, que verifica la sesión activa y el rol del usuario antes de renderizar.

```mermaid
graph LR
    Root["/"] -->|RoleGuard redirect| Dashboard["/dashboard"]
    Root --> Login["/auth/login"]
    Root --> ForgotPwd["/auth/forgot-password"]

    Dashboard --> DashPage["Dashboard Home\n(tarjetas de módulos)"]
    Dashboard --> Empleados["/dashboard/empleados"]
    Dashboard --> Supervisores["/dashboard/supervisores"]
    Dashboard --> Usuarios["/dashboard/usuarios"]
    Dashboard --> Evaluaciones["/dashboard/evaluaciones"]

    Login -->|success| Dashboard
```

### Roles y acceso por módulo

| Ruta | Roles permitidos |
|---|---|
| `/dashboard` | ADMINISTRATIVO, DIRECTOR, COORDINADOR, SUPERVISOR |
| `/dashboard/usuarios` | ADMINISTRATIVO, DIRECTOR |
| `/dashboard/empleados` | ADMINISTRATIVO, DIRECTOR |
| `/dashboard/supervisores` | ADMINISTRATIVO, DIRECTOR |
| `/dashboard/evaluaciones` | SUPERVISOR, DIRECTOR, COORDINADOR |

---

## 6. Control de acceso por rol

`RoleGuard` es un componente cliente que inspecciona la sesión de NextAuth y redirige si el rol del usuario no está en la lista `roles` permitida.

```mermaid
flowchart TD
    A[Render RoleGuard] --> B{status === loading?}
    B -->|sí| C[Mostrar LoadingOverlayScreen]
    B -->|no| D{¿Hay sesión?}
    D -->|no| E[router.push /auth/login]
    D -->|sí| F{¿Rol en lista permitida?}
    F -->|no| G[router.push /dashboard]
    F -->|sí| H[setIsAuthorized true]
    H --> I[Renderizar children]
```

---

## 7. Capa de datos — Hooks y API

### Instancia Axios (`services/api.ts`)

- Base URL configurable hacia `http://localhost:5000/api`.
- Función `setAuthToken` para inyectar el `Authorization: Bearer` header.
- Interceptor de respuesta para refresh token automático en caso de `401`.

### Patrón de hooks por entidad

Cada entidad de dominio tiene su propio módulo de hooks bajo `hooks/<entidad>/`. Todos siguen el mismo patrón:

```mermaid
classDiagram
    class useGetXxx {
        +useQuery(queryKey, queryFn)
        +filters: FilterDTO
        +pagination: object
    }
    class useGetXxxById {
        +useQuery(queryKey, queryFn)
        +id: number
        +enabled: boolean
    }
    class useCreateXxx {
        +useMutation(mutationFn)
        +onSuccess: invalidateQueries
    }
    class useUpdateXxx {
        +useMutation(mutationFn)
        +onSuccess: invalidateQueries
    }
    class useDeleteXxx {
        +useMutation(mutationFn)
        +onSuccess: invalidateQueries
    }

    useGetXxx ..> TanStackQuery
    useGetXxxById ..> TanStackQuery
    useCreateXxx ..> TanStackQuery
    useUpdateXxx ..> TanStackQuery
    useDeleteXxx ..> TanStackQuery
```

### Hooks disponibles por entidad

| Entidad | Query Key | Operaciones extra |
|---|---|---|
| Empleado | `"empleados"` | generateSolicitudPermiso, generateNotificacion, sendSolicitudPermisoEmail, sendNotificacion, generateEmpleadosReport |
| Usuario | `"usuarios"` | — |
| Supervisor | `"supervisores"` | — |
| Evaluación | `"evaluaciones"` | generatePdf, calcularTotales, sendEvaluacionEmail |
| ItemEvaluacion | `"itemsEvaluacion"` | — |
| EvaluacionItemResultado | `"evaluacionItemResultados"` | createOrUpdate, validateODIs |
| Auth | — | forgotPassword, verifyResetToken, resetPassword |

---

## 8. Componentes

### Componente genérico `CRUDTable<T>`

Es la pieza reutilizable central para todos los listados. Recibe columnas, datos y callbacks para operaciones CRUD.

```mermaid
classDiagram
    class CRUDTableProps~T~ {
        +data: T[]
        +total: number
        +page: number
        +limit: number
        +columns: Column~T~[]
        +filters: Filter~T~[]
        +onPageChange(page)
        +onEdit(row)
        +onDelete(row)
        +onDetail(row)
        +onCreate()
        +onFilter(filters)
        +title: string
    }

    class Column~T~ {
        +key: keyof T
        +label: string
        +render(row T) ReactNode
    }

    class Filter~T~ {
        +key: keyof T
        +type: select | date
        +label: string
        +options: SelectOption[]
    }

    CRUDTableProps~T~ --> Column~T~
    CRUDTableProps~T~ --> Filter~T~
```

### Jerarquía de componentes por módulo

```mermaid
graph TD
    subgraph "Módulo Empleados"
        EP[empleados/page.tsx] --> ECTB[EmpleadosCRUDTable]
        EP --> EFM[EmpleadoForm]
        EP --> SPF[SolicitudPermisoForm]
        EP --> DD1[DeleteDialogCRUD]
        ECTB --> CT1[CRUDTable]
    end

    subgraph "Módulo Usuarios"
        UP[usuarios/page.tsx] --> UCTB[UsuariosCRUDTable]
        UP --> UFM[UsuarioForm]
        UP --> DD2[DeleteDialogCRUD]
    end

    subgraph "Módulo Supervisores"
        SP[supervisores/page.tsx] --> SCTB[SupervisoresCRUDTable]
        SP --> SFM[SupervisorForm]
        SP --> DD3[DeleteDialogCRUD]
        SCTB --> CT3[CRUDTable]
    end

    subgraph "Módulo Evaluaciones"
        EVP[evaluaciones/page.tsx] --> EVCTB[EvaluacionesCRUDTable]
        EVP --> EVFM[EvaluacionForm]
        EVP --> DD4[DeleteDialogCRUD]
        EVP --> MEF[EnviarEmailForm]
        EVFM --> EMPS[EmpleadoSelect]
        EVFM --> EVDS[EvaluadorSelect]
        EVFM --> SUPS[SupervisorSelect]
        EVFM --> IEVT[ItemsEvaluacionTable]
    end
```

### Patrón de modal único por página

Cada página de módulo usa un estado `modal: ModalType` para controlar un único `<Modal>` de Mantine con contenido dinámico. Los tipos posibles son: `none`, `create`, `edit`, `view`, `delete`, `mail-*`.

---

## 9. Schemas de validación (Zod)

Los schemas sirven simultáneamente como contrato de tipos TypeScript y como validadores en tiempo de ejecución para formularios y DTOs de red.

```mermaid
classDiagram
    class EmpleadoSchema {
        +nombres: string
        +apellidos: string
        +cedula: string (V/E-XXXXXXXX)
        +tipoPersonal: TipoPersonalEnum
        +sexo: M | F
        +fechaIngreso: Date
        +dependencia: string?
        +cargo: string
        +situacionLaboral: string?
        +horasAcademicas: number?
        +usuarioId: number?
    }

    class UsuarioSchema {
        +email: string
        +password: string (strongPassword)
        +rol: ADMINISTRATIVO | DIRECTOR | COORDINADOR | SUPERVISOR
        +empleado?: EmpleadoCreateSchema
        +supervisor?: SupervisorCreateSchema
    }

    class SupervisorSchema {
        +nombres: string
        +apellidos: string
        +cedula: string
        +dependencia: string
        +cargo: string
        +nivelJerarquico: string
    }

    class EvaluacionSchema {
        +periodoDesde: Date
        +periodoHasta: Date
        +idEvaluado: number
        +idEvaluador: number
        +idSupervisorExterno: number
        +textoCursosRealizados: string?
        +acotacionesSupervisor: string?
        +itemsResultado: EvaluacionItemResultado[]
    }

    class ItemEvaluacionSchema {
        +nombre: string
        +tipoItem: ODI | COMPETENCIA
        +peso: number
        +rangoMinimo: number
        +rangoMaximo: number
    }

    class EvaluacionItemResultadoSchema {
        +evaluacionId: number
        +itemEvaluacionId: number
        +rangoSeleccionado: number
        +pesoXrango: number
        +comentario: string?
    }

    UsuarioSchema --> EmpleadoSchema : discriminatedUnion SUPERVISOR=false
    UsuarioSchema --> SupervisorSchema : discriminatedUnion SUPERVISOR=true
    EvaluacionSchema --> EvaluacionItemResultadoSchema
    EvaluacionItemResultadoSchema --> ItemEvaluacionSchema
```

### Roles del sistema

```mermaid
graph LR
    R1[DIRECTOR]
    R2[ADMINISTRATIVO]
    R3[COORDINADOR]
    R4[SUPERVISOR]

    R1 -->|accede a| M1[Usuarios]
    R1 -->|accede a| M2[Empleados]
    R1 -->|accede a| M3[Supervisores]
    R1 -->|accede a| M4[Evaluaciones]

    R2 -->|accede a| M1
    R2 -->|accede a| M2
    R2 -->|accede a| M3

    R3 -->|accede a| M4

    R4 -->|accede a| M4
```

---

## 10. Helpers

| Archivo | Función principal |
|---|---|
| `evaluacion.ts` | `calcularRangoActuacion(total)` — convierte puntuación a rango textual; `calcularTotales(items)` — suma módulo I (ODI) y módulo II (COMPETENCIAS) |
| `selectsData.ts` | Datos estáticos para todos los `<Select>`: roles, tipos de personal, dependencias, cargos, materias, especialidades |
| `dateHelpers.ts` | `parseDDMMYYYY` — convierte cadenas `DD-MM-YYYY` a objetos `Date` |
| `date-parse.ts` | Utilidades auxiliares de parseo de fechas |
| `badges.ts` | Mapeo de valores de enum a colores de badge para Mantine |

### Lógica de evaluación — Rangos de actuación

```mermaid
graph LR
    A["totalFinal < 100"] --> Z1["Muy bajo"]
    B["100–124"] --> Z2["Deficiente"]
    C["125–149"] --> Z3["Regular"]
    D["150–374"] --> Z4["Bueno"]
    E["375–499"] --> Z5["Muy bueno"]
    F["≥ 500"] --> Z6["Excelente"]
```

---

## 11. Diagrama de componentes por módulo

```mermaid
graph TB
    subgraph "Root Layout (app/layout.tsx)"
        SessP[SessionProvider]
        QP[QueryProvider]
        MP[MantineProvider]
        Notif[Notifications]
    end

    subgraph "Dashboard Layout"
        RG[RoleGuard]
        AS[AppShell Mantine]
        NB[NavbarContent]
    end

    subgraph "Módulos"
        DashPage[Dashboard Page]
        EmplPage[Empleados Page]
        SupPage[Supervisores Page]
        UsrPage[Usuarios Page]
        EvalPage[Evaluaciones Page]
    end

    SessP --> QP --> MP --> Notif
    Notif --> RG --> AS --> NB
    AS --> DashPage
    AS --> EmplPage
    AS --> SupPage
    AS --> UsrPage
    AS --> EvalPage
```

---

## 12. Diagrama de secuencia — Login

```mermaid
sequenceDiagram
    actor U as Usuario
    participant LF as LoginForm
    participant NA as NextAuth (auth.ts)
    participant API as Backend /api/auth/login
    participant RG as RoleGuard
    participant DB as Dashboard

    U->>LF: submit { email, password }
    LF->>NA: signIn("credentials", redirect:false)
    NA->>API: POST /auth/login { email, password }
    API-->>NA: { user, accessToken, refreshToken }
    NA-->>LF: result { url: "/dashboard" }
    LF->>DB: router.push("/dashboard")
    DB->>RG: verificar sesión y rol
    RG-->>DB: renderizar contenido
```

---

## 13. Diagrama de secuencia — CRUD genérico

El siguiente diagrama representa el flujo estándar de cualquier operación de creación en los módulos (Empleados, Usuarios, Supervisores, Evaluaciones):

```mermaid
sequenceDiagram
    actor U as Usuario
    participant Page as Page (dashboard/xxx)
    participant Table as XxxCRUDTable
    participant Modal as Modal + XxxForm
    participant Hook as useCreateXxx
    participant TQ as TanStack Query
    participant API as Backend REST

    U->>Table: Click "Crear"
    Table->>Page: onCreate()
    Page->>Modal: setModal("create")
    Modal-->>U: Muestra formulario vacío

    U->>Modal: Completa y envía form
    Modal->>Page: onSubmit(values)
    Page->>Hook: mutateAsync(dto)
    Hook->>API: POST /api/xxx { ...dto }
    API-->>Hook: 201 Created { data }
    Hook->>TQ: invalidateQueries(queryKey)
    TQ->>API: GET /api/xxx (refetch)
    API-->>TQ: lista actualizada
    TQ-->>Table: nueva data
    Page->>Modal: setModal("none")
    Page-->>U: notification "Creado correctamente"
```

---

## 14. Diagrama de clases — DTOs / Schemas

```mermaid
classDiagram
    class PaginatedResult~T~ {
        +results: T[]
        +total: number
        +page: number
        +limit: number
    }

    class EmpleadoResponseDTO {
        +id: number
        +nombres: string
        +apellidos: string
        +cedula: string
        +tipoPersonal: string
        +cargo: string
        +dependencia: string
        +fechaIngreso: Date
        +usuarioId: number
        +createdAt: Date
        +updatedAt: Date
    }

    class UsuarioResponseDTO {
        +id: number
        +email: string
        +rol: string
        +empleadoId: number
        +supervisorId: number
        +empleado: EmpleadoResponseDTO
        +supervisor: SupervisorResponseDTO
        +createdAt: Date
        +updatedAt: Date
    }

    class SupervisorResponseDTO {
        +id: number
        +nombres: string
        +apellidos: string
        +cedula: string
        +dependencia: string
        +cargo: string
        +nivelJerarquico: string
    }

    class EvaluacionResponseDTO {
        +id: number
        +periodoDesde: string
        +periodoHasta: string
        +idEvaluado: number
        +idEvaluador: number
        +idSupervisorExterno: number
        +totalModuloI: number
        +totalModuloII: number
        +totalFinal: number
        +rangoActuacion: string
        +textoCursosRealizados: string
        +acotacionesSupervisor: string
        +itemsResultado: EvaluacionItemResultadoDTO[]
        +createdAt: string
        +updatedAt: string
    }

    class ItemEvaluacionResponseDTO {
        +id: number
        +nombre: string
        +tipoItem: ODI | COMPETENCIA
        +peso: number
        +rangoMinimo: number
        +rangoMaximo: number
    }

    class EvaluacionItemResultadoDTO {
        +evaluacionId: number
        +itemEvaluacionId: number
        +rangoSeleccionado: number
        +pesoXrango: number
        +comentario: string
        +itemEvaluacion: ItemEvaluacionResponseDTO
    }

    PaginatedResult~EmpleadoResponseDTO~ --> EmpleadoResponseDTO
    PaginatedResult~EvaluacionResponseDTO~ --> EvaluacionResponseDTO
    UsuarioResponseDTO --> EmpleadoResponseDTO
    UsuarioResponseDTO --> SupervisorResponseDTO
    EvaluacionResponseDTO --> EvaluacionItemResultadoDTO
    EvaluacionItemResultadoDTO --> ItemEvaluacionResponseDTO
```

---

## 15. Diagrama de estados — modal CRUD

Todos los módulos de dashboard comparten el mismo patrón de estado de modal:

```mermaid
stateDiagram-v2
    [*] --> none : Carga de página

    none --> create : onClick "Crear"
    none --> edit : onClick "Editar" (selecciona registro)
    none --> view : onClick "Ver detalle" (selecciona registro)
    none --> delete : onClick "Eliminar" (selecciona registro)
    none --> mail : onClick "Enviar email" (selecciona registro)

    create --> none : onClose / onSubmit success
    edit --> none : onClose / onSubmit success
    view --> none : onClose
    delete --> none : onConfirm success / onCancel
    mail --> none : onClose / onSubmit success

    create --> none : error (permanece con notificación)
    edit --> none : error (permanece con notificación)
```

---

> **Generado automáticamente** a partir del análisis del código fuente en `web/`.  
> Última actualización: junio 2026.
