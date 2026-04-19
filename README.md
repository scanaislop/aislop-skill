# aislop-skill

Agent skills for the [aislop](https://github.com/scanaislop/aislop) code-quality CLI, installable across every agent in the [skills.sh](https://skills.sh) ecosystem.

## Install

```bash
npx skills add scanaislop/aislop-skill
```

Flags: `-s <skill>` to pick one skill, `-a <agent>` for a specific agent, `-g` for user-scope, `--list` to preview. See `npx skills add --help`.

## Skills

| Name | Path | Summary |
|---|---|---|
| `aislop` | [`skills/aislop/`](skills/aislop/) | Code-quality gate and coding guardrail. Scans with aislop, fixes the mechanical findings, addresses the rest in-session in the project's own style. |

## Links

- aislop CLI: [github.com/scanaislop/aislop](https://github.com/scanaislop/aislop)
- Docs: [scanaislop.com/docs/skill](https://scanaislop.com/docs/skill)
- Contribute: [CONTRIBUTING.md](CONTRIBUTING.md)

## License

MIT. Copyright 2026 [scanaislop](https://github.com/scanaislop).
