# write_ts_adhere_bun.md — TypeScript / Bun latent façade

This is the prose the AGENT reads before writing any `*.ts` / `*.tsx` / `*.js` / `*.jsx` file. It pairs with the deterministic façade `.orly/dispatch/write_ts_adhere_bun.sh` — the machine half that runs the mechanically-checkable subset and emits one verdict block. This document consolidates the former Bun/TypeScript rules merged with the TypeScript-relevant dissolved gate-card deltas (UI substitution, design-token): every original rule line is preserved verbatim (one forward-looking repoint to `write_zig.md`), each `## ` section now carries exactly one enforcement tag, and the dissolved gate-card prose is appended under "Merged from dissolved gate cards." Mechanical thresholds live once in the `.sh`; this file references rule codes, never restates the numbers.

**Signal legend** (printed by `write_ts_adhere_bun.sh`):

- 🟢 pass — deterministic check passed.
- 🔴 fail — deterministic check failed (or helper absent); STOP, fix, rerun.
- 🤔 DECIDE — judgment-only; no script can decide, the agent reads the section and makes the call (blocks the TURN, not the script).
- 🟣 delegated — the checker runs only in the product repo, not in dotfiles.

**Tag legend** — each section heading below carries one of:

- `> [DETERMINISTIC → <CODE>]` — a machine can pass/fail it; the `.sh` row for `<CODE>` (e.g. `UFS`, `FLL`, `UIS`, `DTK`, `TSC`) enforces it. `TODO-CHECK` marks a mechanizable rule with no helper wired yet (build-the-check). `NEW:<name>` marks a proposed-but-not-yet-existing code.
- `> [JUDGMENT → <CODE>]` — no script can decide; the agent decides at write time against the prose.
- `> [container]` — a non-enforcement wrapper heading (e.g. "Merged from dissolved gate cards"); its tagged subsections carry the real codes, and the coherence audit (§6.3) skips it.

The tag grammar and semantic-anchor model are defined by the tagged sections and
deterministic façade in this snapshot.

---

# Bun / TypeScript discipline

Modelled on `write_zig.md`. Pre-design rules, decisive defaults, and the
anti-patterns each rule exists to prevent. Every rule has a one-line
"why" so you can judge edge cases.

## Scope

> [JUDGMENT → TSJ]

Triggers on every `Edit`/`Write` to:

- `*.ts`, `*.tsx`, `*.js`, `*.jsx` outside `vendor/`, `node_modules/`, `.next/`, `dist/`, `build/`.
- Any Bun-runtime entrypoint (`bun.lock`, `bunfig.toml`, files invoked via `bun run`).

Out of scope: framework-generated files (Next.js generated types, Astro `.astro`).

The **UI Component Substitution Gate** (the UI Component Substitution section below) and **GREPTILE GATE** (which carries RULE UFS, RULE TGU, RULE PRI for TypeScript) sit on top of this file — they fire in addition to these rules, not instead of them.

## §1 · TS FILE SHAPE DECISION (mandatory at PLAN)

> [JUDGMENT → FSD]

Before writing the first line of any new `*.ts`/`*.tsx` file under `src/`, `app/`, or `lib/`, print:

```
TS FILE SHAPE DECISION: <intended-path>
  Trigger: <new file | threshold-cross: <which>>
  Purpose: <one sentence — what this file's job is>
  Primary export: <ClassName | factory function | const | type-only module>
  Behaviour bound to state: <yes/no>
  Verdict: <class | factory | functions-module | type-only>
  Why not the other: <one sentence — required when verdict is conventional>
```

**Decision matrix:**

| Behaviour | State | Verdict |
|---|---|---|
| Operations bound to a single piece of state | yes (mutable, lifecycle) | **class** with `#private` fields |
| Operations on a passive value (validators, parsers, formatters) | no | **functions-module** (named exports) |
| One-shot construction returning a stateful holder | yes (frozen after init) | **factory function** returning `Object.freeze({...})` |
| Only types/interfaces/enums | n/a | **type-only module** |

