# Example 04 — clean scan still gets a manual slop pass

The scan is clean, but the agent catches semantic slop before handing control back.

Commands below use the normal `npx aislop` quick-start path. In repos with strict supply-chain policy, use the project's pinned local script or binary instead.

---

**User:**
> Add a `loadCustomerOrders(customerId)` helper and tests.

**Agent (internal):**

Implements the helper and tests. Runs the scan:

```bash
npx aislop scan --changes --json
```

Score: 100 / 100. Zero findings.

Runs the manual slop pass anyway. Finds two issues:

1. **Fake completeness.** The catch block returns an empty list when the upstream order service fails. That makes failures look like customers have no orders.
2. **Shallow test.** The test only checks the happy path fixture and does not cover upstream failure.

Fixes both:

- Replaces the empty-list fallback with the project's normal wrapped error.
- Adds a failure-path test that asserts the wrapped error is surfaced.

Re-scans:

```bash
npx aislop scan --changes --json
```

Score: 100 / 100. Zero findings.

**Agent reply to user:**

> Added `loadCustomerOrders(customerId)` and tests.
>
> `aislop scan --changes`: 100 / 100, no findings.
>
> Manual slop pass also caught and fixed two issues the scanner could not infer: the first draft returned an empty order list when the upstream service failed, and the test covered only the happy path. It now surfaces the project-standard wrapped error, and the tests cover both success and failure.

---

Notes:

- A clean score did not end the review.
- The agent found semantic slop: a plausible-looking fallback that changed product behavior.
- The fix improved both behavior and test coverage.
