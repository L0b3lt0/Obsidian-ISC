# Contexto del proyecto TMR — Módulo de Permisos

> Documento de traspaso y estado actual del sistema **TMR (Time Report)** y del feature de **Solicitudes de Permisos, Licencias y Vacaciones**.
> Fecha de corte: **2026-11-05**. Rama sincronizada en ambos repositorios (`origin/feat/permisos`): **`feat/permisos`**.
> Este documento **no contiene credenciales**. Pídelas al equipo si las necesitas; nunca las escribas en archivos versionados (`appsettings.json` debe permanecer vacío en Git).

---

## 1. Resumen del Sistema TMR

**TMR (Time Report)** es la plataforma interna de **Integrity Solutions Consulting (ISC)** para registrar, auditar y gestionar las horas que los colaboradores dedican a proyectos de clientes e internos. Sus capacidades principales son:

- **Time Report y Actividades:** Registro diario de horas y actividades por proyecto, cálculo de horas laborables por quincena/mes descontando feriados, seguimiento por líder y exportación de reportes ejecutivos (Excel/PDF).
- **Administración y Configuración:** Gestión de clientes, proyectos, asignaciones de proyecto, líderes (internos ISC y externos de clientes), colaboradores (con historial de contratos y fechas de salida), proveedores, catálogos maestros, feriados y control de acceso basado en roles y permisos (RBAC).
- **Notificaciones de Horas Faltantes:** Detección automática en fechas de corte (Quartz) y envíos manuales por proyecto mediante patrón **Transactional Outbox** (`notificacion.evento_outbox` + `EventDispatcher`), con despacho por correo (**Apprise / MailKit SMTP**) y avisos en tiempo real por **Server-Sent Events (SSE)**.
- **Módulo de Solicitudes de Permisos, Licencias y Vacaciones (`feat/permisos`):** Flujo integral de solicitudes con aprobación de Nivel 1 (Jefe Inmediato interno en TMR o Jefe Externo de cliente vía Magic Link sin sesión), confirmación de Nivel 2 (Talento Humano), consulta gerencial, evidencias en **MinIO S3** y generación de formulario oficial en PDF (**QuestPDF**).

Repositorios (organización GitHub `Integrity-Solutions-Consulting`), ubicados en `C:\Users\allan.erazo\Desktop\TMR\`:

| Repo | Carpeta | Último commit en `origin/feat/permisos` | Stack principal |
|---|---|---|---|
| `tmr-backend` | `tmr-backend/` | `2df12d1` | .NET 10, ASP.NET Core Minimal APIs, EF Core 10 + Npgsql (PostgreSQL 16), Serilog, Quartz, QuestPDF, MailKit, FluentValidation, JWT, BCrypt |
| `tmr-frontend` | `tmr-frontend/` | `1c1b59a` | Angular 21 (standalone components + signals), Angular Material, Tailwind 4, RxJS 7.8 |

---

## 2. Arquitectura del Sistema

### 2.1. Backend (`tmr-backend`)
- **`Program.cs`:** Registro modular de servicios y pipeline HTTP. Autorización con `FallbackPolicy` de usuario autenticado (los endpoints públicos usan `.AllowAnonymous()`), `PermissionPolicyProvider` dinámico para políticas por código RBAC (`.RequireAuthorization("CODIGO_PERMISO")`), middlewares de manejo global de excepciones, auditoría de rendimiento, cabeceras de seguridad, CORS, lista negra de sesiones JWT y rate limiting.
- **`Features/<Modulo>/`:** Arquitectura vertical por funcionalidad (`Permisos`, `TimeReport`, `Notificaciones`, `Colaboradores`, `Proyectos`, `Clientes`, `Lideres`, `Proveedores`, `Configuracion`, `Auth`, `Dashboard`, `Reportes`).
- **`Infrastructure/Messaging/`:** Bus de eventos persistido en PostgreSQL (`notificacion.evento_outbox`) y procesado en segundo plano por `EventDispatcher` con hasta 5 reintentos automáticos.
- **`Documentacion/Scripts_DB/NN-*.sql`:** Scripts SQL numerados embebidos en el ensamblado y ejecutados por `DatabaseInstaller` (`01` al `19`, excepto `06`). Registro de versiones aplicadas en `public.tmr_installation_history`.
- **Esquemas PostgreSQL:**
  - `autenticacion`: Usuarios (`tbl_autenticacion_usuario`), roles (`tbl_autenticacion_rol`), módulos (`tbl_autenticacion_modulo`), permisos (`tbl_autenticacion_permiso`), sesiones.
  - `administracion`: Personas (`tbl_administracion_persona`), empleados (`tbl_administracion_empleado`), contratos (`tbl_administracion_empleado_contrato`), cargos (`tbl_administracion_cargo`), líderes (`tbl_administracion_lider`, `idtipo`: `66 = INT`, `67 = EXT`), clientes (`tbl_administracion_cliente`), catálogos (`tbl_administracion_catalogo` y `tbl_administracion_catalogo_detalle`).
  - `time_report`: Proyectos, asignaciones (`tbl_time_report_asignacion_proyecto`, donde la fila con `idempleado = NULL` e `idlider != NULL` define al líder principal del proyecto), actividades, feriados (`tbl_time_report_feriados`) y las 5 tablas normalizadas de solicitudes de permisos.
  - `notificacion`: Cola `evento_outbox`, historial `notificacion_envio`, fechas de corte y bandeja de campana `notificacion_usuario`.

### 2.2. Frontend (`tmr-frontend`)
- **`src/app/app.routes.ts`:** Enrutamiento con `authGuard` para el layout principal y `roleGuard` activo en cada módulo (incluyendo `/permisos`, `/permisos/equipo`, `/permisos/rrhh` y `/permisos/gerencia`).
- **`src/app/core/`:** Guards, interceptores (`ApiResponseInterceptor` que desenvuelve automáticamente el sobre `{ success, statusCode, message, data }`), `TokenService` y layout (`sidebar`, `navbar` con campana SSE).
- **`src/app/features/time-report/permisos/`:** Componentes standalone del flujo de permisos conectados al backend sin datos simulados.

---

## 3. Estado Actual del Feature `feat/permisos` (Lo que ya está implementado y corregido)

### 3.1. Flujo Funcional y Reglas de Negocio
```
Colaborador registra solicitud (con evidencias opcionales o certificado médico obligatorio RF-004)
        │  Estado inicial: PENDIENTE_JEFE
        ▼
