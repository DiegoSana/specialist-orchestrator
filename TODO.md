# 📋 Specialist — Todo & Roadmap (global)

> Última actualización: 2026-09-23 (agregados 3 pendientes nuevos: bloqueo de multi-perfil MVP,
> navegación sin perfil en social login, mensajes de fotos/buenas prácticas en nueva solicitud)

Este archivo vive en `/var/www/specialist/` (el directorio padre, no es un repo git) y es el
**único** TODO/roadmap del proyecto desde 2026-09-16. Antes había uno por repo:

- El de `specialist-be` se movió acá **tal cual**, historial completo de sesiones incluido (sección
  "Backend" abajo, con su propio título `# 🔧 Tareas Pendientes - Specialist Backend`).
- El de `specialist-fe` se **fusionó**: se trajo su contenido pendiente (sección "Frontend" más
  abajo) y el archivo se borró de ese repo para no tener dos fuentes desincronizadas. Su historial
  de bugs ya resueltos no se copió en detalle — ver los PRs mergeados en GitHub para eso.
- `specialist-admin` y `specialist-shared` no tenían TODO.md propio; se agregó una sección breve
  para cada uno (admin: tomada del checklist "Next Steps" de su `README.md`; shared: el backlog que
  ya estaba documentado en su `CLAUDE.md`/en el `CLAUDE.md` de los repos que lo consumen).

El skill `session-recap` de `specialist-be` (`specialist-be/.claude/skills/session-recap/`) sigue
siendo el mecanismo para cerrar una sesión de trabajo del backend, pero ahora actualiza **este**
archivo (`../TODO.md` desde `specialist-be`) en vez de uno local. El skill `orchestrate-feature`
de este directorio (`.claude/skills/orchestrate-feature/`) lee este archivo como parte de su paso
de investigación al planificar un requerimiento cross-repo.

## 🆕 Pendientes del rediseño de estados (2026-09-22)

Encontrados probando `specialist-fe` PR #20 (rediseño de "Mis Solicitudes"/"Mis Trabajos" al
modelo de 15 estados, ya mergeado) ya en `main` de ambos repos. Pedido explícito del usuario: cada
uno en una **sesión de Claude Code separada**, no en la misma sesión que los descubrió. Abrir una
terminal nueva por ítem (`cd /var/www/specialist && claude`, o el repo específico) y pegar el
bloque de texto de cada uno — ya trae el contexto necesario para no tener que releer esta sesión.

