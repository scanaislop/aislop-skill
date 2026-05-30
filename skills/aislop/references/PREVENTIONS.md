# Prevention catalog — write code that won't trip aislop

Read this before editing. Every pattern below is a rule aislop will flag.

The avoid/prefer examples are illustrative — substitute the project's actual tooling.

## Comments — write code, not prose

`ai-slop/narrative-comment` — `ai-slop/trivial-comment` — `ai-slop/meta-comment`

Avoid: `// This function takes a user ID and returns the user object` — the signature already says this.
Avoid: `// Added for the checkout flow — see issue #123` — belongs in commit message.
Avoid: `// Refactored to use the new API` — belongs in commit message.
Avoid: `// Loop through each item and check if it matches` — the code shows this.

Prefer: `// Keep this before the CORS middleware — credentialed origins reject OPTIONS otherwise.`
Prefer: `// Workaround for knex#1234: bindings silently drop NaN.`

Rule: if deleting the comment wouldn't confuse a future reader, don't write it.

## Types — no `as any`, no `as unknown as X`

`ai-slop/unsafe-type-assertion` — `ai-slop/double-type-assertion` — `ai-slop/ts-directive` — `ai-slop/redundant-type-coercion`

Avoid: `const user = res.data as any;` — hides the real type.
Avoid: `const val = x as unknown as string` — `as any` in disguise.
Avoid: `// @ts-ignore` — you're suppressing a type error, not fixing it.
Avoid: `!!x`, `"" + x`, `+x` where the type system already handles it.

Prefer:
- Declare or derive the real type and annotate the variable.
- If the shape is unknown, use `unknown` plus the project's runtime validator.
- `@ts-expect-error` only with a specific reason comment and only when the type error is genuinely unavoidable (rare).

## Errors — never swallow

`ai-slop/swallowed-exception` — `ai-slop/silent-recovery` — `ai-slop/redundant-try-catch`

Avoid: `try { await thing(); } catch (e) {}` — swallowed, invisible.
Avoid: `catch (e) { log.error(e); return []; }` — silent recovery. The failure is hidden.
Avoid: `catch (e) { throw e; }` — you're not handling the error, don't wrap it.

Prefer: log it via the project's logger, rethrow wrapped in the project's error class, or rethrow as-is. Fire-and-forget is acceptable only when you log (even at warn). Match the catch shape neighbouring code uses.

## Security — use the safe API from the start

`security/sql-injection` — `security/eval` — `security/innerhtml` — `security/hardcoded-secret` — `security/shell-injection` — `ai-slop/hardcoded-url` — `ai-slop/hardcoded-id`

Avoid: `` db.query(`SELECT * FROM users WHERE id = ${id}`) `` — template-string SQL injection.
Prefer: `db.query("SELECT * FROM users WHERE id = $1", [id])` or the ORM builder.

Avoid: `eval(expr)` / `new Function(body)` on user input.
Prefer: `JSON.parse(expr)` for data. Don't execute user code at all.

Avoid: `el.innerHTML = html` on untrusted input.
Prefer: `el.textContent = text`, or sanitise with DOMPurify.

Avoid: `const apiKey = "sk-abc123"` in source.
Prefer: `const apiKey = process.env.OPENAI_API_KEY` — fail loudly if unset.

Avoid: `const apiUrl = "https://api.example.com/v2"` hardcoded in source.
Prefer: Read from environment or config. Use a named constant in a config module.

Avoid: `const orgId = "org_abc123"` hardcoded in business logic.
Prefer: Accept as a parameter, read from config, or derive from context.

## Duplication — search before you write

`code-quality/duplicate-block` — `knip/exports`

- Before writing a new helper, grep the codebase for the operation.
- If two blocks do the same thing with tiny variations, extract once and parameterise the variation.
- If you did extract a helper, make sure it gets used — an unused export is its own rule.

## Dead code — don't leave it

`ai-slop/unused-import` — `code-quality/unused-declaration` — `ai-slop/unreachable-code` — `ai-slop/constant-condition` — `ai-slop/hallucinated-import` — `ai-slop/duplicate-import`

- Remove imports as you remove the code that used them — not later.
- If you wrote a function and ended up not using it, delete it. Don't "leave it in case".
- Commented-out code belongs in git history, not in the file.
- Statements after `return`/`throw`/`break`/`continue` are dead — delete them.
- `if (true)`, `if (false)`, `while (false)` — remove the branch or make the condition real.
- Verify every import resolves in the actual filesystem before writing it. AI agents hallucinate import paths.
- Combine duplicate imports of the same symbol into one statement.

