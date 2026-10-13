---
name: parallel-feature-build
description: Builds software features fast and to senior-engineer standard. Splits a feature into independent units run by parallel agents in separate git worktrees, then merges, reviews, tests, auto-fixes and delivers one feature branch. Every solution names the engineering principles behind it (e.g. Server-Side Encryption, RBAC, Idempotency, Optimistic Locking, Repository Pattern, OWASP) with why and trade-offs. Use whenever the user asks to build, add, implement or ship a feature, module, screen or API touching several layers (DB, API, UI, tests), or says "make it faster", "use parallel agents", "split this up", "build in parallel", "explain the architecture", even without naming this skill. Small fixes get done directly, still with the principle named.
metadata:
  author: Bilal Hadid
  version: 2.0.0
---

# Parallel Feature Build

*Author: Bilal Hadid*

Two goals, both required:

1. **Speed** — independent pieces of a feature are built at the same time by parallel agents.
2. **Engineering quality you can learn from** — every design choice is made on purpose and **named**: the principle, pattern or standard behind it, why it fits here, and what it costs. A developer reading the result should know *what* was built and *which theory it rests on* (e.g. "SSE-KMS encryption at rest", "object-level authorization against IDOR", "transactional outbox").

The engineering catalogue lives in `references/engineering-playbook.md`. Read the sections relevant to the feature during planning.

Parallelism has a real cost: each agent spends its own tokens (multi-agent runs cost several times a single agent), and coding tasks often have tighter dependencies than they look. Badly split work produces merge conflicts and contract drift that cost more than they save. So the first job is deciding *whether* and *how* to split.

## The flow

```
0. Triage      → can agents run in parallel? is this big enough? if not, build directly (still apply step 2e).
1. Recon       → stack, structure, commands, conventions, hotspot files.
2. Plan        → contract + units + file ownership + waves + engineering design decisions.
3. Approve     → show the plan, wait for OK.
4. Build       → wave 0 (contract) committed, then all units of a wave launched in ONE message.
5. Merge       → review for contract drift, merge unit branches one by one.
6. Verify      → checks + engineering/security review; fixer max 2 rounds.
7. Deliver     → feature branch + summary with named engineering decisions and gaps.
```

Track these steps in the task list if one exists.

---

## 0. Triage

**Can you run agents in parallel?** You need an Agent/Task tool. In Claude Code, subagents can usually spawn further subagents (up to a depth limit), but some environments block nesting or don't offer the tool. If you can't launch agents, skip worktrees, unit branches and merging — they're pure overhead without parallelism. Build directly on a feature branch, still doing the contract, engineering design (2e), checks and summary. Say in one line that you built it directly and why.

**Is it worth splitting?** Build directly when:
- it touches roughly 1–3 files, or is a bug fix, rename, config or copy change
- every step depends on the previous one (tight coupling; nothing to run side by side)
- the repo has no clear separation (single script, notebook)

For direct work, still name the principle in your answer in one or two lines (e.g. "Fixed with a parameterized query — prevents SQL injection, OWASP A03"). This habit is what makes answers educational.

If the job is the *same* mechanical change across many files (rename an API everywhere, migrate imports), it's a batch refactor: use a built-in batch skill such as `/batch` if available; otherwise split units by folder.

**Parallelise** when there are 2+ substantial pieces that can be written at the same time — separate layers (schema, API, UI, tests) or separate sub-features (two screens, three endpoint groups).

## 1. Recon

Delegate to a cheap read-only agent (scout/Explore type) when available, so the main context stays small. Ask for under ~40 lines:

- stack, frameworks, package manager, monorepo layout
- folder conventions with 1–2 example files to imitate
- exact install / test / lint / typecheck / build commands
- git status: repo? current branch? clean tree? default branch name?
- **hotspot files** many features touch: DB schema/migrations (`schema.prisma`), route registries, barrel/index files, nav/menu config, shared types, i18n, `package.json`/lockfiles
- existing security & architecture conventions: auth middleware, role checks, validation library, error format, logging, how secrets/config are loaded

If the tree has uncommitted changes, ask whether to commit, stash or include them — worktrees start from a commit and won't see uncommitted work.

## 2. Plan

### 2a. Contract

Define the interfaces units share, concretely enough that agents who never talk produce code that fits:

- data shapes: types/interfaces, DB model fields and types, enums
- API: method, path, request, response, status codes, error shape
- signatures crossing unit boundaries (function names, params, returns, component props)
- names: files, routes, tables, events, permissions/roles

Write it as real code (types file, schema block, OpenAPI snippet), and **put it in a file in the repo** (e.g. `src/contracts/<feature>.ts`). Shared context must live in a file, because each worker starts with a fresh context and sees nothing of this conversation.

### 2b. Work units

Split by whatever gives the cleanest file separation: **by layer**, **by sub-feature**, or **mixed**. Aim for **2–4 units per wave** (5 at most). Beyond that, coordination and merge cost grow faster than time saved.

