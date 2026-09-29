# Specialist (cross-repo orchestrator)

`/var/www/specialist/` is itself a git repo (`DiegoSana/specialist-orchestrator`, added
2026-09-23) that versions only the orchestration layer — this `CLAUDE.md`, `TODO.md`,
`DEPLOYMENT.md`, `SOCIAL_LOGIN_ARCHITECTURE.md`, `.claude/` (settings + cross-repo skills). It
holds four independent git repos as gitignored subdirectories, each with its own remote, history
and `CLAUDE.md`:

```
specialist-be       NestJS 10 + Prisma + PostgreSQL REST API — the canonical domain model.
                     Deployed on Fly.io. Has architecture fitness functions and per-context
                     skills (add-endpoint, add-entity-field, ...).
specialist-fe        Next.js 16 public web app (clients + professionals + companies).
                     Deployed on Vercel. Port 3001 in dev. Keeps its own local types/index.ts,
                     does NOT consume @specialist/shared.
specialist-admin      Next.js 16 internal admin portal (role: isAdmin only). Port 3000 in dev.
                     Consumes @specialist/shared. No test suite.
specialist-shared    Small hand-written TS package (types/schemas/constants/contracts).
                     Ships via committed dist/ + a raw github: dependency URL, not npm — no
                     monorepo linking, no auto-propagation. Only specialist-admin consumes it,
                     and only after a build+commit+push+reinstall cycle there.
```

`DEPLOYMENT.md` and `SOCIAL_LOGIN_ARCHITECTURE.md` in this directory describe the deployed
topology (Vercel → Fly.io → Supabase) and the OAuth flow across `specialist-fe`/`specialist-be`.

**There is no real production yet** (as of 2026-09-29): `specialist-api.fly.dev` /
`specialist-fe`/`specialist-admin` on Vercel are live, but only for functional testing and
validation with real external integrations (Twilio WhatsApp, OAuth, etc.) — not real end users.
Don't treat `NODE_ENV=production` there as "hands off, real prod" the way you would on an app with
actual users; check with the user before assuming a pre-launch safety gate should stay off on this
deploy. This will stop being true once the app actually launches — ask if unsure whether that's
happened yet.

`TODO.md` in this directory is the **global backlog/roadmap**, one `##` section per repo (moved
here from `specialist-be/TODO.md` and merged with `specialist-fe/TODO.md` on 2026-09-16 — neither
repo has its own anymore). Check it during the investigate step of `orchestrate-feature` for
already-known plans/priorities before planning a requirement from scratch. `specialist-be`'s
`session-recap` skill keeps its "Backend" section current; update the other sections by hand.

## Role of a session started here

A Claude Code session whose primary working directory is `/var/www/specialist/` acts as the
**cross-repo orchestrator**: given a feature/bug requirement, it investigates which repos are
affected, plans the contract between them (API shape, DTO fields, branch name), and **delegates
the actual implementation to one subagent per affected repo** via the `orchestrate-feature` skill,
rather than editing files directly across repos itself.

**For any feature or bug request that plausibly touches more than one repo (almost anything
involving an API change, a new field, or new UI backed by data), invoke the `orchestrate-feature`
skill first.** For a change that is obviously confined to one repo (e.g. "fix this admin table's
column width"), just `cd`/work in that repo directly — don't spin up the orchestration machinery
for single-repo work. That's about not over-engineering cross-repo coordination, though, not about
skipping delegation entirely: a single-repo change that's still roughly PR-sized (schema+entity+
service changes, a multi-file rewrite, a bugfix with tests) should still go to a `fork` with a
self-contained directive (branch name, exact scope, files, test/lint/build gate, commit convention)
rather than being implemented inline in this session — keep inline edits for genuinely small,
exploratory, or tightly interactive work instead.

## Commands

There's no root-level build/test — everything runs per-repo. Prefer these forms over `cd` so a
single Bash call stays self-contained:

```bash
git -C specialist-be <cmd>              # instead of: cd specialist-be && git <cmd>
npm --prefix specialist-fe run lint     # instead of: cd specialist-fe && npm run lint
gh pr create --repo DiegoSana/specialist-fe ...   # --repo, not cwd-based detection (see Gotchas)
```

