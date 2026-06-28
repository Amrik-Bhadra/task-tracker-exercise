# NOTES.md

## Summary of changes

### 1. Fixed SQL operator precedence bug in task search logic

Resolved a bug in `TaskRepository.java`, `db/queries/search_tasks.sql`, and `db/oracle/task_search_package.sql` caused by SQL operator precedence.

The original `WHERE` clause depended on SQL's default evaluation order (`AND` before `OR`) without using explicit parentheses. As a result, the query effectively split into two separate logical branches: one branch bypassed the status filter, while the other ignored the archived flag.

This was the root cause behind two seemingly unrelated issues:

* The status dropdown wasn't filtering results correctly.
* Archived tasks were still appearing in search results.

The fix was to wrap the title/description search condition in parentheses so that the archived filter, search term filter, and status filter are evaluated together as a single condition block.

---

### 2. Fixed `useTasks.js` getting stuck on "Loading tasks... after a failed request

`loading` was never reset to `false` on the error path. Added `setError(null)` at the start of each fetch and moved `setLoading(false)` into a `.finally()` so it runs on both success and failure.

---

### 3. Fixed pagination not resetting after filter changes

`App.jsx` was not resetting `page` back to `1` when either `query` or `status` changed.

This could leave users on a page number that no longer existed for the new result set, resulting in `No tasks found` despite matching data being available.

The page reset is handled directly inside the respective `onChange` handlers rather than through a separate `useEffect`. Since both state updates happen within the same event cycle, React batches them into a single render and fetch.

Using a `useEffect` for this would have introduced an unnecessary intermediate request on every filter change.

---

### 4. Added debounce to search requests in `useTasks.js`

Previously, every keystroke triggered an immediate network request. Combined with the backend's artificial latency, this made searching feel slow and wasteful. Added a 300ms debounce using `setTimeout` inside the effect, with a `clearTimeout` cleanup function so only the last change within a 300ms window actually fires a request. `setLoading(true)` still runs immediately, outside the timer, so the UI gives instant feedback while the actual fetch is delayed.

---

### 5. Fixed server crash (500) on invalid `page`/`pageSize` values in `TaskController.java`

`page <= 0` caused `start = (page - 1) * pageSize` to go negative, which made `allResults.subList(start, end)` throw an uncaught `IndexOutOfBoundsException` (HTTP 500). Added clamping right before the pagination math: `page` floors to `1`, and `pageSize` resets to `10` if it's below `1` or above `100`. This makes the crash structurally impossible rather than just handling one bad value, and the response still reports back the actual clamped values used.

---

## What I chose not to change, and why

* Left the artificial `Thread.sleep` in `TaskController` and the `System.out.println` logging as-is. Neither breaks functionality, and the exercise calls for a focused diff, not a rewrite.
* For bug 5, chose to clamp invalid `page`/`pageSize` values instead of rejecting them with a `400 Bad Request`. Rejecting would be more "correct" REST design, but it would also require adding error-handling on the frontend for that new failure case, which is a bigger change than the bug warrants. Clamping fixes the crash with a minimal diff and keeps the existing frontend contract unchanged.
* Did not fix a related issue where an invalid `status` string (e.g. `?status=banana`) also throws an uncaught `IllegalArgumentException` from `TaskStatus.valueOf(...)`, resulting in a 500. I consider this lower severity than the pagination bug because the actual UI (`StatusFilter.jsx`) only ever sends one of the three real enum values via the dropdown, so a normal user can never trigger it — it only matters if someone calls the API directly (Postman, curl, or a malformed client).

---

## Biggest remaining risk

There's a `Thread.sleep()` in TaskController that fakes slow processing for short search terms. I left it as-is since it doesn't break anything right now, but Spring Boot only `has a limited pool` of threads to handle requests. If many users search at once, those threads stay stuck sleeping instead of handling other requests, and under real traffic the whole API could slow down or time out — not just search.

---

## Tools / AI used

- I used Claude primarily to review the codebase incrementally, validate the SQL precedence issue, and sanity-check the fixes in `useTasks.js` and `App.jsx`.
- It was also useful for discussing the trade-offs between resetting pagination inline within event handlers versus handling it through a separate `useEffect`, before finalising the implementation.