Each unit gets: id + goal, files it **owns**, files it may **read** only, contract pieces it implements or consumes, its done-check command, and the **engineering principles it must apply** (from 2e).

### 2c. File ownership

Every edited file has exactly one owner. Hotspot files either go to **wave 0** (done before the parallel wave) or to a single unit; other units report "needed from others" edits that you apply at merge. Never let two agents touch `package.json`/lockfiles: collect dependencies in planning and install them in wave 0.

### 2d. Waves

Group units by dependency. A wave starts only after the previous one is merged. Most features need wave 0 (contract, schema, registrations, deps) plus one parallel wave. Tests can be written in the same wave as the code, against the contract.

### 2e. Engineering design

Read the relevant sections of `references/engineering-playbook.md` (security and data almost always; others as the feature needs) and decide deliberately. Record each significant decision:

| Decision | Principle / pattern | Why here | Trade-off | Standard |
|---|---|---|---|---|
| Order status changes only via allowed transitions | State Machine | prevents shipping unpaid orders | small extra code | — |
| `/orders/:id` checks ownership, not just login | Object-level authorization (IDOR/BOLA) | any user could read others' orders | one query per request | OWASP API1 |
| Payment details encrypted per column | Field-level encryption, envelope keys in KMS | PII at rest | key management | NIST SP 800-57 |

Rules:
- Pick what genuinely fits. A CRUD screen doesn't need CQRS or sagas; name a heavy pattern only to say why it was *not* used, if that's useful.
- Where something important can't be done in scope (no KMS available, no rate limiter yet), mark it as a **gap**. Never present it as done.
- Follow the repo's existing conventions (auth middleware, validation library, error format) unless they're unsafe. Then flag it.
- For decisions that will matter long-term (data model, auth approach, sync strategy), offer to write an ADR (Architecture Decision Record: context, decision, consequences) in `docs/adr/` if the repo uses them.

### Choosing models

Cheap/fast model for recon and running tests, mid-tier for implementation units, strongest only for planning, hard units, tricky merges and the final review. If the environment has named roles (scout, implementer, test-runner, reviewer), use those.

## 3. Approve

Show the plan compactly and wait:

```
Feature: <name>       Branch: feature/<slug>       Units: <n> in <w> wave(s)

Contract  (<path of contract file>)
  <key types / endpoints>

Engineering decisions
  <decision> — <principle> — <why>     (5–10 lines; full table in plan file)
  Gaps: <anything important not covered>

Wave 0 (lead): <contract, schema, registrations, deps>
Wave 1 (parallel):
  A  <goal>   owns: <files>   applies: <principles>
  B  <goal>   owns: <files>   applies: <principles>

Hotspots: <file> → <owner or "applied at merge">
Checks after merge: <commands>
Cost: <n> parallel agents (more tokens, less wall-clock time)
```

Ask: approve, or change something? After approval, run to the end without stopping unless a decision only the user can make comes up. In an unattended run, proceed and state at the top of the final report that the plan wasn't reviewed.

## 4. Build

1. `git checkout -b feature/<slug>` from the current branch.
2. **Wave 0:** create the contract file, schema changes, stubs for new modules, hotspot registrations and dependency installs; save the plan (with the engineering table) to `.parallel/<slug>-plan.md` or similar; **commit** on the feature branch. Workers need this commit.
3. **Create worktrees from the feature branch tip.** Most reliable is to do it yourself:
   `git worktree add ../<repo>-<unit> -b feature/<slug>--<unit> feature/<slug>`
   If you use the Agent tool's `isolation: "worktree"` instead, know that Claude Code branches those worktrees from the **default branch** (e.g. `main`), not your current branch, unless `worktree.baseRef` is set to `"head"`. In that case tell each worker to first run `git merge --ff-only feature/<slug>` and confirm the contract file exists before coding.
4. Worktrees don't share `node_modules`/venvs: tell workers to install deps if their checks need them. Gitignored files like `.env` aren't copied unless the repo has a `.worktreeinclude`. Copy what's needed or tell workers to avoid code paths that need secrets.
5. Launch **every unit of the wave in a single message** with multiple Agent calls. Launching one at a time is sequential work in disguise.
6. No git? Skip worktrees and rely on strict file ownership.

If Claude Code agent teams are enabled (experimental), named subagents may become teammates with shared task lists. The same ownership and contract rules apply.

### Worker prompt template

Workers see nothing of this conversation, so the prompt must stand alone:

