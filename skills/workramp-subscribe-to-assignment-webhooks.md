---
name: Subscribe to assignment webhooks and reconcile completions
description: Register a WorkRamp webhook subscription for assignment lifecycle events, handle the delivered payloads safely without a signature, and reconcile any gaps by polling the date-range assignment endpoints.
api: openapi/workramp-json-api-openapi.yml
operations:
  - elc-webhooks-get-all
  - elc-webhooks-create
  - elc-webhooks-get
  - elc-webhooks-update
  - elc-webhooks-delete
  - get-guide-assignments-in-date-range
  - get-path-assignments-in-date-range
  - get-scorm-assignments-in-date-range
---

# Subscribe to assignment webhooks and reconcile completions

WorkRamp POSTs assignment lifecycle events to an HTTPS endpoint you register. There is **no
signature, no shared secret and no documented retry policy**, so treat every delivery as an
untrusted hint and reconcile against the API. Full event catalog:
`asyncapi/workramp-webhooks.yml`.

## Steps

1. **List what already exists.** `elc-webhooks-get-all`
   (`GET /api/v1/webhook_subscriptions`). Creating a duplicate subscription means duplicate
   deliveries — there is no dedupe.
2. **Create the subscription.** `elc-webhooks-create`
   (`POST /api/v1/webhook_subscriptions`) with:
   ```json
   {
     "name": "completions-sync",
     "url": "https://your-endpoint.example.com/workramp",
     "eventTypes": ["assignmentCompleted.guide", "assignmentCompleted.path", "assignmentCompleted.scorm"]
   }
   ```
   `name`, `url` and `eventTypes` are all required and the URL must be HTTPS. The
   Academy-side equivalent is `create-webhook-subscription`
   (`POST /api/v1/academies/{academy_id}/webhook_subscriptions`) and it uses `webhookName`
   instead of `name` — the two halves of the API do not share the field name.
3. **Handle deliveries.** Every payload carries `eventType`, `timestamp` (UNIX epoch
   milliseconds) and a full `user` object. Assignment events add the resource (`guide`,
   `path`, `challenge`, `resource`, `scorm`) and its assignment object with
   `status`, `completionPercentage`, `isCompleted`, `completedAt` and friends.
4. **Verify, do not trust.** Because there is no signature, confirm the event against the
   API before acting on it — read the assignment back with the matching date-range endpoint
   in step 5, or with `get-path-assignment` for a single path assignment.
5. **Reconcile on a schedule.** No delivery guarantee is published, so run a catch-up sweep:
   - `get-guide-assignments-in-date-range` (`GET /api/v1/assignments/guide`)
   - `get-path-assignments-in-date-range` (`GET /api/v1/assignments/path`)
   - `get-scorm-assignments-in-date-range` (`GET /api/v1/assignments/scorm`)

   Overlap each window with the previous one; do not assume the boundaries are exclusive.
6. **Maintain.** `elc-webhooks-get` to inspect one, `elc-webhooks-update` (`PATCH`) to change
   the URL or event list, `elc-webhooks-delete` to remove it. Delete stale subscriptions —
   an orphaned HTTPS endpoint keeps receiving learner PII.

## Rules

- **Deliveries are unauthenticated.** Put the receiver on a hard-to-guess path, terminate
  TLS, rate-limit it, and validate every field. Do not treat the payload as authorization
  for anything.
- **Payloads contain personal data** — name, email, manager chain, custom attributes. Handle
  under the same controls as the rest of your HR data.
- **Webhook endpoints return `{"type": "...", "message": "..."}` on 403**, unlike most of
  the API which returns `{"error": "..."}` or an empty 400 body.
- **Reconciliation costs calls.** Stay inside 3,000 requests/hour sustained; use the widest
  date window you can rather than per-user polling.
- Event names are dotted and case-sensitive (`assignmentCompleted.guide`). An unknown event
  type in `eventTypes` is not validated for you — check the catalog first.