**Threshold-cross binding** — same as Zig: an existing functions-module that's about to gain its first stateful operation rearchitects to `class` in the same diff. Don't grow a `let cachedX = …` at module scope to dodge the conversion.

**Override:** `TS FILE SHAPE DECISION: SKIPPED per user override (reason: ...)` — user-only, in this turn. Auto-mode does NOT cover it (mirrors Zig FILE SHAPE rule).

## §2 · `const` discipline

> [DETERMINISTIC → UFS]

_`UFS` is a universal rule **enforced once by `write_any`** (which fires for every source file); this façade carries its prose for the author but does not re-run `audit-ufs` — no redundant full-tree scan (`DISPATCH_ARCHITECTURE.md` §16, Decision 6)._

_**ui/ carve-out = manual duty.** `.orly/audits/ufs.sh` skips `ui/packages/*/{src,app,tests,components,lib,hooks}/` for the string-dup check (Tailwind class-strings are noise); cross-runtime `ERR_*` parity still runs. So in ui/ TS/TSX the **author** runs the UFS pass by hand: extract any literal repeated ≥2× in-file to a named const; for discriminated-union tags use an `as const` object so narrowing survives (`const STATUS = { approved: "approved" } as const`, reference `STATUS.approved` at every site). Do not trust the audit to catch ui/ literals._

- **`const` by default. `let` only when the variable is reassigned in the same scope.** Never `let` "because I might change it later". A future-tense `let` is dead-on-arrival.
- **No `var`. Ever.** Hoisting is a bug nursery; the lint rule should be a hard error in `tsconfig`/`eslint.config.ts`.
- **Module-level constants are `SCREAMING_SNAKE_CASE`** when they're tunables (timeouts, limits, magic numbers). `camelCase` when they're configuration values composed at runtime (`const config = {...}`).
- **RULE UFS — Named constants for repeated and semantic literals (applies to all source).**
  - **String literals used in ≥2 sites become a named const.** "It's just a label" is not an exception. Wire-format enum values (`"platform"`, `"self_managed"`, `"receive"`, `"stage"`) MUST be defined once as a SCREAMING_SNAKE const-as-namespace (`PROVIDER_MODE.platform`, `CHARGE_TYPE.receive`) and referenced everywhere else — including test fixtures, mock returns, and assertion arguments. Inline string repeats are a rule violation even if TypeScript narrows them.
  - **Numeric literals carrying semantic meaning become a named const** even at the first use site. Examples that always need a name: conversion factors (`NANOS_PER_USD`, `MS_PER_SECOND`, `BYTES_PER_MIB`), thresholds (`LOW_BALANCE_THRESHOLD_NANOS`, `MAX_RETRY_COUNT`), magic offsets, sub-cent rates. Bare digits like `1_000_000_000` in a `nanos / 1_000_000_000` expression are a smell — the unit suffix names the operation, but the constant names *what the divisor is*. Carve-out: **pin tests** where the literal IS the contract (`expect(formatDollars(4_710_000_000)).toBe("$4.71")`) — those keep the literal, with a `// pin test: literal is the contract` comment.
  - **Cross-runtime constants must be named identically** across Zig, TS, and JS (same SCREAMING_SNAKE identifier; case-only diff allowed for language idioms). When a constant exists in one runtime, the matching one in any sibling runtime carries the same name. `NANOS_PER_USD` is the canonical example: `pub const NANOS_PER_USD` in Zig, `export const NANOS_PER_USD` in TS, `export const NANOS_PER_USD` in JS — never `NANOS_PER_DOLLAR` in one and `NANOS_PER_USD` in another.
  - **Self-audit at CONFORM.** Before declaring done, grep the diff for repeat string literals and bare-numeric divisors/multipliers — every hit either becomes a named const or earns the `// pin test` carve-out comment. Spec dimensions that introduce a new wire-format value, conversion factor, or threshold must define the constant in PLAN, not EXECUTE.

