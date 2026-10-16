# write_sql.md — SQL / schema latent façade

This is the prose the AGENT reads before writing any `schema/*.sql` file. It pairs with the deterministic façade `.orly/dispatch/write_sql.sh` — the machine half that runs the mechanically-checkable subset and emits one verdict block. **Read `.orly/docs/SCHEMA_CONVENTIONS.md` first** — it is the source-of-truth for table/column naming, type choices, and schema-qualification (the analogue of `.orly/docs/REST_API_DESIGN_GUIDELINES.md` for `write_http`). The dissolving **Schema Table Removal Guard** card is merged verbatim below, while the durable SQL rules (`NSQ`, `STS`, `SGR`, `ITF`) stay in `.orly/docs/greptile-learnings/RULES.md` (retained) — referenced here, enforced by the GREPTILE GATE. Mechanical thresholds live once in the `.sh`; this file references rule codes, never restates the numbers.

**Signal legend** (printed by `write_sql.sh`):

- 🟢 pass — deterministic check passed.
- 🔴 fail — deterministic check failed (or helper absent); STOP, fix, rerun.
- 🤔 DECIDE — judgment-only; no script can decide, the agent reads the section and makes the call (blocks the TURN, not the script).
- 🟣 delegated — the checker runs only in the product repo, not in dotfiles.

**Tag legend** — each section heading below carries one of:

- `> [DETERMINISTIC → <CODE>]` — a machine can pass/fail it; the `.sh` row for `<CODE>` (e.g. `FLL`) enforces it. `TODO-CHECK` marks a mechanizable rule with no helper wired yet (build-the-check). `NEW:<name>` marks a proposed-but-not-yet-existing code.
- `> [JUDGMENT → <CODE>]` — no script can decide; the agent decides at write time against the prose.
- `> [container]` — a non-enforcement wrapper heading (e.g. "Merged from dissolved gate cards"); its tagged subsections carry the real codes, and the coherence audit (§6.3) skips it.

The tag grammar and semantic-anchor model are defined by the tagged sections and
deterministic façade in this snapshot.

---

# SQL & schema authoring discipline

## Scope

> [JUDGMENT → SCH]

Triggers on SQL files, embedded SQL and the repository's migration registration.
Before writing schema changes, read `.orly/docs/SCHEMA_CONVENTIONS.md` and locate the
repository's current migrations, registration and deployment policy.
Embedded SQL also fires its selected language façade and the GREPTILE GATE.

## Repository schema policy

> [JUDGMENT → SCH]

Preserve shipped migrations and live data under the repository's own policy.
Destructive changes require the owner's approval for the specific change.
A version number, missing `VERSION` file or another product's rebuild policy
never proves that this repository's data is disposable.


## Companion SQL rules (retained in `.orly/docs/greptile-learnings/RULES.md`)

> [container]

These rules are **not** dissolved — they remain in `.orly/docs/greptile-learnings/RULES.md`. STS/NSQ/SGR are enforced by the GREPTILE GATE until a leaf check is wired (`TODO-CHECK`); ITF additionally surfaces as a `🤔 DECIDE` judgment row in `write_sql.sh`. Summarised here so the SQL author sees the full surface in one place; RULES.md stays canonical (read it for the full text + `Ref:` provenance).

### RULE STS — No static strings in SQL schema

> [DETERMINISTIC → TODO-CHECK]

No string literal in `schema/**.sql` may name a value or identifier the application also names — SQL cannot reference Zig/JS constants, so schema strings drift from code. The surface is every literal, not two syntactic forms: `DEFAULT`, `CHECK (… IN (…))`, **trigger-body comparisons against a column**, **`current_setting()` parameter names**, and **`RAISE EXCEPTION` text a caller matches on**. Enforce value constraints via application constants; where SQL cannot import the constant, pin the pair with a slot-grep test. Carve-out: a literal encoding a structural identity rather than a vocabulary value (`'{}'::jsonb`, `'{}'::text[]`) stays, reason stated inline. A drifted `DEFAULT` yields wrong data; a drifted trigger predicate or setting name fails **silently** — the guard stops guarding. (RULES.md `RULE STS`.)

### RULE NSQ — Named constants, schema-qualified SQL

> [DETERMINISTIC → TODO-CHECK]

No magic numbers; qualify tables with the repository's actual schema —
unqualified names fail when `search_path` differs across environments.
(RULES.md `RULE NSQ`.)

### RULE SGR — Migrations include GRANT statements

> [DETERMINISTIC → TODO-CHECK]

Every `CREATE TABLE` migration grants the required operations to the roles that
actually query the table. PostgreSQL denies by default — a missing grant fails
only at first runtime use. (RULES.md `RULE SGR`.)

### RULE ITF — Integration tests use the real schema

> [JUDGMENT → ITF]

An integration test touching a production table uses shared repository fixture
helpers against the real schema — never a session-local `CREATE TEMP TABLE`
mocking the production shape. Fixtures use semantic scope names.
(RULES.md `RULE ITF`.)
