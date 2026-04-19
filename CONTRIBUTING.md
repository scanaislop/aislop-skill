# Contributing to aislop-skill

Thanks for improving the repo. Each skill lives under `skills/<name>/` and ships its own `SKILL.md`, `skill.json`, and `examples/`. The top-level `README.md`, `LICENSE`, and this file cover the repo as a whole.

The first (and currently only) skill is [`skills/aislop/`](skills/aislop/) — the **code-quality gate and coding guardrail** for AI coding agents.

## What belongs in this repo

- Edits to an existing `skills/<name>/SKILL.md` that sharpen how the agent behaves on real user prompts.
- New examples in `skills/<name>/examples/` that show a realistic flow end-to-end.
- A brand-new skill under `skills/<new-name>/` — must ship `SKILL.md` (with YAML frontmatter: `name`, `description`), a `skill.json` manifest, and at least one example.
- Fixes to `skill.json`, `README.md`, or `LICENSE`.
- Typos, broken links, outdated command flags.

## What does not belong here

- Feature work on the aislop CLI itself — that lives at [scanaislop/aislop](https://github.com/scanaislop/aislop).
- New rule detectors, scoring tweaks, engine changes — same place.
- Integrations with specific agent frameworks that `skills.sh` already handles.

## Style

- **No narrative comments, no emojis, no decorative markers.** The skill tells agents to avoid them; don't violate that in the skill itself.
- Terse and imperative. If a sentence doesn't help the agent decide what to do next, cut it.
- Follow the voice of the existing file. Read the whole section you're editing before adding to it.
- The frontmatter `description` in `SKILL.md` is the router signal. Keep it under ~70 words and in the shape of top skills on [skills.sh](https://skills.sh) (e.g. Vercel's `deploy-to-vercel`, `react-best-practices`): one line of *what*, one line of *when*, optional one-liner of scope.

## Proposing a change

1. Fork the repo and create a branch off `main`: `git checkout -b improve-<area>`.
2. Make your edits. Re-read the whole affected section once end-to-end — is it still coherent?
3. Open a pull request against `main` with:
   - A short title describing the change.
   - A body explaining the *why*. If it's a reply to a real user prompt the skill handled poorly, quote the prompt and the skill's current miss.
4. One maintainer review is required before merge.

## Testing your change

- **Install locally**: `npx skills add ./` (or `./skills/<name>`) from this repo and check the skill lands in your agent's skill directory.
- **SKILL.md readability**: read it top to bottom. If any section reads as filler, delete it.
- **Frontmatter `description`**: paste into a skill router (Claude Code, Cursor's rules search) and check it triggers on the prompts you expect — and doesn't trigger on unrelated ones.
- **`skill.json`**: valid JSON, paths in `examples[]` resolve relative to the `skill.json` itself, keywords and triggers reflect the body.
- **Examples**: the shown commands must match what `npx aislop --help` reports today.

## Publishing new versions

1. Bump `version` in `skill.json` following [semver](https://semver.org) — patch for typos/clarifications, minor for new sections or examples, major for breaking changes to the skill's interface or expectations.
2. Open a PR with the version bump and a short changelog entry in the PR body.
3. After merge, a maintainer tags the release.

## Code of Conduct

Be kind, be specific, argue about the skill not the person. If you see behaviour that violates that, open an issue or email the maintainers.

## License

By contributing, you agree your changes are licensed under the repo's [MIT license](LICENSE).
