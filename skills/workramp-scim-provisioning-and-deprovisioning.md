---
name: Provision and deprovision users with SCIM
description: Use WorkRamp's SCIM 2.0 endpoints to create, search, update and remove users and groups from an identity provider, and fall back to the REST user API where SCIM is thin.
api: openapi/workramp-api-settings-openapi.yml
operations:
  - scim-create-user
  - scim-get-all-users
  - scim-get-user
  - scim-update-user
  - scim-get-all-groups
  - scim-create-group
  - scim-get-group
  - scim-patch-group
  - scim-delete-group
  - get-all-users-in-enterprise
  - delete-user
---

# Provision and deprovision users with SCIM

WorkRamp exposes SCIM 2.0-shaped provisioning at `/scim/v2/…` on the same host and with the
same credential as the REST API — `Authorization: Bearer <API_KEY>`. There is **no SCIM
bearer-token issuance flow and no `/scim/v2/ServiceProviderConfig`, `/Schemas` or
`/ResourceTypes` discovery endpoint**, so an IdP that auto-discovers capabilities will need
manual configuration.

## Surface

| Resource | Operations |
|---|---|
| `/scim/v2/Users` | `GET` list, `POST` create |
| `/scim/v2/Users/{id}` | `GET` read, `PUT` replace |
| `/scim/v2/Groups` | `GET` list, `POST` create |
| `/scim/v2/Groups/{id}` | `GET` read, `PATCH` modify, `DELETE` remove |

Note the asymmetry: Users support `PUT` (full replace) but not `PATCH`; Groups support
`PATCH` but not `PUT`. A standard SCIM client that sends `PATCH` to a user will fail.

## Steps

1. **Search before creating.** `scim-get-all-users` (`GET /scim/v2/Users`) with a filter, or
   fall back to the REST `get-all-users-in-enterprise` (`GET /api/v1/users?email=…&limit=1`),
   which has the better-documented filter surface.
2. **Create.** `scim-create-user` (`POST /scim/v2/Users`). Provisioned identities surface on
   the WorkRamp user object as `userIdentifiers` with namespace `sso_scim` — that is the join
   key back to the REST API.
3. **Update.** `scim-update-user` (`PUT /scim/v2/Users/{id}`) replaces the resource. Send the
   complete representation; a partial `PUT` will drop fields.
4. **Groups.** `scim-create-group` (`POST /scim/v2/Groups`) and `scim-patch-group`
   (`PATCH /scim/v2/Groups/{id}`) for membership changes. WorkRamp's own spec validation
   flags that the Groups `members` array is declared without an items schema, so the member
   object shape is undocumented — mirror the SCIM 2.0 `{"value": "<user id>"}` form and
   verify the result with `scim-get-group`.
5. **Deprovision.** There is **no `DELETE /scim/v2/Users/{id}`**. To remove a user you must
   drop to the REST API: `delete-user` (`DELETE /api/v1/users/{user_id}`). Plan the SCIM →
   REST id mapping before you turn deprovisioning on.

## Rules

- **Deactivating the API key's owner breaks provisioning entirely.** Own integration keys
  with a dedicated service account, never a person's account.
- **The key has no scopes.** A provisioning integration holds full read/write over every
  learner record in the tenant. Isolate it.
- **No idempotency key.** A retried `POST /scim/v2/Users` can create a duplicate — always
  search first (step 1) and re-search after a timeout.
- **Rate limits apply to SCIM too**: 3,000 calls/hour sustained. Bulk IdP syncs are the most
  likely thing to hit the ceiling; a 429 arrives with no `Retry-After`.
- **This is SCIM-shaped, not SCIM-conformant.** Do not assume filter syntax, `ETag`
  concurrency, `/Me`, bulk, or sort support. See `conformance/workramp-conformance.yml`.
