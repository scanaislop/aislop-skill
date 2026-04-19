# aislop skill

**Code-quality gate and coding guardrail for AI coding agents.** A drop-in skill that turns [aislop](https://github.com/scanaislop/aislop) — the open-source CLI that catches 40+ AI-authored code patterns and grades a project 0–100 — into a per-turn quality check for every coding agent in the [skills.sh](https://skills.sh) ecosystem (Claude Code, Cursor, Codex, Gemini CLI, Windsurf, Cline, and 40+ more). If you use this skill, you don't ship slop.

The skill does two jobs:

1. **Prevents slop before it ships.** Teaches the agent the anti-patterns to avoid while writing — narrative comments, `as any`, swallowed errors, template-string SQL, duplicate logic, dead exports, generic names. Equally important: tells the agent to **follow the project's existing conventions** (logger, error classes, naming, tests, architecture), not impose generic ones.
2. **Catches what slips through.** After edits, the agent runs the scoped scan, verifies each finding against the source, fixes the non-mechanical ones in-session using its editing tools, and only surfaces to the user what genuinely needs a decision.

Install once; it fires every turn — not just in CI.

## Install

```bash
npx skills add scanaislop/aislop-skill
```

That's it. The [skills CLI](https://github.com/vercel-labs/skills) installs the skill to every coding agent installed on your machine — Claude Code, Cursor, Codex, Gemini CLI, Windsurf, Cline, Goose, Aider, Warp, OpenCode, Kilo Code, Continue, GitHub Copilot, Antigravity, and 30+ others. Run `npx skills add --help` to see flags (scope with `-g` for global, `-a <agent>` for a specific agent, `--list` to preview).

### Install manually (if you don't use `skills`)

Clone the repo and drop the directory into the agent's skills path:

```bash
# global for every project on this machine
mkdir -p ~/.claude/skills
git clone https://github.com/scanaislop/aislop-skill ~/.claude/skills/aislop

# or project-scoped
mkdir -p .claude/skills
git clone https://github.com/scanaislop/aislop-skill .claude/skills/aislop
```

Other agents' skill paths differ; [skills.sh](https://skills.sh) lists them all.

## What this skill enforces

- **Quality gate** — pre-commit, pre-PR, pre-merge, agent-turn-end. Deterministic score per run.
- **Deduplication** — duplicate logic and copy-paste across files; nudges toward shared helpers.
- **Reusability** — unused exports, unused files, code that should be extracted.
- **Maintainability** — function / file size, cyclomatic complexity, nesting depth, parameter lists.
- **AI-authored patterns** — narrative comments, `as any` casts, swallowed errors, TODO stubs, generic names, leftover `console.log`.
- **Security** — `eval`, template-string SQL, `innerHTML`, hardcoded secrets, vulnerable dependencies.
- **Architecture** — project-specific rules from `.aislop/rules.yaml`.

aislop itself runs six engines through standard tooling (Biome, oxlint, knip, ruff, golangci-lint, rubocop, php-cs-fixer, expo-doctor) plus its own AST + regex detectors. Deterministic, offline, no LLMs.

**Languages:** TypeScript, JavaScript, Python, Go, Rust, Ruby, PHP, Expo/React Native.

## Requirements

- Node.js 20+
- `npx` available (ships with npm)
- aislop pulled on demand via `npx aislop` — no global install required

## Underlying CLI

The scanner lives at [github.com/scanaislop/aislop](https://github.com/scanaislop/aislop). Docs at [scanaislop.com/docs](https://scanaislop.com/docs).

## License

MIT. Copyright 2026 [scanaislop](https://github.com/scanaislop). See [LICENSE](LICENSE).
