# Example 01 — before commit

An agent implements a small feature, then runs aislop before handing control back.

---

**User:**
> Add a `formatBytes(n)` helper in `src/lib/format.ts` that returns "1.2 KB", "3.4 MB", etc. Export it from the barrel.

**Agent (internal):**

Implements `formatBytes`, updates `src/lib/index.ts` barrel, writes a unit test. Before replying, invokes the aislop skill because edits are finished and a commit is implicit.

```bash
npx aislop scan --changes --json
```

Parses the JSON. Three findings:

- `ai-slop/narrative-comment` · `src/lib/format.ts:3` · fixable
- `ai-slop/unused-import` · `src/lib/format.ts:1` · fixable
- `ai-slop/as-any-cast` · `src/lib/format.ts:18` · not fixable

Runs auto-fix for the mechanical two:

```bash
npx aislop fix
```

Re-reads the file. The `as any` cast is on `n as any` inside the magnitude branch. Replaces it with a proper numeric guard:

```ts
if (typeof n !== "number" || !Number.isFinite(n)) {
  throw new TypeError("formatBytes expects a finite number");
}
```

Re-scans:

```bash
npx aislop scan --changes
```

Score: 96 / 100. Zero errors, zero warnings.

**Agent reply to user:**

> Added `formatBytes` in `src/lib/format.ts` and re-exported it from `src/lib/index.ts`. Unit test covers bytes, KB, MB, GB, and the non-finite guard.
>
> Ran `aislop scan --changes`: 96 / 100, no findings. Ready to commit.

---

Notes on what the agent did right:

- Scoped the scan to `--changes` rather than the whole project.
- Used `--json` when acting programmatically.
- Ran `fix` for the mechanical findings before doing manual work.
- Replaced `as any` with a real type guard, not a rule suppression.
- Re-scanned after fixes before claiming done.
- Reported the score and finding count in the reply.
