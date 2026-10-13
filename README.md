# Parallel Feature Build — a Claude skill

**Build features faster with parallel AI agents, and get senior-engineer reasoning with every solution.**

Author: **Bilal Hadid** · License: MIT · Version 2.0.0

---

## What it does

When you ask Claude to build a feature, this skill makes it work like a small engineering team instead of one developer typing step by step:

```
 Triage → Recon → Plan (contract + file ownership + engineering design) → You approve
   → Wave 0: shared contract committed
   → Wave 1: 2–4 agents build in parallel, each in its own git worktree
   → Merge → Review (contract drift + security) → Test → Auto-fix (max 2 rounds)
   → Deliver: one feature branch + summary
```

It also fixes a common complaint about AI coding: *"it gives me the solution but not the theory behind it."* Every decision is named with the engineering principle it rests on, why it fits, its trade-off, and the standard to look up. For example:

> **Field-level encryption (AES-256-GCM, authenticated encryption)** — phone stored as `v1:iv:tag:ciphertext` with the row id as AAD, so a ciphertext copied to another row fails to decrypt. *Trade-off: phone numbers can't be searched; key must be managed.* — NIST SP 800-38D
>
> **RBAC + object-level authorization (prevents IDOR/BOLA)** — one `canAccessCustomer` check on every `/:id` route. — OWASP API Security Top 10 #1

Gaps are reported honestly ("no rate limiting on login yet → add token-bucket limiting"), not hidden.

## Key ideas

| Idea | Why it matters |
|---|---|
| **Contract-first** | Types/API shapes are written to a file before agents start, so independently built parts fit together. |
| **One owner per file** | Hotspot files (schema, route index, `package.json`) get a single owner, so agents never conflict. |
| **Waves** | Shared groundwork first (wave 0), then the parallel build. |
| **Parallel launch** | All agents of a wave start in one message, in separate git worktrees. |
| **Review before trust** | A clean git merge can still hide mismatched assumptions, so there's a drift + security review, then real tests. |
| **Smart triage** | Small fixes skip agents entirely. Parallelism only pays off on bigger features. |
| **Engineering playbook** | `references/engineering-playbook.md`: ~150 named principles across architecture, API design, security (incl. Server-Side Encryption: SSE-S3 / SSE-KMS / SSE-C, envelope encryption, KMS), data, reliability, performance, frontend, testing and DevOps, with standards to cite (OWASP, NIST, RFCs, WCAG). |

## Install

### Claude Code (recommended: this is where parallel agents run)

Personal (all your projects):
```bash
git clone https://github.com/BilalHadid/parallel-feature-build.git
mkdir -p ~/.claude/skills
cp -r parallel-feature-build/skills/parallel-feature-build ~/.claude/skills/
```

Per project (shared with your team through git):
```bash
mkdir -p .claude/skills
cp -r path/to/parallel-feature-build/skills/parallel-feature-build .claude/skills/
```

### Claude app (claude.ai / desktop)

Download [`dist/parallel-feature-build.skill`](dist/parallel-feature-build.skill) and upload it from your skills settings. In the chat app the skill still gives the full engineering explanations. Parallel agents depend on the environment offering sub-agents.

## Use it

Just ask for a feature. The skill triggers on its own:

```
Add a customer orders module: Prisma model, CRUD API, an Orders page with a create form, and tests.
```
```
Build supplier management and low-stock alerts in parallel.
```

Claude shows you a plan (contract, units, file ownership, engineering decisions, cost), waits for your OK, then runs to the end and delivers:

```
✅ Customer orders — branch feature/orders (7 commits)
Built            … per unit …
Engineering decisions
- State Machine — order status transitions …   Trade-off: …
- Idempotency Keys — POST /orders …            Trade-off: …
Standards referenced: OWASP ASVS V4, RFC 9457 …
Gaps / recommended next  …
Checks: lint ✓ tests ✓ (16 passed)   Merge conflicts: 0   Fix rounds: 0
```

It never pushes or opens a PR unless you ask.

## Honest notes on speed and cost

- **Parallel agents use more tokens.** Each agent has its own context; multi-agent runs cost several times a single agent. You trade usage for wall-clock time.
- **Speed wins on big features, not tiny ones.** In our test on a toy repo, 3 parallel workers built their parts in ~18 s, but planning and merging added a fixed ~30 s, so a single agent was faster there. On real modules that take a single agent 15–30 minutes, the parallel build saves real time.
- **Accuracy is the consistent win.** In tests, the skill's runs worked on feature branches, wrote 2–3× more tests, merged with zero conflicts, and named every security decision. Baseline runs committed to `main` and explained no theory.

Results from our evals are in [`docs/benchmark.md`](docs/benchmark.md).

## Tips

- In Claude Code, built-in `isolation: worktree` branches from your **default branch**. The skill handles this by creating worktrees from the feature branch (or fast-forwarding workers to it). If you prefer, set `worktree.baseRef` to `"head"`.
- Worktrees don't share `node_modules`; add a `.worktreeinclude` for gitignored files like `.env` if workers need them.
- 2–4 agents per wave is the sweet spot. More adds coordination cost.

## Repo layout

```
skills/parallel-feature-build/
  SKILL.md                          # the workflow
  references/engineering-playbook.md
dist/parallel-feature-build.skill   # packaged for upload
evals/evals.json                    # test prompts used during development
docs/benchmark.md                   # test results
```

## Contributing

Issues and PRs are welcome: new playbook entries, better split heuristics, and eval cases from real projects.

## License

MIT © 2026 Bilal Hadid
