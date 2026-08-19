---
name: Enroll an Academy contact in a certification
description: Create or invite a customer/partner contact into a WorkRamp Academy, put them in a segment, assign a certification or path, and read back their registrations.
api: openapi/workramp-json-api-openapi.yml
operations:
  - get-academy-customer-users
  - create-contact
  - invite-users
  - update-academy-user
  - add-users-to-segment
  - get-all-certifications-2
  - assign-certificates
  - assign-paths
  - get-all-registrations
---

# Enroll an Academy contact in a certification

Academy is the customer/partner half of WorkRamp (sold by Confirm as "Academy"). Every
operation is nested under `/api/v1/academies/{academy_id}/…`, so the first thing you need is
the Academy ID — the docs page *Locate your Academy ID* explains where to find it in the
admin UI. Academy **Contacts** are a different identity space from Employee **Users**; do
not assume an id from one works in the other.

Auth is the same tenant-wide bearer key as everywhere else:

```
Authorization: Bearer <API_KEY>
Content-Type: application/json
```

## Steps

1. **Look the contact up first.** `get-academy-customer-users`
   (`GET /api/v1/academies/{academy_id}/users`). It filters on name, email, email domain,
   status and custom registration fields; filters combine with AND, and results are always
   sorted by email ascending. Both Active and Deactivated contacts come back by default,
   including pending invitations — check the status before deciding the contact is new.
2. **Create or invite.**
   - `create-contact` (`POST /api/v1/academies/{academy_id}/users`) creates the contact
     directly. The docs describe it as safe to use just to "ensure that the user has been
     created" — useful when SSO handles login and you only need the record to exist so you
     can segment them.
   - `invite-users` (`POST /api/v1/academies/{academy_id}/invite`) sends the Academy
     invitation instead.
3. **Update attributes if needed.** `update-academy-user`
   (`PATCH /api/v1/academies/{academy_id}/users/{contact_id}`) takes a JSON dictionary of
   default or custom attributes.
4. **Segment them.** `add-users-to-segment`
   (`POST /api/v1/academies/{academy_id}/segments/{segment_id}/add_users`). Segments are the
   dynamic grouping Academy uses to target content.
5. **Resolve the certification.** `get-all-certifications-2`
   (`GET /api/v1/academies/{academy_id}/certifications`) lists what the academy offers.
6. **Assign it.** `assign-certificates`
   (`POST /api/v1/academies/{academy_id}/certifications/{certification_id}/invite`). Paths
   and trainings have the same shape — `assign-paths` and `assign-trainings`. All three
   accept an optional `dueAt` field.
7. **Confirm.** `get-all-registrations`
   (`GET /api/v1/academies/{academy_id}/registrations`) — a registration is the record of a
   contact's enrollment. Fetch a single one with `get-registration`.

## Rules

- **No idempotency key exists.** Step 2 and step 6 are not safe to blind-retry. On a
  timeout, re-run step 1 or step 7 and only re-send if the record is missing.
- **403 means the key's owner lost admin**, or the feature is not enabled for the tenant.
  The body is `{"type": "forbidden", "message": "Error. Access denied."}`.
- **Assign endpoints take arrays of users** — batch rather than looping one call per contact.
  You have 3,000 calls/hour sustained.
- **`dueAt` and other dates** are UNIX epoch milliseconds on most endpoints; some Academy
  endpoints take ISO 8601. Check the specific operation.
- Do not confuse `certification_id` with `cert_id` — the contract uses both spellings on
  different certification endpoints.
