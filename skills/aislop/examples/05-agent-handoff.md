# Example 05 — agent hand-off with `aislop fix --claude`

A large PR has too many non-fixable findings for manual editing. The agent hands off remaining issues to Claude Code.

---

**User:**
> I need to clean up this 30-file PR before opening it. Mostly API layer and database migrations.

**Agent (internal):**

Full scan first:

```bash
aislop scan --json
```

Twenty findings. Runs auto-fix:

```bash
aislop fix
```

8 mechanical findings fixed (imports, comments, console logs). 12 remain — too many to hand-edit efficiently. Hands off:

```bash
aislop fix --claude
```

This launches Claude Code in a subprocess with the findings as context. Claude Code fixes 10 of the 12.

Agent reviews the remaining two:

1. `security/sql-injection` in `src/migrations/seed.ts:22` — false positive (safe test fixture values).
2. `complexity/file-too-large` in `src/api/handlers.ts` — legitimate but cannot split without breaking the API contract; files a follow-up issue.

Re-scans:

```bash
aislop scan --changes --json
```

Score: 94 / 100. Zero errors. The remaining warning is the known file-size issue.

**Agent reply to user:**

> Cleaned up the PR.
>
> Auto-fixed 8 mechanical findings with `aislop fix`, then handed off 12 remaining to Claude Code with `aislop fix --claude`.
>
> Claude fixed 10: unsafe type assertions (3), swallowed exceptions (4), redundant wrappers (2), excess parameters (1).
>
> Remaining (2):
>   - sql-injection src/migrations/seed.ts — false positive; safe test fixture values
>   - file-too-large src/api/handlers.ts — known debt, filed #342 to split
>
> Re-scanned: 94 / 100, 0 errors. Ready to open the PR.

---

Notes:

- `aislop fix --claude` (and `--codex`, `--cursor`, `--gemini`) is for high-volume cleanups where hand-editing would be too slow.
- Always review what the agent did — don't trust findings blindly.
- Remaining findings still need the agent's judgement layer even after hand-off.
- Use `-p` to print a hand-off prompt without launching any agent (for copy-paste into a different tool).
