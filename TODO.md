# 📋 Specialist — Todo & Roadmap (global)

> Backlog vivo, no historial. Qué se hizo, cuándo y en qué PR vive en el `git log`/PRs mergeados de
> cada repo (y en el `git log` de este propio archivo, si hace falta reconstruir una versión vieja
> con contexto) y en la memoria del proyecto. Acá solo se lista lo que sigue **pendiente** — los
> ítems ya resueltos se sacan de este archivo en vez de quedar tachados, para que siga siendo
> rápido de leer.

Vive en `/var/www/specialist/` (el directorio padre, no es parte de ninguno de los cuatro repos) y
es el único TODO/roadmap del proyecto. El skill `orchestrate-feature`
(`.claude/skills/orchestrate-feature/`) lo lee en su paso de investigación al planificar un
requerimiento cross-repo. El skill `session-recap` de `specialist-be`
(`specialist-be/.claude/skills/session-recap/`) sigue siendo el mecanismo para cerrar una sesión de
trabajo del backend y debe actualizar **este** archivo (no uno local); las otras secciones se
actualizan a mano.

## Índice

- [🚦 Bloqueantes para lanzamiento (MVP)](#-bloqueantes-para-lanzamiento-mvp)
- [🆕 Sin investigar](#-sin-investigar)
- [🎯 Decisión / diseño pendiente](#-decisión--diseño-pendiente)
- [🔧 Backend (specialist-be)](#-backend-specialist-be)
- [🎨 Frontend (specialist-fe)](#-frontend-specialist-fe)
- [🛠️ Admin (specialist-admin)](#️-admin-specialist-admin)
- [📦 Shared (specialist-shared)](#-shared-specialist-shared)
- [🧪 E2E (specialist-e2e)](#-e2e-specialist-e2e)

---

## 🚦 Bloqueantes para lanzamiento (MVP)

Vista curada de lo que un relevamiento de producto (2026-10-01) identificó como necesario antes de
abrir la app a usuarios reales — no son features nuevas, son cierres sobre lo que ya existe. Cada
ítem vive en detalle en su sección de abajo; esto es solo el resumen priorizado.

1. **Aprobar templates de WhatsApp en Meta/Twilio para producción.** Todo el flujo de seguimiento
   del pedido (liberación de contacto, avisos de cierre/calificación, etc.) depende de templates
   que hoy solo están habilitados para testing. No es tarea de código — gestión externa en Meta
   Business Manager/Twilio — pero sin esto el flujo central del producto no funciona con usuarios
   reales. Ver Backend → Notificaciones/WhatsApp.
2. **Contenido legal con placeholders reales por completar.** Email de contacto y razón social en
   T&C/Contacto siguen siendo placeholders de etapa MVP — ver "🎯 Decisión / diseño pendiente".
3. **Validación de Company sin definir** (CUIT/AFIP/documentación). Si las empresas van a operar
   como proveedores desde el día uno, hoy no hay ninguna verificación de que existan realmente —
   ver "🎯 Decisión / diseño pendiente".
4. **Rate limiting ausente en toda la API**, incluyendo login/registro y el webhook público de
   WhatsApp (sin JWT). Riesgo de abuso día uno, no deuda técnica de largo plazo — ver Backend →
   Seguridad.

---

## 🆕 Sin investigar

Pedidos directos del usuario, todavía no investigados — candidatos para `orchestrate-feature`.

(Vacío por ahora.)

---

## 🎯 Decisión / diseño pendiente

- **Auditoría de usuarios y perfiles + flujos de estado.** Entregable: documento de hallazgos, no
  código (puede incluir los diagramas de estado de `Request` que faltan). Cubrir la relación `User`
  ↔ `Professional`/`Company`/`Client`, los estados de cada perfil y su cruce con
  `emailVerified`/`phoneVerified` y `ProfileActivationService`/`canOperate()`. Ver
  `specialist-be/docs/architecture/PROFILE_ACTIVATION_ORCHESTRATION.md` y ADR-006
  (`docs/decisions/ADR-006-REQUEST-STATE-MACHINE.md`).
- **Validaciones extra de Company.** Definir primero qué se exige (CUIT, AFIP, documentación) antes
  de implementar — bloqueante para lanzar si las empresas operan como proveedores desde el día uno
  (ver "🚦 Bloqueantes para lanzamiento").
- **Futuro de `specialist-shared`.** Solo `specialist-admin` lo consume, y solo para
  `loginSchema`/`LoginDTO`/`AuthResponse` — el resto (`AdminContract`, `User`, etc.) no tiene uso
  real hoy. Opciones: (1) adoptarlo de verdad (requeriría que `specialist-fe` también lo use), (2)
  encogerlo a solo lo que se usa, (3) eliminarlo y mover esas pocas líneas a `specialist-admin`
  (recomendada si solo admin lo sigue consumiendo), (4) generar tipos desde el OpenAPI del backend
  en vez de mantener un espejo manual.
- **Email de contacto y razón social de `specialist-fe` siguen siendo placeholders** (T&C/Contacto,
  `specialist-fe`#39, 2026-09-29) — definir el canal real y la razón social, y decidir si sumar
  Política de Privacidad (no estaba en el alcance original). Bloqueante para lanzar — ver "🚦
  Bloqueantes para lanzamiento".

---

## 🔧 Backend (specialist-be)

**Seguridad**
- **Rate limiting ausente en toda la API**, incluyendo login/registro y el webhook público de
  WhatsApp (`POST /api/webhooks/twilio`, sin JWT). Bloqueante antes de exponer la app a tráfico
  real — ver "🚦 Bloqueantes para lanzamiento". (Validación de inputs más estricta y audit log de
  acciones administrativas siguen como mejoras post-MVP, ver "Ideas futuras" abajo.)

**Bugs / permisos**
- Verificar acceso a solicitudes completadas mostradas en perfiles públicos de otros especialistas.
- Revisar validación de permisos en fotos de solicitudes (¿son públicas las de un trabajo
  completado? ¿quién ve las de uno en progreso?).
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

**Notificaciones / WhatsApp**
- **Aprobar templates de WhatsApp en Meta/Twilio para producción** (hoy solo habilitados para
  testing) — ver "🚦 Bloqueantes para lanzamiento".
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
- Revisar `specialist-be/src/identity/infrastructure/verification/twilio-verify.service.ts`: hace
  referencia directa a Twilio en vez de estar abstraído detrás de una interfaz de provider
  (acoplamiento innecesario a un proveedor concreto de verificación/WhatsApp).
- **`NotificationDispatchService.dispatchPending()` nunca implementó el branch de WhatsApp**
  (`dispatch-service.ts`, solo despacha `EMAIL` — el de `WHATSAPP` es un comentario
  `// WhatsApp pending dispatch comes later.`), a pesar de que `NotificationChannel`/
  `ExternalNotificationChannel` ya incluyen `WHATSAPP` desde hace tiempo. La notificación de
  "nuevo pedido matchea tu rubro" (`REQUEST_MATCHING_TRADE_CREATED`, `specialist-be`#100,
  2026-10-01) necesitaba WhatsApp real y no pudo esperar a que esto se completara, así que manda
  el mensaje **directo** contra `WhatsAppMessagingPort` desde el propio handler
  (`requests-notifications.handler.ts`), salteando el pipeline genérico — deuda técnica deliberada
  y marcada en el código. Cuando se implemente el dispatch real de WhatsApp acá, migrar ese envío
  para que pase por el pipeline genérico como cualquier otro canal.

**Limpieza / tests**
- Scripts duplicados en `package.json` (`db:seed` y `prisma:seed` son el mismo comando).
- Tests de integración para permisos y tests unitarios de `TwilioVerifyService`/`Phone` (el resto —
  E2E de flujos críticos: solicitud directa, pública, moderación de reviews — ya está cubierto, ver
  "🧪 E2E").
- **Falta el endpoint de limpieza dev-only para `specialist-e2e`**: nuevo endpoint (admin-only +
  gateado por env var `E2E_TEST_UTILS_ENABLED`, nunca seteada en Fly/producción) que borre en
  cascada los `Request` cuyo `title` empiece con un prefijo dado (el `Review` asociado ya tiene
  `onDelete: Cascade` desde el rediseño de reviews bidireccional, así que alcanza con borrar el
  `Request` — mismo orden FK-safe que ya usa `test/test-setup.ts#cleanDatabase`). Hasta que exista,
  el `global-teardown.ts` de `specialist-e2e` intenta llamarlo, falla silenciosamente (try/catch) y
  deja los datos `[E2E]` sin limpiar en la DB de dev.
- **El test suite nunca bootea la app real de Nest** (los tests unitarios mockean el wiring de
  módulos) — un `forwardRef` faltante en `ReputationModule`/`ReviewService` pasó 851 tests y el
  build del PR de reviews bidireccional sin ser detectado, y solo se encontró al levantar el
  backend de verdad para correr E2E. Evaluar agregar un smoke test liviano que compile el
  `AppModule` completo (`Test.createTestingModule({imports: [AppModule]}).compile()`) a la suite,
  para agarrar este tipo de error de DI/import circular antes de mergear.
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
- Seguridad: validación de inputs más estricta, audit log de acciones administrativas (el rate
  limiting ya no es "futuro" — ver sección Seguridad arriba).
- Mecanismo de soporte in-app (botón "reportar un problema" / chat con admin desde el detalle de un
  request) — evaluar si conviene unificarlo con el soporte por WhatsApp ya existente en vez de tener
  dos canales separados.
- Notificaciones push (web), tiempo real con WebSockets, búsqueda avanzada de especialistas.

---

## 🎨 Frontend (specialist-fe)

- Manejo de 403 al acceder a una solicitud por URL directa: confirmar que el FE muestra un mensaje
  apropiado (hoy el backend ya devuelve el código correcto).
- Revisar completitud de traducciones `es`/`en` (`messages/*.json`) — keys candidatas a faltar,
  a re-verificar (puede estar ya resuelto): `navigation.specialists`,
  `specialist.requestDetail.interestExpressed`, `specialist.requestDetail.interestExpressedDescription`,
  `specialist.requestDetail.removeInterest`.
- UX: loading states consistentes (skeletons en vez de spinners), empty states con
  call-to-action, optimistic updates en acciones frecuentes, toast notifications, animaciones y
  transiciones (menor prioridad).
- **Responsive roto a ~360px** en pantallas core: Job Board, perfil de especialista, formulario de
  nueva solicitud, lista de especialistas interesados — si el canal principal es mobile, afecta el
  flujo central, no una pantalla secundaria.
- Tests: ampliar cobertura de hooks y de componentes críticos.
- Job Board: filtro por palabra clave (trade o nombre del proveedor), con debounce.
- Company profiles: mostrar tipo de proveedor (Professional/Company) en la lista de interesados;
  página de perfil público de empresa (hoy solo existe la pantalla de setup, no una vista pública).
- Evaluar eliminar traducciones de quotes (`acceptQuote`, `quote`, `amount`) si no se van a
  implementar.
- **Test roto en `main`, preexistente**: `hooks/__tests__/use-reviews.test.tsx` →
  `useReviewByRequestId › should return null when no review exists (404)` falla de forma
  determinística (no es flaky, falla igual corriéndolo solo). Causa: `useReviewByRequestId`
  (`hooks/use-reviews.ts:100-109`) no atrapa el 404 en su `queryFn` — el query queda en `isError`
  en vez de resolver `isSuccess` con `data: null` como espera el test. Fix de una línea
  (`try/catch` alrededor del `apiClient.get`), sin relación con ninguna feature en curso —
  detectado 2026-10-01 corriendo la suite completa antes de un commit no relacionado.
- **Cerrar `/professionals` (hoy público, sin auth guard) detrás de login para el MVP**, igual que
  el resto de la app. Decisión PO 2026-10-01: dejar todo privado salvo el landing hasta sumar
  usuarios — mostrar el directorio de profesionales vacío o con pocos registros da peor primera
  impresión que pedir registro antes de navegar. Reabrir como directorio público cuando haya masa
  crítica de oferta (umbral propuesto: ~15-20 profesionales activos en alguna categoría), ya que
  ahí sí conviene el SEO/descubrimiento orgánico que hoy se resigna a propósito.

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
- Vista de detalle de Request: agregar la posibilidad de bloquear una imagen individual (definir si
  el flag vive en backend o es solo UI de admin).
- Evaluar qué tan complejo es hacer el admin mobile responsive (hoy no está pensado para mobile).
- `DEPLOYMENT.md` (raíz) no documenta el deploy de `specialist-admin` en Vercel — solo tiene el
  paso a paso de `specialist-fe`. Agregar la sección equivalente (root directory
  `specialist-admin`, `NEXT_PUBLIC_API_URL` con `/api` incluido).
- ~~En Admin Settings debe figurar información de cómo está configurado el envío de mensajes por
  WhatsApp, tanto para follow-up como para soporte.~~ **Resuelto 2026-09-29**
  (`specialist-admin` [#21](https://github.com/DiegoSana/specialist-admin/pull/21), mergeado, con
  companion `specialist-be` [#83](https://github.com/DiegoSana/specialist-be/pull/83)): card
  "WhatsApp delivery" en `/admin/settings` muestra el provider activo (Twilio/local), el número de
  origen y un aviso si sigue siendo el número de sandbox por defecto. Un solo provider/número sirve
  tanto a follow-up como a soporte, así que no hace falta desdoblar la vista por canal. De paso
  (`specialist-admin` [#25](https://github.com/DiegoSana/specialist-admin/pull/25) +
  `specialist-be` [#91](https://github.com/DiegoSana/specialist-be/pull/91)) se agregó una segunda
  card análoga para el provider de verificación de teléfono/email (Twilio/local).
- Nueva pantalla en el admin, accesible desde la vista de detalle de usuario, que muestre toda su
  interacción por WhatsApp en orden cronológico: mensajes de los distintos `Request` en los que
  participó, conversaciones de soporte (`SupportConversation`) y cualquier otra comunicación — hoy
  no hay ninguna vista que junte todo eso por usuario, solo se puede ver por request individual.
  Definir de dónde sale la data en specialist-be (probablemente un endpoint nuevo que una
  `RequestInteraction`/`SupportConversation` por teléfono/usuario) antes de encarar el front.

---

## 📦 Shared (specialist-shared)

Ver "Futuro de `specialist-shared`" en [🎯 Decisión / diseño pendiente](#-decisión--diseño-pendiente).

---

## 🧪 E2E (specialist-e2e)

Repo nuevo (2026-09-29), quinto sibling de este directorio (`gh repo create DiegoSana/specialist-e2e
--private`), suite Playwright + TypeScript que cubre los flujos core cruzando `specialist-fe` y
`specialist-admin` contra las cuentas seed fijas de `specialist-be/prisma/seed.ts` (no registra
usuarios nuevos).

**Specs**: `auth.spec.ts`, `create-request-public.spec.ts`, `create-request-direct.spec.ts`,
`job-board-interest.spec.ts`, `match-notification.spec.ts` (2026-10-01, `specialist-e2e`#5: opt-in
"avisame cuando haya un pedido para mí" — un profesional opt-in recibe la notificación in-app
`REQUEST_MATCHING_TRADE_CREATED` al crearse un pedido público que matchea su rubro; opt-in y
aserción van por API directa, sin depender del toggle de FE; busca el id del pedido recién creado
vía `findRequestIdByTitle` en vez de navegar la UI del dashboard — con los datos de E2E acumulados
sin limpiar, esa carrera se puso lo bastante lenta como para superar incluso un
`test.setTimeout(60_000)`) (los cinco sobre `specialist-fe`), `review-moderation.spec.ts` (cruza
`specialist-fe` + `specialist-admin`, smoke test chico de un solo sentido: login → crear solicitud
directa → avanzar hasta `FINISHED` → confirmar (`CLOSED`) → dejar review → aprobar en
`/admin/reviews`), `whatsapp-followup.spec.ts` (cruza `specialist-fe` + `specialist-admin`: ciclo
de vida completo de una solicitud impulsado enteramente por respuestas de WhatsApp simuladas — el
admin fuerza cada regla de seguimiento desde el panel real "Forzar seguimiento", la respuesta del
cliente/proveedor se simula pegándole directo al webhook real `POST /api/webhooks/twilio` —
agnóstico al provider — en vez del endpoint dev-only `simulate-reply`;
`CONTACT_RELEASED → IN_PROGRESS → FINISHED → CLOSED`), `review-bidirectional.spec.ts` (cobertura
del rediseño de reviews bidireccional — ambas partes califican, doble-ciego oculto hasta que admin
aprueba las dos reviews, reveal sincrónico, columna "Dirección" y toggle "Destacar" en
`/admin/reviews`; deja fuera de alcance el timeout de 14 días del doble-ciego y las reviews de
proveedor `Company`, ver "Pendiente" abajo), `password-reset.spec.ts` (2026-10-01: flujo de
"olvidé mi contraseña" de punta a punta — registra un usuario descartable como única excepción
documentada a "nunca registra usuarios" ya que resetear la password de una cuenta seed rompería
specs posteriores en la misma corrida serial, pide el reset, lee el email real desde la API de
Mailpit, extrae el token, resetea y confirma login con la contraseña nueva). Los 9 specs
verificados en verde contra el stack real.

**Pendiente**
- El endpoint de limpieza dev-only en `specialist-be` (ver entrada en la sección Backend arriba)
  todavía no existe — hasta que se implemente, los datos `[E2E]` (requests + su cascada) quedan sin
  borrar en la DB de dev después de cada corrida. Además, ese endpoint solo filtra por prefijo de
  título de `Request`, no por email — los usuarios descartables que crea `password-reset.spec.ts`
  (único spec que registra usuarios) tampoco quedan cubiertos hasta que se extienda.
- CI: todavía no existe ningún workflow de GitHub Actions en el repo (verificado 2026-10-01, no hay
  directorio `.github/` ni en `main` ni en ninguna rama) — falta escribirlo desde cero. Va a
  necesitar, además, un Personal Access Token nuevo que el usuario tiene que crear a mano y cargar
  como secret (`CROSS_REPO_PAT`) para el checkout cruzado de `specialist-be`/`specialist-fe`/
  `specialist-admin` (los tres privados). Por ahora la suite solo corre local, contra las tres apps
  levantadas a mano.
- `review-moderation.spec.ts`, `whatsapp-followup.spec.ts` y `review-bidirectional.spec.ts`
  necesitan `WHATSAPP_PROVIDER=local` en el `specialist-be` contra el que corren (solo para que el
  envío *saliente* de `trigger-followup` no intente mandar un WhatsApp real a un teléfono falso del
  seed — la simulación de respuesta en sí no tiene esa dependencia, ver `specialist-e2e/CLAUDE.md`).
- `review-bidirectional.spec.ts` no cubre el timeout de 14 días del doble-ciego (no testeable sin
  manipular tiempo real, solo se ejercita el reveal sincrónico al aprobar ambas reviews) ni reviews
  de proveedor `Company` (las cuentas seed fijas no incluyen una company fácil de llevar por todo
  el ciclo de vida de un request sin agregar seed data nueva).
- Ampliar cobertura más allá del alcance inicial (registro de usuario nuevo, perfiles de empresa)
  queda para después.
- `match-notification.spec.ts` no verifica el envío de WhatsApp en sí (el handler lo manda directo
  contra el adapter, salteando el pipeline genérico — ver Backend → Notificaciones/WhatsApp), solo
  la notificación in-app; suficiente señal por ahora, pero revisar si vale la pena agregar
  verificación del envío cuando el dispatch genérico de WhatsApp se complete.
