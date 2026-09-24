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

**Notificaciones / WhatsApp**
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

- **`npm run lint` roto** (encontrado 2026-09-24): Next 16.0.10 sacó el subcomando `next lint`
  (`npx next --help` ya no lo lista), y `eslint-config-next` sigue en `^15.0.0` — el `CLAUDE.md` ya
  documentaba el desalineamiento de versiones como gotcha, pero ahora el script directamente falla
  ("Invalid project directory provided"). Workaround usado mientras tanto: `npx eslint <archivo>`
  directo. Definir reemplazo real (flat config de ESLint, o el paquete que Next 16 recomienda).
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
  email-teléfono).
- Gestión de solicitudes y de perfiles Profesional/Empresa (acciones administrativas más allá de
  ver).
- Generar `components/ui/` de shadcn (ya configurado en `components.json`, falta generar).
- Mejorar la vista de detalle de Request (alcance específico por definir).
- Evaluar qué tan complejo es hacer el admin mobile responsive (hoy no está pensado para mobile).
- `DEPLOYMENT.md` (raíz) no documenta el deploy de `specialist-admin` en Vercel — solo tiene el
  paso a paso de `specialist-fe`. Agregar la sección equivalente (root directory
  `specialist-admin`, `NEXT_PUBLIC_API_URL` con `/api` incluido).

---

## 📦 Shared (specialist-shared)

Ver "Futuro de `specialist-shared`" en [🎯 Decisión / diseño pendiente](#-decisión--diseño-pendiente).