## §3 · Import discipline

> [DETERMINISTIC → TSC]

- **`import type { X } from "..."`** for type-only imports. Bun's TypeScript transpile drops type-only imports without analysis — using value imports for types adds bundle weight and complicates tree-shaking.
- **No default exports** for application code. Default exports break renaming, hide rebinding, and confuse `import` autocomplete. Exception: framework conventions that require it (Next.js page/layout, Astro components).
- **Side-effect imports last** (`import "./bootstrap"`). Distinguish `import "./reset.css"` from value imports by physical position — eyeballs need that signal.
- **Path aliases over `../../../`.** Configure in `tsconfig.json`/`bunfig.toml`. If you find yourself counting dots, the file is in the wrong directory.

## §4 · Bun-native primitives over Node compat

> [JUDGMENT → TSJ]

| Task | Use | Avoid (unless explicit reason) |
|---|---|---|
| HTTP server | `Bun.serve({ fetch, port })` | `express`, `fastify`, `hono` (unless a specific middleware is needed) |
| Filesystem read | `Bun.file(path).stream()` for unknown-size input; `.text()` / `.json()` / `.bytes()` only after size is bounded per §11 | `fs.promises.readFile` |
| Filesystem write | `await Bun.write(path, data)` | `fs.promises.writeFile` |
| SQLite | `import { Database } from "bun:sqlite"` | `better-sqlite3`, `sqlite3` |
| Test runner | `import { test, expect } from "bun:test"` | `vitest`, `jest`, `mocha` |
| Process spawn | `Bun.spawn(["cmd", "arg"])` | `child_process.spawn` |
| Hashing | `Bun.password.hash` / `Bun.CryptoHasher` | `crypto.createHash` for new code |
| Globbing | `new Bun.Glob(pattern).scan(...)` | `glob` package |
| Env | `Bun.env` (immutable view of `process.env`) | reading `process.env` directly when Bun-native works |

The Bun runtime is the constraint. If a dependency forces Node-compat, document the reason in a comment (`// using fs.promises because pdf-lib expects Node Buffer`).

## §5 · Type discipline at parse boundaries

> [JUDGMENT → TGU]

- **Never `any`.** Use `unknown` and narrow.
- **Narrow at parse boundaries** — when JSON crosses a network/storage boundary, validate with a real schema (Zod / Valibot / TypeBox). Trust nothing from outside the process.
- **Branded types for IDs** (`type WorkspaceId = string & { readonly _brand: "WorkspaceId" }`). Prevents passing a `UserId` where a `WorkspaceId` is expected — a compile-time bug class that's expensive at runtime.
- **No `as` casts** except: (a) const assertions (`as const`), (b) narrowing after a runtime check the compiler can't see (`x as NonEmpty<T>` after asserting `x.length > 0`).
- **Tagged unions over optional-field structs** (RULE TGU, applies to TypeScript too):
  ```ts
  // ❌ optional-field
  type Result = { ok: boolean; value?: T; error?: string };
  // ✅ tagged union
  type Result<T> = { ok: true; value: T } | { ok: false; error: string };
  ```

## §6 · File ordering

> [DETERMINISTIC → TSC]

```ts
// 1. Type-only imports
import type { Workspace, User } from "./types";

// 2. Value imports
import { z } from "zod";
import { Database } from "bun:sqlite";

// 3. Module-local types
type Internal = { … };

// 4. Module-level consts
const TIMEOUT_MS = 5_000;
const STATEMENTS = { listAll: "SELECT * FROM ws" } as const;

// 5. Main export (class | factory | functions-module)
export class WorkspaceStore { … }

// 6. Helpers (private to module)
function parseRow(row: unknown): Workspace { … }
```

Side-effect imports land **after** value imports if order matters; otherwise group with value imports.

## §7 · Naming

> [DETERMINISTIC → TSC]

