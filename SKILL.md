---
name: aislop
description: Code-quality gate and coding guardrail for AI coding agents. Use when the user finishes a feature, prepares to commit / push / open a PR, or asks a quality question — "any slop", "any duplicates", "is this reusable", "is this maintainable", "is this safe", "review my changes", "score my code". Scans with aislop, fixes what is mechanical, addresses the rest in-session in the project's style, reports what changed.
---

# aislop — agent skill

[aislop](https://github.com/scanaislop/aislop) is a deterministic, open-source CLI that scans a project across six engines (format, lint, code-quality, ai-slop, security, architecture), grades it 0–100, and lists every finding with file, line, rule, severity, and a `fixable` flag. It catches 40+ AI-authored patterns that compile fine, pass tests, and still ship broken: duplicate logic, dead imports, narrative comments, `as any` casts, swallowed errors, oversized functions, `eval`, template-string SQL, `innerHTML`, hardcoded secrets, vulnerable deps, architecture drift.

**The promise of this skill: if you follow it, you don't ship slop.** It does two jobs:

1. **Prevention** — the *Write code that won't trip this scan* section below. Internalise those patterns and your first draft won't trip aislop's detectors.
2. **Detection + fix** — when you finish editing, scan, verify findings, fix the non-mechanical ones yourself using your editing tools, and report only what needs the user's decision.

This skill is not "run `aislop` and report back" — the user could do that themselves. The value is everything around the commands: writing right the first time, reading each finding against the source, and fixing what you can before escalating.

Supports TypeScript, JavaScript, Python, Go, Rust, Ruby, PHP, Expo/React Native.

## Follow the project's conventions, not this skill's examples

This skill tells you which **anti-patterns** to avoid. It does NOT tell you the one right way to structure your code — that's the project's decision, and it's already encoded in the repo. Before acting on any prevention pattern below, check what the project actually does and match it:

- **Logger** — don't assume `log.error` exists. Grep the repo for `pino`, `winston`, `bunyan`, `logger.ts`, `log.ts`, or a shared log wrapper and use that. If the project uses `console.error` by convention in its scripts, fine.
- **Error wrapping** — use whatever error class / cause pattern the project already uses (`AppError`, `DomainError`, `HttpError`, bare `Error`, `{ cause }`). Don't introduce a new one.
- **Types / validation** — match the project's choice (`zod`, `valibot`, `ajv`, TypeScript-only, Python `pydantic`, Go structs, etc.). Don't pull in a new validator.
- **Naming** — follow the project's casing (`camelCase` vs `snake_case` vs `kebab-case`), file naming (`user.service.ts` vs `userService.ts`), and folder layout.
- **Tests** — use the test framework and patterns already in the repo (vitest vs jest; `describe`/`it` vs table-driven; mocks vs fakes).
- **Architecture** — if `.aislop/rules.yaml` exists, it's authoritative. If it doesn't, read a few existing files in the area you're editing and match their import patterns and module boundaries.

The "prefer" examples in the prevention section below are illustrative. Substitute the project's actual tool, class, or function name wherever you see a generic one. A fix in the project's style beats a fix in this skill's style.

## Write code that won't trip this scan

Read this section before editing. Every pattern below is a rule aislop will flag — internalise the preventive version and your first draft passes. Remember the principle above: the shape is illustrative; use the project's actual tools.

### Comments — write code, not prose

`ai-slop/narrative-comment` and `ai-slop/trivial-comment` fire on comments that restate what the next line does or explain the history of the change.

Avoid:
- `// This function takes a user ID and returns the user object` — the signature already says this.
- `// Added for the checkout flow — see issue #123` — belongs in the PR description or commit message.
- `// Loop through each item and check if it matches` — the code shows this.

Prefer:
- `// Keep this before the CORS middleware — credentialed origins reject OPTIONS otherwise.` — non-obvious ordering constraint.
- `// Workaround for knex#1234: bindings silently drop NaN.` — specific bug, specific reason.

Rule: if deleting the comment wouldn't confuse a future reader, don't write it.

### Types — no `as any`, no `as unknown as X`

`ai-slop/as-any-cast` fires on every `as any`. Find the real type up-front.

Avoid: `const user = res.data as any; user.name.toUpperCase();`

Prefer:
- Declare or derive the real type and annotate the variable.
- If the shape is unknown, use `unknown` plus the project's runtime validator (check for `zod`, `valibot`, `ajv`, pydantic, or whatever is already in the repo).
- For third-party libraries without types, install `@types/<lib>` or add a minimal `.d.ts` in the project's existing types location.

### Errors — never swallow

`ai-slop/swallowed-exception` fires on empty `catch` blocks or ones that do nothing with the error.

Avoid: `try { await thing(); } catch (e) {}`

Prefer: log it via the project's logger, rethrow wrapped in the project's error class, or rethrow as-is. Fire-and-forget is acceptable only when you log (even at warn) so a future debugger can trace it. Match whatever pattern the neighbouring code uses — if the file already has a particular catch shape, use that.

### Security — use the safe API from the start

Avoid: `db.query(\`SELECT * FROM users WHERE id = ${id}\`)` — template-string SQL.
Prefer: `db.query("SELECT * FROM users WHERE id = $1", [id])` or the ORM builder.

Avoid: `eval(expr)` / `new Function(body)` on user input.
Prefer: `JSON.parse(expr)` for data. Don't execute user code at all.

Avoid: `el.innerHTML = html` on untrusted input.
Prefer: `el.textContent = text`, or sanitise with DOMPurify.

Avoid: `const apiKey = "sk-abc123"` in source.
Prefer: `const apiKey = process.env.OPENAI_API_KEY` — fail loudly if unset.

### Duplication — search before you write

`ai-slop/duplicate-code` and `code-quality/unused-export` fire on copy-paste and on extracted helpers nobody imports.

- Before writing a new helper, grep the codebase for the operation. If a function for it exists, import it.
- If two blocks do the same thing with tiny variations, extract once and parameterise the variation.
- If you did extract a helper, make sure it gets used — an unused export is its own rule.

### Dead code — don't leave it

`ai-slop/unused-import`, `ai-slop/unused-variable`, `ai-slop/unused-declaration` all fire on leftovers.

- Remove imports as you remove the code that used them — not later, not at the end.
- If you wrote a function and ended up not using it, delete it. Don't "leave it in case".
- Commented-out code belongs in git history, not in the file.

### Console — use the project's logger (or add one)

`ai-slop/console-log` fires on leftover `console.log`.

- Grep for `pino`, `winston`, `bunyan`, `logger.ts`, `log.ts`, or a shared wrapper and use it.
- If the project has no logger and it needs one, pick a sensible one that fits the stack and add it yourself — don't ask the user first. Note the addition in your reply.
- If it's a script or CLI where `console.error` is the convention, keep `console.error`. `console.log` is flagged regardless.
- Never leave debug logs committed. Delete before the final scan.

### TODO stubs — finish or file

`ai-slop/todo-stub` fires on `// TODO` without an owner or ticket.

Avoid: `// TODO: handle error case`

Prefer:
- Finish it in the same turn.
- If genuinely out of scope: `// TODO(#234): handle the retry path — currently we fail-fast.` with a real issue number.

### Naming — name for intent

`ai-slop/generic-naming` fires on `data`, `result`, `temp`, `handleClick`, `thing`, `stuff`.

Avoid:
- `const data = await fetch(...)`
- `function handleClick()`
- `const temp = items.filter(...)`

Prefer:
- `const orders = await fetchOrders(...)`
- `function confirmDeletion()`
- `const unpaidInvoices = items.filter(...)`

The name should tell the reader what's in it, not the type or how it was derived.

### Function and file size — split at the logical seam

`complexity/function-too-long` (default max 80 LOC), `complexity/file-too-large` (default max 400 LOC; JSX/TSX 2x), `complexity/max-nesting`, `complexity/max-params`, `complexity/high-complexity`.

- Write short functions from the start. If you're over 40 lines, there's usually a seam.
- Extract the seam — a group of statements that does one sub-task and could be named. Don't extract arbitrarily to hit a line count.
- A file should have one cohesive responsibility. If you're reaching for a second "section" divider, split it.
- If you need 7+ parameters, pass an options object: `{ user, org, branch, ...opts }`.
- Deep nesting (`if (a) { if (b) { if (c) { ... } } }`) → early returns: `if (!a) return; if (!b) return; if (!c) return; ...`.

### Architecture — respect `.aislop/rules.yaml`

If the project has `.aislop/rules.yaml` (custom architecture rules — import bans, layering, module boundaries), open it before writing. A rule you break now is a finding you'll have to undo later.

### Dependencies — use what's there

`code-quality/unused-dependency`, `code-quality/missing-dependency`, `security/vulnerable-dependency`.

- Before adding a package, check `package.json` — the project often already has a library for the job.
- Never install a package for a 5-line utility.
- If `pnpm audit` / `npm audit` flags a vulnerability with a clean upgrade, take it. If it needs an override, stage that via `aislop fix -f`.

## When to invoke

Invoke whenever any of these is true:

1. You have finished editing code and are about to hand control back.
2. The user is preparing to **commit, push, open a PR, merge, or ship**.
3. The user asks a quality-gate question: "is this ready", "is this clean", "any slop", "score this".
4. The user asks a **deduplication** question: "any duplicate code", "can we dedupe this", "is this already somewhere else".
5. The user asks a **reusability** question: "can we reuse this", "should this be a helper", "are we duplicating work".
6. The user asks a **maintainability** question: "is this too long", "should I split this", "too much nesting".
7. The user asks a **security** question: "is this safe", "any sql injection", "any leaking secrets", "audit my deps".
8. The user asks an **architecture** question: "does this follow our rules", "any layering violations".
9. The user references a quality regression ("score dropped", "why is the PR red").
10. A CI job running `aislop ci` failed and the user wants it green.

### When to skip

- User is only reading code; hasn't edited.
- User is mid-refactor and asked you to hold off on checks.
- User explicitly disabled aislop for the turn.

## Core workflow — your job, not the CLI's

For every command in this workflow, run `npx aislop <command> --help` first if you need the exact flag syntax — this skill stays focused on the judgement layer.

### 1. Scope the scan — match the blast radius

```bash
npx aislop scan --changes --json     # default mid-session: only what you touched
npx aislop scan --staged --json      # right before a commit
npx aislop scan --json               # full project: pre-release, PR, wide refactor
```

Always `--json`. The TTY output is for humans; you need structured findings to act on.

### 2. Auto-fix the mechanical findings

```bash
npx aislop fix
```

Let the CLI handle everything marked `fixable: true`. Don't hand-edit things the CLI will fix mechanically. Use `aislop fix -f` only when the user asked to clean up dependencies or unused files.

### 3. Re-scan — everything with `fixable: false` is your job

```bash
npx aislop scan --changes --json
```

The CLI has done its half. Everything still in the output — the `fixable: false` findings — is what this skill exists for. That's where the agent earns its keep: verifying the finding is real and fixing it in-session.

### 4. For each remaining finding, do the judgement work

**a. Verify the finding is real.** Open the file. Read the cited line and a few lines of context. Ask:
- Does the rule description actually apply to this line?
- Could this be a false positive? (regex-based rules sometimes match inside strings, comments, or identifier names that happen to contain the pattern)
- Is there a reason the existing code is intentional?

**b. Decide what to do:**

| What you found | Action |
|---|---|
| Real issue (any shape — obvious or structural) | **Fix it in-session using your editing tools.** This is the skill's core work. Don't ask the user first. |
| False positive (regex-matched a string literal, comment, or identifier name — not real code) | Surface in your reply with file:line and one sentence explaining why it's a false positive. Do not silence the rule in `.aislop/config.yaml`. |
| Legitimately intentional (rare — pre-existing code the user clearly wants kept) | Note it in your reply with file:line. Let the user decide whether to add a local suppression. |

The old reflex "ask the user before doing a non-obvious fix" doesn't belong here. If the finding is real, fix it. Only escalate when there's a genuine *product* decision — see **Default: invent, apply, report** below.

**c. Do the fix.** You have editing tools. Typical non-mechanical fixes:

- `ai-slop/as-any-cast` — find the real type; if it's a third-party shape, import or derive it. Only keep `as any` if there's no better shape — and surface that.
- `ai-slop/swallowed-exception` — add `log.error(e)` / `console.error(e)` / rethrow. Swallow only with an explicit `// intentional: <reason>` comment.
- `complexity/function-too-long` — extract a helper with a clear name. Find the logical seam, don't break in the middle.
- `complexity/file-too-large` — split by concern, not line count.
- `security/sql-injection` — swap template string for parameterised query. Verify the driver's placeholder syntax before editing.
- `security/eval` / `security/inner-html` — replace with safe API (`JSON.parse`, `textContent`, DOMPurify).
- `code-quality/unused-export` — check if it's part of the module's public API; wire it up at the call site or delete.
- `ai-slop/duplicate-code` — extract to a shared helper. Pick the name carefully.

### 5. Re-scan until the bar is met

Loop steps 4–5 until:
- Zero `error`s.
- Zero `fixable: true` warnings.
- Every remaining `fixable: false` warning has been fixed by you, proposed in your reply with rationale, or flagged as a false positive with explanation.

Do not say "done" before this point.

### 6. Report — triaged, not dumped

Your reply is a triage summary, not a copy-paste of the JSON. Default voice is past-tense "I did X" — not "should I do X?":

```
Ran aislop — 12 findings, score 73 → 95.

Auto-fixed (7): unused imports (3), narrative comments (2), formatting (2).

Fixed in-session (4):
  - ai-slop/as-any-cast            src/api/normalize.ts:47   (added ApiUser type)
  - complexity/function-too-long   src/lib/reconcile.ts:14   (extracted matchByHash helper)
  - security/sql-injection         src/db/search.ts:30       (parameterised with $1)
  - ai-slop/swallowed-exception    src/workers/cleanup.ts:28 (added log.warn — still fire-and-forget, now traceable)

False positives (1):
  - security/eval                  src/labels.ts:12
    The match is the string "eval() call" inside a display-name table, not an actual eval(). Left as-is.

Re-scanned: 95 / 100, 0 errors, 0 warnings.
```

What the CLI auto-fixed. What you fixed yourself, with the specific change. False positives with the one-line reason. Final score.

If something genuinely needs a product-level call, put it in its own section at the end and be specific about what the choice is — but default to having already made the call and reporting it.

## How to verify a false positive

Before flagging one, check:

1. Is the match inside a string literal, template literal, or comment? Regex rules don't always distinguish code from strings.
2. Is the match part of an identifier name that happens to contain the pattern? `evaluateScore()` is not `eval()`; `consoleView` is not `console.log`.
3. Is the match in a test fixture, mock data, or example? Sometimes intentional patterns show up in test files.
4. Would a reader of the code agree this isn't the pattern the rule describes? If yes, it's a false positive. If you have to squint, it's not — fix it.

Include a one-sentence reason when surfacing. Never silence the rule on the user's behalf.

## Severity and score interpretation

| Severity | Fixable | Action |
|---|---|---|
| `error` | any | MUST fix this turn. No exceptions. |
| `warning` | `fixable: true` | The CLI's `aislop fix` handles these. Never leave them. |
| `warning` | `fixable: false` | **The agent fixes this in-session.** This is where the skill earns its keep. Verify, then edit the file. |
| `info` | any | Note; act only if the user asked for a high bar. |

Score bands: **90–100** healthy; **75–89** healthy with debt — fix errors + fixable warnings; **60–74** degraded — structural findings likely; **< 60** failing gate (`aislop ci` exits non-zero at default threshold 70).

## Default: invent, apply, report — don't ask

The agent is here to *do* the work, not hand choices back. When there's a sensible fix, make it, and report what you did in the reply. Do not ask the user for permission on anything you can decide yourself:

- Missing type → invent the type, apply it, mention the type name in the reply.
- Missing logger → pick one that fits the stack, add it, mention the choice in the reply.
- Swallowed error → add a log call and rethrow (or wrap) in the project's error style. Apply. Mention.
- Duplicate logic → extract to a helper with a good name. Apply. Mention.
- Oversized function → extract at the logical seam. Apply. Mention.
- Generic variable name → rename to something that reads. Apply. Mention.
- `// Should I rename this?` — don't. Just rename.

Escalate to the user **only** when there's a genuine branch the user has to pick:

- The fix is a **product decision** (what the behaviour should actually be — not which library to use).
- The fix would **change observable semantics** the user should review (e.g. an error that used to bubble now gets swallowed intentionally, or a security change alters who has access).
- The finding is a **false positive** worth flagging (with reason) so the user knows you saw it.
- The finding is **legitimately intentional** (the user's code, not yours) and the user should decide whether to add a local suppression.

Never:
- Ask "should I delete this unused import / fix this typo / rename this variable" — just do it.
- Ask "which library should I use" — pick the one that fits the project and say why.
- Ask "is this too long to extract" — if it trips the rule, extract.

## Anti-patterns — do not do these

- Do NOT treat this skill as a wrapper around `aislop fix`. The CLI does mechanical fixes; your job is the judgement layer.
- Do NOT ask the user for permission on a fix you can make yourself. Invent, apply, report.
- Do NOT paste raw JSON into your reply. Triage and summarise.
- Do NOT surface every finding to the user as if it were a question. Most are your job to resolve.
- Do NOT silence rules in `.aislop/config.yaml` to make a scan pass. Fix the issue or flag as a false positive with a reason.
- Do NOT delete `.aislop/config.yaml` or `.aislop/rules.yaml`.
- Do NOT add blanket `// aislop-disable` comments. Suppress the narrowest line with an explicit reason.
- Do NOT re-introduce a duplicate after `fix` extracted the shared version.
- Do NOT claim the task is complete without a post-fix re-scan.
- Do NOT run `aislop fix -f` silently on unrelated turns — it rewrites dependency manifests and can delete files.
- Do NOT rewrite narrative comments you just deleted.
- Do NOT fight the detector by editing regex patterns in the source. Your job is clean code, not a quieter detector.
