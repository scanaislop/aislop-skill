# Fix guide — non-mechanical fixes by rule

When `aislop fix` has done its work, remaining findings need judgement. For each rule below, open the cited file, verify the finding, and apply the fix.

| Rule | Fix |
|---|---|
| `unsafe-type-assertion` / `double-type-assertion` | Find the real type. No `as any`, no `as unknown as X`. |
| `swallowed-exception` / `silent-recovery` | Log and rethrow, or surface the failure. Don't log and continue. |
| `redundant-try-catch` | Remove the wrapper. Let the error propagate naturally. |
| `hallucinated-import` | Verify the module exists in the filesystem. Create or use what's there. |
| `duplicate-import` | Merge into one import statement. |
| `meta-comment` / `narrative-comment` | Delete comments that describe what the code does or why it changed. |
| `trivial-comment` | Delete comments that restate the obvious. |
| `unreachable-code` | Delete statements after `return`/`throw`/`break`. |
| `constant-condition` | Remove the branch or make the condition real. |
| `empty-function` | Implement the function or delete it. No stubs. |
| `ts-directive` | Fix the underlying type issue instead of suppressing it. |
| `console-leftover` | Replace with the project's logger or remove. |
| `thin-wrapper` | Inline the call if the wrapper adds no value. |
| `generic-naming` | Rename `data`/`result`/`temp`/`handleClick` to convey intent. |
| `function-too-long` | Extract a helper at the logical seam. |
| `file-too-large` | Split by concern, not line count. |
| `sql-injection` | Swap template string for parameterised query. |
| `eval` / `innerhtml` | Replace with `JSON.parse`, `textContent`, or DOMPurify. |
| `unused-export` | Wire it up at the call site or delete. |
| `duplicate-code` | Extract to a shared helper. |
| `hardcoded-secret` | Move to environment variable. Fail loudly if unset. |
| `hardcoded-url` / `hardcoded-id` | Read from environment, config, or accept as a parameter. |
| `vulnerable-dependency` | Upgrade to the fixed version or add an override. |
