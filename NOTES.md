# NOTES.md

## Summary of changes

### 1. Fixed SQL operator precedence bug in task search logic

Resolved a bug in `TaskRepository.java`, `db/queries/search_tasks.sql`, and `db/oracle/task_search_package.sql` caused by SQL operator precedence.

The original `WHERE` clause depended on SQL's default evaluation order (`AND` before `OR`) without using explicit parentheses. As a result, the query effectively split into two separate logical branches: one branch bypassed the status filter, while the other ignored the archived flag.

This was the root cause behind two seemingly unrelated issues:

* The status dropdown wasn't filtering results correctly.
* Archived tasks were still appearing in search results.

The fix was to wrap the title/description search condition in parentheses so that the archived filter, search term filter, and status filter are evaluated together as a single condition block.


### 2. Fixed `useTasks.js` getting stuck on "Loading tasks... after a failed request

`loading` was never reset to `false` on the error path. Added `setError(null)` at the start of each fetch and moved `setLoading(false)` into a `.finally()` so it runs on both success and failure.


### 3. Fixed pagination not resetting after filter changes

`App.jsx` was not resetting `page` back to `1` when either `query` or `status` changed.

This could leave users on a page number that no longer existed for the new result set, resulting in `No tasks found` despite matching data being available.

The page reset is handled directly inside the respective `onChange` handlers rather than through a separate `useEffect`. Since both state updates happen within the same event cycle, React batches them into a single render and fetch.

Using a `useEffect` for this would have introduced an unnecessary intermediate request on every filter change.

---

## Tools / AI used

I used Claude primarily to review the codebase incrementally, validate the SQL precedence issue, and sanity-check the fixes in `useTasks.js` and `App.jsx`.

It was also useful for discussing the trade-offs between resetting pagination inline within event handlers versus handling it through a separate `useEffect`, before finalising the implementation.