1. ~~**[BE+FE] Reorganizar documentación del rediseño de estados.**~~ **Resuelto 2026-09-22**
   (`specialist-be` [#68](https://github.com/DiegoSana/specialist-be/pull/68), rama
   `docs/estados-pedido-cleanup`, mergeado):
   - Doc de diseño movido a `specialist-be/docs/architecture/EspecialistBRC — Estados del
     pedido.md` (antes suelto en `docs/`), linkeado desde `docs/README.md`.
   - Escrita `specialist-be/docs/decisions/ADR-006-REQUEST-STATE-MACHINE.md` (decisión, PRs
     #59-63/#65-67, tradeoffs y limitaciones conocidas), linkeada desde `docs/README.md` y
     `docs/architecture/ARCHITECTURE.md`.
   - Actualizadas todas las referencias a la ruta vieja del doc (7 archivos de código en
     `src/requests/`, más `API.md`/`DOMAIN_MODEL.md`/`ARCHITECTURE.md`/`whatsapp/README.md`).
   - Causa raíz del TODO duplicado: el skill `session-recap` de `specialist-be` seguía
     escribiendo en un `TODO.md` local (`.claude/skills/session-recap/SKILL.md`,
     `.claude/rules/08-docs-and-backlog.md`) en vez de `../TODO.md` — la migración del
     2026-09-16 actualizó la descripción en este archivo pero no el skill ni la regla. Corregido
     en ambos; `specialist-be/TODO.md` (que había vuelto a acumular contenido propio durante
     PR1-PR5, en parte más viejo que este archivo) se eliminó tras fusionar acá lo que tenía de
     único (ver recap de "Estados del pedido" más abajo en la sección Backend).
   - `specialist-fe` no tenía docs sueltos de este rediseño (confirmado, no había nada que mover).

2. ~~**[FE, bug] "Calificar" aparece en Cerrados aunque ya calificaste.**~~ **Resuelto 2026-09-22**
   (`specialist-fe` [#21](https://github.com/DiegoSana/specialist-fe/pull/21), rama
   `fix/rate-button-closed-already-rated`, mergeado). Causa confirmada: `getPrimaryAction()`
   en `lib/request-status.ts` devolvía `'RATE'` fijo para `CLOSED` sin mirar si ya existe una review
   del viewer. Agregado `RequestStatusContext.alreadyReviewed`; `CLOSED` devuelve `'NONE'` cuando es
   true (cliente vía `useReviewByRequestId` con `enabled` condicional, especialista vía
   `request.clientRating != null` ya presente en el objeto, sin fetch extra). Tests verdes
   (134/135; la 1 que falla es un caso pre-existente en `main`, no relacionado, confirmado con
   `git stash`).

3. ~~**[BE o FE, bug] Pedido `4b493f2e-12fa-4b90-bbee-68a376447922` — "esperando respuesta del
   especialista" pero el detalle no muestra el especialista ni interesados.**~~ **Resuelto
   2026-09-22** (`specialist-fe` [#22](https://github.com/DiegoSana/specialist-fe/pull/22), rama
   `fix/sent-request-specialist-not-shown`, mergeado). No era bug de backend (`RequestResponseDto`
   ya devolvía `professional`/`company` en cualquier estado) ni faltaba "interesados" (correcto,
   un pedido directo no tiene interesados). Causa real: `ContactCard`
   (`components/requests/contact-card.tsx`) devolvía `null` entero salvo que el estado estuviera en
   una lista hardcodeada de "liberado" que no incluía `SENT`, mezclando "sabemos quién es la
   contraparte" con "podemos mostrar su teléfono". Ahora la tarjeta se muestra con el nombre en
   cuanto hay contraparte; el teléfono sigue gateado a `CONTACT_RELEASED` en adelante. Mismo fix
   sirve simétricamente para la vista del especialista (componente compartido).

4. ~~**[FE, feature] Login: poder loguearse como cualquier usuario, no solo los 6-7 botones
   fijos.**~~ **Resuelto 2026-09-22** (`specialist-fe`
   [#23](https://github.com/DiegoSana/specialist-fe/pull/23), rama
   `feat/login-dev-user-dropdown`, mergeado). Decisión: lista hardcodeada en el FE (no
   endpoint nuevo). `DEV_SEED_USERS` mirror de los 15 usuarios de `specialist-be/prisma/seed.ts`
   (el bloque viejo solo cubría 8) en un `<select>` agrupado por rol (Clientes/Especialistas
   /Empresas/Admin) arriba de los botones existentes, que se dejaron como quick-picks. Sin gate
   nuevo — el bloque viejo tampoco lo tenía, no se tocó ese punto.

5. ~~**[FE, feature] Al elegir especialista, abrir el mismo popup de detalle que en la lista de
   especialistas.**~~ **Resuelto 2026-09-22** (`specialist-fe`
   [#24](https://github.com/DiegoSana/specialist-fe/pull/24), rama
   `feat/interested-specialist-detail-popup`, mergeado). Extraído el modal inline de
   `app/[locale]/professionals/page.tsx` a `components/providers/provider-detail-modal.tsx`
   (prop `showCreateRequestCta`, default true); `InterestedSpecialists` suma botón "Ver perfil"
   por fila que abre el mismo modal con `showCreateRequestCta={false}`. La fila de interés solo
   trae nombre/rating (no trades/descripción/ciudad), así que el perfil completo se busca en el
   catálogo público `/providers` matcheando por `serviceProviderId`. **Encontrado en el camino**:
   ese match no funcionaba porque `specialist-be` `ProfessionalService.sanitizeForPublic()`
   perdía `serviceProviderId` silenciosamente (Company sí lo devolvía bien) — bug real, no
   cosmético, corregido en `specialist-be`
   [#69](https://github.com/DiegoSana/specialist-be/pull/69) (rama
   `fix/professional-catalog-missing-service-provider-id`, mergeado). De paso, arreglado
   un bug preexistente descubierto al mover el modal: `t('reviews')` resolvía a un objeto
   (`professionals.reviews.loginRequired`) en vez de string, crasheaba al abrir el perfil de un
   especialista con rating — era `t('details.reviews')`.

## 🆕 Nuevos pendientes (2026-09-23)

Pedidos por el usuario directamente sobre este archivo, sin investigar todavía — quedan acá como
backlog crudo para retomar con `orchestrate-feature` (los tres son potencialmente cross-repo).

1. **[BE+FE] Bloquear multi-perfil de usuario para el MVP.** Hoy un mismo usuario puede terminar
   con más de un tipo de perfil (cliente/profesional/empresa); para el MVP eso debe restringirse:
   - El usuario que se registró como **cliente** no debe poder crear un perfil profesional ni un
     perfil empresa.
   - El usuario que se registró como **especialista (profesional)** sí debe poder crear además un
     perfil **empresa** (caso válido: especialista que además arma su empresa).
   - Revisar puntualmente el **flujo del especialista que quiere registrar su empresa** — hoy no
     está claro que funcione bien de punta a punta.
   - Relacionado con el ítem ya existente "Auditoría de usuarios y perfiles + flujos de estado"
     (ver "Por dónde retomar" más abajo) y con "Perfil activo (MVP)" en la sección Backend — capaz
     conviene resolverlos juntos en la misma investigación en vez de por separado.

2. **[FE, posiblemente BE] Social login: bloquear navegación sin perfil elegido.** Cuando un
   usuario se loguea con social login y no existe un `User` registrado, hoy se le crea uno **sin
   ningún rol**, y se le muestra una pantalla para elegir rol y continuar el registro. Problema: si
   en vez de completar esa pantalla el usuario navega a otra URL directamente, queda logueado
   igual, sin perfil, y puede acceder a pantallas que no debería. Hay que evitar esa navegación: si
   el usuario no tiene perfil, no debe poder acceder a ninguna pantalla que no sea pública — debe
   ser redirigido siempre a la pantalla de selección de rol/perfil hasta que la complete.

3. **[FE] Nueva solicitud: destacar el beneficio de agregar fotos + mejores prácticas.** En la
   pantalla de creación de una solicitud, informar al usuario que si agrega imágenes tiene muchas
   más chances de ser elegido por algún especialista — el mensaje tiene que ser **bien visible**
   (evaluar un modal pre-guardado/pre-save como una opción, no la única). Además, agregar en esa
   misma pantalla una breve descripción de cuáles son las mejores prácticas para crear una
   solicitud (qué información conviene incluir para conseguir mejores respuestas).

## ▶️ Por dónde retomar (2026-09-18)

Sesión cerrada con todo lo chico resuelto y mergeado. Lo que queda son ítems que piden decisión o
plan antes de tocar código; orden sugerido (los detalles están en cada ítem, buscar por el título):

1. ~~**Opt-out de WhatsApp visible en el perfil del usuario**~~ **Resuelto 2026-09-23** (sesión de
   orquestador, rama `feat/whatsapp-optout-self-service` en ambos repos): `specialist-be`
   [#73](https://github.com/DiegoSana/specialist-be/pull/73), mergeado — `UserProfileResponseDto`
   expone `whatsappOptedOut`/`whatsappOptedOutAt`; nuevo `POST /users/me/whatsapp-reactivate`
   (self-service, unidireccional, reutiliza `setWhatsAppOptedOut`/`UserWhatsAppReactivatedEvent` y
   el email ya existente). `specialist-fe` [#30](https://github.com/DiegoSana/specialist-fe/pull/30),
   mergeado — tarjeta en `profile/page.tsx` (mismo patrón visual que verificación de teléfono/email)
   con botón "Reactivar". Pendiente: click-through manual con un usuario opted-out real (no se hizo
   en esta sesión, solo build/type-check/tests).
2. ~~**Estado `ASIGNADO` en Request**~~ — descartado 2026-09-23, ítem previo al rediseño de estados
   (ya cubierto por el modelo de 15 estados, ver Backend).
3. **Auditoría de usuarios y perfiles + flujos de estado** (Backend → "Revisar el diseño de
   usuarios y perfiles"). Entregable: documento con hallazgos, no código; puede incluir los
   diagramas de estado de Request que faltan.
4. **Validaciones extra de Company** (Backend → Perfil activo, Fase C): primero definir qué se exige
   (CUIT, AFIP, documentación).
5. Más chicos sin dueño: opt-out a nivel plataforma en Twilio/Meta, ADR de la feature de IA de
   follow-up, "el usuario no quiere ser contactado → request a review", Playwright E2E, admin
   mobile responsive, sección de admin en `DEPLOYMENT.md`.

## Índice

- [🔧 Backend (specialist-be)](#-tareas-pendientes---specialist-backend)
- [🎨 Frontend (specialist-fe)](#-frontend-specialist-fe)
- [🛠️ Admin (specialist-admin)](#️-admin-specialist-admin)
- [📦 Shared (specialist-shared)](#-shared-specialist-shared)

---

# 🔧 Tareas Pendientes - Specialist Backend

> Última actualización: 2026-09-22 (recap del rediseño de estados del pedido, PR1-PR5 + fixes)

> **Nueva sección:** [Perfil activo (MVP): reglas y restricciones](#-perfil-activo-mvp-reglas-y-restricciones) — definición de activo (email + teléfono usuario + confirmación admin), restricciones por perfil activo, y orden de implementación.

---

## 🧹 Limpieza pendiente (chico)

- [ ] **Revisar scripts de `package.json` del backend, sacar/limpiar duplicados.** Ej.: `db:seed` y
  `prisma:seed` son el mismo comando (`npx tsx prisma/seed.ts`) duplicado con dos nombres distintos;
  ver si conviene dejar uno solo y actualizar referencias (README, `db:reset`, CI/scripts que lo
  llamen).

---

## ✅ Cierre – Lo realizado hasta ahora (resumen ejecutivo)

**Perfil activo y permisos (backend):**
- **ProfileActivationService:** único punto que define “perfil activo” (cliente y proveedor). RequestService y RequestInterestService usan solo este servicio; no se llama `user.isFullyVerified()` repartido.
- **Crear solicitud:** exige cliente activo. **Expresar interés:** exige proveedor activo. **Asignar proveedor:** exige cliente activo (`canAssignProviderBy` + `hasActiveClientProfile`). **Job board (ver lista):** no exige activo; solo expresar interés lo exige.
- **GET /providers (catálogo):** solo muestra perfiles activos (usuario verificado + perfil canOperate). Filtro `userVerified` en repos de Professional y Company; `onlyActiveInCatalog: true` en ProvidersController.
- **RequestsController:** soporte Company en findMyRequests y findAvailable; vista limitada en `findByIdForInterestedProvider` + `fromEntityLimited`; RateClientDto con validación; TODOs de excepciones eliminados.
- **Admin:** `PUT /admin/users/:id/verification` para marcar email/teléfono verificados (override manual).

**Contacto unificado (si ya aplicaste la migración):** contacto solo en User; perfiles sin phone/email/whatsapp; canOperate = solo status ACTIVE/VERIFIED.

**Tests:** 291 pasando (suite completa verificada antes de rama/commits).

**Pendiente para otra sesión:** tests opcionales (ProfileActivationService, “assign rechazado si no activo”); Frontend B.3 mensajes; Admin Fase D moderación reviews; mejora “notificar a clientes cuando proveedor cambia teléfono/email”.

---

## 📌 Donde quedamos hoy (recap para seguir mañana)

### ✅ Hecho (2026-09-15): Harness de Claude Code + docs al día

- **Harness**: `CLAUDE.md` raíz, `src/<contexto>/CLAUDE.md`, `.claude/rules/01..08`, `.claude/skills/*`, `.claude/settings.json`.
- **Docs refrescados contra el código**: `DOMAIN_MODEL.md`, `ROLES_ARCHITECTURE.md`, `COMPANY_PROFILES.md`, `REVIEW_MODERATION.md`, `ADR-004` (`reviewCount` → `totalReviews`), `docs/README.md` (índice completo), `whatsapp/README.md` (regla PENDING 3 días), ports/defaults en `DOCKER.md`, `ENVIRONMENT_VARIABLES.md`, `NOTIFICATIONS.md`; `whatsapp-followup-implementation-status.md` marcado histórico; `admin-portal-plan.md` usa `specialist-be`.
- **Fix**: `identity.module.ts` lee `JWT_EXPIRES_IN` (antes `JWT_EXPIRATION`, que `.env`/compose/docs no definían); `JWT_EXPIRATION` queda como fallback.

### ✅ Hecho (2026-09-21/22): Rediseño de estados del pedido (15 estados)

Spec: `docs/architecture/EspecialistBRC — Estados del pedido.md`. Decisión documentada en
`docs/decisions/ADR-006-REQUEST-STATE-MACHINE.md`. Mergeado a `main` en `specialist-be` (PRs
#59-63 = PR1-PR5, más #65-67 de fixes encontrados integrando el FE) y en `specialist-fe` (PR #20).
Detalle completo del contenido de cada PR y de los gaps conocidos (`IN_PROGRESS→ABANDONED` sin
mapear, sin timestamp de entrada a estado, templates de WhatsApp sin aprobar en Meta/Twilio) está
en el ADR — no se duplica acá. Pendientes encontrados probando el resultado: ver sección
"🆕 Pendientes del rediseño de estados" al principio de este archivo.

### ⬜ Hallazgos pendientes (2026-09-15)

- [x] `ReviewService.updateServiceProviderRating` no recalcula rating para Company (2026-09-16: ahora resuelve Professional o Company por `serviceProviderId` y llama `CompanyService.updateRating`; se agregó `updateRating` a la interfaz `CompanyRepository`, sacando el cast `as any`).
- [x] `CreateReviewDto`: `professionalId` requerido pero ignorado; `requestId` opcional en DTO pero requerido en el service (2026-09-16: `professionalId` ahora opcional/deprecated, `requestId` ahora requerido — verificado contra `specialist-fe` que ya envía ambos, sin romper el contrato).
- [x] Lint: repo formateado completo con `npm run lint` (2026-09-15); 0 errores eslint, prettier limpio. Mantenerlo así (correr `npm run lint` antes de cada commit).
- [x] `@specialist/shared`: tipos desactualizados (`User.role`, `UserStatus`, `AdminContract` con PATCH vs PUT) — corregidos a mano el 2026-09-18 (`specialist-shared` #2). Sigue sin generarse desde el backend; solo lo usa el login de `specialist-admin`.
- [x] `GET /professionals/:id/reviews` no tiene equivalente para companies (2026-09-16: agregado `GET /companies/:id/reviews` + `ReviewService.findByCompanyId`, docs actualizadas en API.md/API_STRUCTURE.md).


### ✅ Hecho (2026-02-06): Contacto unificado y solo status en perfiles

- Contacto en User: Company sin phone/email; Professional sin whatsapp. Migración `20260206000000_remove_profile_contact_and_active`.
- Solo status en perfiles: sin `active`; canOperate = status ACTIVE/VERIFIED. Docs y scripts en MIGRATION_GUIDE.

### ✅ Hecho anteriormente

1. **Servicio de orquestación “perfil activo”**
   - **ProfileActivationService** (`src/profiles/application/services/profile-activation.service.ts`): único punto que define `hasActiveClientProfile` y `hasActiveProviderProfile` (componiendo User + Professional/Company + `isFullyVerified` / `canOperate`).
   - Ningún otro servicio llama a `user.isFullyVerified()` para permisos; todos usan este servicio.

2. **RequestAuthContext**
   - Añadido `hasActiveClientProfile`; ya existía `hasActiveProviderProfile`. Ambos se rellenan desde la orquestación.
   - Ver `src/requests/domain/entities/request.entity.ts`.

3. **RequestService**
   - **create():** usa `profileActivationService.getActivationStatus(clientId).hasActiveClientProfile` en lugar de `user.isFullyVerified()`.
   - **buildAuthContext():** llama a `getActivationStatus(userId)` y devuelve `hasActiveClientProfile`, `hasActiveProviderProfile` y `serviceProviderId`.

4. **RequestInterestService**
   - **buildAuthContext():** usa `profileActivationService.getActivationStatus(userId)` para `hasActiveProviderProfile`; se quitó la composición inline y la dependencia de UserService.

5. **Documentación**
   - **PROFILE_ACTIVATION_ORCHESTRATION.md** (`docs/architecture/`): diseño del servicio, uso en AuthContexts, auditoría de endpoints, orden de implementación.
   - **PERMISSIONS_BY_ROLE.md** y **AUTHORIZATION_PATTERN.md**: referencias al servicio de orquestación.

6. **Tests**
   - Request y RequestInterest specs actualizados (mock de ProfileActivationService). **291 tests pasando.**

### ✅ Hecho además (controller y restricciones)

- **RequestsController:** Company en findMyRequests/findAvailable; vista limitada → `findByIdForInterestedProvider` + `fromEntityLimited`; TODOs de excepciones eliminados; **RateClientDto** (rating 1–5, comment opcional).
- **Restricciones:** `canAssignProviderBy` exige `hasActiveClientProfile`; expresar interés exige `hasActiveProviderProfile`. Ver lista de requests disponibles (job board) no exige perfil activo; sí lo exige el listado de proveedores (GET /providers) para aparecer en catálogo.

### ⬜ Siguiente (cuando retomes)
- **Fase A:** GET /providers ya filtra por usuario verificado + perfil activo. A.4 hecho: contacto solo en User (sin phone/email en Company, sin whatsapp en Professional).
- **Tests:** opcional spec para `ProfileActivationService`; opcional test "assign rechazado si cliente no activo".
- **Frontend / Admin:** B.3 mensajes al rechazar por perfil no activo; Fase D pantalla moderación de reviews.

### Archivos clave para seguir

| Qué | Dónde |
|-----|--------|
| Orquestación | `src/profiles/application/services/profile-activation.service.ts` |
| Diseño y auditoría | `docs/architecture/PROFILE_ACTIVATION_ORCHESTRATION.md` |
| Contexto Request | `src/requests/domain/entities/request.entity.ts` (RequestAuthContext) |
| Uso en create/buildAuthContext | `request.service.ts`, `request-interest.service.ts` |

---

## 📋 Resumen de Estado

| Módulo | Permisos | Tests | Documentado |
|--------|----------|-------|-------------|
| Requests | ✅ | ✅ | ⬜ |
| Request Interest | ✅ | ✅ | ⬜ |
| Reviews | ✅ | ✅ | ⬜ |
| Notifications | ✅ | ✅ | ✅ |
| Profiles | ✅ | ✅ | ⬜ |
| Identity | ✅ | ✅ | ⬜ |
| **Companies** | ✅ | ✅ | ✅ |
| **RequestInterest** | ✅ | ✅ | ✅ |

---

## 🔐 Refactoring de Permisos

### ✅ Completado

- [x] **Requests Module**
  - [x] Crear `RequestAuthContext` interface en dominio
  - [x] Agregar métodos de autorización a `RequestEntity`:
    - `canBeViewedBy(ctx)`
    - `canManagePhotosBy(ctx)`
    - `canChangeStatusBy(ctx, newStatus)`
    - `canRateClientBy(ctx)`
    - `canExpressInterestBy(ctx)`
    - `canAssignProfessionalBy(ctx)`
  - [x] Refactorizar `RequestService` para usar métodos de dominio
  - [x] Refactorizar `RequestInterestService` para usar métodos de dominio
  - [x] Agregar `buildAuthContext()` helper en servicios
  - [x] Simplificar `RequestsController` (solo construye contexto y delega)
  - [x] Soporte para Admin en todos los permisos
  - [x] Actualizar tests

### ✅ Completado

- [x] **Requests Module** (completado anteriormente)

- [x] **Reviews Module**
  - [x] Crear `ReviewAuthContext` interface en dominio
  - [x] Agregar métodos de autorización a `ReviewEntity`:
    - `canBeViewedBy(ctx)` - APPROVED: público, PENDING/REJECTED: solo reviewer + admin
    - `canBeModifiedBy(ctx)` - solo reviewer y solo si PENDING
    - `canBeModeratedBy(ctx)` - solo admins y solo si PENDING
  - [x] Agregar `buildAuthContext()` helper a entidad
  - [x] Refactorizar `ReviewService`:
    - `findByIdForUser()` - con validación de permisos
    - `findByRequestIdForUser()` - con validación de permisos
    - `update()` / `delete()` - valida canBeModifiedBy
    - `approve()` / `reject()` - valida canBeModeratedBy
  - [x] Actualizar `ReviewsController`
  - [x] Actualizar tests (37 tests pasando)

### ✅ Completado

- [x] **Notifications Module**
  - [x] Crear `NotificationAuthContext` interface en dominio
  - [x] Agregar métodos de autorización a `NotificationEntity`:
    - `canBeViewedBy(ctx)` - owner o admin
    - `canBeMarkedReadBy(ctx)` - solo owner
    - `canBeResentBy(ctx)` - solo admin con delivery fallido
  - [x] Refactorizar `NotificationService.markRead()` para usar métodos de dominio
  - [x] Agregar métodos admin: `findByIdForUser()`, `listAll()`, `getDeliveryStats()`, `resendNotification()`
  - [x] Crear `AdminNotificationsController`:
    - `GET /admin/notifications` - listar todas con filtros
    - `GET /admin/notifications/stats` - estadísticas de delivery
    - `GET /admin/notifications/:id` - ver detalle
    - `POST /admin/notifications/:id/resend` - reenviar fallidas
  - [x] Actualizar tests

- [x] **Profiles Module**
  - [x] Crear `ProfessionalAuthContext` interface en dominio
  - [x] Agregar métodos de autorización a `ProfessionalEntity`:
    - `isOwnedBy(userId)` - helper para verificar propiedad
    - `canViewFullProfileBy(ctx)` - owner o admin
    - `canBeEditedBy(ctx)` - owner o admin
    - `canManageGalleryBy(ctx)` - owner o admin
    - `canChangeStatusBy(ctx)` - solo admin
  - [x] Refactorizar `ProfessionalService`:
    - `updateProfile()` - usa `canBeEditedBy()`
    - `addGalleryItem()` / `removeGalleryItem()` - usa `canManageGalleryBy()`
    - `updateStatus()` - usa `canChangeStatusBy()` y requiere user
  - [x] Actualizar `AdminService.updateProfessionalStatus()` para pasar user
  - [x] Actualizar tests (222 tests pasando)
  - [x] `ClientService` no requiere refactor (solo activa perfil propio)

- [x] **Identity Module**
  - [x] Crear `UserAuthContext` interface en dominio
  - [x] Agregar métodos de autorización a `UserEntity`:
    - `isSelf(ctx)` - helper para verificar si es el mismo usuario
    - `canBeViewedBy(ctx)` - self o admin
    - `canBeEditedBy(ctx)` - self o admin
    - `canChangeStatusBy(ctx)` - solo admin
    - `canBeDeletedBy(ctx)` - self o admin
  - [x] Agregar `buildAuthContext()` static helper
  - [x] Agregar métodos permission-aware a `UserService`:
    - `findByIdForUser()` - con validación de permisos
    - `updateForUser()` - con validación de permisos
    - `updateStatusForUser()` - solo admin
  - [x] Actualizar `AdminService`:
    - `getUserById()` usa `findByIdForUser()`
    - `updateUserStatus()` usa `updateStatusForUser()`
  - [x] Actualizar `AdminController` para pasar `@CurrentUser()`
  - [x] Actualizar tests (222 tests pasando)

### ⬜ Pendiente

---

## 🐛 Bug Fixes

### ✅ Completados

- [x] **FE: Solicitudes no aparecían en "mis solicitudes" del especialista**
  - Causa: `useProfessionalRequests` no pasaba `role=professional`
  - Fix: Agregar `?role=professional` al endpoint

- [x] **FE: Botón "Aceptar Presupuesto" visible (no es MVP)**
  - Fix: Removido de `client/requests/[id]/page.tsx`

- [x] **BE: Cualquier usuario podía ver cualquier solicitud por URL**
  - Fix: Agregar `canBeViewedBy()` y validar en `findByIdForUser()`

### ⬜ Pendiente

- [ ] **Verificar acceso a solicitudes desde perfil de otros especialistas**
  - Revisar cómo se muestran las solicitudes completadas en perfiles públicos

- [ ] **Revisar validación de permisos en fotos de solicitudes**
  - ¿Las fotos de trabajo completado son públicas?
  - ¿Quién puede ver las fotos durante el trabajo en progreso?

- [x] **BUG: usuario con perfil Company verificado y activo no ve su empresa ni puede operar**
  (2026-09-18, **resuelto**: `specialist-be` [#55](https://github.com/DiegoSana/specialist-be/pull/55) +
  `specialist-fe` [#14](https://github.com/DiegoSana/specialist-fe/pull/14), ambos mergeados).
  Causa: las respuestas de login/register/OAuth no incluían `hasCompanyProfile`, el FE cacheaba el
  usuario sin ese flag y nunca pedía la empresa; además el FE mostraba `ACTIVE` como "Rechazada"
  (enum `CompanyStatus` sin ACTIVE/INACTIVE/SUSPENDED). También se agregó la notificación al dueño
  cuando un admin verifica/rechaza/suspende la empresa (`CompanyStatusChangedEvent`). Sigue
  pendiente el "Company Dashboard" del FE (sección Frontend). Descripción original: en local, `empresapendiente@test.com` tiene un perfil `Company` asociado; tanto el
  `User` como el `Company` están verificados/activos, pero el usuario no ve su empresa en su perfil
  y no puede acceder a las funcionalidades de perfil empresa. Investigar (sospecha: algo en
  `ProfileActivationService`, en cómo `/users/me` arma `hasCompanyProfile`, o en el estado real en
  BD vs. lo que asumen los checks — revisar `canOperate()`/`isFullyVerified()` para este caso
  puntual). Además: verificar si hoy se notifica al usuario cuando su empresa queda verificada por
  un admin; si no se notifica, crear esa notificación (ver contexto `notifications` y el patrón de
  `add-domain-event`).

---

## 🗑️ Código a Eliminar (No MVP)

### ✅ Eliminado

- [x] `POST /requests/:id/accept` endpoint
- [x] `acceptQuote()` método en `RequestService`
- [x] `updateStatusByClient()` (unificado en `updateStatus()`)
- [x] `useAcceptQuote` hook en frontend (import removido)

### ⬜ Pendiente Evaluar

- [ ] Campos `quoteAmount` y `quoteNotes` en Request
  - ¿Mantener en schema para futuro MVP+?
  - ¿Eliminar completamente?

---

## 📝 Pull Requests

### ✅ Mergeados

> Nota: esta tabla no se mantuvo al día entre 2026-02 y 2026-09 (hay muchos más PRs mergeados
> mencionados sueltos en el resto de este archivo, p.ej. #14, #19, #55, #56 de BE y #2 de shared).
> Se retoma acá con el lote más reciente; no se reconstruyó el historial completo.

| PR | Repo | Descripción |
|----|------|-------------|
| #10 | BE | feat: Request title + notificaciones mejoradas |
| #11 | BE | refactor: Permission validation hybrid pattern |
| #3 | FE | fix: Campanita mobile responsive |
| #4 | FE | fix: Professional profile edit + permissions |
| #59-63 | BE | feat: rediseño de estados del pedido (15 estados), PR1-PR5 — ver ADR-006 |
| #65 | BE | fix: `GET /requests/interested` route order + `statusReason`/`interestsCount` en responses |
| #66 | BE | fix: drift de migración schema, gating de contacto por `canViewCounterpartContactBy`, cliente puede reportar `IN_PROGRESS→INTERRUPTED`, `canManagePhotosBy` en `CLOSED` |
| #67 | BE | fix: leak de contacto de interesados no elegidos en `GET /requests/:id/interests` |
| #20 | FE | feat: rediseño de estados del pedido (15 estados) en dashboards y detalle |
| #68 | BE | docs: reorganize state-machine spec, add ADR-006, fix TODO desync root cause |
| #21 | FE | fix: hide Calificar in Cerrados once the viewer already reviewed |
| #22 | FE | fix: show assigned specialist/client name before contact release |
| #23 | FE | feat: dev-login dropdown covering all seeded users |
| #69 | BE | fix: restore serviceProviderId on public professional search results |
| #24 | FE | feat: open the specialist profile popup from interested-specialists |
| #73 | BE | feat: self-service WhatsApp opt-out reactivation from own profile |
| #30 | FE | feat: show WhatsApp opt-out status + reactivate button on profile page |

### 🟡 Pendiente Merge

_Ninguno por ahora_

---

## 🔄 Refactoring de DTOs

### Problema Actual

Los controladores retornan directamente entidades de dominio o respuestas de servicios, generando:
- **Acoplamiento**: Cambios en el dominio afectan la API pública
- **Seguridad**: Posible exposición de campos internos/sensibles
- **Flexibilidad**: No se puede formatear la respuesta sin modificar el dominio

### Patrón Sugerido

```
Controller → Request DTO → Service → Domain Entity → Response DTO → Client
```

### ⬜ Controladores a Revisar

- [x] **RequestsController** ✅
  - [x] Crear `RequestResponseDto` en `presentation/dto/`
  - [x] Crear `InterestedProfessionalResponseDto` en `presentation/dto/`
  - [x] `findById` - Retorna `RequestResponseDto`
  - [x] `findMyRequests` - Retorna `RequestResponseDto[]`
  - [x] `findAvailable` - Retorna `RequestResponseDto[]`
  - [x] `create` - Retorna `RequestResponseDto`
  - [x] `update` - Retorna `RequestResponseDto`
  - [x] `addPhoto` / `removePhoto` - Retorna `RequestResponseDto`
  - [x] `expressInterest` - Retorna `InterestedProfessionalResponseDto`
  - [x] `getInterestedProfessionals` - Retorna `InterestedProfessionalResponseDto[]`
  - [x] `assignProfessional` - Retorna `RequestResponseDto`
  - [x] `rateClient` - Retorna `RequestResponseDto`
  - [x] Swagger decorators actualizados con tipos de respuesta

- [x] **ProfessionalsController** ✅
  - [x] Crear `ProfessionalResponseDto` en `presentation/dto/`
  - [x] Crear `ProfessionalSearchResultDto` (sin campos sensibles como whatsapp/address)
  - [x] `search` - Retorna `ProfessionalSearchResultDto[]` (campos públicos)
  - [x] `findById` - Retorna `ProfessionalResponseDto` (datos completos)
  - [x] `getMyProfile` - Retorna `ProfessionalResponseDto`
  - [x] `createMyProfile` - Retorna `ProfessionalResponseDto`
  - [x] `updateMyProfile` - Retorna `ProfessionalResponseDto`
  - [x] `addGalleryItem` / `removeGalleryItem` - Retorna `ProfessionalResponseDto`
  - [x] Swagger decorators actualizados con tipos de respuesta

- [x] **ReviewsController** ✅
  - [x] Crear `ReviewResponseDto` en `presentation/dto/`
  - [x] Crear `PublicReviewDto` (para endpoints públicos, sin info de moderación)
  - [x] `create` - Retorna `ReviewResponseDto`
  - [x] `findById` - Retorna `ReviewResponseDto`
  - [x] `findByRequestId` - Retorna `ReviewResponseDto`
  - [x] `update` - Retorna `ReviewResponseDto`
  - [x] `delete` - Retorna void
  - [x] `findPending` (admin) - Retorna `ReviewResponseDto[]`
  - [x] `approve` / `reject` (admin) - Retorna `ReviewResponseDto`
  - [x] `ProfessionalReviewsController.findByProfessionalId` - Retorna `PublicReviewDto[]`
  - [x] Swagger decorators actualizados con tipos de respuesta

- [x] **NotificationsController** ✅ (ya tenía DTOs implementados)

- [x] **ClientsController** ✅
  - [x] `createClientProfile` - Retorna `UserProfileResponseDto`
  - [x] Swagger decorators actualizados

- [x] **Identity/AuthController** ✅ (ya tenía DTOs implementados)
  - [x] `register` / `login` - Ya usan `AuthResponseDto`
  - [x] OAuth callbacks - Redireccionan con token

- [x] **UsersController** ✅ (refactorizado)
  - [x] `getMyProfile` - Usa `UserProfileResponseDto.fromEntity()`
  - [x] `updateMyProfile` - Usa `UserProfileResponseDto.fromEntity()`
  - [x] `activateClientProfile` - Usa `UserProfileResponseDto.fromEntity()`
  - [x] Eliminado método privado `toResponseDto()` duplicado

### Consideraciones

- Los DTOs de respuesta pueden usar `class-transformer` para `@Expose()` y `@Exclude()`
- Considerar usar mappers automáticos o manuales
- Los DTOs deben vivir en `presentation/dto/`
- Un DTO puede ser reutilizado en múltiples endpoints si tiene sentido

---

## 🧪 Tests a Mejorar

- [ ] Agregar tests de integración para permisos
- [ ] Agregar tests E2E para flujos críticos:
  - [ ] Flujo completo de solicitud directa
  - [ ] Flujo completo de solicitud pública
  - [ ] Flujo de moderación de reviews
- [ ] Verificar cobertura de código

---

## 🟢 Perfil activo (MVP): reglas y restricciones

> **Objetivo:** Redefinir “activo” para Cliente, Profesional y Empresa: email y teléfono del **usuario** verificados + perfil confirmado (manual por admin). Solo perfiles activos pueden: aparecer en listado de proveedores, crear solicitudes (cliente), expresar interés (proveedor).

### Definición de “perfil activo”

Para **todos** los perfiles (Cliente, Profesional, Empresa):

- **Usuario:** `emailVerified === true` y `phoneVerified === true` (datos del **User**, no del perfil).
- **Perfil:** confirmado manualmente por admin (teléfono, email y perfil pueden ser confirmados/override por admin).
- **MVP:** Solo se usan teléfono y email del **usuario**. En perfiles (Professional/Company) los campos de contacto se consideran no requeridos y se pueden ocultar en el FE.
- **Empresa:** además tendrá validaciones extra (por definir; ej. CUIT, documentación).

**Requisitos para acciones:**

| Acción | Requisito |
|--------|-----------|
| Aparecer en listado de proveedores (`GET /providers`, búsquedas) | Perfil activo (usuario verificado + perfil confirmado por admin) |
| Cliente: crear solicitudes (`POST /requests`) | Perfil de cliente activo (email y teléfono del usuario verificados). **Aplica tanto a solicitud pública como a solicitud directa.** |
| Proveedor: expresar interés (`POST /requests/:id/interest`) | Perfil proveedor activo |

### Orden de implementación sugerido

#### Fase A: Backend – definición de “activo” y confirmación por admin

- [x] **A.1** Definir en dominio/servicios “usuario con perfil activo”:
  - [x] `UserEntity.isFullyVerified()` (emailVerified && phoneVerified). Confirmación de perfil por admin queda para más adelante.
- [x] **A.2** Admin puede confirmar manualmente:
  - [x] Endpoint: marcar teléfono del usuario como verificado (override) — `PUT /admin/users/:id/verification` con `{ phoneVerified?: boolean }`.
  - [x] Endpoint: marcar email del usuario como verificado (override) — mismo endpoint con `{ emailVerified?: boolean }`.
  - [ ] Endpoint: marcar perfil (Professional/Company/Client) como “confirmado” por admin (puede requerir nuevo campo o flag en BD).
- [x] **A.3** Restricciones por verificación (guards o validación en servicios):
  - [x] Crear solicitud (`POST /requests`): exigir perfil de cliente activo (ProfileActivationService.hasActiveClientProfile).
  - [x] Expresar interés (`POST /requests/:id/interest`): exigir perfil proveedor activo (hasActiveProviderProfile).
  - [x] Asignar proveedor (`POST /requests/:id/assign-provider`): exigir cliente activo (canAssignProviderBy usa hasActiveClientProfile).
  - [ ] Job board (`GET /requests/available`): no exige perfil activo; solo ver la lista. Expresar interés sí exige perfil activo (canExpressInterestBy).
  - [x] Listado de proveedores (`GET /providers`): solo incluir perfiles **activos** (usuario verificado + perfil canOperate). Implementado con `userVerified` en repositorios y `onlyActiveInCatalog: true` en ProvidersController. Búsquedas directas `/professionals` y `/companies` siguen mostrando por active+status sin exigir usuario verificado (comportamiento previo).
- [x] **A.4** Contacto solo en User: eliminados `phone`/`email` de Company y `whatsapp` de Professional. DTOs de creación/actualización ya no incluyen esos campos; contacto se obtiene del User (ver migración `20260206000000_remove_profile_contact_and_active`).

#### Fase B: Frontend

- [x] **B.1** Ocultar en FE los campos de teléfono/email de **perfil** (Professional/Company) o mostrarlos como no requeridos; usar solo teléfono/email del usuario para verificación y contacto en MVP.
- [x] **B.2** Pantalla de especialistas: usar **solo** `GET /providers` (unificado). La página de catálogo y nueva solicitud ya usan `useSearchProviders`.
- [x] **B.3** Mensajes claros cuando una acción se rechaza por perfil no activo (ej. "Verificá tu email y teléfono para crear una solicitud"). Implementado: crear solicitud, expresar interés (detalle y job board) con enlace Ir a mi perfil. (ej. “Verificá tu email y teléfono para crear una solicitud”).

#### Fase C: Empresa – validaciones extra (por definir)

- [ ] **C.1** Definir qué validaciones extra requiere el perfil de empresa (ej. CUIT, documentación, AFIP). Documentar en TODO o en COMPANY_PROFILES.
- [ ] **C.2** Implementar cuando estén definidas.

#### Fase D: Admin – moderación de reviews

- [x] **D.1** **Backend:** Endpoints de moderación confirmados y estables: `GET /reviews/admin/pending`, `POST /reviews/:id/approve`, `POST /reviews/:id/reject` (con `JwtAuthGuard` + `AdminGuard`), documentados en `docs/API.md`.
- [x] **D.2** **Admin portal (FE, 2026-09-16):** Pantalla `/admin/reviews` en `specialist-admin` (rama `feat/review-moderation-screen`, commit `2ebc99e`): lista de reviews pendientes (sin paginación, el endpoint devuelve array plano) con acciones inline Approve/Reject. Nuevo `lib/api/admin.ts` (`Review` interface + `getPendingReviews`/`approveReview`/`rejectReview` — únicos calls que pegan a `/reviews/...` en vez de `/admin/...`), `hooks/use-reviews.ts`, entrada "Reviews" en el sidebar. Corregida la nota desactualizada en `.claude/rules/01-data-fetching.md` que decía que todo vivía bajo `/admin/*`. Mergeado: `specialist-admin` PR #5.

### Resumen rápido

- **Activo** = usuario con email + teléfono verificados (+ perfil confirmado por admin cuando se implemente).
- **MVP:** Contacto = solo usuario; perfiles sin exigir teléfono/email propios; admin puede confirmar manualmente.
- **Listado proveedores** = solo perfiles activos.
- **Crear solicitud** = cliente activo (email + teléfono verificados); **expresar interés** = proveedor activo. Crear solicitud aplica igual a solicitud pública y solicitud directa.
- **Pantalla especialistas** = usar `GET /providers`.
- **Moderación reviews** = ya en BE; falta pantalla en admin FE.

---

## 📚 Documentación Pendiente

- [x] Documentar patrón de autorización `AuthContext` + métodos de dominio
  - Creado `docs/architecture/AUTHORIZATION_PATTERN.md`
- [x] Actualizar README con nuevos endpoints
  - Companies, providers, verification, notifications; enlace a API Structure
- [x] Documentar flujos de permisos por rol (Cliente, Especialista, Admin)
  - Creado `docs/guides/PERMISSIONS_BY_ROLE.md`
- [ ] Agregar diagramas de estado de Request

---

## 🏢 Nueva Feature: Perfil de Empresa

### Descripción

Nuevo tipo de perfil para empresas (ej: constructoras, empresas de mantenimiento, etc.).
Misma funcionalidad que especialistas pero diferenciado para evolución futura.

---

### 🏗️ Arquitectura: ServiceProvider

Para desacoplar `Request` y `Review` del tipo de proveedor, introducimos una capa abstracta:

```
┌─────────────────────────────────────────────────────────────┐
│                      ServiceProvider                         │
│  - id: UUID                                                  │
│  - type: PROFESSIONAL | COMPANY                              │
│  - averageRating: Float (calculado)                          │
│  - reviewCount: Int                                          │
│  - createdAt, updatedAt                                      │
├─────────────────────────────────────────────────────────────┤
│         ▲                              ▲                     │
│         │ 1:1                          │ 1:1                 │
│    ┌────┴─────┐                  ┌─────┴─────┐               │
│    │Professional│                │  Company  │               │
│    │  - userId  │                │  - userId │               │
│    │  - bio     │                │  - name   │               │
│    │  - trades  │                │  - trades │               │
│    └────────────┘                └───────────┘               │
└─────────────────────────────────────────────────────────────┘
                          │
            ┌─────────────┴─────────────┐
            │ 1:N                       │ 1:N
            ▼                           ▼
┌───────────────────────┐    ┌───────────────────────┐
│       Request         │    │        Review         │
│  - providerId (FK)    │    │  - requestId (FK)     │
│  - clientId           │    │  - serviceProviderId  │
│  - status             │    │  - rating, comment    │
└───────────────────────┘    └───────────────────────┘
```

**Beneficios:**
- ✅ FK constraints reales en BD
- ✅ Un solo campo `providerId` en Request (no `professionalId` + `companyId`)
- ✅ Reviews siempre atadas a Request completado
- ✅ Rating se agrega a ServiceProvider
- ✅ Escala a N tipos de proveedores futuros

---

### ✅ Fase 1: Migración a ServiceProvider (COMPLETADO)

#### 1.1 Schema Changes

```prisma
// NUEVO
model ServiceProvider {
  id            String       @id @default(uuid())
  type          ProviderType
  averageRating Float        @default(0)
  reviewCount   Int          @default(0)
  createdAt     DateTime     @default(now())
  updatedAt     DateTime     @updatedAt

  professional  Professional?
  company       Company?
  requests      Request[]
  reviews       Review[]
}

enum ProviderType {
  PROFESSIONAL
  COMPANY
}

// MODIFICADO
model Professional {
  id                String   @id @default(uuid())
  userId            String   @unique
  serviceProviderId String   @unique  // ← NUEVO
  serviceProvider   ServiceProvider @relation(...)
  // ... resto igual
}

// MODIFICADO
model Request {
  // ANTES: professionalId String?
  // DESPUÉS:
  providerId        String?
  provider          ServiceProvider? @relation(...)
  // ... resto igual
}

// MODIFICADO  
model Review {
  // ANTES: professionalId String
  // DESPUÉS:
  requestId         String
  request           Request @relation(...)
  serviceProviderId String   // Denormalizado para queries
  serviceProvider   ServiceProvider @relation(...)
  // ... resto igual
}
```

#### 1.2 Migración de Datos

- [ ] Crear tabla `ServiceProvider`
- [ ] Para cada `Professional` existente:
  - Crear `ServiceProvider` con `type=PROFESSIONAL`
  - Actualizar `Professional.serviceProviderId`
- [ ] Migrar `Request.professionalId` → `Request.providerId`
- [ ] Migrar `Review.professionalId` → `Review.serviceProviderId`
- [ ] Eliminar columnas viejas

#### 1.3 Domain Layer

- [ ] Crear `ServiceProviderEntity`
  ```typescript
  class ServiceProviderEntity {
    constructor(
      public readonly id: string,
      public readonly type: ProviderType,
      public readonly averageRating: number,
      public readonly reviewCount: number,
    ) {}
    
    canReceiveRequest(): boolean
    canBeReviewed(): boolean
    updateRating(newReview: Review): void
  }
  ```

- [ ] Modificar `ProfessionalEntity` para componer `ServiceProviderEntity`
- [ ] Actualizar `RequestEntity`:
  - Cambiar `professionalId` → `providerId`
  - Actualizar métodos `canXxxBy` para usar `providerId`

- [ ] Actualizar `ReviewEntity`:
  - Cambiar relación a `serviceProviderId`
  - Review siempre requiere `requestId`

#### 1.4 Application Layer

- [ ] Crear `ServiceProviderService` (queries comunes)
- [ ] Actualizar `ProfessionalService`:
  - `create()` también crea `ServiceProvider`
  - Queries incluyen `serviceProvider` relation
- [ ] Actualizar `RequestService`:
  - Cambiar `professionalId` → `providerId` en todas las operaciones
- [ ] Actualizar `ReviewService`:
  - Al crear review, actualizar `ServiceProvider.averageRating`

#### 1.5 Presentation Layer

- [ ] Actualizar DTOs (transparente para clientes API)
- [ ] Mantener backward compatibility si es necesario

---

### ⬜ Fase 2: Modelo Company

#### 2.1 Schema

```prisma
model Company {
  id                String   @id @default(uuid())
  userId            String   @unique
  serviceProviderId String   @unique
  serviceProvider   ServiceProvider @relation(...)
  user              User     @relation(...)
  
  // Datos de empresa
  companyName       String
  legalName         String?
  taxId             String?  // CUIT/RUT
  description       String?
  foundedYear       Int?
  employeeCount     Int?
  
  // Contacto
  website           String?
  phone             String?
  email             String?
  
  // Ubicación
  address           String?
  city              String?
  state             String?
  country           String?
  
  // Verificación
  verified          Boolean  @default(false)
  verifiedAt        DateTime?
  
  // Relaciones
  trades            Trade[]  @relation("CompanyTrades")
  photos            CompanyPhoto[]
  
  createdAt         DateTime @default(now())
  updatedAt         DateTime @updatedAt
}
```

#### 2.2 Domain Layer ✅

- [x] Crear `CompanyEntity` - `src/profiles/domain/entities/company.entity.ts`
  ```typescript
  class CompanyEntity {
    constructor(
      public readonly id: string,
      public readonly userId: string,
      public readonly serviceProviderId: string,
      public readonly companyName: string,
      // ... campos implementados
    ) {}
    
    // Métodos de autorización implementados
    canBeViewedBy(ctx: CompanyAuthContext): boolean
    canBeEditedBy(ctx: CompanyAuthContext): boolean
    // ... más métodos
  }
  ```

- [x] Crear `CompanyAuthContext` interface

#### 2.3 Application Layer ✅

- [x] Crear `CompanyService`
  - `createProfile(userId, data)` - crea Company + ServiceProvider
  - `updateProfile(user, id, data)` - actualiza con permisos
  - `search(params)` - búsqueda pública
  - `findById(id)` - acceso público sanitizado
  - `findByUserId(userId)` - acceso dueño
  - `addGalleryItem(user, url)` / `removeGalleryItem(user, url)`
  - `verifyCompany(user, id)` - solo admin

- [x] Crear DTOs:
  - `CreateCompanyDto`
  - `UpdateCompanyDto`
  - `SearchCompaniesDto`
  - `CompanyResponseDto`
  - `CompanySearchResultDto` (sin datos sensibles)

#### 2.4 Presentation Layer ✅

- [x] Crear `CompaniesController` - endpoints implementados:
  ```
  GET    /companies           - buscar empresas (público)
  GET    /companies/:id       - ver perfil público
  POST   /companies/me        - crear mi perfil
  PATCH  /companies/me        - actualizar mi perfil
  POST   /companies/me/gallery - agregar foto galería
  DELETE /companies/me/gallery - eliminar foto galería
  ```

#### 2.5 Identity Integration ✅

- [x] Agregar a `User`:
  ```prisma
  model User {
    // existente
    company           Company?
  }
  ```

- [x] Actualizar `UserEntity`:
  - Agregar `hasCompanyProfile: boolean`
  - Métodos: `isCompany()`, `isServiceProvider()`, `hasAnyProviderProfile()`, `canCreateCompanyProfile()`

- [x] Actualizar `/users/me` response con `hasCompanyProfile`

#### 2.6 Notifications

- [x] Actualizar handlers para soportar Company como provider
- [x] Notificaciones cuando empresa recibe interés/asignación
- [x] Actualizar eventos con `serviceProviderId`, `providerUserId`, `providerType`
- [x] Documentar cambios en `docs/guides/NOTIFICATIONS.md`

---

### ✅ Fase 3: Testing

- [x] Actualizar tests existentes para nuevo schema (completado)
- [x] Tests unitarios para `ServiceProviderEntity` (20 tests)
- [x] Tests unitarios para `CompanyEntity` (31 tests)
- [ ] Tests de integración para migración
- [x] Tests E2E para flujo completo de empresa
  - [x] `test/test-setup.ts` - Infraestructura y helpers para E2E
  - [x] `test/companies.e2e-spec.ts` - CRUD, búsqueda, galería, verificación
  - [x] `test/requests.e2e-spec.ts` - Flujo completo de interest (Professional + Company)
- [ ] Agregar E2E tests al CI pipeline (GitHub Actions con PostgreSQL service)

### ✅ Fase 4: Documentación

- [x] Actualizar `docs/API.md` con endpoints de Companies
- [x] Crear `docs/decisions/ADR-004-SERVICE-PROVIDER-ABSTRACTION.md`
- [x] Actualizar `docs/README.md` con nueva estructura

---

### ✅ Reglas de Negocio - Dual Profile

#### Dual Profile: Professional + Company

> 📖 **Diseño completo:** [docs/architecture/COMPANY_PROFILES.md](./docs/architecture/COMPANY_PROFILES.md)

**Resumen de decisiones:**
- Solo UN perfil proveedor activo a la vez (Professional XOR Company)
- Al verificar Company → Professional se desactiva automáticamente
- Usuario puede alternar entre perfiles desde dashboard
- CUIT único (error si ya existe)
- Company usa mismos flujos que Professional (Job Board, Reviews, Solicitudes)

**Implementación Backend ✅:**
- [x] Lógica de activación/desactivación de perfiles (`ProfileActivationPolicy` + `ProfileToggleService`)
- [x] Validación de CUIT único (en `CompanyService.createProfile`)
- [x] Endpoints para toggle de perfil:
  - `POST /api/professionals/me/activate` - Activar perfil profesional
  - `POST /api/companies/me/activate` - Activar perfil empresa
  - `GET /api/users/me/provider-profiles` - Ver estado de ambos perfiles
- [x] Catálogo unificado con filtro (`GET /api/providers?providerType=ALL|PROFESSIONAL|COMPANY`)

**Pendiente Frontend:**
- [ ] Toggle de perfil activo en dashboard (FE)
- [x] Filtro "Tipo" en catálogo de especialistas (2026-09-18, `specialist-fe` #19)
- [x] Badge "Empresa" en tarjetas de proveedor

---

### ✅ Arquitectura de Empresas

> 📖 **Diseño completo:** [docs/architecture/COMPANY_PROFILES.md](./docs/architecture/COMPANY_PROFILES.md)

**MVP (actual):**
- [x] Company como ServiceProvider
- [x] Estados: PENDING_VERIFICATION → ACTIVE → VERIFIED (+ INACTIVE, REJECTED, SUSPENDED)
- [x] Validación de CUIT único
- [x] Company no opera hasta ACTIVE (verificado por admin)

**Post-MVP:**
- [ ] Multi-usuario por empresa (CompanyMember con roles)
- [ ] Verificación avanzada (AFIP, documentación)
- [ ] Transferencia de ownership

---

### Consideraciones Futuras (No MVP)

- [ ] Dashboard de empresa con métricas
- [ ] Verificación de empresa (documentos legales, AFIP)
- [ ] Planes de suscripción para empresas
- [ ] Portal de empleados de la empresa
- [ ] Asignación de solicitudes a empleados específicos
- [ ] Transferencia de ownership de empresa

### Prioridad

🟡 **Media** - Implementar después de estabilizar permisos y DTOs

### Orden de Implementación Sugerido

1. **Fase 1** (ServiceProvider) - ~2-3 días
2. **Fase 2** (Company model) - ~2-3 días  
3. **Fase 3** (Testing) - ~1-2 días
4. **Frontend** - ~3-4 días

**Total estimado: ~10-12 días**

---

## 🚀 Mejoras Futuras (Backlog)

### Performance
- [ ] Revisar N+1 queries en listados
- [ ] Implementar caché para perfiles públicos
- [ ] Optimizar queries de notificaciones

### Seguridad
- [ ] Rate limiting por endpoint
- [ ] Validación de inputs más estricta
- [ ] Audit log para acciones administrativas

### UX / Soporte
- [ ] **Botón "Reportar un problema"**
  - Agregar botón en la app (ubicación y flujo por definir).
  - Lógica, backend y canal de reporte por definir más adelante.

- [ ] **Crear un mecanismo de soporte al usuario** (2026-09-18): hoy no existe ningún canal para
  que un usuario (cliente, profesional o empresa) pida ayuda o reporte un problema desde la app.
  Definir alcance y forma (¿chat interno, ticket, email, WhatsApp?) — ver también la sección
  "Soporte y Chat" más abajo (dentro del Portal de Administración), que ya plantea un chat con
  administrador acotado a la pantalla de detalle de request; evaluar si conviene unificar ambos
  esfuerzos en un solo diseño en vez de dos mecanismos de soporte separados.

### Verificación de Usuario (Email & Teléfono) ✅
- [x] **Validación de Email**
  - Implementado flujo de verificación usando Twilio Verify
  - Agregado campo `emailVerified: boolean` a User
  - Endpoints: `POST /identity/verification/email/request` y `/confirm`
  
- [x] **Validación de Teléfono**
  - Implementado flujo de verificación usando Twilio Verify
  - Agregado campo `phoneVerified: boolean` a User
  - Endpoints: `POST /identity/verification/phone/request` y `/confirm`
  - Validación de formato E.164 para números telefónicos
  - Invalidación automática cuando cambia el teléfono/email

- [ ] **Tests para Verificación**
  - [x] Tests unitarios para `VerificationService` (application layer)
  - [ ] Tests unitarios para `TwilioVerifyService` (infrastructure layer)
  - [ ] Tests unitarios para `Phone` value object
  - [ ] Tests de integración para endpoints de verificación
  - [ ] Tests E2E para flujo completo de verificación de teléfono
  - [ ] Tests E2E para flujo completo de verificación de email
  - [x] Tests de validación: prevenir código si ya está verificado (cubierto en VerificationService spec)
  - [ ] Tests de invalidación: verificar que se invalida al cambiar teléfono/email

- [ ] **Deployment y Configuración**
  - [ ] Subir credenciales de Twilio a fly.io (secrets)
    - `TWILIO_ACCOUNT_SID`
    - `TWILIO_AUTH_TOKEN`
    - `TWILIO_VERIFY_SERVICE_SID`

- [ ] **Restricciones de Acciones por Verificación** → Ver sección [Perfil activo (MVP)](#-perfil-activo-mvp-reglas-y-restricciones)
  - Requisito unificado: **perfil activo** = usuario con email + teléfono verificados (+ perfil confirmado por admin).
  - Acciones que requieren perfil activo: crear solicitud (cliente), expresar interés (proveedor), aparecer en listado de proveedores.
  - [ ] Implementar guards/validación en servicios según fases A.2 y A.3 de la sección Perfil activo.

### Notificaciones y Comunicaciones
- [x] **No notificar al propio usuario por sus propias acciones** (2026-09-17, resuelto 2026-09-18 en specialist-be #56, mergeado): `RequestsNotificationsHandler.onStatusChanged`
  (`src/notifications/application/handlers/requests-notifications.handler.ts`) crea una notificación in-app también para
  quien hizo el cambio de estado (ramas `clientMadeChange`/`providerMadeChange` — solo cambia el texto a "Moviste..." y
  se pone `includeExternal: false`, pero la notificación in-app se crea igual). Sacar esa notificación in-app cuando el
  usuario es el autor de su propia acción, no solo desactivar el canal externo. Revisar si el mismo patrón aparece en
  otros handlers (`reviews-notifications.handler.ts` no lo tenía al revisar, pero conviene confirmar el resto).
- [ ] **Notificar a clientes cuando un proveedor cambia teléfono o email (mejora)**
  - Cuando un provider (Professional o Company) actualiza su número de teléfono o email en el usuario, notificar a los clientes de los **requests activos** en los que ese proveedor participa (asignado o con interés expresado).
  - Permite que el cliente tenga el dato de contacto actualizado para solicitudes en curso.
- [ ] **Integración de Twilio WhatsApp en Notificaciones**
  - [ ] Incorporar Twilio al módulo de notificaciones
  - [ ] Crear adapter para envío de mensajes por WhatsApp usando Twilio API
  - [ ] Agregar canal `WHATSAPP` a tipos de notificación
  - [ ] Configurar preferencias de usuario para recibir notificaciones por WhatsApp
  - [ ] Validar que usuario tenga teléfono verificado antes de enviar por WhatsApp
  - [ ] Implementar fallback a email si WhatsApp falla
  - [ ] Agregar tests para envío de notificaciones por WhatsApp

- [ ] **Follow-up Interactivo por WhatsApp**
  - [ ] **Tracking de clicks en botón de WhatsApp**
    - [ ] Crear endpoint para registrar click en botón de contacto por WhatsApp
    - [ ] Modelo de datos para almacenar eventos de click (RequestContactClick)
      - Campos: requestId, clickedByUserId, providerId (quien fue contactado), timestamp, source (client/professional)
    - [ ] Integrar tracking en frontend: llamar endpoint cuando se hace click en botón WhatsApp
    - [ ] Agregar tracking tanto para clicks desde cliente hacia provider como viceversa
    - [ ] Considerar usar eventos de dominio para desacoplar tracking del flujo principal
  - [ ] Investigar funcionalidad de follow-up automático para Requests
  - [ ] Definir triggers: tiempo sin actividad después del primer contacto
  - [ ] Usar datos de tracking para determinar cuándo hacer follow-up (ej: si hubo click pero no respuesta)
  - [ ] Diseñar flujo de preguntas interactivas por WhatsApp
  - [ ] Implementar webhook endpoint para recibir respuestas de Twilio
  - [ ] Procesar respuestas y actualizar estado del Request según respuesta
  - [ ] Crear sistema de templates de mensajes para follow-up
  - [ ] Agregar configuración de tiempos de follow-up (ej: 3 días, 7 días)
  - [ ] Implementar lógica para evitar múltiples follow-ups
  - [ ] Agregar tests para webhook de Twilio y procesamiento de respuestas
  - [ ] Agregar tests para tracking de clicks
  - [ ] Documentar flujo completo de follow-up interactivo

  > Nota (2026-09-17): varios de estos ítems ya están implementados en la práctica — hay reglas de
  > follow-up por status (`src/requests/application/follow-up/rules/`), webhook de Twilio, detección
  > de intención de respuesta (`DetectResponseIntentUseCase`) y templates
  > (`message-templates.json`, ver `docs/guides/whatsapp/README.md`). Falta principalmente el
  > tracking de clicks y lo que se agrega abajo. Re-auditar este checklist contra el código antes de
  > seguir marcando ítems.

- [ ] **Repasar el flujo de preguntas/respuestas del follow-up interactivo** (2026-09-17):
  - [ ] Qué pasa cuando la respuesta del usuario no matchea ninguna opción válida — hoy cae en el
    template `unknown_response` (`message-templates.json`), revisar si alcanza o si hace falta un
    límite de reintentos / detección de que el usuario está confundido.
  - [ ] Definir cómo delegar/escalar la conversación a soporte humano cuando el bot no puede resolver
    la respuesta (¿nuevo estado en `RequestInteraction`, notificación a admin, ambos? ver también
    el ítem "Chat con Administrador en Request" más abajo, puede ser el mismo mecanismo).
  - [x] Darle al usuario la opción de bloquear/optar por no recibir más mensajes de WhatsApp
    (opt-out): implementado como parte del ítem "IA para follow-up" de abajo.

- [x] **IA para follow-up: clasificador LLM + opt-out + panel de atención admin** (analizado e
  implementado 2026-09-17, sesión de orquestador; rama `feat/whatsapp-ai-followup` en
  `specialist-be` y `specialist-admin`, commits locales, **sin pushear/sin PR todavía**). Alcance
  elegido: **Opción C (híbrida)** — el envío programado de nudges sigue siendo 100% templates fijos
  (restricción de WhatsApp Business API), se reemplaza el clasificador de intención
  (`DetectResponseIntentUseCase`, keyword matching) por un LLM (Anthropic, `claude-haiku-4-5`)
  detrás de `IntentDetectionPort` (mismo patrón que `WhatsAppMessagingPort`/`EmailSender`: adapter
  real + adapter `local`/keyword — **default `local`**, a diferencia de `WHATSAPP_PROVIDER`, para
  no pegarle a una API paga en un entorno mal configurado — más el adapter real con tool use
  forzado, nunca texto libre). Detalle completo en `specialist-be/src/requests/CLAUDE.md`
  ("AI reply classification" / "Admin request attention") y `docs/guides/ENVIRONMENT_VARIABLES.md`.

  Los tres objetivos (viabilidad, sync de status, opt-out) más el agregado de esta sesión
  (escalate + panel admin) quedaron todos implementados:
  1. **Viabilidad**: `FollowUpSchedulerJob` flaguea `AT_RISK` cuando se agota la escalera de
     follow-ups de un status sin ninguna interacción `RESPONDED`; el LLM devuelve además
     `viability: 'ACTIVE'|'AT_RISK'|'ABANDONED'|null` sobre respuestas evasivas reales. **Solo flag
     + aviso a admin, nunca auto-cancela** (decisión de producto confirmada).
  2. **Sync de status**: el LLM reemplaza el keyword matching para `ResponseIntent`, mismo handler
     downstream (`mapIntentToStatus` sin cambios). Contexto: últimas interacciones del request +
     template disparador + status actual. Umbral de confianza (`INTENT_CLASSIFIER_CONFIDENCE_THRESHOLD`,
     default 0.6) — por debajo, cae a `UNKNOWN` en vez de arriesgar una transición de status mala.
  3. **Opt-out**: `whatsappOptedOut`/`whatsappOptedOutAt` en `User` (migración manual — ver nota de
     riesgo abajo). Fast-path de keywords explícitas ("BAJA"/"STOP"/"CANCELAR SUSCRIPCION") +
     detección no explícita por LLM. Gatea `ProfileActivationService.getActivationStatus` (bloquea
     tomar/solicitar requests nuevos) y corta el envío puntual de WhatsApp a ese user. Los requests
     ya en curso **no se tocan** (decisión confirmada).
  4. **Escalate + panel admin** (agregado en esta misma sesión, después del primer análisis): las
     tres razones (`AT_RISK`/`ABANDONED`/`ESCALATED`) comparten un mecanismo único
     `RequestAttentionFlag` (association-store) + `RequestAttentionService.flag()` (idempotente —
     no duplica ni renotifica mientras haya un flag abierto) + notificación a todos los admins
     (`UserService.findAdminUserIds()`). Nuevos endpoints `GET/POST /admin/requests/attention`
     (`:id/resolve`) y pantalla dedicada `/admin/attention` en `specialist-admin` (tabla + acción
     "Marcar resuelto", mismo patrón que la pantalla de moderación de reviews). El mecanismo de
     chat/mensajería admin↔usuario en sí **no** se construyó — el admin sigue la conversación
     manualmente desde `/admin/whatsapp`; eso sigue siendo el ítem separado "Chat con Administrador
     en Request" de abajo.

  **Pendiente antes de mergear/desplegar:**
  - [x] Las dos migraciones nuevas (`whatsappOptedOut`/`whatsappOptedOutAt`, `RequestAttentionFlag`)
    se probaron contra la DB real de dev local (2026-09-17): aplicadas vía `prisma db execute` +
    `migrate resolve --applied` (no vía `migrate dev`, ver ítem nuevo de causa raíz abajo),
    `prisma migrate status` limpio, columnas/tabla verificadas por `psql`, los 432 tests en verde,
    y la app arranca sin errores con `AdminRequestAttentionController` registrado.
  - [ ] Investigar si Twilio/Meta ya manejan opt-out a nivel de plataforma para WhatsApp Business
    API (podría ya bloquear entrega o exponer un estado) — no se investigó, el fast-path de
    keywords propio se implementó igual mientras tanto.
  - [ ] Evaluar si el feature amerita un ADR (patrón nuevo de puerto LLM, semántica de opt-out,
    mecanismo de attention flags) — sugerido pero no escrito.
  - [x] **Resuelto (2026-09-18)**: `RequestAttentionFlaggedHandler` crea la notificación in-app para
    cada admin (`findAdminUserIds` + `createForUser`) y `specialist-admin` #12 agregó la campana.
    Descripción original: **¿Hay notificaciones in-app para admins? Si las hay, no se ven en la UI** (2026-09-18):
    `RequestAttentionService.flag()` notifica "a todos los admins" (`UserService.findAdminUserIds()`)
    cuando se abre un attention flag — confirmar que efectivamente crea `Notification` in-app (no
    solo email/WhatsApp) para esos admin users, y si es así, verificar que `specialist-admin` tenga
    alguna forma de verlas (¿campanita/bandeja como en `specialist-fe`? ¿solo la pantalla dedicada
    `/admin/attention` cubre esto y las notificaciones in-app quedan huérfanas?). El backend ya
    expone `GET /admin/notifications` (`AdminNotificationsController` — listar todas, filtros,
    stats, reenviar fallidas), pero no está claro si el admin FE lo consume para mostrarle al propio
    admin logueado sus notificaciones, o si ese endpoint es solo para auditar notificaciones de
    otros usuarios.
  - [x] Probado contra la API real de Anthropic (2026-09-17, `claude-haiku-4-5-20251001`,
    `INTENT_CLASSIFIER_PROVIDER=anthropic` local). Confirmó el valor del feature: un mensaje
    evasivo real ("no sé todavía, estoy viendo otras opciones") hubiera sido mal clasificado como
    `CANCELLED` por el keyword matching viejo (matchea "no" como substring) — el LLM lo clasificó
    bien como `NEEDS_INFO` (confidence 0.75) + `viability: AT_RISK`. **Encontró y arregló 2 bugs
    reales en el camino**, ambos commiteados:
    - El fast-path de keywords explícitas de opt-out **no estaba implementado** — con el adapter
      local (default), el opt-out nunca se detectaba, ni con "BAJA"/"STOP" textual. Fix:
      `opt-out-keywords.ts` (commit `e67dffa`), independiente del provider configurado.
    - El prompt del adapter de Anthropic **confundía "escalar a un humano" con "opt-out"**: un
      mensaje real ("no doy más con estos mensajes automáticos, quiero hablar con una persona")
      devolvió `optOut: true` y **bloqueó de verdad** a un provider real en la DB de test, cuando
      el usuario solo quería un humano, no dejar de recibir WhatsApp. Fix: prompt más estricto +
      ejemplo de contraste (commit `cdb438c`), reverificado contra la API real — mismo mensaje
      ahora da `escalate: true, optOut: false`.
    - **Bug preexistente encontrado (no introducido por este trabajo, confirmado por logs del día
      anterior)**: `RequestInteractionRespondedHandler` construye el `RequestAuthContext`
      manualmente en vez de usar `ProfileActivationService`, y dos casos borde chocaban con
      `canChangeStatusBy` y tiraban `ForbiddenException` en silencio (atrapada y logueada, sin
      avisar a nadie): (1) un proveedor pidiendo cancelar un request `IN_PROGRESS` (correcto que
      el dominio lo rechace — solo cliente/admin cancelan — pero se perdía sin dejar rastro), y
      (2) el fallback de "asignar especialista por número" cuando la respuesta del cliente no
      parsea como número, que intentaba un `PENDING`→`ACCEPTED` directo sin asignar proveedor
      (siempre destinado a fallar, `ACCEPTED` requiere pasar por `assignProvider`). **El flujo
      central (proveedor confirma que empezó/terminó) se verificó funcionando correctamente contra
      la API real** — esto es acotado a esos dos casos borde. Fix: ambos casos ahora llaman
      `RequestAttentionService.flag(ESCALATED)` en vez de tragarse el error (commit `e6efdd5`),
      verificado en vivo para los dos casos.
    - Otros casos probados en vivo sin problemas: confirmación clara de reseña (`CONFIRMED`
      conf=0.95), reply "1" a la selección de especialista (viability `ACTIVE`, aunque el
      `statusIntent` que le asigna a un "1" suelto es algo arbitrario — inofensivo porque ese flujo
      no usa `statusIntent`, se resuelve aparte por `tryAssignProviderByNumber`), jerga
      rioplatense sin tildes, queja seria de seguridad/cobro (`escalate: true` correctamente sin
      falso positivo de `optOut`), mensaje totalmente off-topic (`CONFIRMED` con confidence 0.75 —
      un poco generoso para chat intrascendente, pero no causó ningún cambio de estado indebido
      gracias al fix de arriba).
    - **Riesgo residual sin resolver**: `confidence` solo mide certeza del `statusIntent`; no hay
      una señal de confianza separada para `optOut`/`escalate`/`viability`, así que esos tres
      dependen enteramente de que el prompt esté bien calibrado, sin red de seguridad numérica.
      Vale la pena vigilar en producción real antes de confiar ciegamente en `optOut` del LLM.
  - [x] Push + PR en ambos repos (2026-09-17): `specialist-be`
    [#50](https://github.com/DiegoSana/specialist-be/pull/50), `specialist-admin`
    [#9](https://github.com/DiegoSana/specialist-admin/pull/9). Ambos mergeados.

- [x] **Bug preexistente: `prisma migrate dev` roto por orden del historial de migraciones**
  (**resuelto 2026-09-18**: carpeta renombrada a `20251215200252_add_request_interactions`,
  `specialist-be` [#57](https://github.com/DiegoSana/specialist-be/pull/57), y el registro de
  `_prisma_migrations` ya fue actualizado con `UPDATE` en local y en Supabase. Verificado replay
  desde DB vacía (15 migraciones). Cualquier DB nueva con las migraciones ya aplicadas necesita el
  mismo `UPDATE`, ver `docs/guides/MIGRATION_GUIDE.md`. El detalle de abajo queda como historial.)
  (encontrado 2026-09-17, no relacionado con el trabajo de IA de arriba — solo se topó con él ahí).
  `prisma migrate dev` (no `migrate status` ni `migrate deploy`, que están OK) revalida **todo** el
  historial contra una shadow DB vacía, replicando las carpetas en orden alfabético/timestamp:
  `20250127000000_add_request_interactions` ordena antes que `20251215200251_init`, pero tiene una
  FK a `requests`, tabla que recién crea `init` — falla con `P3006`/`P1014` ("underlying table for
  model `requests` does not exist") en cualquier bootstrap desde cero (DB nueva, `migrate dev` para
  crear una migración nueva, CI, disaster recovery). La DB real de dev local está sana (probado:
  `migrate status` limpio, la app arranca bien) porque ya tiene todo aplicado — el bug solo pega en
  el paso de validación de `migrate dev`, no en producción vía `migrate deploy`.
  - [ ] Arreglo real: renombrar la carpeta `20250127000000_add_request_interactions` a un timestamp
    posterior a `20251215200251_init` (ver patrón ya usado en `scripts/fix-migration-record.sql`
    para casos similares).
  - [ ] **Antes de renombrar**: esa migración casi seguro ya está aplicada en el deploy de
    Fly.io/Supabase (es de enero 2025) — hay que corregir el registro `_prisma_migrations` ahí
    también (UPDATE del `migration_name`, no DELETE) en el mismo momento, o el próximo `prisma
    migrate deploy` en producción va a intentar reaplicar esa tabla/columnas y romper el deploy.
    Requiere acceso a esa DB remota — coordinar antes de tocar la carpeta local.
  - [ ] Workaround usado mientras tanto (2026-09-17, para las 2 migraciones de la feature de IA):
    aplicar el SQL a mano vía `prisma db execute --file <migration.sql>` +
    `prisma migrate resolve --applied <nombre>`, sin pasar por `migrate dev`. Válido para agregar
    migraciones nuevas puntuales, pero no arregla el problema de fondo.

### Seguimiento WhatsApp / Opt-out — pendientes detectados usando la feature (2026-09-18)

- [x] **Mensajes directos (no respuesta a un follow-up) no se procesan / WhatsApp como canal de
  soporte general** (2026-09-18, sesión de orquestador — investigación + diseño en plan mode +
  implementación, rama `feat/whatsapp-support-conversations` en `specialist-be` y
  `specialist-admin`, **PRs abiertos, sin mergear**: `specialist-be`
  [#52](https://github.com/DiegoSana/specialist-be/pull/52), `specialist-admin`
  [#11](https://github.com/DiegoSana/specialist-admin/pull/11)). El bug puntual escaló a una
  decisión de producto más grande (WhatsApp también como canal de soporte, no solo automatización)
  — ver `specialist-be/docs/decisions/ADR-005-SUPPORT-CONVERSATIONS.md` y
  `specialist-be/src/support/CLAUDE.md` para el diseño completo. Resumen:
  - Nuevo bounded context `support` (`SupportConversation` + `SupportMessage`), separado a
    propósito de `RequestInteraction`/`requests` — un mensaje de soporte nunca puede ser
    malinterpretado por el clasificador de intención de follow-ups.
  - El matching de follow-ups ahora tiene ventana de antigüedad (`WHATSAPP_REPLY_MATCH_WINDOW_DAYS`,
    default 14 días — antes no tenía límite, ese era el bug de fondo) y filtra explícitamente
    `interactionType: FOLLOW_UP` (antes implícito/casual, no exigido por la query).
  - Si un mensaje entrante no matchea ningún follow-up vigente, y `SUPPORT_CONVERSATIONS_ENABLED=true`
    (flag apagado por defecto para rollout seguro), cae a `SupportConversationService` en vez de
    perderse en silencio.
  - Admin: pantalla `/admin/support` (specialist-admin) — lista + hilo de conversación + responder
    (respetando la ventana real de 24hs de WhatsApp Business API, que no se trackeaba en ningún
    lado) + resolver/reabrir. Notificación in-app cuando entra un mensaje que necesita atención.
  - **Probado en local** (2026-09-18): flag prendido en `.env`, contenedor reiniciado, mensaje
    simulado vía `POST /api/webhooks/twilio` con `--data-urlencode` (¡importante! con `-d` un `+`
    en el teléfono se interpreta como espacio) → confirmado en logs y en la tabla
    `support_conversations` que el flujo entero funciona.
  - Mergeados: `specialist-be` #52, `specialist-admin` #11. `SUPPORT_CONVERSATIONS_ENABLED` ya está
    activado en el deploy de Fly (`specialist-be` #54).

- [ ] **Cómo se atan (o no) los flujos de follow-up automático y de soporte cuando coexisten para
  el mismo teléfono** (2026-09-18, caso planteado durante la implementación de arriba — **a
  diseñar en una sesión nueva**, no resuelto todavía). Caso concreto: un cliente responde un
  mensaje de follow-up (flujo normal, sin problema) y **después** nos escribe espontáneamente algo
  que el sistema cataloga como soporte (crea/reabre una `SupportConversation`). Preguntas abiertas:
  - Si mientras esa `SupportConversation` sigue `OPEN` el scheduler dispara un follow-up nuevo para
    ese mismo teléfono/request (`FollowUpSchedulerJob`), y el cliente responde — hoy el matching de
    `RequestInteractionService.processInboundMessage` **siempre prioriza** un follow-up vigente
    sobre la conversación de soporte (el fork a `support` es solo el fallback cuando no hay match).
    ¿Es el orden correcto? ¿O debería ganar la conversación de soporte mientras esté abierta, ya
    que hay un humano potencialmente esperando esa respuesta?
  - Cuando el follow-up "gana" el match, la `SupportConversation` sigue `OPEN` sin ningún cambio —
    no se resuelve sola, no queda ninguna nota/señal de que el cliente "se fue" al flujo
    automático. ¿Debería auto-resolverse? ¿Dejar un mensaje de sistema en el hilo? ¿Avisar al
    admin que el hilo que tenía abierto ahora está "dividido" entre dos flujos?
  - `SupportConversation.relatedRequestId` existe en el schema pero hoy nadie lo completa
    automáticamente — ¿debería `SupportConversationService.receiveInboundMessage` resolverlo solo
    (buscando el request más reciente para ese teléfono, aunque no haya matcheado como follow-up)
    para darle contexto al admin, en vez de quedar siempre en `null`?
  - Desde la perspectiva del cliente es **el mismo chat de WhatsApp** — no sabe que por dentro hay
    dos sistemas separados. Si el admin le está respondiendo por `/admin/support` y en paralelo el
    scheduler le manda un template de follow-up automático, al cliente le puede llegar como dos
    "voces" contradictorias en la misma conversación. ¿Hace falta pausar los follow-ups automáticos
    de un request mientras haya una `SupportConversation` abierta para ese teléfono? ¿O es
    sobre-ingeniería para MVP y alcanza con que el admin lo maneje a ojo por ahora?
  - **Decisión (2026-09-18)**: pausar follow-ups por teléfono mientras haya una `SupportConversation`
    `OPEN` (guard en `FollowUpSchedulerJob`). **Mergeado**: `specialist-be`
    [#53](https://github.com/DiegoSana/specialist-be/pull/53). Sin auto-resolver, sin cambiar el
    orden del matching. Además, `specialist-admin`
    [#12](https://github.com/DiegoSana/specialist-admin/pull/12): campana de notificaciones in-app
    (antes el admin no tenía dónde leerlas) y contador de conversaciones abiertas en "Soporte".
    `relatedRequestId` ya se completa automáticamente (2026-09-18, specialist-be #56, mergeado).
  - Investigar/diseñar en modo plan antes de implementar, como se hizo con el ítem de arriba —
    tiene forma de tocar tanto `requests` (scheduler, matching) como `support` (relatedRequestId,
    posible pausa) y quizás `specialist-admin` (señal visual de "este teléfono tiene las dos cosas
    abiertas").

- [x] **Opt-out: visibilidad y accionabilidad desde el admin** (2026-09-18, sesión de orquestador,
  rama `feat/whatsapp-optout-admin` en `specialist-be` y `specialist-admin`, **mergeada**: `specialist-be`
  [#51](https://github.com/DiegoSana/specialist-be/pull/51), `specialist-admin`
  [#10](https://github.com/DiegoSana/specialist-admin/pull/10)). `GET /admin/users`(`/:id`) expone
  `whatsappOptedOut`/`whatsappOptedOutAt`; nuevo `PUT /admin/users/:id/whatsapp-opt-out` (admin,
  `{ whatsappOptedOut: boolean }`) para marcar/desmarcar manualmente. Pantalla de listado con pill
  "WhatsApp opt-out"; detalle de usuario con tarjeta de override (mismo patrón que la de
  verificación) + confirmación antes de aplicar. **Cómo se le avisa al usuario**: email automático
  en ambos sentidos — al quedar opted-out (`UserWhatsAppOptedOutEvent`, primer domain event del
  contexto Identity) y al ser reactivado por un admin (`UserWhatsAppReactivatedEvent`, agregado en
  un segundo paso de la misma sesión después de que el usuario nos avisó que el toggle no
  notificaba en el sentido de reactivación) — ambos forzando `EMAIL` como canal externo vía el
  nuevo parámetro `forceExternalChannel` de `NotificationService.createForUser`.
  - [ ] Pendiente (no abordado en esta sesión): relacionado — "cuando el usuario expresa que no
    quiere ser contactado, el request va a 'review'" — revisar qué implica esto exactamente hoy y
    si hace falta un estado/flag dedicado en el request en sí (más allá del `RequestAttentionFlag`
    que ya existe) o alcanza con eso.

- [x] ~~**Mostrar el estado de opt-out al usuario en su propio perfil**~~ **Resuelto 2026-09-23**
  — ver detalle en "🆕 Pendientes del rediseño de estados" / "▶️ Por dónde retomar" al principio de
  este archivo (`specialist-be` #73, `specialist-fe` #30, ambos mergeados).

- [x] ~~**Revisar el flujo de estados de `Request`**: evaluar agregar un estado `ASIGNADO` entre
  `PENDING` y `ACCEPTED`.~~ **Descartado 2026-09-23**: ítem escrito contra el modelo viejo de 5
  estados; el rediseño a 15 estados (2026-09-21/22, ver ADR-006) ya cubre ese hueco intermedio. Sin
  acción.

- [ ] **Revisar el diseño de usuarios y perfiles, incluyendo los flujos de estado** (2026-09-18):
  auditoría más amplia que el bug puntual de arriba — cubrir la relación `User` ↔
  `Professional`/`Company`/`Client`, los estados de cada perfil (`PENDING_VERIFICATION`, `ACTIVE`,
  `VERIFIED`, `INACTIVE`, `REJECTED`, `SUSPENDED`) y su cruce con `emailVerified`/`phoneVerified`
  del usuario y con `ProfileActivationService`/`canOperate()`. Buen punto de partida: el bug de
  Company arriba puede ser síntoma de una inconsistencia de diseño más general, no solo un caso
  aislado. Relacionado con el ítem pendiente "Agregar diagramas de estado de Request" (sección
  Documentación Pendiente) y con [Perfil activo (MVP)](#-perfil-activo-mvp-reglas-y-restricciones).

### UX
- [ ] Notificaciones push (web)
- [ ] Tiempo real con WebSockets
- [ ] Búsqueda avanzada de especialistas

### Historial de estados de Request (diferido 2026-09-16)
- [ ] **Timestamps por cada cambio de estado** (accepted/in_progress/done): hoy `Request` solo
  tiene `createdAt`/`updatedAt`, sin historial de transiciones. Surgió como parte de "Pantalla de
  Requests del admin necesita más información" (ver sección Admin) pero se dejó explícitamente
  fuera de ese alcance por requerir migración (columnas nuevas o tabla de historial de estados).

### Portal de Administración

> 📖 **Plan completo:** [docs/plans/admin-portal-plan.md](./docs/plans/admin-portal-plan.md)

**Estado:** Planificación - Pendiente decidir stack tecnológico FE/UI

**Decisiones pendientes:**
- [ ] Decidir stack tecnológico frontend (Next.js, React Admin, AdminJS, Shadcn UI)
- [ ] Decidir UI framework/component library
- [ ] Definir funcionalidades básicas MVP
- [ ] Crear mockups/wireframes básicos

**Funcionalidades MVP planificadas:**
- [ ] Dashboard con métricas y KPIs
- [ ] Gestión de usuarios (listar, ver, editar, cambiar estado, confirmar email/teléfono manualmente)
- [ ] Gestión de solicitudes (listar, ver, acciones administrativas)
- [ ] Gestión de perfiles profesionales y empresas (verificar, suspender, confirmar perfil)
- [x] **Moderación de reviews pendientes** (2026-09-16) — `GET /reviews/admin/pending`, `POST /reviews/:id/approve`, `POST /reviews/:id/reject`; pantalla en admin FE lista (ver Fase D).
- [ ] Gestión de notificaciones (estadísticas, reenviar fallidas)

**Fases de implementación:**
- [ ] Fase 1: Setup y Autenticación
- [ ] Fase 2: Dashboard y Gestión de Usuarios
- [ ] Fase 3: Gestión de Solicitudes y Perfiles
- [ ] Fase 4: Moderación y Notificaciones
- [ ] Fase 5: Polish y Mejoras

### Soporte y Chat
- [ ] **Chat con Administrador en Request**
  - [ ] Agregar botón de chat con administrador en pantalla de detalle de request
  - [ ] Botón visible tanto para clientes como para especialistas
  - [ ] Implementar sistema de chat/mensajería con administradores
  - [ ] Considerar opciones:
    - Integración con servicio de chat externo (Intercom, Crisp, etc.)
    - Chat interno con notificaciones a administradores
    - Sistema de tickets de soporte
  - [ ] Contexto del chat debe incluir información del request (ID, título, estado)
  - [ ] Permitir que usuarios reporten problemas específicos del request
  - [ ] Notificaciones a administradores cuando hay nuevos mensajes
  - [ ] Panel de administración para gestionar conversaciones de soporte
  - [ ] Agregar tests para funcionalidad de chat
  - [ ] Documentar flujo de soporte

---

## 📅 Prioridades Sugeridas

### Corto plazo
1. Avanzar con [Perfil activo (MVP)](#-perfil-activo-mvp-reglas-y-restricciones): Fase A (backend) y Fase D (moderación reviews en admin FE).
2. Confirmar/migrar pantalla de especialistas a usar solo `GET /providers` (FE).

### Esta Semana
1. ~~Crear PRs pendientes~~ ✅ BE #10, #11 | FE #3, #4
2. ~~Merge de PRs existentes~~ ✅
3. Revisar módulo de Reviews (permisos de moderación)

### Próxima Semana
1. ~~Refactorizar Notifications module~~ ✅
2. ~~Revisar Profiles module~~ ✅
3. ~~Revisar Identity module~~ ✅
4. Revisar DTOs en controladores principales

### Mes
1. DTOs completos en todos los controladores
2. Documentación de arquitectura
3. Tests E2E
4. Perfil activo: restricciones por verificación y confirmación admin (Fases A–B)

---

## 📌 Notas

- El patrón de `AuthContext` puede ser extraído a un módulo compartido
- Considerar crear un guard de NestJS genérico para permisos comunes
- Los métodos `canXxxBy()` en entidades siguen el principio de "tell, don't ask"


---

## 🎨 Frontend (specialist-fe)

> Fusionado desde `specialist-fe/TODO.md` (última actualización previa: 2026-01-06, archivo
> borrado de ese repo el 2026-09-16). Solo se trajo lo pendiente; el detalle de bugs ya resueltos
> quedó en el historial de PRs de ese repo (#3, #4, #5).

### 📋 Resumen de Estado (a 2026-01-06, puede haber quedado desactualizado)

| Feature | Estado | Tests | Mobile |
|---------|--------|-------|--------|
| Dashboard Cliente | ✅ | ⬜ | ✅ |
| Dashboard Especialista | ✅ | ⬜ | ✅ |
| Detalle Solicitud (Cliente) | ✅ | ⬜ | ✅ |
| Detalle Solicitud (Especialista) | ✅ | ⬜ | ✅ |
| Notificaciones | ✅ | ⬜ | ✅ |
| Job Board | ✅ | ⬜ | ⬜ |
| Perfiles | ✅ | ⬜ | ⬜ |

### ⬜ Bugs pendientes

- [ ] **Revisar acceso a solicitudes por URL directa**
  - El backend ahora devuelve 403, ¿el FE lo maneja bien? Mostrar mensaje apropiado al usuario.
- [ ] **Traducciones faltantes identificadas** — verificar que todas las keys estén completas en
  `messages/es.json` y `messages/en.json`.
- [x] **Bug mobile (2026-09-18): botón "Contactar por WhatsApp" roto en la vista de detalle de
  request** — resuelto en `specialist-fe` [#17](https://github.com/DiegoSana/specialist-fe/pull/17):
  info del proveedor y botones se apilan en mobile y el texto ya no se parte.

### ⬜ Pendiente (2026-09-18)

- [x] **"Mis solicitudes"**: título del request y nombre del proveedor asignado (profesional o
  empresa) en la tarjeta (2026-09-18, `specialist-fe` #18).

### ⬜ Código a evaluar eliminar

- [ ] ¿Eliminar traducciones relacionadas a quotes? (`acceptQuote`, `quote`, `amount`, etc. —
  mantener si se planea implementar en el futuro.)

### 🎯 Mejoras de UX pendientes

**Alta prioridad:**
- [ ] Manejo de errores 403/401: mensaje amigable + redirigir a dashboard si no puede ver una
  solicitud.
- [ ] Loading states consistentes: skeletons en lugar de spinners donde aplique.
- [ ] Empty states: mejores mensajes + call-to-action relevante cuando no hay datos.

**Media prioridad:**
- [ ] Optimistic updates en acciones frecuentes (marcar como leído, etc.)
- [ ] Toast notifications para feedback de acciones completadas y errores.

**Baja prioridad:**
- [ ] Animaciones y transiciones (page transitions, list animations).

### 📱 Responsive pendiente

- [ ] Job Board page (revisar en 360px)
- [ ] Perfil de especialista
- [ ] Formulario de nueva solicitud
- [ ] Lista de especialistas interesados

### 🧪 Tests pendientes

- [ ] Tests unitarios para hooks (parcialmente hecho — ver `hooks/__tests__/`, ampliar cobertura)
- [ ] Tests de componentes críticos
- [ ] Tests E2E con Playwright/Cypress (no existe suite e2e en este repo)
- [ ] **Evaluar Playwright para tests E2E** (2026-09-18): decidir si se adopta Playwright puntualmente
  (en vez de dejarlo como opción genérica junto a Cypress arriba) — cubriría flujos críticos de
  `specialist-fe` (y potencialmente `specialist-admin`, que tampoco tiene suite de tests) que hoy
  solo están validados por los E2E de backend (`specialist-be/test/*.e2e-spec.ts`, que no ejercitan
  UI real). Definir alcance inicial (login, crear solicitud, expresar interés) y si corre en CI.

### 🌐 Traducciones — verificar completitud

- [ ] `messages/es.json` / `messages/en.json` — keys identificadas como faltantes (a re-verificar,
  pueden estar resueltas):
  ```
  navigation.specialists
  specialist.requestDetail.interestExpressed
  specialist.requestDetail.interestExpressedDescription
  specialist.requestDetail.removeInterest
  ```

### 🎨 UI/UX pendiente

**Alta prioridad:**
- [x] Tarjetas de especialista: badge "Empresa" si tiene perfil de empresa (comprobado 2026-09-18 en
  `professionals/page.tsx`).
- [x] Tarjetas de especialista: texto del botón "Solicitar" → "Contactar" (comprobado 2026-09-18).
- [x] Formulario de solicitud directa: descripción aclara que el proveedor puede ser un especialista
  o una empresa (2026-09-18, `specialist-fe` #19).

**Media prioridad:**
- [ ] Consistencia en badges de estado, iconografía unificada, colores de estado estandarizados,
  tipografía responsive.

### 🔍 Feature: Job Board Search

- [ ] Filtro por palabra clave (nombre del trade, nombre del especialista/empresa), input con
  debounce, integrar con endpoint existente (query param `search`).

### 🏢 Feature: Company Profiles

> Base implementada en PR #5 (types, hooks, setup page, sección en profile, nav, traducciones).

- [x] Company Dashboard (2026-09-18, comprobado): en vez de `/company/dashboard`, la empresa usa
  "Mis Trabajos" (`/specialist/dashboard`, specialist-fe#16) y la bolsa de trabajo (#15). Opcional
  pendiente: que el login de una empresa la mande a "Mis Trabajos" en vez de a la bolsa de trabajo.
- [x] Company accede al Job Board (specialist-fe#15, comprobado); falta comprobar de punta a punta
  el flujo de expresar interés y ser asignada
- [ ] Mostrar tipo de proveedor (Professional/Company) en interesados
- [ ] Company public profile page

### 📌 Notas (frontend)

- Usar `@tanstack/react-query` devtools para debugging.
- El hook `useRequest` puede devolver 403 — verificar manejo de error en todos los call sites.
- Considerar extraer lógica de permisos a un hook `useCanViewRequest`.
- Company usa colores `emerald` para diferenciarse de Professional (`blue`/`green`).

---

## 🛠️ Admin (specialist-admin)

> No tenía `TODO.md` propio; esto es el checklist "Next Steps" de su `README.md` (Fase 1
> completada: auth, layout, integración con `@specialist/shared`).

### ⬜ Pendiente

- [ ] Dashboard con métricas reales (hoy es un placeholder)
- [ ] Gestión de usuarios (CRUD completo)
- [ ] Gestión de solicitudes
- [ ] Gestión de perfiles Profesional/Empresa
- [x] Pantalla de moderación de reviews (2026-09-16) — ver Fase D más arriba.
- [ ] `components/ui/` de shadcn no está generado todavía pese a que `components.json` ya lo
  configura (ver `specialist-admin/CLAUDE.md`).

### ⬜ Pendiente (2026-09-18)

- [x] **Grilla de Requests**: reordenar columnas y sacar la de ubicación. Orden final: status,
  title, client, provider, created, actions.
- [x] **Filtros en el listado de Requests del admin**: filtrar por cliente, por proveedor y por
  título (hoy solo hay filtro por status). Cross-repo: `GET /admin/requests` en `specialist-be`
  necesita los query params nuevos (ver cómo `GET /admin/users` ya resolvió `?search=` server-side,
  mismo patrón) y `specialist-admin` los controles de filtro (con debounce, como en Users).
  Mergeado (2026-09-18): `specialist-be` #56 + `specialist-admin` #13.
- [ ] **Mejorar la vista de detalle de Request** (a definir en más detalle qué específicamente).
- [x] **Vista de conversación de WhatsApp** (`/admin/whatsapp/[requestId]`): nombre del cliente y del
  proveedor clickeables hacia `/admin/users/:id` (2026-09-18, `specialist-be` #58 expone
  `provider.userId` en `GET /admin/requests/:id` + `specialist-admin` #14).
- [x] **Opt-out**: UI en el admin para verlo/accionarlo (2026-09-18) — ver el ítem completo en la
  sección Backend ("Opt-out: visibilidad y accionabilidad desde el admin"), PR
  [#10](https://github.com/DiegoSana/specialist-admin/pull/10) mergeado.
- [ ] **¿Qué tan complejo es hacer el admin mobile responsive?** — evaluar alcance antes de
  encararlo (hoy no está pensado para mobile, ver también `.claude/rules/` de este repo).

### ⬜ Pendiente (2026-09-16, feedback tras probar `/admin/whatsapp`)

- [x] **Tarjeta de info del request en `/admin/whatsapp/[requestId]`** (2026-09-16): tarjeta con
  título (linkeado), status, cliente, proveedor+tipo, fechas, vía `GET /admin/requests/:id`.
- [x] **Pantalla de Requests del admin necesita mucha más información** (2026-09-16): list ahora
  muestra proveedor (nombre + badge tipo); detail nuevo vía `GET /admin/requests/:id` (nuevo
  endpoint — antes no existía, el FE pegaba contra el `GET /requests/:id` no-admin) trae proveedor
  completo con trades, sección de proveedores interesados, y link directo a la conversación de
  WhatsApp. **Fechas de cada cambio de estado quedaron explícitamente fuera de alcance** (el schema
  de `Request` solo tiene `createdAt`/`updatedAt`; requeriría columnas nuevas o tabla de historial
  — ver ítem nuevo en Backlog más abajo).
- [x] **Unificar el menú Users/Professionals/Companies en un solo "Usuarios"** (2026-09-16):
  `app/admin/users` reescrito con tabs de filtro (All/Client/Professional/Company) + búsqueda,
  ambos server-side (`?search=` y `?type=` nuevos en `GET /admin/users`, debounce 350ms en la
  búsqueda). Sidebar con una sola entrada "Users". De paso se corrigieron dos bugs encontrados al
  probar: `GET /admin/users` nunca mandaba `isAdmin`/`hasCompanyProfile` (el FE ya los leía, así
  que los badges estaban rotos); y `getUsers` en el FE no desempaquetaba `{data, meta}` de la
  respuesta, así que "Total" mostraba 0 y la paginación nunca aparecía. Sin tab "Admin": `isAdmin`
  es un boolean plano en `User`, no una entidad de perfil como Client/Professional/Company — sigue
  mostrándose como badge por fila, solo se sacó del filtro. Las pantallas `[id]` de detalle de
  Professional/Company (`app/admin/professionals/[id]`, `app/admin/companies/[id]`) se mantuvieron
  pero **quedaron huérfanas de navegación** — resuelto después: `specialist-admin` #8 las linkea
  desde el detalle de usuario.
  PRs: `specialist-be` [#44](https://github.com/DiegoSana/specialist-be/pull/44),
  `specialist-admin` [#6](https://github.com/DiegoSana/specialist-admin/pull/6) (rama
  `feat/admin-portal-overhaul` en ambos repos, sin mergear).
- [x] **Desplegar `specialist-admin` en Vercel** (2026-09-16): ya está deployado. Pendiente menor:
  `DEPLOYMENT.md` (raíz) todavía solo documenta el paso a paso de Vercel para `specialist-fe`
  (Step 3, root directory `specialist-fe`) — falta agregar ahí la sección equivalente para admin
  (root directory `specialist-admin`, `NEXT_PUBLIC_API_URL` con el sufijo `/api` incluido a
  diferencia de `specialist-fe`, ver Gotchas de `specialist-admin/CLAUDE.md`) para que quede
  documentado el paso a paso real.

---

## 📦 Shared (specialist-shared)

> No tenía `TODO.md` propio. Backlog ya documentado en `specialist-shared/CLAUDE.md` y repetido
> en los `CLAUDE.md` de sus consumidores.

### ⬜ Pendiente

- [x] **`types/user.ts` desactualizado** (resuelto 2026-09-18, `specialist-shared` #2: `User` con la
  forma real del backend, `UserStatus` corregido, `UserRole`/`USER_ROLES` deprecados,
`AuthResponse.user.hasCompanyProfile`). Dependencia bumpeada en `specialist-admin` y
  `app/test-shared/page.tsx` adaptada (`specialist-admin` #15, 2026-09-18).
  Descripción original: `User`/`UserRole` modelan un `role` único + `name`, pero
  el modelo real del backend usa `firstName`/`lastName` + booleans independientes
  (`hasClientProfile`/`hasProfessionalProfile`/`hasCompanyProfile`/`isAdmin`). `AuthResponse.user`
  en el mismo paquete ya tiene la forma correcta. `specialist-admin` lo esquiva declarando su
  propio tipo local en `lib/api/admin.ts`. Decidir: arreglar `types/user.ts` (y actualizar los call
  sites en `specialist-admin`), o directamente retirar ese export si no aporta valor.
- [x] `AdminContract` alineado con `specialist-be` (2026-09-18, `specialist-shared` #2): status
  updates son `PUT`, la moderación de reviews vive bajo `/reviews`, y se agregaron las rutas de
  verification, whatsapp-opt-out y notifications que faltaban. Nadie lo consume hoy.

- [ ] **Decidir el futuro de `specialist-shared`** (análisis 2026-09-18, retomar después). Hallazgos:
  - Solo `specialist-admin` lo declara (`github:DiegoSana/specialist-shared#main`); `specialist-be` y
    `specialist-fe` no lo usan (fe mantiene su `types/index.ts` con su propio `UserStatus`).
  - Uso real en admin: solo `loginSchema`, `LoginDTO` y `AuthResponse` (`lib/api.ts`,
    `hooks/use-admin-auth.ts`, `app/admin/login/page.tsx`). Sin uso: `AdminContract` (112 de 257
    líneas; admin hardcodea ~50 paths en `lib/api/admin.ts`), `User`, `UserStatus`, `USER_STATUS`,
    `REQUEST_STATUS`. `UserRole`/`USER_ROLES` (deprecated) solo los toca `app/test-shared/page.tsx`,
    que arma un `User` con forma vieja (`name`, `role`).
  - Riesgo de deriva: `loginSchema` exige `min(6)` en password; el `LoginDto` del backend no. El
    `RegisterDto` sí tiene `MinLength(6)`.
  - Costo: ciclo build+commit+push+reinstall (`postinstall` corre `tsc`, `dist/` versionado) para
    compartir muy poco.
  - Opciones: (1) usarlo de verdad en admin (`AdminContract` + tipos compartidos, borrar
    `test-shared` y lo deprecated); solo vale la pena si `fe` también lo adopta, y `specialist-be`
    sigue siendo la fuente de verdad, así que `shared` queda como espejo manual. (2) Encogerlo a
    `loginSchema` + `AuthResponse`. (3) Eliminarlo y mover esas líneas a admin (recomendada si solo
    admin lo consume). (4) Alternativa a un espejo manual: generar tipos/paths desde el OpenAPI del
    backend (`@nestjs/swagger` ya está en `main.ts`, ej. `openapi-typescript`); pendiente verificar
    cuán completos están los `@ApiProperty` en los DTOs.
  - No se recomienda que `specialist-be` consuma `shared`: usa `class-validator` + Prisma, no zod.
