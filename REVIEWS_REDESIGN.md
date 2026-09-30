# Rediseño del sistema de reviews/ratings

Estado: **plan aprobado, pendiente de ejecución**. Branch único a usar en todos los repos
tocados: `feat/bidirectional-reviews`.

Este documento es la especificación completa para implementar el rediseño. Un agente que lo lea
no necesita más contexto que este archivo + el `CLAUDE.md` de cada repo que le toque.

## 1. Objetivo

Hoy el sistema de calificaciones es asimétrico:

- **Cliente → Especialista**: pasa por `ReviewEntity` (`specialist-be/src/reputation/`), con
  moderación de admin (PENDING/APPROVED/REJECTED) y agrega a `ServiceProvider.averageRating` /
  `totalReviews`.
- **Especialista → Cliente**: son dos campos sueltos `clientRating`/`clientRatingComment` en el
  `Request` (`request.entity.ts:150-151`), **sin moderación, sin tabla propia, sin ningún
  agregado a nivel de cliente**. No existe siquiera un modelo `Client` — la identidad del cliente
  es solo `User`.
- Ninguna automatización empuja a calificar — ambos flujos son 100% auto-iniciados desde el
  dashboard, sin recordatorio de WhatsApp (por eso se puede dar el caso "el cliente calificó, el
  especialista nunca entró a hacerlo").

Se va a rediseñar a un modelo simétrico y estándar de marketplace (Uber/Airbnb/Upwork):
calificación bidireccional obligatoria al cerrar un request, ambas direcciones moderadas, **doble
ciego con reveal simultáneo o por timeout**, agregados de rating tanto para especialista como para
cliente, y comentarios destacados curados por admin.

## 2. Decisiones tomadas (con el usuario, 2026-09-30)

| Decisión | Elegido |
|---|---|
| Dónde mostrar rating+comentarios del cliente | **Sin página de perfil propia** — solo en contexto, donde un especialista ya ve la solicitud/interés hoy (detalle de request, listado de intereses) |
| Reveal | **Doble-ciego con timeout** — ninguna parte ve la calificación de la otra hasta que ambas calificaron o vence un timeout |
| Selección de comentarios destacados | **Curado por admin** — flag `isFeatured` togglable desde la UI de moderación existente, no algoritmo automático |
| Datos legacy (`clientRating`/`clientRatingComment` en `Request`) | **Backfill como `APPROVED`** — migrar a la nueva tabla con `revealedAt = createdAt` (ya estaban reveladas), para no arrancar el agregado de cliente en cero |

Además incluido en este plan: fix del bug preexistente donde el modal de perfil de especialista
**no muestra reviews para `Company`**, solo para `PROFESSIONAL`.

## 3. Estado actual (referencias de código, no re-derivar)

- `Review` model: `specialist-be/prisma/schema.prisma:247-266`. Campos: `id, reviewerId,
  serviceProviderId, requestId (unique, 1:1 con Request), rating, comment?, status
  (@default(PENDING)), moderatedAt, moderatedBy, createdAt, updatedAt`. `ReviewStatus` enum en
  `:268-272` (PENDING/APPROVED/REJECTED).
- `Request.clientRating`/`clientRatingComment`: `schema.prisma:208-209` (comentario arriba: "Client
  rating by provider (after work is done)", línea 207). Back-relation `review Review?` en `:218`.
- `ServiceProvider.averageRating`/`totalReviews`: `schema.prisma:82-83`, ya agregadas — único
  write-path es `ReviewService.approve()` → `updateServiceProviderRating()`
  (`src/reputation/application/services/review.service.ts:208-210`). No tocar este mecanismo para
  especialistas, solo filtrarlo por dirección cuando se generalice `Review`.
- Moderación: `ReviewService.approve()`/`.reject()` (`review.service.ts:197-259,261+`), gate
  `ReviewEntity.canBeModeratedBy(ctx)` (`review.entity.ts:180`), endpoints admin-only
  `GET /reviews/admin/pending`, `POST /reviews/:id/approve`, `POST /reviews/:id/reject`
  (`reviews.controller.ts:137-193`).
- `POST /reviews` (cliente→especialista): `reviews.controller.ts:36-52`,
  `ReviewService.create` (`review.service.ts:132-191`) exige `reviewer.hasClientProfile` y
  `request.clientId === reviewerId`.
- `POST /requests/:id/rate-client` (especialista→cliente): `requests.controller.ts:459-486`,
  `RequestService.rateClient` (`request.service.ts:492-527`). Gate:
  `RequestEntity.canRateClientBy(ctx)` (`request.entity.ts:363-374`) exige `isClosed()`,
  `clientRating === null` (una sola vez), `isAssignedProvider(ctx)`.
- Ambas solo posibles con `Request.status === CLOSED` — `canBeReviewed()` en `request.entity.ts:249`.
- **UI de moderación ya existe** en `specialist-admin`: `app/admin/reviews/page.tsx` (tabla
  reviewer/provider/rating/comentario/fecha + Approve/Reject) + `hooks/use-reviews.ts`
  (`usePendingReviews`, `useApproveReview`, `useRejectReview`). Endpoints reales bajo
  `/reviews/admin/*`, no `/admin/reviews/*` (ver `specialist-admin/.claude/rules/01-data-fetching.md`
  y `specialist-shared/src/contracts/admin.contract.ts:73-85`).
- Perfil de especialista (modal, no página): `specialist-fe/components/providers/
  provider-detail-modal.tsx`. Muestra `averageRating`/`totalReviews` (~líneas 123-138) y lista de
  reviews **solo si `provider.type === 'PROFESSIONAL'`** (comentario "Reviews - Only show for
  professionals", ~línea 161) — **este es el bug a corregir**: `ServiceProvider.reviews` ya
  incluye reviews de `Company` también (la entidad padre es la misma), el filtro de tipo es
  innecesario. Lista bloqueada/oculta para usuarios no logueados (~línea 178+).
- Rating de cliente hoy: solo visible privadamente en el detalle del request vía
  `ReceivedRatingCard` (`components/requests/received-rating-card.tsx`), usado en
  `client/requests/[id]/page.tsx:144-148` y `specialist/requests/[id]/page.tsx:119-142`.
- **No existe perfil público de cliente** — no hay ruta `/client/[id]` ni equivalente. Consistente
  con la decisión de no crear una página nueva.
- `RequestExpirationJob` (patrón de referencia para el nuevo job de reveal):
  `src/requests/application/jobs/request-expiration.job.ts:47-54`, `@Cron`, gateado por env flag,
  días configurables por env var.
- `specialist-shared/src/contracts/admin.contract.ts:73-85` — únicas referencias a
  review/moderación en shared; no hay tipos `Review`/`Rating` en otro lado de shared.
- `TODO.md` no tiene ítems pendientes de moderación-UI (ya existe); sí referencias a E2E
  (`review-moderation.spec.ts`, líneas 350-372) y a que `Review` no tiene `onDelete: Cascade` desde
  `Request` (líneas 244-250) — tenerlo en cuenta al migrar el schema.

## 4. Diseño técnico

### 4.1 Schema (`specialist-be/prisma/schema.prisma`)

- Nuevo enum `ReviewDirection { CLIENT_TO_PROVIDER PROVIDER_TO_CLIENT }`.
- Generalizar `Review` para soportar ambas direcciones. La forma exacta de las FKs (mantener
  `serviceProviderId` nullable + agregar `revieweeUserId` para el caso `PROVIDER_TO_CLIENT`, vs.
  generalizar del todo a un `revieweeUserId` único y resolver `ServiceProvider` por join) queda a
  criterio del agente que implemente, siguiendo las convenciones de `add-entity-field` — pero el
  **contrato de salida** (sección 4.2) no puede cambiar sin reportarlo.
- Unique constraint pasa de `requestId` (1:1) a `(requestId, direction)` — ahora puede haber hasta
  2 reviews por request (una por dirección).
- Agregar `revealedAt DateTime?` (null = oculta hasta que ambas partes calificaron o vence el
  timeout) e `isFeatured Boolean @default(false)`.
- Agregar a `User`: `clientAverageRating Float @default(0)`, `clientTotalReviews Int @default(0)`.
  No tocar `ServiceProvider.averageRating/totalReviews` — solo filtrar su trigger de actualización
  por `direction: CLIENT_TO_PROVIDER`.
- Confirmar/agregar `onDelete: Cascade` de `Review` → `Request` al generalizar (gap ya anotado en
  `TODO.md:244-250`).
- **Migración de datos** (script de migración, no a mano): por cada `Request` con
  `clientRating IS NOT NULL`, crear `Review(direction: PROVIDER_TO_CLIENT, rating: <valor>,
  comment: <clientRatingComment>, status: APPROVED, revealedAt: <Request.updatedAt o createdAt>,
  moderatedBy: null)`, y recalcular `User.clientAverageRating/clientTotalReviews` para cada
  cliente afectado a partir de las reviews migradas. Mantener las columnas viejas en `Request`
  por compatibilidad de lectura histórica, pero dejar de escribirlas desde el código nuevo.

### 4.2 Endpoints y contrato

- `POST /requests/:id/rate-client` — mismo path/método (no romper el hook `useRateClient` del
  FE más de lo necesario), pero pasa a crear una `Review(direction: PROVIDER_TO_CLIENT)` con
  `status: PENDING` en vez de escribir los campos planos. Pasa por moderación igual que
  `POST /reviews`.
- `POST /reviews` (cliente→especialista, sin cambios de firma) — la dirección se infiere del rol
  del caller en el backend, **nunca** viene del body (evita spoofing de dirección).
- Moderación (`/reviews/admin/approve|reject`) extendida con acción opcional para togglear
  `isFeatured` (puede ser parámetro del mismo endpoint de approve, o un endpoint nuevo
  `POST /reviews/:id/feature` — a criterio del agente BE, documentarlo en el reporte final).
- **Reveal**: nuevo `RevealReviewsJob`, mismo patrón que `RequestExpirationJob` (cron +
  feature flag `REVIEW_REVEAL_ENABLED`, off por default como el otro job). Timeout configurable
  vía `REVIEW_REVEAL_TIMEOUT_DAYS` (default propuesto: 14, como Airbnb). Revela ambas reviews de
  un request cuando: (a) las dos están `APPROVED`, o (b) vence el timeout desde que se envió la
  primera.
- DTO de detalle de request: reemplazar los campos planos `clientRating`/`clientRatingComment`
  expuestos por algo simétrico, p. ej. `myReview` / `counterpartReview` (nombres finales a
  confirmar por el agente BE y reportar). El contenido de `counterpartReview` debe venir oculto
  (`null` o con un marcador `pending: true`) si `revealedAt` es `null`, incluso para el propio
  autor de la otra review — pero el autor **sí** debe poder ver que su propia review ya fue
  enviada (sin exponer el contenido de la del otro).
- DTO de request/interés visto por el especialista (antes de aceptar, y en el detalle): incluir
  `client: { averageRating, totalReviews, featuredReviews: [...] }` para alimentar la vista "en
  contexto" decidida en la sección 2 — no se crea endpoint de perfil de cliente nuevo.
- **Nudge**: extender el sistema de follow-up existente
  (`src/requests/application/follow-up/rules/`) para notificar a ambas partes por WhatsApp cuando
  el request llega a `CLOSED` y aún no calificaron — hoy no existe ningún recordatorio (grep
  confirmado sobre templates y reglas, sin resultados para "reseña"/"review"/"calificar").

### 4.3 Bug fix incluido: modal de especialista no muestra reviews de `Company`

En `specialist-fe/components/providers/provider-detail-modal.tsx` (~línea 161), quitar el filtro
que condiciona la lista de reviews a `provider.type === 'PROFESSIONAL'`. `ServiceProvider.reviews`
ya es la relación padre compartida por `Professional` y `Company`, así que no hace falta ningún
cambio de backend — es puramente un fix de condicional en el componente. Verificar también
`professionals/page.tsx:180-197` (cards del catálogo) por si tiene el mismo filtro indebido.

## 5. Plan de ejecución por repo (orden de dependencia)

### 5.1 `specialist-be` (primero, todo lo demás depende de esto)

- Migración de schema (4.1) + script de backfill de datos legacy.
- Generalizar `ReviewEntity`/`ReviewService` para ambas direcciones, moderación, `isFeatured`.
- Nuevo `RevealReviewsJob` (4.2).
- Actualizar `POST /requests/:id/rate-client` para crear `Review` en vez de escribir campos planos.
- Actualizar DTOs de detalle de request e interés (4.2).
- Extender reglas de follow-up con el nudge de ambas partes al llegar a `CLOSED`.
- Actualizar `docs/API.md` y `docs/guides/PERMISSIONS_BY_ROLE.md` con el nuevo contrato.
- DoD: tests + lint + build verdes, migración corrida en dev, `architecture.spec.ts` (fitness
  functions) en verde.
- **Reportar el shape final real** (nombres de campos, endpoints, cualquier desvío de este plan)
  — lo siguiente se delega con eso, no con lo planeado acá.

### 5.2 `specialist-shared` (después de BE, antes de admin)

- Actualizar `admin.contract.ts` (o nuevo `reputation.contract.ts`) con `direction`, `isFeatured`,
  `revealedAt` en la forma de respuesta de moderación, usando el shape real reportado por BE.
- **Build + commit `dist/` + push directo a `main`** — única excepción de "no pushear hasta el
  final" (ya documentada en `CLAUDE.md` del orquestador), porque `specialist-admin` no puede
  consumir el cambio de otra forma.

### 5.3 `specialist-fe` (paralelo con admin, después de shared no aplica — fe no depende de shared)

- Adaptar `useRateClient`/`ReceivedRatingCard` al nuevo shape (`myReview`/`counterpartReview`,
  estado "pendiente de revelar").
- Vista de especialista: mostrar `clientAverageRating/clientTotalReviews` + reviews `isFeatured`
  del cliente en el detalle del request / listado de intereses.
- Fix del bug del modal (4.3).
- DoD: lint + build verdes (fe no tiene enforcement de tests tan estricto como be, pero correr lo
  que haya).

### 5.4 `specialist-admin` (paralelo con fe, después de shared pusheado)

- `/admin/reviews`: columna `direction`, toggle "Destacar" (`isFeatured`), usando los tipos
  actualizados de `@specialist/shared`.
- DoD: build verde (sin test suite).

## 6. Fuera de alcance (explícitamente, para no scope-creep)

- Página de perfil público de cliente (descartada en la decisión de la sección 2).
- Bloqueo funcional de otras acciones hasta que ambas partes califiquen ("obligatorio" acá
  significa recordatorio activo vía WhatsApp, no un gate que impida crear nuevos requests).
- Cambios a `ServiceProvider.averageRating`/`totalReviews` más allá de filtrar por dirección.
- E2E (`specialist-e2e`) — no incluido en este plan; su `review-moderation.spec.ts` existente
  probablemente necesite ajustes después, pero es un seguimiento aparte.