```
You are building one part of a larger feature, in parallel with other agents.

Worktree: <path>   Branch: <unit branch> (work only in this directory)
First: confirm <contract file> exists; if not, run `git merge --ff-only feature/<slug>`.
Stack and conventions: <2–5 lines from recon + 1–2 example files to imitate>

Your unit: <id> — <goal>

Contract (match exactly; others depend on it): <contract file path + key snippets>

Engineering principles you must apply in this unit:
- <principle> — <what it means concretely here>
  (e.g. "Object-level authorization: every :id route loads the record and checks
   record.ownerId === user.id or role permits, else 404")
- ...

You OWN (create/edit only these): <paths>
READ only: <paths or "everything else">
Need a change in a file you don't own? Don't make it; list it under NEEDED FROM OTHERS.

Done means:
- goal implemented, contract matched exactly, principles above applied in the code
- <done-check command> passes (install deps in this worktree if needed)
- committed on your branch with a clear message

Report (short — no long explanations):
STATUS: done | blocked
FILES: <created/changed>
CHECK: <command> → <pass/fail + key lines only>
PRINCIPLES APPLIED: <principle> → <file:function where applied>
CONTRACT CHANGES: none | <deviation + why>
NEEDED FROM OTHERS: none | <edits>
NOTES: <what the merger must know, incl. anything you could not apply>
```

## 5. Merge

1. Read every report. If a unit is `blocked`, changed the contract, or couldn't apply a principle, decide before merging: fix and re-run that unit, or adapt the others.
2. **Check for semantic drift before merging.** Clean git merges can still hide mismatched assumptions: duplicated helpers, different field names, one side returning `null` where the other expects `404`. Skim each unit's diff against the contract (`git diff feature/<slug>...feature/<slug>--<unit>`), or have a reviewer agent do it for large diffs.
3. Merge one at a time, smallest/most independent first: `git merge --no-ff feature/<slug>--<unit>`. Resolve conflicts yourself (you know the contract) or hand conflicted files plus contract to one agent.
4. Apply NEEDED FROM OTHERS edits and commit.
5. `git worktree remove` each worktree; delete merged unit branches.
6. Next wave, if any, from the updated feature branch.

## 6. Verify

**Checks:** on the merged feature branch run install (if deps changed), typecheck, lint, tests, build. Delegate to a test-runner agent if available, asking only for pass/fail and the failing lines. Keep output short.

**Engineering and security review:** for anything non-trivial, have a reviewer agent (or a `/code-review` / `/security-review` skill if installed, or yourself) review the full feature diff against base with this checklist:
- contract matched everywhere; no leftover TODOs, debug code, dead stubs
- each decision in the 2e table is actually in the code (not just claimed)
- security: authZ on every route incl. object-level checks, input validated at the boundary, no string-built SQL, output escaped, secrets not hardcoded or logged, sensitive data encrypted/masked as planned (map findings to OWASP Top 10 / ASVS)
- data: transactions where multi-write, money not in floats, indexes for new query paths, no N+1
- tests cover boundaries and failure paths (400/401/403/404/409), not just happy paths

**Fix loop:** failures or review findings go to one fixer agent on the feature branch with exact failures + contract + relevant files.
- Round 1, then round 2 if needed.
- After 2 rounds, stop. Report what still fails, likely cause, and what you'd try next. Blind retries beyond this rarely help.

Pre-existing failures (also failing on base) are listed separately, not fixed silently.

## 7. Deliver

Deliverable: the merged feature branch, checks run, and this summary:

```
✅ <Feature> — branch feature/<slug> (<n> commits, based on <base>)

Built
- <unit>: <what it does> (<key files>)

Engineering decisions
- <Principle / pattern> — <where in code> — <why it was the right call here>. Trade-off: <cost>.
- <e.g. Server-Side Encryption (SSE-KMS) — uploads bucket config — documents encrypted at rest with
  customer-managed keys, access auditable in KMS. Trade-off: KMS request cost, key policy to manage.>
- ...
Standards referenced: <OWASP ASVS V4 Access Control, RFC 9457, WCAG 2.2 AA, ...>

Gaps / recommended next
- <important principle not applied + why + suggested fix>

Checks: typecheck ✓  lint ✓  tests ✓ (<x> passed)  build ✓   Review: <clean | n findings fixed>
Fix rounds: <0–2>   Merge conflicts: <n>
Time: <wall-clock> · Agents: <n> across <w> wave(s)
```

Write the engineering decisions for a developer who wants to learn: name the concept as it's known in the industry, so they can look it up, then one plain sentence tying it to this code. Five sharp decisions beat fifteen name-drops.

Don't push or open a PR unless asked; offer it in one line.

---

## Common mistakes

- **Splitting too fine.** Fewer, meatier units. 2–4 per wave.
- **Contract only in the conversation.** Workers can't see it; commit it to a file in wave 0.
- **Worktrees from the wrong base.** Built-in worktree isolation may start from `main`; verify the contract commit is present.
- **Two owners for one file**, especially schema, route index, package.json.
- **Sequential launching.** All units of a wave go out in one message.
- **"Merged cleanly" ≠ correct.** Review for semantic drift, then run checks.
- **Name-dropping.** Only claim principles that are in the code; list the rest as gaps.
- **Skipping the explanation on small tasks.** Even a one-line fix names its principle.
