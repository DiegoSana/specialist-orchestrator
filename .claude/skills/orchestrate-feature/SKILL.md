---
name: orchestrate-feature
description: Investigate a feature/bug requirement across the Specialist repos, plan the cross-repo contract, and delegate implementation to one subagent per affected repo (specialist-be, specialist-fe, specialist-admin, specialist-shared). Use when a request in /var/www/specialist arrives that plausibly spans more than one repo — a new API + UI, a changed DTO shape, anything touching both backend and a frontend.
---

# orchestrate-feature

You are the orchestrator, not the implementer. Your job is investigate → plan → delegate →
aggregate. Don't edit application code in a sub-repo yourself unless a delegated agent got stuck
and you're fixing that specific agent's mistake.

## 0. Scope check

If the requirement is obviously confined to one repo (a UI tweak, a copy change, a single-file
fix), skip this whole skill and just do the work directly in that repo — spinning up subagents for
single-repo work is pure overhead.

## 1. Investigate (do this yourself)

- Re-read the relevant sections of each candidate repo's `CLAUDE.md` (already summarized in the
  root `CLAUDE.md`'s repo map, but go deeper for the specific feature area).
- Grep/glob across the repos for existing related code — an existing similar endpoint, hook,
  component, or Prisma model — so you can point delegated agents at concrete files instead of
  making them rediscover the codebase from scratch. `grep -rn "<keyword>" specialist-be/src
  specialist-fe/hooks specialist-fe/components specialist-admin/hooks specialist-admin/app` is a
  reasonable starting sweep.
- Check `docs/API.md`, `docs/guides/PERMISSIONS_BY_ROLE.md` in `specialist-be` for whether the
  capability (or something close to it) already exists.
- Check `TODO.md` (in this directory — the global backlog, one section per repo) for existing
  plans, priorities, or known related discrepancies that should shape the plan.
- If the requirement is ambiguous in a way that changes the plan materially (which roles get
  access, whether it applies to Companies as well as Professionals, whether it needs a migration),
  ask the user now with `AskUserQuestion` — don't guess on anything that would make a delegated
  agent build the wrong thing.

## 2. Plan the cross-repo contract

Write this down explicitly (in your own response, not just in your head) before delegating:

- **Which repos are affected**, and in dependency order. Typical order:
  `specialist-shared` (only if a shared type/contract needs to change) → `specialist-be` → then
  `specialist-fe` and/or `specialist-admin` in parallel.
- **The interface between repos**: exact endpoint path + HTTP method, request/response field
  names and types, new enum values, new i18n keys, any new domain event. This is the contract
  every delegated agent will build against — if backend and frontend agents each guess
  independently, they'll drift.
- **Branch name**: one name, reused across every repo touched (e.g. `feat/portfolio-videos`).
- **A one-paragraph task spec per affected repo** — this becomes the core of that repo's delegated
  prompt.

For anything destructive or hard to reverse (a schema migration, a breaking API change, deleting
data, a change to auth/permission logic) confirm the plan with the user before delegating. For
routine additive work, proceed directly.

## 3. Delegate

Use the `Agent` tool, `subagent_type: "general-purpose"`, one call per affected repo. Batch
independent repos (e.g. `specialist-fe` + `specialist-admin`, once the backend contract is settled)
in a single message with multiple `Agent` calls so they run in parallel; run the dependency-root
repo(s) first and wait for that result before delegating the dependents, since they need the
confirmed real shape (not just the planned one — a backend agent may reasonably adjust a field name
while implementing; carry the *actual* final shape forward, not the plan, when it diverges).

Each prompt must be self-contained (the subagent starts with zero context) and should include:

1. **What this is part of**: one sentence naming the overall feature, so the agent understands
   it's implementing a slice, not the whole thing.
2. **Where to work**: the absolute repo path, e.g. `/var/www/specialist/specialist-be`.
3. **The exact task**: framed in that repo's own vocabulary, referencing concrete files found in
   step 1 where possible.
4. **The contract**: the specific endpoint/DTO/field shape from step 2 — tell it not to deviate
   without flagging it back to you in its final report.
5. **Process instructions**:
   - Read that repo's own `CLAUDE.md` and `.claude/rules/` first.
   - Use that repo's own skills where one applies (e.g. `add-endpoint`, `add-entity-field` in
     `specialist-be` — invoke them via the `Skill` tool).
   - Check `git -C <repo> branch --show-current` and `git status`. If the repo is clean on `main`,
     branch directly (`git checkout -b <branch-name>`). If it has other uncommitted work or is on
     an unrelated branch, use `git worktree add /tmp/<repo>-<branch> main -b <branch-name>` instead
     of disturbing it, then remove the worktree once committed.
   - Verify `git config user.email` is `diegohsanabria@gmail.com` in that repo before committing;
     if not, set `user.name "Diego Sanabria"` / `user.email "diegohsanabria@gmail.com"` locally
     first.
   - Run that repo's definition-of-done gate (tests/lint/build — see its `CLAUDE.md`) before
     finishing.
   - Commit locally with a Conventional Commit message (context-scoped where the repo uses that
     convention) ending in the standard `Co-Authored-By` line. **Do not push, do not open a PR** —
     report back instead.
6. **What to report back**: files changed, final endpoint/DTO shape actually implemented (flag any
   deviation from the planned contract), test/lint/build status, the commit hash.

Exception: if the plan touches `specialist-shared`, that agent's instructions must additionally
say to `npm run build`, commit `dist/` alongside `src/`, and **push to `main`** — the only
legitimate push-before-the-end, since `specialist-admin` can't pick up the change otherwise. Note
this exception explicitly in that agent's prompt and in your own plan write-up.

## 4. Aggregate & report

- Collect every subagent's report. Cross-check that the shapes each repo actually implemented
  agree with each other (e.g. the field name the backend agent used matches what the frontend
  agent consumed) — if they don't, that's a bug to fix now, before handing back to the user.
- Give the user one unified summary: what changed per repo, test/lint/build status per repo,
  branch name(s), commit hashes, and anything you had to decide or deviate on.
- Ask before pushing/opening PRs for the remaining repos (unless the original request already said
  to do that) — mirror the manual flow: `git push -u origin <branch>` then `gh pr create --repo
  DiegoSana/<name> --head <branch> --base main ...` (use `--repo` explicitly, see root `CLAUDE.md`
  Gotchas).
