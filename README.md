# Sistema de Gestión de Proyectos de Graduación UNI

Plataforma web para registrar, revisar, aprobar y dar seguimiento a los proyectos de graduación de la Universidad Nacional de Ingeniería. Documentada con el **Modelo C4** (niveles 1 y 2), **diagramas de secuencia** y una arquitectura **API-First**, usando **Mermaid** para que la arquitectura quede versionada junto al código.

> Asignatura: Diseño de Sistemas en Internet — Tema: Arquitectura de Sistemas Web con Modelo C4, API-First y Mermaid.

## Problema

Las facultades administran las propuestas de graduación en hojas de cálculo y correo electrónico. Ese flujo no deja rastro auditable: no se sabe quién revisó qué, en qué estado está cada propuesta ni cuándo venció una observación. El sistema centraliza el ciclo de vida del proyecto (propuesta → revisión → aprobada → en desarrollo → defensa → cerrada) y deja cada transición registrada.

## Arquitectura

El proyecto utiliza el Modelo C4 para documentar el contexto y los contenedores principales, complementado con diagramas de secuencia para describir las interacciones API-First.

### C4 Nivel 1 — Contexto

```mermaid
C4Context
title Sistema de Gestion de Proyectos de Graduacion UNI - Contexto

Person(estudiante, "Estudiante", "Registra su propuesta, entrega documentos y consulta el avance de su proyecto")
Person(tutor, "Tutor", "Revisa propuestas y documentos, registra observaciones y valida avances")
Person(coordinador, "Coordinador", "Asigna tutores, administra estados y seguimiento academico")
Person(admin, "Administrador", "Administra usuarios, roles y catalogos del sistema")

System(uni, "Sistema de Gestion de Proyectos de Graduacion UNI", "Plataforma web para registrar, revisar, aprobar y dar seguimiento a los proyectos de graduacion")

System_Ext(registro, "Registro Academico UNI", "Fuente oficial de datos de estudiantes, carreras y ciclos academicos")
System_Ext(email, "Correo institucional", "Envia notificaciones academicas de estado y de asignacion")
System_Ext(storage, "Almacenamiento de documentos", "Conserva propuestas, avances, anexos y actas de defensa")

Rel(estudiante, uni, "Registra propuestas, entrega documentos y consulta avances")
Rel(tutor, uni, "Revisa proyectos y registra observaciones")
Rel(coordinador, uni, "Gestiona asignaciones, estados y seguimiento")
Rel(admin, uni, "Administra usuarios y catalogos")

Rel(uni, registro, "Consulta datos academicos oficiales", "HTTPS/JSON")
Rel(uni, email, "Envia notificaciones", "SMTP")
Rel(uni, storage, "Almacena y recupera documentos", "HTTPS")
```

### C4 Nivel 2 — Contenedores

```mermaid
C4Container
title Sistema de Gestion de Proyectos de Graduacion UNI - Contenedores

Person(estudiante, "Estudiante", "Usuario academico")
Person(tutor, "Tutor", "Docente tutor")
Person(coordinador, "Coordinador", "Responsable academico")
Person(admin, "Administrador", "Gestor de usuarios y catalogos")

System_Boundary(uni, "Sistema de Gestion de Proyectos de Graduacion UNI") {
    Container(web, "Aplicacion Web", "React / TypeScript", "Interfaz para estudiantes, tutores, coordinadores y administradores")
    Container(api, "API REST", "ASP.NET Core Web API / C#", "Autenticacion, reglas de negocio, DTO y servicios HTTP")
    Container(worker, "Servicio de Reportes", "Worker en C# / Hangfire", "Genera reportes de avance y consolidados por carrera de forma asincrona")
    ContainerDb(db, "Base de Datos", "SQL Server", "Usuarios, proyectos, revisiones, estados y catalogos")
    ContainerDb(files, "Gestor de Documentos", "Almacenamiento de objetos", "Propuestas, avances, anexos y actas de defensa")
}

System_Ext(email, "Correo institucional", "SMTP / proveedor de correo")
System_Ext(registro, "Registro Academico UNI", "Datos academicos oficiales")

Rel(estudiante, web, "Usa", "HTTPS")
Rel(tutor, web, "Usa", "HTTPS")
Rel(coordinador, web, "Usa", "HTTPS")
Rel(admin, web, "Usa", "HTTPS")

Rel(web, api, "Consume API REST", "HTTPS/JSON")
Rel(api, db, "Lee y escribe", "EF Core / SQL")
Rel(api, files, "Sube y recupera documentos", "HTTPS")
Rel(api, worker, "Encola trabajos de reporte", "HTTPS/JSON")
Rel(worker, db, "Lee datos de proyectos", "EF Core / SQL")
Rel(api, email, "Envia notificaciones", "SMTP/API")
Rel(api, registro, "Consulta datos academicos", "HTTPS/JSON")
```

### Secuencia — Login con JWT y refresco de token

