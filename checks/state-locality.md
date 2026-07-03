---
id: state-locality
label: "State Locality & Cache Coherence"
---

Review whether state that is *conceptually shared* actually lives somewhere all instances can see it, and whether caches derived from mutable data stay coherent across workers and survive a failing backend without poisoning themselves. These bugs are nearly invisible in development because a single process with a warm cache and a healthy backend behaves correctly; they only surface under multiple workers/replicas, across sessions, or when a dependency blips — exactly the conditions tests rarely reproduce.

The litmus test for the whole check: **if two instances (two web workers, two replicas, two browser tabs, two sessions) could end up with different values after independent, legitimate operations, and the value is presented as global/shared, that is the defect.**

First determine whether the project can run as more than one instance or hold shared mutable state: a web/API backend that can be scaled to multiple workers or replicas, a service with in-process caches over mutable data, or a frontend that persists settings. A single-process CLI, a stateless pure-function library, or a batch script with no shared runtime state has no surface here — report that and skip.

Look for these patterns:

1. **False persistence — instance-local storage presented as shared config.** A setting, toggle, roster, or piece of configuration that is conceptually *global* (every operator/user/replica should see the same value) is written to per-instance memory (a module-level dict, a process global, an in-memory singleton), to one replica's local disk, or to browser `localStorage`/`sessionStorage`, while the UI or API presents it as a shared setting that was "saved." The next request handled by a different worker, or the next session, doesn't see it. Fix: persist genuinely-shared state to a store every instance reads (the primary datastore, a shared cache/Redis, an ES doc, a config service). Genuinely per-instance or per-session state (a request cache, a UI-only preference, a preview) is fine — but it must be *labeled and treated* as local/ephemeral, not surfaced as shared config.

2. **Per-worker cache with no cross-instance invalidation.** An in-process cache derived from data that can change at runtime (a config document, a policy/rules index, a feature roster, a lookup table) is populated once per worker and never invalidated when the underlying data is mutated on a *different* worker. Worker A handles the write and refreshes its own copy; workers B..N keep serving the old value indefinitely. Works perfectly with one worker in dev. Fix: give the cache a cross-instance invalidation path — a version marker every worker polls and every mutation bumps, a pub/sub invalidation message, a short TTL with a shared source of truth, or a broadcast. Every mutation site of the underlying data must trigger the invalidation. If a cache is *deliberately* single-worker-only, it must say so explicitly at its definition and the deployment must actually be single-worker.

3. **Cached degrade value — a failure response stored as if it were fresh truth.** A hand-rolled cache (`_cache`, `_cache_ts`, `_refresh_if_stale`) refreshes from a backend, the backend momentarily fails or returns empty, and the cache stores that *degraded* result (an empty list, all-zero weights, default/blank config, an error sentinel) as the new cached value with a fresh timestamp — so every subsequent read serves the degraded value until the TTL expires, long after the backend recovered. The correct contract is **keep last-known-good on refresh failure**: on a failed or suspicious refresh, retain the previous good value (and log/metric the staleness) rather than caching the failure. Fix: make the refresh path distinguish "got a real new value" from "refresh failed" and never overwrite good data with a degrade result. Watch for this contract being implemented correctly in one cache and then re-implemented (incorrectly) by a sibling cache that copied the structure but not the guard.

4. **Read-modify-write on shared state across instances.** A value is read from a shared store, modified in one worker's memory, and written back, with no atomicity or versioning — so two workers racing lose one update. (This overlaps with `concurrency`; flag it here specifically when the shared store is the coherence mechanism — an ES doc, a config row, a counter — and the lost update manifests as divergence rather than a classic in-process race.) Fix: use an atomic operation, optimistic concurrency (version/seq check on write), or a compare-and-set path provided by the store.

For each issue:
- Identify the value, why it is conceptually shared (or why a cache over mutable data needs cross-instance coherence), and the concrete two-instance divergence it produces.
- Apply the smallest fix that restores coherence: move the state to a shared store, register the cache with the project's existing invalidation mechanism (reuse it — do not invent a second one), add the last-known-good guard, or make the write atomic. Prefer routing through a primitive the project already has for this (a version poller, a config-read cache, a shared client) over hand-rolling a new one.
- Where the project already documents the correct pattern (an invalidation registry, a "version the shared doc" rule, a cache primitive that owns the degrade contract), consuming that pattern *is* the fix; a new cache that bypasses it is the violation.

Do NOT flag:
- State that is legitimately per-instance or per-session and is treated as such — a per-request memoization, a connection pool, a UI-only preference that isn't claimed to be shared, a warm read cache whose staleness window is documented and acceptable.
- Immutable or deploy-time config that is identical on every instance because it ships in the artifact / environment and changes only on redeploy (version-controlled config deployed in lockstep is a valid way to be "shared"; it does not need a runtime store).
- Idempotent caches where a stale read is provably harmless and bounded, and the code says so.

If the project cannot run as multiple instances and holds no shared mutable state, report that and skip.
