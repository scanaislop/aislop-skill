# Example 02 — agent handles the non-mechanical remainder

`aislop fix` cleans up 8 of 12 findings. The agent addresses the remaining 4 by hand.

---

**User:**
> I just merged in a large refactor of the billing module. Can you clean it up before I open the PR?

**Agent (internal):**

Runs a broader scan because the refactor touched many files:

```bash
npx aislop scan --json
```

Twelve findings:

- 3 × `ai-slop/narrative-comment` (fixable)
- 2 × `ai-slop/unused-import` (fixable)
- 3 × `ai-slop/trivial-comment` (fixable)
- 2 × `ai-slop/as-any-cast` in `src/billing/normalize.ts` (not fixable)
- 1 × `complexity/function-too-long` in `src/billing/reconcile.ts:14` (`reconcile`, 134 lines) (not fixable)
- 1 × `security/sql-injection` in `src/billing/search.ts:30` (not fixable)

Runs auto-fix:

```bash
npx aislop fix
```

8 fixed: narrative comments, unused imports, trivial comments.

Remaining 4, handled one by one:

1. **`src/billing/normalize.ts:47` — `as any`.** The cast was hiding a `BillingLine | null` where `BillingLine` was expected. Tightened the return type of `parseLine` to `BillingLine | null` and handled null at the call site.
2. **`src/billing/normalize.ts:82` — `as any`.** Same root cause; removed after fix #1.
3. **`src/billing/reconcile.ts:14` — function too long.** Extracted two helpers: `matchByHash(lines)` and `matchByAmount(lines)`. `reconcile` now sits at 38 lines and delegates.
4. **`src/billing/search.ts:30` — SQL injection.** The query was built with a template literal interpolating `req.query.term`. Replaced with a parameterised `?` placeholder and passed `term` as a bound value.

Re-scans:

```bash
npx aislop scan --json
```

Score: 93 / 100. Zero errors, zero warnings.

**Agent reply to user:**

> Cleaned up the billing refactor.
>
> Auto-fixed 8 (narrative comments, unused imports, trivial comments).
>
> Manual fixes:
> - `src/billing/normalize.ts` — removed two `as any` casts by tightening `parseLine` return type to `BillingLine | null`.
> - `src/billing/reconcile.ts` — extracted `matchByHash` and `matchByAmount` from the 134-line `reconcile` function.
> - `src/billing/search.ts` — replaced template-literal SQL with a parameterised query.
>
> Re-scanned: 93 / 100, no remaining findings. Safe to open the PR.

---

Notes:

- Full-project `scan` was the right scope for a wide refactor.
- Agent grouped the remainder by rule id in its reply instead of dumping the JSON.
- Every fix was a real change. No rule suppressions, no config edits.
- Re-scan after manual work is non-optional.
