# 📋 Specialist — Todo & Roadmap (global)

> Backlog vivo, no historial. Qué se hizo, cuándo y en qué PR vive en el `git log`/PRs mergeados de
> cada repo (y en el `git log` de este propio archivo, si hace falta reconstruir una versión vieja
> con contexto). Acá solo se lista lo que sigue **pendiente**.

Vive en `/var/www/specialist/` (el directorio padre, no es parte de ninguno de los cuatro repos) y
es el único TODO/roadmap del proyecto. El skill `orchestrate-feature`
(`.claude/skills/orchestrate-feature/`) lo lee en su paso de investigación al planificar un
requerimiento cross-repo. El skill `session-recap` de `specialist-be`
(`specialist-be/.claude/skills/session-recap/`) sigue siendo el mecanismo para cerrar una sesión de
trabajo del backend y debe actualizar **este** archivo (no uno local); las otras secciones se
actualizan a mano.

## Índice

- [🆕 Sin investigar](#-sin-investigar)
- [🎯 Decisión / diseño pendiente](#-decisión--diseño-pendiente)
- [🔧 Backend (specialist-be)](#-backend-specialist-be)
- [🎨 Frontend (specialist-fe)](#-frontend-specialist-fe)
- [🛠️ Admin (specialist-admin)](#️-admin-specialist-admin)
- [📦 Shared (specialist-shared)](#-shared-specialist-shared)

---

## 🆕 Sin investigar

Pedidos directos del usuario, todavía no investigados — candidatos para `orchestrate-feature`.

1. ~~**[BE+FE] Bloquear multi-perfil de usuario para el MVP.**~~ **Resuelto 2026-09-24** (rama
   `feat/block-multi-profile-mvp` en ambos repos): `specialist-be`
   [#74](https://github.com/DiegoSana/specialist-be/pull/74), mergeado — `UserEntity.canCreate
   ProfessionalProfile()`/`canCreateCompanyProfile()` (antes código muerto) ahora implementan la
   regla simétrica (cliente puro bloqueado; quien ya es profesional o empresa puede crear el otro
   tipo de proveedor aunque también tenga perfil cliente) y se hacen cumplir en
   `ProfessionalService.createProfile`/`CompanyService.createProfile` (403). `specialist-fe`
   [#31](https://github.com/DiegoSana/specialist-fe/pull/31), mergeado — `profile/page.tsx` oculta
   por completo las secciones "Perfil de Especialista"/"Perfil de Empresa" (no solo el botón) para
   un usuario cliente puro; vuelven a aparecer solas cuando se permita multi-perfil post-MVP
   (gateado por el mismo `isPureClient`). El flujo "especialista registra su empresa" se investigó
   y no tenía ningún bug — ya funcionaba de punta a punta antes de este cambio y sigue funcionando
   después, gracias a la regla simétrica.
2. ~~**[FE, posiblemente BE] Social login: bloquear navegación sin perfil elegido.**~~ **Resuelto
   2026-09-24** (`specialist-fe` [#32](https://github.com/DiegoSana/specialist-fe/pull/32),
   mergeado, sin cambios de backend). Causa: `hooks/use-require-profile.ts` ya implementaba el
   redirect correcto pero era código muerto (no se usaba en ningún lado); `notifications/page.tsx`
   y `profile/page.tsx` renderizaban con `AppLayout` (sin ningún guard) en vez de `ProtectedLayout`
   como el resto de las páginas autenticadas. Se reescribió el hook (resuelve el usuario en un
   efecto para evitar hydration mismatch, redirige a `/login` si no hay sesión y a
   `/profile-setup` si no hay ningún perfil) y se enganchó en esas dos páginas. De paso, arreglado
   un bug chico encontrado en el camino: `profile-setup/page.tsx` no contemplaba
   `hasCompanyProfile` al decidir si el usuario ya tenía perfil (un usuario solo-empresa veía la
   pantalla de selección de rol de nuevo).
3. ~~**[FE] Nueva solicitud: destacar el beneficio de agregar fotos + mejores prácticas.**~~
   **Resuelto 2026-09-24** (`specialist-fe` [#33](https://github.com/DiegoSana/specialist-fe/pull/33),
   `specialist-be` [#75](https://github.com/DiegoSana/specialist-be/pull/75), ambos mergeados). La
   subida de fotos en "nueva solicitud" era un placeholder no funcional ("próximamente") — se
   implementó de punta a punta: banner de beneficio bien visible, lista de mejores prácticas, modal
   opcional si se envía sin fotos, y soporte real de fotos **y video** (multi-selección, hasta 6
   archivos). En el camino se encontraron y arreglaron 3 bugs reales, ninguno introducido por esta
   sesión: (1) `request-photo` nunca permitía video en el backend, tirando 500 en vez de subir (BE
   #75); (2) subir antes de crear la solicitud daba 403 porque el storage no tiene metadata en
   base de datos — se resolvió subiendo fotos/videos recién después de creada la solicitud (mismo
   patrón que ya usa la pantalla de detalle), no tocando la lógica frágil de permisos; (3) al regex
   de detección de video le faltaba `.mov` (el formato que graba un iPhone por default), rompiendo
   silenciosamente esos videos también en la pantalla de detalle ya en producción. Además: si un
   archivo excede el tamaño máximo (10MB foto / 100MB video) ahora avisa al elegirlo, y si falla
   adjuntar algo después de crear la solicitud ya no navega en silencio — muestra qué falló y por
   qué. **Pendiente**: click-through manual con backend real (no se hizo esta sesión).
4. ~~**[BE, bug] Permisos de imágenes/archivos: el admin recibe "access denied".**~~ **Resuelto
   2026-09-24** (`specialist-admin` [#18](https://github.com/DiegoSana/specialist-admin/pull/18),
   mergeado). No era un bug de backend: `FileAccessGuard`/`canAccessFile()` en `specialist-be` ya
   le daban acceso total al admin. La causa real era en `specialist-admin`:
   `app/admin/requests/[id]/page.tsx` mostraba las fotos (privadas, `storage/private/...`) con un
   `<img src>` plano, que el navegador pide sin el header `Authorization` — el backend lo trataba
   como no autenticado y devolvía 403. Se portó el patrón que `specialist-fe` ya usa para esto
   (`AuthenticatedImage`: fetch con bearer token + blob URL) a este repo. **Pendiente**: click-through
   manual en el navegador con una solicitud que tenga fotos privadas — no se hizo en esta sesión
   (este repo no tiene suite de tests).
5. ~~**[BE+FE] Configuración de visibilidad para especialistas y empresas.**~~ **Resuelto
   2026-09-28** (rama `feat/provider-visibility-toggle` en ambos repos): `specialist-be`
   [#79](https://github.com/DiegoSana/specialist-be/pull/79), mergeado — campo `isVisible: Boolean
   @default(true)` agregado a `Professional` y `Company`, editable por el dueño vía
   `PATCH /professionals/me`/`/companies/me`, filtrado (`where.isVisible = true`) en
   `search()` de ambos repositorios — cubre búsqueda pública y selección de destinatario para
   solicitudes directas (mismo código). No afecta `findById` directo, `Request.providerId` ni
   `RequestInterest` ya existentes; listados de admin usan un query path separado y no se tocaron.
   `specialist-fe` [#38](https://github.com/DiegoSana/specialist-fe/pull/38), mergeado — toggle en
   `profile/page.tsx` (secciones "Perfil de Especialista"/"Perfil de Empresa"), clonando el patrón
   visual del opt-out de WhatsApp.

## 🎯 Decisión / diseño pendiente

- **Auditoría de usuarios y perfiles + flujos de estado.** Entregable: documento de hallazgos, no
  código (puede incluir los diagramas de estado de `Request` que faltan). Cubrir la relación `User`
  ↔ `Professional`/`Company`/`Client`, los estados de cada perfil y su cruce con
  `emailVerified`/`phoneVerified` y `ProfileActivationService`/`canOperate()`. Ver
  `specialist-be/docs/architecture/PROFILE_ACTIVATION_ORCHESTRATION.md` y ADR-006
  (`docs/decisions/ADR-006-REQUEST-STATE-MACHINE.md`).
- **Validaciones extra de Company.** Definir primero qué se exige (CUIT, AFIP, documentación) antes
  de implementar.
- **Futuro de `specialist-shared`.** Solo `specialist-admin` lo consume, y solo para
  `loginSchema`/`LoginDTO`/`AuthResponse` — el resto (`AdminContract`, `User`, etc.) no tiene uso
  real hoy. Opciones: (1) adoptarlo de verdad (requeriría que `specialist-fe` también lo use), (2)
  encogerlo a solo lo que se usa, (3) eliminarlo y mover esas pocas líneas a `specialist-admin`
  (recomendada si solo admin lo sigue consumiendo), (4) generar tipos desde el OpenAPI del backend
  en vez de mantener un espejo manual.
- ~~**Contenido legal/institucional de `specialist-fe`.**~~ **Resuelto 2026-09-29**
  ([specialist-fe#39](https://github.com/DiegoSana/specialist-fe/pull/39), mergeado). Decisión de
  contenido (con el usuario): T&C redactados desde cero como boilerplate de etapa MVP con
  disclaimer visible (plataforma en prueba, razón social a definir); "Quiénes somos" con texto
  genérico de misión, sin historia personal; "Contacto" con email placeholder marcado para
  reemplazar cuando se defina un canal real (no existía ningún mecanismo de contacto público en el
  repo, solo WhatsApp deep-links atados a una solicitud). Nuevas rutas `/about`, `/terms`,
  `/contact`; `Footer` reusable con esos tres links, visible en toda la app (se agregó a
  `ProtectedLayout`/`AppLayout` — dashboards, requests, job board, professionals, notifications,
  profile — y a `login`/`register`, que no tenían shell compartido). Deliberadamente sin footer:
  `profile-setup`/`company/setup`/`specialist/setup` (onboarding sin chrome por diseño existente) y
  `auth/callback` (solo redirect). Pendiente real: reemplazar el email placeholder y la razón
  social cuando existan, y sumar Política de Privacidad si se decide (no estaba en el alcance
  pedido).
- ~~**Visibilidad de fotos/videos de solicitudes.**~~ **Decidido 2026-09-24**: sirven para que el
  especialista pueda valuar el trabajo, así que **mientras la solicitud es pública y no tiene
  proveedor asignado**, las fotos/videos son visibles para cualquier usuario autenticado (no
  anónimo). **En el momento en que se asigna un proveedor** (`providerId`, sea profesional o
  empresa), las fotos pasan a ser privadas — solo las ve el cliente, ese proveedor asignado, y los
  admins; el resto de especialistas (incluso los que expresaron interés) pierde el acceso ahí
  mismo. Las solicitudes **directas** (no públicas, dirigidas a un especialista puntual) son
  privadas desde el vamos, sin la ventana pública. Implementado en `specialist-be`
  (`fix/request-photo-privacy`, ver Backend) y reflejado en el mensaje informativo del formulario de
  nueva solicitud en `specialist-fe`.

---

## 🔧 Backend (specialist-be)

**Bugs / permisos**
- Verificar acceso a solicitudes completadas mostradas en perfiles públicos de otros especialistas.
- Revisar validación de permisos en fotos de solicitudes (¿son públicas las de un trabajo
  completado? ¿quién ve las de uno en progreso?) — relacionado con el bug de admin de arriba.
- `prisma migrate dev` roto por orden del historial de migraciones:
  `20250127000000_add_request_interactions` ordena antes que `20251215200251_init` pero depende de
  una tabla que recién crea `init`, falla con `P3006` en cualquier bootstrap desde cero (DB nueva,
  CI, disaster recovery). Arreglo real: renombrar la carpeta a un timestamp posterior — pero antes
  hay que corregir el registro `_prisma_migrations` en la DB de producción (Fly/Supabase, la
  migración es de enero 2025 y probablemente ya está aplicada ahí), si no el próximo `migrate
  deploy` la reintenta y rompe. Workaround actual para migraciones nuevas: aplicar a mano vía
  `prisma db execute` + `migrate resolve --applied`, sin pasar por `migrate dev`. Ver
  `docs/guides/MIGRATION_GUIDE.md`.
- Decidir si mantener o eliminar `quoteAmount`/`quoteNotes` en `Request` (hoy sin uso, no es MVP).

**Deploy / performance**
- ~~Reportado 2026-09-24: la API en Fly.io se sentía lenta en *todos* los requests.~~ **Resuelto
  2026-09-24**: región movida de `gru` (São Paulo) a `lax` (Los Ángeles, cerca del proyecto de
  Supabase en `us-west-2`), `DATABASE_URL` apuntado al pooler de Supabase (6543) con
  `?pgbouncer=true`, `DIRECT_URL` (conexión directa 5432) agregado para las migraciones, y
  `min_machines_running: 1` para sacar los cold starts. Mergeado en
  [specialist-be#76](https://github.com/DiegoSana/specialist-be/pull/76) y deployado con éxito
  (CI verde, health check 200, release sin errores). Dos gotchas de Supabase que costaron un
  intento fallido cada uno, documentados en la memoria del proyecto: (1) sin `?pgbouncer=true` en
  la URL del pooler, Prisma tira `26000: prepared statement does not exist`; (2) el usuario del
  pooler (`postgres.<ref>`) no sirve para la conexión directa, que usa `postgres` a secas — mezclar
  los dos da `P1000: Authentication failed`.
- Detectado 2026-09-29 probando Twilio sandbox localmente: `docker-compose.dev.yml` (servicio
  `app`) nunca recibió el mismo `DIRECT_URL` que se agregó para producción (ver arriba) — el
  comando de arranque corre `prisma db push`, que valida `directUrl = env("DIRECT_URL")` en
  `schema.prisma` y no está definido ni en el `environment:` del compose ni en `.env` local, así
  que el contenedor falla con `P1012` antes de levantar. Agregar una línea `DIRECT_URL: ...`
  (mismo valor que `DATABASE_URL`, apuntando a `localhost:5432` como el resto del servicio) al
  `environment:` de `app` en `docker-compose.dev.yml` para que `docker compose up` funcione out of
  the box.

**Notificaciones / WhatsApp**
- **[Prioridad]** Investigar qué pasa cuando un cliente o especialista tiene **varias requests
  abiertas en simultáneo**: los follow-ups matchean mensajes entrantes por número de teléfono
  (`findMostRecentByPhone`, ver specialist-be#87 del 2026-09-29, que ya resolvió el caso de una
  interaction vieja sin responder robando un mensaje nuevo **dentro del mismo request** — esto es
  el caso más amplio, **entre requests distintos** del mismo teléfono). Si el mismo número tiene
  follow-ups pendientes en dos requests a la vez, ¿cómo se evita que un mensaje se le asigne al
  request equivocado? ¿Hay que desambiguar en el propio mensaje (pedirle al usuario que aclare a
  cuál request se refiere), limitar a un follow-up activo por teléfono a la vez, o algo distinto?
  Definir el comportamiento esperado antes de tocar código.
- Separar el guard de `AdminWhatsAppDevController`: hoy tanto `simulate-reply` como
  `trigger-followup` requieren `WHATSAPP_PROVIDER=local` (`AdminWhatsAppService.isDevMode()`), así
  que ambos quedan inaccesibles apenas se usa Twilio real (visto 2026-09-29 al activar Twilio en
  Fly, specialist-be#81). Tiene sentido para `simulate-reply` (finge un inbound que con Twilio
  real llega por webhook), pero no para `trigger-followup` ("mandar esta regla ahora" usa el mismo
  path de envío real y sería útil dejarlo disponible con cualquier provider) — separar los dos
  guards.
- Notificar a clientes cuando un proveedor cambia teléfono o email, para los requests activos donde
  participa.
- Tracking de clicks en el botón de contacto por WhatsApp (para decidir follow-up según si hubo
  click sin respuesta).
- Definir qué pasa cuando la respuesta del usuario al follow-up interactivo no matchea ninguna
  opción válida (hoy cae en un template genérico) — ¿límite de reintentos o detección de que está
  confundido?
- Definir el orden de prioridad cuando coexisten un follow-up automático vigente y una
  `SupportConversation` abierta para el mismo teléfono (hoy siempre gana el follow-up).
- Investigar si Twilio/Meta ya manejan opt-out a nivel de plataforma para WhatsApp Business API.
- Evaluar si la feature de IA para follow-up (clasificador LLM, opt-out, attention flags) amerita un
  ADR propio.
- Definir qué implica "el usuario no quiere ser contactado" para el estado del request en sí —
  ¿un flag/estado dedicado más allá de `RequestAttentionFlag`?
- Timestamps por cada cambio de estado de `Request` (hoy solo `createdAt`/`updatedAt`, sin
  historial de transiciones) — requiere migración.

**Limpieza / tests**
- Scripts duplicados en `package.json` (`db:seed` y `prisma:seed` son el mismo comando).
- Tests de integración para permisos; tests E2E de flujos críticos (solicitud directa, solicitud
  pública, moderación de reviews); tests unitarios de `TwilioVerifyService` y del value object
  `Phone`.
- `test/scripts/whatsapp/testing/test-single-followup.ts` no cierra el `NestApplicationContext` al
  terminar (visto 2026-09-29): cada corrida deja un proceso `ts-node` colgado dentro del contenedor
  `especialistas-api-dev`. Liviano (~13s CPU cada uno) pero se acumulan si se corre varias veces
  seguidas para probar Twilio sandbox — falta un `await app.close()` (o similar) al final del
  script.

**Ideas futuras (no MVP)**
- Multi-usuario por empresa (roles), verificación avanzada (AFIP/documentación), transferencia de
  ownership de empresa, dashboard de empresa con métricas, planes de suscripción, portal de
  empleados.
- Performance: revisar N+1 en listados, caché de perfiles públicos, optimizar queries de
  notificaciones.
- Seguridad: rate limiting por endpoint, validación de inputs más estricta, audit log de acciones
  administrativas.
- Mecanismo de soporte in-app (botón "reportar un problema" / chat con admin desde el detalle de un
  request) — evaluar si conviene unificarlo con el soporte por WhatsApp ya existente en vez de tener
  dos canales separados.
- Notificaciones push (web), tiempo real con WebSockets, búsqueda avanzada de especialistas.

---

## 🎨 Frontend (specialist-fe)

- ~~**`npm run lint` roto**~~ **Resuelto** (`specialist-fe` PR #34, `fix/lint-next16`, mergeado):
  reemplazado `next lint` por flat config de ESLint nativo, compatible con Next 16.
- Manejo de 403 al acceder a una solicitud por URL directa: confirmar que el FE muestra un mensaje
  apropiado (hoy el backend ya devuelve el código correcto).
- Revisar completitud de traducciones `es`/`en` (`messages/*.json`) — keys candidatas a faltar,
  a re-verificar (puede estar ya resuelto): `navigation.specialists`,
  `specialist.requestDetail.interestExpressed`, `specialist.requestDetail.interestExpressedDescription`,
  `specialist.requestDetail.removeInterest`.
- UX: loading states consistentes (skeletons en vez de spinners), empty states con
  call-to-action, optimistic updates en acciones frecuentes, toast notifications, animaciones y
  transiciones (menor prioridad).
- Responsive pendiente en ~360px: Job Board, perfil de especialista, formulario de nueva solicitud,
  lista de especialistas interesados.
- Tests: ampliar cobertura de hooks y de componentes críticos; evaluar adoptar Playwright para E2E
  (no hay suite hoy, ni acá ni en `specialist-admin`) — definir alcance inicial (login, crear
  solicitud, expresar interés) y si corre en CI.
- Job Board: filtro por palabra clave (trade o nombre del proveedor), con debounce.
- Company profiles: mostrar tipo de proveedor (Professional/Company) en la lista de interesados;
  página de perfil público de empresa.
- Evaluar eliminar traducciones de quotes (`acceptQuote`, `quote`, `amount`) si no se van a
  implementar.

---

## 🛠️ Admin (specialist-admin)

- Dashboard con métricas reales (hoy placeholder).
- Gestión de usuarios: CRUD completo (hoy solo listar/ver/cambiar estado/confirmar
  email-teléfono). Agregar bloquear y eliminar usuarios. Sacar las acciones rápidas de la grilla de
  usuarios — todas las acciones (incluidas las nuevas) van en la vista de detalle del usuario, no
  en la grilla.
- Gestión de solicitudes y de perfiles Profesional/Empresa (acciones administrativas más allá de
  ver).
- Generar `components/ui/` de shadcn (ya configurado en `components.json`, falta generar).
- Mejorar la vista de detalle de Request (alcance específico por definir). Puntos ya identificados:
  ~~(1) en la grilla de requests, mover la columna status para que quede antes de la columna
  actions~~ **Resuelto 2026-09-28** ([specialist-admin#19](https://github.com/DiegoSana/specialist-admin/pull/19),
  mergeado); ~~(2) en la vista de detalle, la visualización de imágenes se ve cortada (arreglar el
  layout/crop)~~ **Resuelto 2026-09-29** ([specialist-admin#20](https://github.com/DiegoSana/specialist-admin/pull/20)):
  miniaturas ahora en `aspect-square` sin recorte fijo, detección de video en `request.photos`
  (incluye `.mov`) con componente `AuthenticatedVideo` nuevo, y modal de vista completa con
  navegación prev/next, construido sobre el primer primitivo shadcn del repo (`Dialog`/`Button`,
  antes `components/ui/` no existía). Validado localmente por el usuario contra backend + seed
  reales; (3) agregar la posibilidad de bloquear una imagen individual desde esa vista (definir si
  el flag vive en backend o es solo UI de admin) sigue pendiente, alcance separado.
- Evaluar qué tan complejo es hacer el admin mobile responsive (hoy no está pensado para mobile).
- `DEPLOYMENT.md` (raíz) no documenta el deploy de `specialist-admin` en Vercel — solo tiene el
  paso a paso de `specialist-fe`. Agregar la sección equivalente (root directory
  `specialist-admin`, `NEXT_PUBLIC_API_URL` con `/api` incluido).

---

## 📦 Shared (specialist-shared)

Ver "Futuro de `specialist-shared`" en [🎯 Decisión / diseño pendiente](#-decisión--diseño-pendiente).
