---
id: access-control
label: "Authorization & Access Control"
---

Review whether the code verifies that the authenticated caller is *allowed to act on the specific object or perform the specific operation* — not merely that they are logged in. This is authorization (can this user do this to this resource?), which is distinct from authentication (who is this user?). Broken access control — an endpoint that returns or mutates a resource without checking the caller owns it or has the right role — is one of the most common and highest-impact web vulnerabilities, and it is invisible to input-validation and injection checks because every value on the wire is well-formed and the caller is genuinely logged in.

First determine whether this project has an authorization surface at all. It does if it has authenticated endpoints that operate on per-user, per-tenant, or per-role resources (a web API, a SaaS backend, an admin panel). It does **not** if it is a single-user CLI, a library with no request context, a fully public read-only service, or a static site. If there is no authenticated multi-actor surface, report that and skip.

If there is, identify the project's authorization model before looking for violations: how is the caller's identity established (session, JWT, API key), what does an authenticated principal carry (user id, tenant/org id, roles/scopes), and where is the *allowed-to* decision supposed to be made (a decorator, a dependency, a middleware, a policy/guard layer, a per-query tenant filter)? Then check that every state-changing or resource-returning path actually goes through it.

Look for these violation types:

1. **Insecure direct object reference (IDOR / BOLA)** — a handler takes an object id from the path, query, or body (`/orders/{id}`, `?document_id=`, `{ "account": ... }`), loads it by id alone, and returns or mutates it **without checking the caller owns it or may access it**. The fix is to scope the lookup to the caller — `WHERE id = :id AND owner_id = :current_user` — or to load-then-authorize and return 404/403 on mismatch. Returning 404 (not 403) for objects the caller may not even know exist avoids leaking their existence.

2. **Missing tenant isolation** — in a multi-tenant system, a query filters by the requested id but not by the caller's tenant/org, so one tenant can read or write another tenant's rows. Every query on tenant-scoped data must carry the tenant predicate; a shared connection or ORM base query that "usually" includes it but is bypassed on one path is the classic leak. Prefer enforcing the tenant scope at a single choke point (a session-bound query filter, row-level security) over re-remembering it in every handler.

3. **Function/endpoint-level authorization gaps** — an admin-only or elevated operation (delete any user, change roles, read audit logs, impersonate) that checks *authentication* but not the caller's *role/scope*. Also: a new endpoint added next to protected ones that never picked up the guard the others share (a route registered outside the authenticated router, a handler missing the `@requires_role` decorator its siblings have).

4. **Client-side-only enforcement** — the UI hides a button or route for unauthorized users, but the backend endpoint it would call performs no check of its own. The server is the only trust boundary; a hidden control is not access control. Flag backend operations whose only gate is that the frontend chose not to offer them.

5. **Mass-assignment / privilege escalation via input** — a create/update handler binds the request body straight onto a model (`Model(**body)`, `model.update(body)`, spread assignment) so a caller can set fields they should never control — `role`, `is_admin`, `owner_id`, `tenant_id`, `verified`, `balance`. The fix is an explicit allow-list of writable fields (a typed input schema with only the user-settable fields), never the raw body.

6. **Broken-object-property / over-fetching at the field level** — an endpoint correctly authorizes access to the object but serializes fields the caller shouldn't see (another user's email, internal flags, PII, other tenants' data embedded in a shared record). Return a caller-appropriate projection, not the whole row.

7. **Authorization decided before the resource is loaded** — a check that trusts an id or role claim from the request rather than re-deriving it from the loaded resource and the authenticated principal (e.g. trusting a `tenant_id` in the body instead of the one bound to the session). Never let the request tell you what it's allowed to do.

For each violation:
- Trace the actual path from the route to the data access and confirm the missing check — read the handler, the query, and any shared middleware/dependency before concluding a gate is absent; the check may live one layer up. A finding you can't trace to a concrete unauthorized access is not a finding.
- Apply the fix at the right layer: prefer a shared guard/dependency/query-scope over a per-handler `if` that the next new endpoint will forget. Scope object lookups to the caller; allow-list writable fields; move the decision server-side and post-load.
- Add a test that pins it: the resource owner (or correctly-roled caller) succeeds, and a *different* authenticated principal gets 403/404. A test that only checks "logged out → 401" does not prove object-level authorization; the regression must exercise an authenticated-but-unauthorized caller.

Do NOT:
- Add authentication where the project intentionally has none (a deliberately public endpoint, a health check, a webhook that authenticates by signature instead — that's the `schema-validation`/`security` concern, not this one). Confirm an endpoint is *meant* to be protected before flagging it.
- Invent a role/permission system the project doesn't have. Enforce the model that exists; if the right model is genuinely absent (a multi-tenant app with no tenant scoping anywhere), flag it as a high-severity design gap for human review rather than building an authorization framework unprompted.
- Flag internal service-to-service calls behind a trusted network boundary the project's design documents as trusted, or admin scripts that run with full privileges by design.
- Change auth/session/CORS configuration absent a concrete access-control defect — that overlaps with `security` and is easy to break.

If the project has no authenticated multi-actor surface, report that and skip.
