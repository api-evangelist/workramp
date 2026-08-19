---
name: Onboard an employee and assign learning
description: Provision a new employee in WorkRamp (Confirm Learn:Up), place them in the right group, set their custom attributes, and assign them a learning path and a course — then verify the assignments landed.
api: openapi/workramp-json-api-openapi.yml
operations:
  - create-user
  - get-all-users-in-enterprise
  - create-group
  - assign-a-user-to-a-group-1
  - add-custom-user-attributes
  - create-path-assignment
  - create-assignment-2
  - get-all-user-assignments
---

# Onboard an employee and assign learning

Base URL `https://app.workramp.com` — EU tenants must use `https://app.eu.workramp.com` for
every call. Auth is a single header on every request:

```
Authorization: Bearer <API_KEY>
Content-Type: application/json
```

The key belongs to an admin user and everything you do is attributed to that user. There
are no scopes: this key can read and change anything in the tenant. Never put it in a
browser or an agent context you do not control.

## Steps

1. **Check whether the person already exists.** `get-all-users-in-enterprise`
   (`GET /api/v1/users`) with the `email` query parameter. Always send `limit` — set it to
   `1` for a single-user lookup. Unbounded reads are explicitly called out in the docs as a
   rate-limiting risk.
2. **Create the user if they are new.** `create-user` (`POST /api/v1/users`). Managers are
   supplied as `managers` (an array) — the older `collaborators` parameter was replaced in
   2022 and is gone.
3. **Place them in a group.** Read `get-all-groups` (`GET /api/v1/groups`) to resolve the
   group id, create one with `create-group` (`POST /api/v1/groups`) if needed, then
   `assign-a-user-to-a-group-1` (`POST /api/v1/groups/{GroupID}/users`).
4. **Set custom attributes.** `add-custom-user-attributes`
   (`POST /api/v1/attributes/user/{UserID}`) adds or updates values, so it is safe to send
   the full attribute set. Note the path parameter casing changes here — `{UserID}`, not
   `{user_id}`.
5. **Assign the learning path.** `create-path-assignment`
   (`POST /api/v1/paths/{path_id}/assignments`). Assign a single course instead with
   `create-assignment-2` (`POST /api/v1/guides/{GuideID}/assignments`).
6. **Verify.** `get-all-user-assignments` returns every Course, Training Series (Path),
   Challenge and SCORM assignment for a user. Confirm the new rows are present before
   reporting success.

## Rules

- **There is no idempotency key.** Nothing in this API accepts one. Do not blind-retry a
  POST: on a timeout, re-read (step 1 or step 6) and only re-send if the record is absent.
  Step 2 and step 5 will otherwise create duplicates.
- **Rate limits.** 3,000 calls/hour sustained, 13,000/hour burst. There is no
  `Retry-After` and no `RateLimit-*` header — a 429 returns
  `{"error": "Rate limited. See info at ..."}` and nothing else. Back off exponentially.
- **Timestamps are UNIX epoch milliseconds**, not seconds and not ISO 8601, except on the
  endpoints the provider has already migrated. Check the field before parsing.
- **Errors are thin.** Most 400s carry an empty body. Log the raw response; do not try to
  branch on an error code that is not there.
- **Verbs are inconsistent.** Update User is a `POST`, Update Contact is a `PATCH`. Read the
  operation, do not infer from the name.