| Construct | Convention | Example |
|---|---|---|
| Variable / function | camelCase | `parseWorkspace`, `cachedTokens` |
| Class / type / interface | PascalCase | `WorkspaceStore`, `RouteHandler` |
| Enum & enum members | PascalCase / PascalCase | `Status.Pending`, `Status.Active` |
| Module-level tunable constants | SCREAMING_SNAKE | `MAX_RETRIES`, `TIMEOUT_MS` |
| Filename | kebab-case matching default-ish export | `workspace-store.ts` |
| Test file | `<unit>.test.ts` (Bun convention) | `workspace-store.test.ts` |
| Private class field | `#name` (real private, not `_name`) | `#db`, `#cache` |

## §8 · `bun:test` discipline

> [DETERMINISTIC → TODO-CHECK]

- **One test file per source file**, sibling-named (`workspace-store.test.ts` next to `workspace-store.ts`). Mirrors RULE TST-NAM.
- **Use `bun:test` exports** — `test`, `expect`, `describe`, `beforeEach`, `afterEach`, `mock`. Never import from `vitest`/`jest` even if the API looks identical.
- **No milestone IDs in test names** (RULE TST-NAM). Tests describe behaviour, not provenance.
- **Integration tests opt out via filename**: `workspace-store.integration.test.ts` — Bun pattern for filtered runs (`bun test --test-name-pattern '\.integration\.'`).
- **Snapshot tests are forbidden by default.** They drift, the diff is opaque, and "looks right" isn't a behaviour assertion. Only land a snapshot when you genuinely have no other way to assert (rare).

## §9 · Error handling

> [JUDGMENT → TSJ]

Pick one style per module — never mix:

- **Throw** (`throw new TypeError(...)`) when the caller cannot reasonably handle the error and the program should die or unwind.
- **Result-style** (`type Result<T,E> = { ok: true; value: T } | { ok: false; error: E }`) when the caller routinely handles the failure and a thrown error would force every caller to wrap in `try/catch`.

In a Result-style module, **never throw**. In a throw-style module, **never return `Result<…>`**. Mixing forces every caller to know which mode each function is in — unmaintainable.

`Error` subclasses must be named after *what is wrong*, not *when it happened* (mirrors RULE NLG): `WorkspaceNotFound`, not `LegacyWorkspaceLookupError`.

## §10 · Anti-patterns (named, banned)

> [DETERMINISTIC → TSC]

| Pattern | Why banned |
|---|---|
| `let` for "I might change it later" | Future-tense reassignment is a bug pretending to be flexibility. |
| `any` to silence the type-checker | The check exists to find this. |
| `// @ts-ignore` / `// @ts-expect-error` without an issue link | Comments rot; a permanent `@ts-ignore` is a permanent lie. |
| `!` non-null assertion (or `as any` / `as unknown as T`) to silence strictness | Strict mode exists to catch null/undefined — that is the migration's whole intent. "Could be undefined" is a found bug: handle with `??` / `if` / early return / array-method narrowing, never assert it away. `noUncheckedIndexedAccess` returning `T \| undefined` for `arr[i]` is the feature, not the obstacle. |
| Default exports for application code | Renames cascade. Imports lie about what they import. |
| Re-exporting everything from `index.ts` | Hides dependency graph, defeats tree-shaking. Re-export only what's part of the public surface. |
| `Object.assign` for shallow merging | Use spread (`{ ...a, ...b }`). `Object.assign` mutates the first arg — bug class waiting to happen. |
| Top-level `console.log` left in source | Use a real logger, or remove before commit. Audit catches this in pre-commit if a logger gate exists. |
| `process.env.X` reads scattered through code | Read once at module init, freeze, export typed. Scattered reads make config opaque. |

## §11 · Runtime resource discipline

> [DETERMINISTIC → TODO-CHECK]

TypeScript makes unbounded work look cheap. Bun will still allocate, queue, block, and leak process handles if the code shape is sloppy.

### Promise concurrency

