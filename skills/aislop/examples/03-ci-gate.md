# Example 03 — CI gate with GitHub Actions

Enforcing the aislop score as a required check on every PR.

Commands below use the normal local CLI shape. CI pins the package through the project's lockfile instead of using a floating runtime fetch.

---

**User:**
> Add aislop as a CI gate. Fail the build if the score drops below 70.

**Agent (internal):**

Checks for `.aislop/config.yml`. Missing. Writes the minimal config directly:

```yaml
# .aislop/config.yml
version: 1
ci:
  failBelow: 70
```

Ensures the project has a pinned `aislop` dev dependency in its lockfile and a committed package script:

```json
{
  "scripts": {
    "quality:ci": "aislop ci ."
  },
  "devDependencies": {
    "aislop": "0.10.1"
  }
}
```

Creates the workflow from that pinned dependency path:

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
      - run: npm ci
      - run: npm run quality:ci
```

Verifies locally that `aislop ci` exits non-zero when the score is under the configured threshold:

```bash
npx aislop ci
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
> - `.aislop/config.yml` — sets `ci.failBelow: 70`.
> - `.github/workflows/aislop.yml` — installs from the project lockfile and runs `npm run quality:ci` on every PR and push to `main`. Fails the job if the score drops below 70.
>
> Verified locally: `npx aislop ci` exits 0 at the current score of 98 / 100. To tighten the gate, bump `failBelow` in `.aislop/config.yml`.

---

Notes:

- The gate threshold lives in config, not hardcoded in the workflow, so teams can tune it without editing YAML in `.github/`.
- `aislop ci` exits non-zero when the score is below `ci.failBelow` or any error-severity finding is present, which is what fails the job.
- `aislop ci` emits JSON to stdout — pipe it to an artifact if you want historical tracking.
- Keep the gate low at first, raise it once the project baselines. Starting strict is how teams learn to silence rules.
