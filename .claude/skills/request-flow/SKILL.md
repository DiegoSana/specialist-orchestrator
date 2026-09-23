---
name: request-flow
description: Domain knowledge for the Request lifecycle ("Estados del pedido") — the 15-state RequestStatus machine spanning specialist-be and specialist-fe, who owns each transition, where each piece lives per repo, and known gaps. Use whenever a task touches a Request's status, the client/specialist dashboards or detail pages, WhatsApp follow-ups tied to a request, or anything that says "estado del pedido" / "request status".
---

# request-flow

Domain map for the Request state machine, not an implementation how-to (for that, use
`specialist-be`'s own skills — `add-entity-field`, `add-follow-up-rule`, etc. — once you're
working inside that repo). Read this first so you don't have to re-derive the shape of the
system from scratch every time this area comes up.

## The state machine

15 states, defined once in `specialist-be/prisma/schema.prisma` (`enum RequestStatus`, ~line 404)
and mirrored in `specialist-fe/types/index.ts`. Full spec:
`specialist-be/docs/architecture/EspecialistBRC — Estados del pedido.md`.

Main path: `DRAFT → PUBLISHED (bolsa) | SENT (directo) → CONTACT_RELEASED → IN_PROGRESS →
FINISHED → CLOSED`, with `UNDER_REVIEW` a transitory support-handled detour off `FINISHED`.
Terminal alternates that never reach `CLOSED`: `EXPIRED`, `NO_RESPONSE`, `REJECTED`, `CANCELLED`,
`NOT_COMPLETED`, `INTERRUPTED`, `ABANDONED`.

Interests (a Request can have multiple interested providers before one is chosen) have their own
sub-states — see the domain entities under `specialist-be/src/requests/domain/entities/`.

Actors that can move a status: `Client`, `Provider` (Professional/Company), `Sistema` (the
`RequestExpirationJob` cron, flag `REQUEST_EXPIRATION_ENABLED`), `Soporte` (resolves
`UNDER_REVIEW`). "Quién puede mover cada cosa" table lives in the spec doc above — it was updated
during the rollout to diverge from the original doc in one place: the **client**, not just the
provider, can report `IN_PROGRESS → INTERRUPTED` (matches the FE handoff brief, not the original
spec).

## Where each piece lives

**specialist-be** (`src/requests/`):
- `domain/entities/` — status transitions as entity methods (`canXxxBy(ctx)` auth checks per
  `add-entity-field`/architecture conventions), interest sub-states.
- `application/jobs/` — `RequestExpirationJob` (auto-expiry, off by default).
- `application/follow-up/rules/` — WhatsApp follow-up ladder per status (P1-P3 reply mapping).
  For the follow-up system itself (ladders, scheduler, response→status mapping), use the
  `follow-ups` skill; for the step-by-step of adding one rule, `specialist-be`'s own
  `add-follow-up-rule` skill.
- `presentation/dto/` — response DTOs; `statusReason` and `interestsCount` were added here in the
  rollout (PR #65).
- Contact info gating: `canViewCounterpartContactBy` — phone/whatsapp only visible from
  `CONTACT_RELEASED` onward. Any endpoint returning interest/participant data must respect this
  (a pre-rollout leak in `GET /requests/:id/interests` was closed for exactly this reason).

**specialist-fe** (`lib/`, `components/requests/`):
- `lib/request-status.ts` — **single source of truth** for presentation: label, badge, ball-owner,
  tab-bucket, timeline-step, primary-action per status (`REQUEST_STATUS_META`). Never switch on
  the raw enum in a component; derive from this module instead. Labels live in
  `messages/{es,en}.json` under `requestStatus`.
- `lib/request-participants.ts` — counterpart contact extraction (respects the BE's contact
  gating).
- `components/requests/request-timeline.tsx` — 4-step post-contact timeline
  (`CONTACT_RELEASED → IN_PROGRESS → FINISHED → CLOSED`), matches `TIMELINE_STATUSES` in
  `request-status.ts`.
- Both dashboards (client/specialist) use three tabs derived from `RequestBucket`: "Te toca a
  vos" (`yours`) / "Esperando" (`waiting`) / "Cerrados" (`closed`, with a collapsed `final` strip
  for no-agreement terminal states).
- "Republish" is a plain `POST /requests` with copied data — there is **no** backend republish
  endpoint, intentionally.

**specialist-admin**: only reads `RequestStatus` in a couple of admin pages
(`app/admin/requests/`, `hooks/use-requests.ts`); has **not** been updated for the 15-state model
— its request list/filter still assumes the old flat 5-state enum
(`PENDING/ACCEPTED/IN_PROGRESS/DONE/CANCELLED`). This is a known, deliberately out-of-scope gap
(flagged in root `TODO.md`), not a bug to silently fix — confirm with the user before touching it.

## Known gaps / deferred (don't assume these are done)

- `specialist-admin`'s status filter (see above) — still old 5-state.
- `IN_PROGRESS → ABANDONED` transition intentionally unmapped ("por definir" in the spec doc).
- The "Crear Solicitud" screen (bolsa vs. directo choice) was never redesigned for this model.
- No state-entry timestamp exists. Follow-up ladders use `Request.updatedAt`, which any unrelated
  save resets — a documented limitation, not a bug to fix reflexively.
- WhatsApp templates still need Meta/Twilio approval before they work for real sends.

## Local dev gotcha

Seeding the `especialistas` dev Postgres DB via `docker exec especialistas-api-dev npm run
db:seed` (Alpine/musl container) reliably `SIGSEGV`s on the first Prisma write. Workaround: run
`npm run db:seed` from the host (or a worktree) with `DATABASE_URL` pointed at
`localhost:5432/especialistas` instead. Also note the dev Postgres container hosts two DBs
(`specialistas` and `especialistas`) — `docker-compose.dev.yml`'s `app` service connects to
`especialistas`, but the committed `.env`'s `DATABASE_URL` targets `specialistas` — a pre-existing
naming mismatch, unrelated to this feature.

## Cross-repo consistency check

Before delegating or finishing any change here, `RequestStatus` in
`specialist-be/prisma/schema.prisma` and `specialist-fe/types/index.ts` must stay in lockstep —
if you add/rename/remove a value on one side, `request-status.ts`'s `REQUEST_STATUS_META` (every
key, it's a `Record<RequestStatus, ...>` so TS will fail to compile if a key is missing) and the
message catalogs need the matching update. If the change plausibly spans both repos, use
`orchestrate-feature` rather than editing both repos ad hoc.