- **No unbounded `Promise.all(items.map(...))`** on data from users, files, network, databases, or command output. Use a concurrency limiter, chunked loop, or streaming transform.
- **Every fan-out declares its bound** near the call site: `MAX_CONCURRENT_FETCHES`, `MAX_PARALLEL_FILES`, etc. Numeric literals follow RULE UFS in §2.
- **Retry loops are bounded**: max attempts, delay/backoff, jitter when multiple clients may retry, and an abort path.
- **Long-running async work accepts an `AbortSignal`** or has an equivalent owner-controlled cancellation path.
- **`Promise.race` timeouts cancel the loser.** A timer-only race that leaves work running is a leak.

### Memory shape

- **Do not materialize unknown-size input.** Avoid `await Bun.file(path).text()`, `response.text()`, `response.json()`, or `Array.from(...)` when size is not bounded. Stream, chunk, or reject at a named limit.
- **Do not accumulate transform results by default.** Prefer `for await` streaming or chunked processing for large file, HTTP, or command-output flows.
- **Large JSON parse/stringify is both memory and Central Processing Unit (CPU) work.** Put it behind a size limit or move it out of request-critical code.
- **Clear resource handles in `finally`.** Timers, intervals, subprocesses, file handles, and temporary listeners must be cleaned even on thrown errors.

### Timeout and cancellation

Every network request, subprocess wrapper, file watcher, long poll, or task queue wait added by a diff needs a timeout or cancellation owner.

```ts
const controller = new AbortController();
const timeout = setTimeout(() => controller.abort(), REQUEST_TIMEOUT_MILLISECONDS);
try {
  return await fetch(url, { signal: controller.signal });
} finally {
  clearTimeout(timeout);
}
```

Use a local helper only when it preserves the same visible ownership: who starts the timer, who aborts, and who clears it.
Returning a `Response` from a helper transfers body consumption outside this timeout; only do that when the caller owns a second timeout.

### Bun subprocess cleanup

- `Bun.spawn` callers must consume, redirect, or intentionally ignore stdout/stderr. Do not leave pipes unread.
- Every spawned process is awaited, killed on timeout, or tied to an owner cleanup path.
- Wrapper tests cover the timeout/kill path, not just the success exit code.

### CPU-bound work

- CPU-heavy synchronous work does not live directly in request handlers without a named bound. Examples: large JSON serialization, hashing loops, compression, directory walks, and bulk formatting.
- Worker usage declares input size, concurrency, termination, and error propagation. A worker that can fail silently is a hidden hang.
- Prefer one parse/config load at module startup over repeated parse inside handlers or command loops.

## §12 · Override syntax

> [JUDGMENT → TSJ]

Per-rule override:

```
BUN RULE <§N>: SKIPPED per user override (reason: ...)
```

immediately preceding the edit. **User-invokable only.** Generic "scope creep" / "save for later" not valid — name a concrete external constraint (third-party-API shape, framework requirement, dependency mismatch).

## §13 · Family

> [DETERMINISTIC → FLL]

- Universal rules (RULE UFS, RULE TGU, RULE PRI, RULE FLL, RULE ORP, RULE TST-NAM) live in `.orly/docs/greptile-learnings/RULES.md` and apply to TypeScript via the **GREPTILE GATE**.
- Frontend visual primitives live in the UI Component Substitution section of this façade (UI GATE).
- Length caps live in `.orly/dispatch/write_any.md` (LENGTH GATE — file ≤ 350, fn ≤ 50, method ≤ 70 — same as every other source language).
- This file is the **TypeScript / Bun-specific** layer that those universal rules cannot express. Scan its section headers at the first TS/JS touch (`grep -nE "^##+ "` — two-plus hashes, so nested cards surface) and read the sections the diff shape touches; on sub-task shape change, re-scan and read only newly-touched sections — never the full file.

## Repository design system

> [JUDGMENT → ARCH]

Use the consuming repository's declared component library and design tokens.
Locate its exports and styles before substituting primitives; do not import
another product's package or invent its directory layout.
Run the repository's declared checks and review substitutions against that source.
