# the-seedskill

A [Claude Code](https://claude.com/claude-code) skill (formerly `multitenant-saas-playbook`): an engineering playbook for building and evolving a real, paying-customer-facing multi-tenant SaaS. The reference stack is Next.js + Supabase/Postgres + Tailwind + Vercel, but the lifecycle and diagnostic habits generalize to other stacks.

It captures the operating discipline that came out of actually shipping a multi-tenant SaaS solo, over many rounds of: build → a real user tries it → something's subtly wrong → find the actual mechanism → fix it → check whether the same mechanism is lurking elsewhere. On top of that domain-specific residue — gotchas that cost real debugging time, design calls that turned out right, and calls that had to be reversed after real usage — it now carries a self-contained core of general engineering discipline adapted from [obra/superpowers](https://github.com/obra/superpowers): systematic debugging, evidence-before-claims verification, test-first development, a design → plan → execute workflow with approval gates, and code review / parallel-agent practices. You don't need superpowers installed to use it; if you have it, the two compose.

## What's inside

- [`SKILL.md`](SKILL.md) — the main playbook: the rules that don't bend, push/merge authorization discipline, "verify by seeing it, not by inferring it," how to diagnose a "this is broken" report, auditing for the same bug shape after a fix.
- [`references/debugging-and-verification.md`](references/debugging-and-verification.md) — four-phase root-cause debugging, the three-fix rule, defense in depth, and the verification gate before any "done" claim.
- [`references/testing-discipline.md`](references/testing-discipline.md) — red→green→refactor, tests that name the break they catch, and multi-tenant tests worth writing every time.
- [`references/design-plan-execute.md`](references/design-plan-execute.md) — classifying a request (spike / bounded / architectural), approval gates, writing implementation plans, and finishing a branch.
- [`references/review-and-agents.md`](references/review-and-agents.md) — receiving and requesting code review, and dispatching parallel agents.
- [`references/nextjs-supabase-gotchas.md`](references/nextjs-supabase-gotchas.md) — concrete bug patterns in the Next.js/Supabase/Postgres stack (redacted Server Action errors, RLS that silently empties joins, column-level grants, ignored gate results…), each with a broken/fixed code pair.
- [`references/money-and-ledger-integrity.md`](references/money-and-ledger-integrity.md) — patterns from auditing code that moves money and stock: void flags and their compensating entries, lock order and missing locks, hard-delete vs `activo`, `NaN`/rounding traps, rules that must hold on every write path, stale-base `create or replace`, and the checklist before calling such a fix done.
- [`references/multitenant-architecture.md`](references/multitenant-architecture.md) — how to design one codebase that serves structurally different businesses without forking.
- [`references/responsive-design-method.md`](references/responsive-design-method.md) — a breakpoint strategy and mockup-first process for retrofitting responsiveness onto an existing app, plus the small repeating shapes found in a second pass on an accordion-style app.

## Install

Clone it directly into your Claude Code skills folder:

```bash
git clone https://github.com/lean19r-cell/the-seedskill ~/.claude/skills/the-seedskill
```

On Windows, that's `%USERPROFILE%\.claude\skills\the-seedskill` (or `C:\Users\<you>\.claude\skills\the-seedskill`).

To use it inside a single project instead of globally, clone it into that project's `.claude/skills/the-seedskill` and commit it — anyone who clones the project repo gets the skill automatically.

No build step, no registration — Claude Code auto-discovers any folder under `.claude/skills/` with a valid `SKILL.md`. Start a new session (or restart) after installing so it picks it up.

To update later: `git -C ~/.claude/skills/the-seedskill pull`.

## Credits

The general-discipline references (`debugging-and-verification.md`, `testing-discipline.md`, `design-plan-execute.md`, `review-and-agents.md`) are adapted from [Superpowers](https://github.com/obra/superpowers) by Jesse Vincent and Prime Radiant, used under the MIT License (Copyright (c) 2025 Jesse Vincent).