Nivel 1: Jefe Inmediato
   ├─ Líder Interno (INT) → Aprueba / Observa / Rechaza en TMR (/permisos/equipo)
   └─ Líder Externo (EXT) → Aprueba / Observa / Rechaza por Magic Link (/permisos/revision-externa?token=...)
        │  Si aprueba → PENDIENTE_TH | Si observa → OBSERVADA | Si rechaza → RECHAZADA
        ▼
Nivel 2: Talento Humano (RRHH — /permisos/rrhh)
   ├─ Confirmar → APROBADA (aplica impacto en Time Report y habilita PDF final con firmas)
   └─ Devolver  → OBSERVADA (con comentario obligatorio para subsanación)
        ▼
Gerencia (/permisos/gerencia): Tablero consolidado de consulta y descarga de PDFs
```

### 3.2. Correcciones Arquitectónicas y de Seguridad Completadas (Commit `2df12d1` y `1c1b59a`)
1. **Seguridad de Credenciales (`appsettings.json`):** `appsettings.json` no contiene ninguna credencial ni cadena de conexión. Las configuraciones locales viven únicamente en `appsettings.Development.json` (no commiteado) o variables de entorno.
2. **Sin DDL en Runtime:** Se eliminó `AsegurarEsquemaSolicitudesAsync` de `PermisosEndpoints.cs`; el esquema depende exclusivamente del Script 19 (`19-Inv_tmr_db_modulo_solicitudes.sql`, ya ejecutado en BD).
3. **Validación Estricta de Ficha de Colaborador:** Se eliminó la creación silenciosa de registros ficticios (`USR-xxxxx` / `EMP-xxx`). Si un usuario sin ficha activa en `administracion.tbl_administracion_empleado` intenta crear una solicitud, recibe `400 Bad Request` con un mensaje claro.
4. **Almacenamiento MinIO S3 Nativo y Compensación en Rollback:**
   - `PermisosStorageService.cs` opera exclusivamente contra el API S3 (puerto `9002` en Dokploy) usando firma **AWS Signature V4 (`AWS4-HMAC-SHA256`)**, validando tamaño (`<= 5 MB`), extensión y cabecera binaria (magic bytes).
   - Se eliminaron las ~550 líneas del puente frágil a la consola web de MinIO y se agregó `EliminarAdjuntoAsync` para borrar objetos subidos a MinIO si falla el commit de la transacción en PostgreSQL.
5. **Generación de Código `PER-{ANIO}-{NNN}` Segura ante Concurrencia:** Calcula el máximo sufijo numérico anual e incorpora hasta 5 reintentos automáticos ante `23505` (`UniqueViolation`).
6. **Máquina de Estados y Reglas Anti-Autodecisión:**
   - Nadie puede aprobar, observar, rechazar o confirmar su propia solicitud (`403 Forbidden`).
   - `decision-lider` exige estado `PENDIENTE_JEFE`.
   - `decision-rrhh` valida acción (`confirmar` | `devolver`), rechaza solicitudes ya cerradas (`APROBADA`, `RECHAZADA`, `OBSERVADA`) y exige `PENDIENTE_TH` (salvo solicitudes sin líder asignado).
   - `TokenExterno` nunca se expone en las respuestas JSON de la API; solo se inyecta internamente al encolar el correo para el líder externo.
7. **Feriados, Traslape Propio y Saldo de Vacaciones por Antigüedad:**
   - Descuenta fines de semana y días feriados (fijos y recurrentes de `time_report.tbl_time_report_feriados`) tanto en los días solicitados como en la fecha de reincorporación.
   - Bloquea solicitudes con fechas solapadas del mismo colaborador (permitiendo medias jornadas complementarias `MANANA` y `TARDE` el mismo día).
   - Calcula los días disponibles de vacaciones según la antigüedad real (`Fechaingreso`, Art. 69 Código del Trabajo: 15 días base + 1 día por año adicional a partir del 6.º año, hasta 15 extra) menos los días consumidos/en trámite del año.
8. **Políticas RBAC en Backend y `roleGuard` en Frontend:**
   - Todos los endpoints de `PermisosEndpoints.cs` aplican las políticas del Script 19 (`MIS_PERMISOS_READ`, `MIS_PERMISOS_CREATE`, `PERMISOS_VIEW`, `PERMISOS_LIDER_READ`, `PERMISOS_LIDER_APPROVE`, `PERMISOS_RRHH_READ`, `PERMISOS_RRHH_CONFIRM`, `PERMISOS_GERENCIA_READ`).
   - `/api/permisos/storage/estado` está restringido a Administradores y Talento Humano.
   - En `app.routes.ts` todas las rutas internas de permisos tienen `canActivate: [roleGuard]`.
9. **Unificación de PDF Oficial (QuestPDF):**
   - `PermisosPdfService` en el frontend descarga directamente el PDF generado por el backend (`GET /api/permisos/{id}/pdf` y `GET /api/permisos/externo/{token}/pdf`), eliminando la duplicación en `jsPDF`.
10. **Notificaciones por `EventoOutbox`, Destinatario Único de Talento Humano y Fallback SMTP:**
    - `PermisosNotificacionService` publica eventos `CorreoPermisoEncolado` en `notificacion.evento_outbox`, despachados en segundo plano por `EventDispatcher` y `EnviarCorreoPermisoHandler`.
    - **Destinatario de Talento Humano dinámico por Cargo:** Tanto los correos (`ObtenerDestinatariosRrhhAsync`) como la campana SSE (`ObtenerIdsUsuariosRrhhAsync`) buscan primero al colaborador activo con el cargo **`Líder de Talento Humano`** (`CargoId = 29`, asignado actualmente en BD a **Diego Emilio López Espinoza** — `diego.lopez@integritysolutions.com.ec`, quien también cuenta con el rol `RECURSOS HUMANOS`). Si en el futuro cambia el jefe de Talento Humano, basta con reasignar dicho cargo en el módulo **Colaboradores**. Solo si ningún empleado activo tiene ese cargo, el sistema usa como respaldo a los usuarios con rol `RECURSOS HUMANOS`.
    - **Respaldo automático Apprise → MailKit SMTP:** Si el contenedor de Apprise responde error (p. ej. `404`), `EnviarCorreoPermisoHandler` cae automáticamente a `IEmailService` (MailKit SMTP, compatible con el puerto `1025` de MailDev).

---

## 4. Usuarios y Datos de Prueba en Base de Datos (`isc-tmr-migration`)

| Rol en el Flujo | Correo (`email`) | Vínculo en BD | Ruta en Frontend |
|---|---|---|---|
| **Colaborador (Solicitante)** | `colaborador@tmr.com` | `prueba sanchez` (`EmpId=6`, `RPS0006`, Nómina/Pasantía) | `/permisos` |
| **Colaborador Alterno** | `prueba@integritysolutions.com.ec` | `Prueba Colaborador` (`EmpId=84`, `ISC0669`, Nómina/Pasantía) | `/permisos` |
| **Jefe Inmediato Interno** | `lider@tmr.com` | `Líder Prueba Interno` (`LiderId=60`, `INT`, proyecto `PRUEBA PERMISOS INTERNO` `Id=39`) | `/permisos/equipo` |
| **Jefe Inmediato Externo** | *(Sin sesión TMR — Magic Link)* | `Líder Prueba Externo` (`LiderId=61`, `EXT`, `lider.externo@tmr.com`, proyecto `PRUEBA PERMISOS EXTERNO` `Id=40`) | `/permisos/revision-externa?token=...` |
| **Líder de Talento Humano (RRHH)** | `diego.lopez@integritysolutions.com.ec` *(o `rec1@tmr.com`)* | `Diego Emilio Lopez Espinoza` (`EmpId=2021`, Cargo: `Líder de Talento Humano`, Roles: `LIDER`, `RECURSOS HUMANOS`) | `/permisos/rrhh` |
| **Gerencia** | `ger1@tmr.com` | Rol `GERENCIA` | `/permisos/gerencia` |
| **Administrador** | `admin@tmr.com` / `bde@tmr.com` | Rol `ADMINISTRADOR` (`bde@tmr.com` no tiene ficha de empleado, útil para probar bloqueo #5) | Todas las rutas |

Consulta SQL rápida para obtener el Magic Link de una solicitud externa en pruebas:
```sql
SELECT codigo, 'http://localhost:4200/permisos/revision-externa?token=' || tokenexterno AS magic_link
FROM time_report.tbl_time_report_solicitud
ORDER BY id DESC
LIMIT 5;
```

---

## 5. En qué se debe continuar (Próximos Pasos)

1. **Ejecución de Pruebas Funcionales y Casos Negativos (End-to-End):**
   - **Flujo con Líder Interno:** Crear solicitud con `colaborador@tmr.com` en el proyecto `PRUEBA PERMISOS INTERNO` → aprobar/observar/rechazar con `lider@tmr.com` en `/permisos/equipo` → confirmar/devolver en `/permisos/rrhh` con `diego.lopez@integritysolutions.com.ec` (o `rec1@tmr.com`) → verificar visualización y descarga de PDF oficial en `/permisos/gerencia` con `ger1@tmr.com`.
   - **Flujo con Líder Externo (Magic Link):** Crear solicitud con `colaborador@tmr.com` en `PRUEBA PERMISOS EXTERNO` → abrir el enlace `/permisos/revision-externa?token=...` sin sesión → aprobar/observar/rechazar → verificar que el token quede invalidado tras el primer uso y probar el botón **Reenviar enlace** desde `/permisos/rrhh`.
   - **Casos negativos a validar:**
     - Intentar crear solicitud con `bde@tmr.com` (sin ficha de empleado) → debe responder `400`.
     - Crear solicitud en las mismas fechas de otra solicitud activa del mismo colaborador → debe responder `400` por traslape.
     - Solicitar **Licencia por enfermedad > 1 día hábil** con colaborador de Nómina sin adjuntar certificado (`RF-004`) → debe responder `400`.
     - Intentar subir archivo `> 5 MB` o con extensión/cabecera binaria inválida (`RNF-003`) → debe responder `400`.
     - Intentar aprobar/confirmar una solicitud propia (`403`) o confirmar dos veces en RRHH una solicitud ya `APROBADA` (`400`).

2. **Infraestructura de Correo y Base de Datos en Dokploy:**
   - **Contenedores Apprise y MailDev en Dokploy:** Verificar y levantar en el panel de Dokploy el contenedor de **Apprise** (`https://apprise.dokploy.integritysolutions.com.ec/notify` actualmente responde `404 page not found` en Traefik) y la interfaz web de **MailDev** (`https://smtp-maildev.dokploy.integritysolutions.com.ec`, cuyo puerto SMTP `1025` sí está activo y recibe los correos vía fallback SMTP).
   - **Limpieza de datos y secuencias en BD de migración:**
     - Eliminar o desactivar la ficha duplicada `EMP-005` (`EmpId = 98`) de `admin@tmr.com` que había sido generada por la versión anterior.
     - Sincronizar con `setval` las secuencias de `administracion.tbl_administracion_persona` y `time_report.tbl_time_report_proyecto` al `MAX(id)` actual para evitar colisiones de clave primaria en futuros registros.
   - **Variables de entorno en despliegue Dokploy:** Asegurar que en el servicio de `tmr-backend` en Dokploy estén definidas las variables `Minio__Endpoint` (apuntando al puerto S3 `9002`), `Minio__AccessKey`, `Minio__SecretKey`, `Minio__BucketName=permisos`, `EmailSettings__FrontendUrl` y `Apprise__*`.

3. **Integración con Time Report y Cierre del Feature (`PR` a `develop`):**
   - Verificar cómo el estado `APROBADA` (con `ImpactoTimeReport = 'RESTA_META'`) se refleja en los reportes de cumplimiento de horas del período si el equipo requiere descuento automático en la meta quincenal/mensual.
   - Una vez superadas las pruebas E2E, abrir los Pull Requests de `feat/permisos` hacia `develop` en `tmr-backend` y `tmr-frontend`.