Each repo's own `CLAUDE.md` has its exact commands (`npm test`, `npm run lint`, `npx prisma
generate`, etc.) — see `specialist-be/CLAUDE.md`, `specialist-fe/CLAUDE.md`,
`specialist-admin/CLAUDE.md`, `specialist-shared/CLAUDE.md`.

## Conventions

- One GitHub identity across all four repos: `DiegoSana` (personal), not `diego-sanabria-azumo`
  (work). Every repo's local `git config user.name`/`user.email` should be `Diego Sanabria` /
  `diegohsanabria@gmail.com` — check with `git -C <repo> config user.email` before committing; if
  it's unset or shows the azumo address, set it locally (never touch the global config, which is
  intentionally the azumo identity for other projects on this machine).
- Before creating a new feature branch (`git checkout -b` or `git worktree add ... -b`) in any of
  the four repos, fetch/pull `main` first and confirm local `main` is current with `origin/main` —
  do this even for single-repo, non-orchestrated changes, not just cross-repo work.
- Use one consistent branch name across every repo touched by a given feature (e.g.
  `feat/portfolio-videos` in `specialist-be` **and** `specialist-fe`) so the work is easy to
  correlate later — there's no monorepo tooling tying them together otherwise.
- If a repo already has uncommitted work on another branch when you need to start a new one, don't
  disturb it: use `git worktree add <tmp-dir> main -b <branch>`, do the work there, commit, then
  `git worktree remove <tmp-dir>`. This is the same trick used to add this harness to
  `specialist-fe` without touching its in-progress feature branch.
- Default to **implement + test/lint/build green + commit locally**, then stop and report before
  pushing or opening PRs — one confirmation point for the whole cross-repo change, not four. Push
  and open PRs only once asked (or if the original request already said to).
- **Exception**: changes to this orchestrator repo itself (`CLAUDE.md`, `TODO.md`,
  `DEPLOYMENT.md`, `.claude/`) push **directly to `main`**, no feature branch, no PR, no
  confirmation round-trip — commit and `git push origin <local-branch>:main` (or `git push` if
  already on `main`) straight away. This only applies to the orchestrator repo's own files; the
  four sibling repos (`specialist-be`/`fe`/`admin`/`shared`) still follow the default above
  (branch, commit locally, confirm before push/PR).

## Gotchas

- **A bare `git status`/`git commit` here now targets the orchestrator repo itself**, not one of
  the four sibling repos — that's usually not what you want; target a specific repo explicitly
  (`git -C specialist-be ...`) unless the change is actually to `CLAUDE.md`/`TODO.md`/`.claude/`.
- **`gh` is unreliable when cwd-detection is involved across these sibling directories** — pass
  `--repo DiegoSana/<name>` explicitly on every `gh pr create`/`gh pr view`/etc. rather than relying
  on it to infer the repo from the working directory (`gh pr create` failed with "not a git
  repository" via `cd`-then-run in this exact layout; `--repo` fixed it). Root cause found
  2026-09-16: `gh` here is installed as a **snap** (`which gh` → `/snap/bin/gh`), confined to only
  the `home`/`network`/`ssh-keys`/`desktop` interfaces (`snap connections gh`) — it has no
  filesystem access to `/var/www` at all, so it can't read local git state there even with the
  right cwd, and reports a misleading "not a git repository (or any parent up to mount point
  /var/lib)". `--repo` alone isn't enough once a PR's branch must be pushed already: also pass
  `--head <branch> --base main` explicitly on `gh pr create` so it never tries to run `git` against
  this directory to detect the current branch — that combination is what actually works. This
  snap confinement is about `gh` specifically, not `git`: `git` itself has full filesystem access
  here, so pushing/pulling the orchestrator repo (`git -C /var/www/specialist push`, no `-C` needed
  when already cwd'd there) works fine — only `gh repo create --source=. --push` would try to shell
  out to `git` under the confined process and is expected to fail the same way; create the repo
  with plain `gh repo create <owner>/<name> --private` (no `--source`, network-only) and push
  separately with plain `git`.
- `specialist-shared` has no build-on-change propagation: a plan that touches it needs that repo's
  own build+commit+push done (and merged to `main`) **before** delegating to `specialist-admin`,
  which is the one legitimate exception to "don't push until everything's done."
- `specialist-fe` does not depend on `specialist-shared` at all — don't route type changes for the
  public web app through that package.
- Only `specialist-be` has enforced architecture rules (`architecture.spec.ts` fitness functions)
  and a full skill set (`add-endpoint`, `add-entity-field`, ...). `specialist-fe`/`specialist-admin`
  are convention-guided via `.claude/rules/`, not test-enforced; `specialist-shared` has no tests
  at all. Calibrate how much you trust "it compiled" accordingly per repo.
- The `claude()` shell function that switches to the personal Claude account
  (`CLAUDE_CONFIG_DIR=~/.claude-personal`) triggers on any `$PWD` starting with
  `/var/www/specialist`, so it already covers this directory — no extra setup needed to launch a
  session here under the right account.
