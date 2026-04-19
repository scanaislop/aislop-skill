# Example 03 — CI gate with GitHub Actions

Enforcing the aislop score as a required check on every PR.

---

**User:**
> Add aislop as a CI gate. Fail the build if the score drops below 70.

**Agent (internal):**

Checks for `.aislop/config.yaml`. Missing. Creates it with `npx aislop init` (or writes the minimal config directly):

```yaml
# .aislop/config.yaml
version: 1
ci:
  failBelow: 70
```

Creates the workflow:

```yaml
# .github/workflows/aislop.yml
name: aislop
on:
  pull_request:
  push:
    branches: [main]
jobs:
  quality:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
      - run: npx aislop@latest ci .
```

Or, using the composite action:

```yaml
name: aislop
on:
  pull_request:
  push:
    branches: [main]
jobs:
  quality:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: scanaislop/aislop@v0.5
```

Verifies locally that `aislop ci` exits non-zero when the score is under threshold:

```bash
npx aislop ci --fail-below 70
echo $?
```

Runs the agent's own self-check before finishing:

```bash
npx aislop scan --changes
```

Score: 98 / 100.

**Agent reply to user:**

> Wired up the CI gate.
>
> - `.aislop/config.yaml` — sets `ci.failBelow: 70`.
> - `.github/workflows/aislop.yml` — runs `npx aislop ci` on every PR and push to `main`. Fails the job if the score drops below 70.
>
> Verified locally: `aislop ci` exits 0 at the current score of 98 / 100. To tighten the gate, bump `failBelow` in the config or pass `--fail-below 80` to the CLI directly.

---

Notes:

- The gate threshold lives in config, not hardcoded in the workflow, so teams can tune it without editing YAML in `.github/`.
- The `--fail-below` CLI flag lets you override per-environment (e.g. stricter on `main`, looser on preview branches).
- `aislop ci` emits JSON to stdout — pipe it to an artifact if you want historical tracking.
- Keep the gate low at first, raise it once the project baselines. Starting strict is how teams learn to silence rules.