## Console — use the project's logger

`ai-slop/console-leftover`

- Grep for `pino`, `winston`, `bunyan`, `logger.ts`, `log.ts`, or a shared wrapper and use it.
- If the project has no logger and it needs one, pick one that fits the stack and add it.
- If it's a script or CLI where `console.error` is the convention, keep `console.error`. `console.log` is always flagged.
- Never leave debug logs committed. Delete before the final scan.

## TODO stubs — finish or file

`ai-slop/todo-stub`

Avoid: `// TODO: handle error case`

Prefer:
- Finish it in the same turn.
- If genuinely out of scope: `// TODO(#234): handle the retry path — currently we fail-fast.` with a real issue number.

## Empty functions — stub or finish

`ai-slop/empty-function`

Avoid: `function onSuccess() {}` — empty handler.
Avoid: `async function migrate() { throw new Error("not implemented"); }` — stub that compiles but does nothing.

Prefer: Implement the function with real behavior. If the task is out of scope, remove the function entirely.

## Naming — name for intent

`ai-slop/generic-naming`

Avoid: `const data = await fetch(...)`, `function handleClick()`, `const temp = items.filter(...)`

Prefer: `const orders = await fetchOrders(...)`, `function confirmDeletion()`, `const unpaidInvoices = items.filter(...)`

The name should tell the reader what's in it, not the type or how it was derived.

## Function and file size — split at the logical seam

`complexity/function-too-long` (default max 80 LOC) — `complexity/file-too-large` (default max 400 LOC; JSX/TSX 2x) — `complexity/deep-nesting` — `complexity/too-many-params`

- Write short functions from the start. If you're over 40 lines, there's usually a seam.
- Extract the seam — a group of statements that does one sub-task and could be named.
- A file should have one cohesive responsibility.
- If you need 7+ parameters, pass an options object.
- Deep nesting → early returns: `if (!a) return; if (!b) return;`.

## Architecture — respect `.aislop/rules.yml`

If the project has `.aislop/rules.yml` (custom architecture rules — import bans, layering, module boundaries), open it before writing. A rule you break now is a finding you'll have to undo later.

## Dependencies — use what's there

`knip/dependencies` — `knip/unlisted` — `security/vulnerable-dependency`

- Before adding a package, check `package.json` — the project often already has a library.
- Never install a package for a 5-line utility.
- If `pnpm audit` / `npm audit` flags a vulnerability with a clean upgrade, take it.

## Fake completeness — no placeholders dressed as features

Scans catch many textual patterns but cannot always tell whether code is only pretending to be complete.

Avoid:
- Hardcoded success paths: `return true`, `return []`, fake IDs, canned API responses.
- "Temporary" fallback behavior that silently hides integration failures.
- UI or API handlers that implement only the happy path.
- Tests that assert only that something renders or returns the same fixture it was given.

Prefer:
- Wire the real dependency or fail loudly with the project's error pattern.
- Cover at least one success path and one meaningful failure or edge path.
- Delete scaffolding that is no longer needed once the real implementation exists.

## Abstractions — don't future-proof imaginary callers

`ai-slop/thin-wrapper` — `ai-slop/duplicate-type-declaration` — `ai-slop/redundant-type-coercion`

AI code often creates generic layers before the repo needs them.

Avoid:
- New factories, registries, strategy objects, adapters, hooks, or base classes with one caller.
- Options objects full of unused flags.
- Helpers exported from barrels before any external module imports them.
- Wrappers that do nothing except delegate with the same signature — inline the call.

Prefer:
- Keep logic local until there are at least two real callers or an existing project pattern requires extraction.
- Name extracted helpers after the concrete domain behavior, not the implementation technique.
- Reuse existing type declarations — import them if they're in another file.

## Wiring — finish the path, not just the file

Before calling the task done, check:
- Public API: route, command, export, screen, or handler is reachable from the intended entry point.
- Types and generated artifacts: schemas, clients, migrations, snapshots, docs, or indexes are updated.
- State transitions: loading, empty, error, retry, permission-denied, and success states are coherent.
- Backwards compatibility: existing callers still compile and keep their previous behavior.
- Observability: important failures are traceable through the project's logger, metrics, or error surface.
