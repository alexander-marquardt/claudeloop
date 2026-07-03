---
id: frontend-data-flow
label: "Frontend Data-Fetching & Effect Correctness"
---

Review the frontend's data-fetching and effect logic for correctness bugs — the class of defect where the UI shows stale, missing, duplicated, or out-of-order data because an effect, subscription, or fetch was wired up wrong. These are correctness bugs, not style issues: the component renders, the types check, and it works in the happy path, but it shows the wrong thing when requests race, inputs change quickly, the component unmounts mid-flight, or the effect re-runs when it shouldn't (or fails to when it should).

This check applies to component-based frontends with effects/lifecycle and asynchronous data loading — React (hooks or classes), and by analogy Vue, Svelte, Solid, Angular. If the project has no such frontend (backend/library/CLI only, or server-rendered pages with no client-side data fetching), report that and skip. The patterns below are written in React terms; map them to the equivalent primitive in the framework in use (Vue `watch`/`watchEffect`, Svelte `$:`/`onMount`, Angular `ngOnChanges`/`OnDestroy`).

Look for these patterns:

1. **Stale or missing effect dependencies.** An effect (`useEffect`, `useMemo`, `useCallback`) reads props/state/values that are not in its dependency array, so it runs with a captured stale value and never re-runs when that value changes — the view keeps showing data for the *previous* id/filter/query. Conversely, an object/array/function literal recreated every render placed in the deps makes the effect re-run every render (a refetch loop). Fix: include every reactive value the effect reads; stabilize offending dependencies (memoize the object, hoist the constant, move the function in), rather than deleting a needed dep to silence the linter — a dependency removed to stop a loop usually hides the real bug.

2. **Fetch race / out-of-order responses.** An effect fires a request when its input changes, the input changes again before the first response lands, and the two responses resolve out of order — so the UI displays results for a stale input (type "ab", then "abc", see results for "ab"). Fix: make the effect ignore stale responses — an `ignore`/`cancelled` flag flipped in cleanup, an `AbortController` whose `signal` is passed to `fetch` and aborted in cleanup, or the request/response keying that the project's data layer provides. Every effect that fetches on a changing input needs a stale-response guard.

3. **Missing cleanup — updates after unmount, leaked subscriptions/timers.** An effect starts a fetch, subscription, `setInterval`/`setTimeout`, event listener, or observer and returns no cleanup, so it sets state after the component unmounted (a leak, and in React a warning) or accumulates duplicate subscriptions across remounts. Fix: return a cleanup function that aborts the request, unsubscribes, and clears the timer/listener. Pair every subscribe/add/start inside an effect with its teardown.

4. **Response→state→refetch feedback loop.** A fetch result is written to state (or to the URL/query params), and that state is itself a dependency of the effect that fetched — so the write re-triggers the fetch, producing an infinite loop or a UI that flickers between two values with no fixed point. Also the URL-sync variant: a fetch result updates the URL, and a `useEffect` on the URL re-fetches. Fix: break the cycle — derive the value instead of storing it, separate the "user changed input" trigger from the "data arrived" write, or guard the write so it doesn't fire when the value is unchanged.

5. **Entangled effects — unrelated concerns in one effect.** A single mega-effect does URL sync *and* data fetching *and* analytics *and* a subscription, sharing one dependency array, so a change to any one input re-runs all of them (refetching on an analytics-only change, resubscribing on a filter change). Fix: split into one effect per concern, each with the narrow dependency set that concern actually needs.

6. **Double-fire / duplicate requests.** Under React StrictMode (dev double-mount) or from an effect that lacks a guard, a mount fires the same request twice — usually harmless for idempotent GETs but a real bug for non-idempotent effects (double POST, double analytics event, a lock acquired twice). Fix: make mount effects idempotent, or add a short-lived signature/ref dedupe gate (and a proper abort on the superseded call) rather than disabling StrictMode.

7. **Unhandled loading / error states.** An async fetch renders as though it always succeeds — no loading indicator (so the UI flashes empty or shows a stale value), and no error branch (so a failed request silently shows nothing or a blank list indistinguishable from "no results"). Fix: render explicit loading and error states; distinguish "empty result" from "not loaded yet" from "failed."

For each issue found:
- Identify the concrete wrong-data symptom (which stale/missing/duplicated value the user sees, under what interaction), not just the deviation from convention.
- Apply the minimal correct fix (right deps, stale-response guard, cleanup, broken cycle, split effect, idempotent mount, explicit states).
- Where the project already uses a data-fetching library (React Query, SWR, RTK Query, Apollo, TanStack Query), prefer fixing the bug by using that library's built-in mechanism (query keys, automatic cancellation, cache invalidation) over hand-rolling a flag — a raw `useEffect` fetch sitting next to a codebase that standardizes on a query library is often itself the smell.
- Add or update a test where the framework's testing tools can express it (a rerender-with-new-input test asserting the stale response is ignored, an unmount test asserting no post-unmount state update). If the interaction genuinely can't be expressed in the project's test setup, say so with an explicit `no test added: <missing harness>` note rather than skipping silently.

Do NOT:
- Flag purely cosmetic effect style where behavior is correct (an effect that could be a `useMemo` but produces right results, ordering of hooks that doesn't affect output).
- Add `AbortController`/cancellation to effects whose requests are cheap, idempotent, and whose out-of-order resolution is provably harmless — note them and move on; over-guarding every effect is its own noise.
- Introduce a data-fetching library the project doesn't use, or restructure the component tree; work within the existing patterns.
- Remove a dependency from a deps array just to stop a re-run — fix the underlying instability instead. Silencing the exhaustive-deps lint rule is itself a suppression (see `suppressed-failures`).

If the project has no client-side component/effect layer, report that and skip.