```mermaid
sequenceDiagram
    autonumber
    actor E as Estudiante
    participant W as Aplicacion Web
    participant A as API REST
    participant DB as SQL Server

    E->>W: Ingresa correo y contrasena
    W->>A: POST /api/auth/login
    activate A
    A->>DB: SELECT usuario por correo
    DB-->>A: Usuario + hash PBKDF2 + rol
    A->>A: Verificar contrasena con PBKDF2
    alt Credenciales validas
        A->>A: Emitir access token (JWT) y refresh token
        A->>DB: Registrar refresh token vigente
        DB-->>A: Confirmacion de registro
        A-->>W: 200 OK + access token + refresh token
        W->>W: Conserva los tokens en memoria
        W->>A: GET /api/projects/mine
        Note right of A: Header Authorization con el access token emitido
        A->>A: Validar firma y vigencia del token
        A->>DB: Consultar proyectos del estudiante
        DB-->>A: Lista de proyectos
        A-->>W: 200 OK + JSON
        W-->>E: Muestra el panel con sus proyectos
    else Credenciales invalidas
        A->>A: Incrementa el contador de intentos
        A-->>W: 401 Unauthorized
        W-->>E: Muestra mensaje de autenticacion
    end
    deactivate A

    Note over E,W: El access token expira y la sesion debe renovarse
    W->>A: POST /api/auth/refresh
    A->>DB: Validar refresh token vigente
    DB-->>A: Token valido
    A-->>W: 200 OK + nuevo access token
    W->>A: GET /api/projects/mine
    A-->>W: 200 OK + JSON
    W-->>E: Continua navegando sin volver a digitar la clave
```

### Secuencia — Aprobación de propuesta por el coordinador

```mermaid
sequenceDiagram
    autonumber
    actor C as Coordinador
    participant W as Aplicacion Web
    participant A as API REST
    participant DB as SQL Server
    participant M as Servicio de correo

    C->>W: Abre una propuesta en estado revision
    W->>A: GET /api/projects/{id}
    activate A
    A->>DB: Consultar proyecto, documentos y revisiones
    DB-->>A: Proyecto + documentos + historial
    A-->>W: 200 OK + JSON
    W-->>C: Muestra la propuesta completa
    deactivate A

    C->>W: Registra observaciones y aprueba
    W->>A: PATCH /api/projects/{id}
    activate A
    A->>A: Validar transicion de estados permitida
    A->>DB: UPDATE estado = aprobada, tutor_id
    DB-->>A: Fila actualizada
    A->>M: Notificar al estudiante y al tutor asignado
    M-->>A: 202 Accepted
    A-->>W: 200 OK + proyecto actualizado
    W-->>C: Muestra confirmacion del cambio
    deactivate A

    Note over A,M: El estudiante y el tutor reciben el correo con el enlace al proyecto

    rect rgba(6, 78, 59, 0.08)
        W->>A: GET /api/projects/{id}
        A->>DB: Reconsultar proyecto
        DB-->>A: Proyecto con estado aprobada
        A-->>W: 200 OK + JSON
        W-->>C: Refresca la tabla de seguimiento
    end
```

## Contrato de la API

La API se diseña antes que la interfaz (API-First). El contrato se expone con OpenAPI/Swagger y se valida antes de implementar los consumidores.

| Recurso | Método | Endpoint | Éxito | Errores principales |
|---|---|---|---|---|
| Autenticación | POST | `/api/auth/login` | 200 | 400, 401 |
| Refresco de token | POST | `/api/auth/refresh` | 200 | 401 |
| Mis proyectos | GET | `/api/projects/mine` | 200 | 401 |
| Proyecto | GET | `/api/projects/{id}` | 200 | 401, 404 |
| Crear propuesta | POST | `/api/projects` | 201 | 400, 401 |
| Actualizar proyecto | PATCH | `/api/projects/{id}` | 200 | 400, 401, 403, 404 |
| Revisión | POST | `/api/projects/{id}/reviews` | 201 | 400, 401, 403, 404 |

## Ejecución

```bash
git clone https://github.com/JhonAlon/graduacion-uni-c4.git
cd graduacion-uni-c4
dotnet restore
dotnet run --project src/Graduacion.Api
```

Swagger queda disponible en `/swagger` una vez levantado el servidor.

## Decisiones técnicas

- **API-First**: el contrato HTTP se define en OpenAPI y a partir de él se genera el cliente de la aplicación web, de modo que frontend y backend evolucionan contra el mismo acuerdo.
- **DTO y separación de responsabilidades**: los controladores quedan delgados; la lógica de negocio vive en servicios, y las entidades de persistencia nunca se devuelven al cliente.
- **Inyección de dependencias**: los servicios se registran en `Program.cs` y se inyectan por constructor, lo que facilita sustituir el almacén de documentos o el proveedor de correo.
- **Autenticación**: contraseñas con PBKDF2 (nunca en texto plano), access token JWT de vida corta y refresh token vigente para renovar la sesión sin volver a digitar la clave.
- **Documentos separados de la base relacional**: los PDF pesados se guardan en un almacén de objetos; SQL Server conserva solo metadatos transaccionales.
- **Arquitectura como código**: los diagramas Mermaid viven en el README, por lo que se versionan y se revisan en el mismo PR que cambia la implementación.
