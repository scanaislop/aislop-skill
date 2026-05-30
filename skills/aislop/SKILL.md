---
name: aislop
description: Code-quality gate and coding guardrail for AI coding agents. Always invoke when you finish editing and prepare to hand control back, even if the user did not explicitly ask. Trigger on commit, push, PR, or quality questions like any slop, is this clean, review my changes, score my code, any duplicates, or is this safe. Scans with aislop, fixes what is mechanical, addresses the rest in-session in the project style, then reports what changed.
---

# aislop — agent skill

[aislop](https://github.com/scanaislop/aislop) is a deterministic, open-source CLI that scans across six engines, grades 0–100, and catches 50+ rules across 7 languages: swallowed exceptions, hallucinated imports, silent recovery, narrative comments, `as any` casts, oversized functions, SQL injection, eval, hardcoded secrets, and architecture drift.

**If you follow this skill, you don't ship slop.** It does two jobs:

1. **Prevention** — internalise the rule catalog below so your first draft doesn't trip detectors.
2. **Detection + fix** — scan after editing, fix mechanical findings with the CLI, fix the rest yourself, report what changed.

This skill is not "run `aislop` and report back" — the user could do that themselves. The value is the judgement layer: writing right the first time, reading each finding against source, and fixing what you can before escalating.

Supports TypeScript, JavaScript, Python, Go, Rust, Ruby, PHP, Expo/React Native.

## Remote execution & supply-chain policy

This skill uses the `aislop` CLI, distributed through the public npm registry. Running `npx aislop` fetches and executes the published package — treat this as an untrusted remote dependency.

**Default: prefer a locally installed, version-pinned aislop over `npx`.**

```bash
# Preferred — pinned in the project's lockfile
npm install --save-dev aislop && aislop scan --changes --json

# Acceptable fallback
npx aislop scan --changes --json

# Never acceptable: GitHub clones, curl|sh installers, unpinned npx in CI
```

Decision order:
1. **Local binary or script** in `devDependencies` — pinned by the lockfile.
2. **`npx aislop@<version>`** (pinned) — acceptable when no local install exists.
3. **`npx aislop`** (floating) — last resort for interactive sessions. Never in CI.
4. **Blocked environment** — skip the scan and report the CLI could not be run. Do not install outside the project's dependency workflow.

`aislop fix --claude` (and `--cursor`, `--codex`, `--gemini`) executes a local process against the project's code only — no additional network fetch.

## Follow the project's conventions, not this skill's examples

This skill tells you which anti-patterns to avoid. It does NOT tell you the one right way to structure your code — that's encoded in the repo. Before acting on any prevention pattern, check what the project actually does:

- **Logger** — grep for `pino`, `winston`, `bunyan`, `logger.ts`, or a shared wrapper. If `console.error` is the convention, fine.
- **Error wrapping** — use the project's error class (`AppError`, `DomainError`, `HttpError`, or bare `Error` with `{ cause }`).
- **Types / validation** — match the project's choice (`zod`, `valibot`, `ajv`, pydantic, Go structs, etc.).
- **Naming** — follow the project's casing, file naming, and folder layout.
- **Tests** — use the framework and patterns already in the repo.
- **Architecture** — if `.aislop/rules.yml` exists, it's authoritative. Otherwise, match import patterns and module boundaries from existing files.

The avoid/prefer examples in the prevention catalog are illustrative. A fix in the project's style beats a fix in this skill's style.

## Quick prevention catalog

Read this before editing. Each rule aislop will flag. See [`references/PREVENTIONS.md`](references/PREVENTIONS.md) for full avoid/prefer examples.

Categories: Comments, Types, Errors, Security, Dedup, Dead code, Console, TODO, Empty functions, Naming, Size, Architecture, Deps, Fake completeness, Abstractions, Wiring.

## When to invoke

Invoke when you've finished editing and are handing control back, the user is preparing to **commit/push/open a PR**, or they ask any quality question: "any slop", "is this clean", "any duplicates", "is this reusable", "can we dedupe this", "is this too long", "is this safe", "audit my deps", "does this follow our rules", "score my code", "score dropped", "add a badge".

Skip when: user is only reading, is mid-refactor and asked you to hold off, or explicitly disabled aislop.

## CLI commands reference

| Command | Use |
|---|---|
| `aislop scan [dir]` | Full scan. Flags: `--changes`, `--staged`, `--json`, `--sarif` |
| `aislop fix [dir]` | Auto-fix mechanical issues. `-f` for aggressive. `--claude`/`--codex`/`--cursor`/`--gemini` for agent hand-off. `-p` to print prompt. |
| `aislop ci [dir]` | CI mode: JSON output + exit code by threshold |
| `aislop init [dir]` | Generate `.aislop/config.yml` |
| `aislop doctor [dir]` | Check installed tools and environment |
| `aislop rules [dir]` | List all rules with severity and fixability |
| `aislop trend [dir]` | Show score history from `.aislop/history.jsonl` |
| `aislop badge [dir]` | Generate a public score badge URL + README markdown |
| `aislop hook install` | Install per-edit quality hooks. Supports Claude Code, Cursor, Codex, Gemini, pi, and others (run `aislop hook install --help` for the full list). |

## Core workflow

For every command, use `aislop` (local install, preferred) or `npx aislop` (fallback).

### 1. Scope the scan

```bash
aislop scan --changes --json     # mid-session: only what you touched
aislop scan --staged --json      # before a commit
aislop scan --json               # full project: pre-release, PR, wide refactor
```

Always `--json`. The TTY output is for humans; you need structured findings.

### 2. Auto-fix the mechanical findings

```bash
aislop fix
```

CLI handles everything marked `fixable: true`. Don't hand-edit what the CLI fixes. Use `-f` only when the user asked to clean up deps or unused files.

After auto-fix, optionally hand off remaining findings:
```bash
aislop fix --claude          # launch Claude Code with remaining findings
aislop fix --codex           # launch Codex CLI
aislop fix -p                # print a prompt to paste into any agent
```

### 3. Re-scan — everything fixable:false is your job

```bash
aislop scan --changes --json
```

The CLI has done its half. Remaining `fixable: false` findings are what this skill exists for.

### 4. For each remaining finding, do the judgement work

**a. Verify it's real.** Open the file. Read the cited line and context. Is the rule description accurate? Could it be a false positive (regex matched inside a string/comment/identifier)? Is there a reason the code is intentional?

**b. Decide:**

| What you found | Action |
|---|---|
| Real issue | **Fix it in-session.** Don't ask first. |
| False positive | Surface with file:line and one-sentence reason. Do not silence the rule. |
| Legitimately intentional | Note it. Let the user decide on suppression. |

**c. Fix it.** See [`references/FIX_GUIDE.md`](references/FIX_GUIDE.md) for the fix pattern for each rule. Open the file, verify the finding, apply the fix.

### 5. Run the manual slop pass

A clean scan is not enough. Review your diff for patterns the CLI cannot infer:

1. **Fake completeness** — no hardcoded success paths, canned data, placeholder behavior.
2. **Test quality** — tests exercise at least one edge/failure path, not just rendering.
3. **Unneeded abstraction** — no new generic layer with one caller.
4. **Incomplete wiring** — exports, routes, generated files, call sites are updated.
5. **State handling** — loading, empty, error, timeout, cancellation states are coherent.
6. **Dependency discipline** — no new package for trivial logic.

Fix what you find. If something is genuinely out of scope, say why.

### 6. Re-scan until the bar is met

Loop steps 4–6 until:
- Zero `error`s.
- Zero `fixable: true` warnings.
- Every `fixable: false` warning is fixed, proposed with rationale, or flagged as false positive.
- Manual slop pass has no unresolved in-session issue.

Do not say "done" before this point.

### 7. Report — triaged, not dumped

Default voice is past-tense "I did X":

```
Ran aislop — 12 findings, score 73 → 95.

Auto-fixed (7): unused imports (3), narrative comments (2), formatting (2).

Fixed in-session (4):
  - unsafe-type-assertion       src/api/normalize.ts:47  (added ApiUser type)
  - function-too-long           src/lib/reconcile.ts:14  (extracted matchByHash)
  - sql-injection               src/db/search.ts:30      (parameterised with $1)
  - swallowed-exception         src/workers/cleanup.ts:28 (added log.warn)

False positives (1):
  - security/eval               src/labels.ts:12
    The match is inside a display-name table, not an actual eval().

Re-scanned: 95 / 100, 0 errors, 0 warnings.
```

If clean but manual pass found issues, report those too. If something needs a product-level call, put it at the end with the specific choice.

## How to verify a false positive

Before flagging, check:
1. Is the match inside a string literal, template literal, or comment?
2. Is it part of an identifier name that happens to contain the pattern?
3. Is it in a test fixture, mock data, or example?
4. Would a reader agree this isn't the pattern the rule describes?

Include a one-sentence reason. Never silence the rule on the user's behalf.

## Severity and score interpretation

| Severity | Fixable | Action |
|---|---|---|
| `error` | any | MUST fix this turn. |
| `warning` | `fixable: true` | CLI's `aislop fix` handles these. Never leave them. |
| `warning` | `fixable: false` | **Agent fixes in-session.** Verify, then edit. |
| `info` | any | Note; act only if the user asked for a high bar. |

Score bands: **90–100** healthy; **75–89** healthy with debt; **60–74** degraded; **< 60** failing gate.

## Default: invent, apply, report — don't ask

Make the fix and report what you did. Missing type → invent it. Swallowed error → log and rethrow. Oversized function → extract at the seam. Duplicate logic → extract a helper. Generic name → rename. Don't ask "should I" for any of these — just do it and mention what changed in the reply.

Escalate **only** when the fix is a product decision, would change observable semantics you shouldn't override, is a false positive worth flagging, or is legitimately intentional code the user should decide on.

Never ask permission for fixes you can make yourself. Pick the right tool and apply it.

## Anti-patterns

- Do NOT treat this skill as a wrapper around `aislop fix`. Your job is the judgement layer.
- Do NOT ask for permission on fixes you can make yourself.
- Do NOT paste raw JSON into your reply — triage and summarise.
- Do NOT silence rules in config or add blanket suppression comments.
- Do NOT delete `.aislop/config.yml` or `.aislop/rules.yml`.
- Do NOT claim completion without a post-fix re-scan.
- Do NOT treat a 100 score as proof the code is good — do the manual pass.
- Do NOT run `aislop fix -f` silently on unrelated turns — it rewrites manifests.
- Do NOT fight the detector by editing regex patterns — your job is clean code.
