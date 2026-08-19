---
name: Receive Pixc order webhooks instead of polling
description: >-
  Register, inspect, update and remove Pixc webhooks so an agent is notified when an order is
  created, when its status changes, and when a download is ready — rather than polling.
api: openapi/pixc-webhooks-api-openapi.yml
operations: [apiListWebhook, apiAddWebhook, apiShowWebhook, apiUpdateWebhook, apiRemoveWebhook]
generated: '2026-08-13'
method: generated
source: >-
  https://pixc.com/api/ plus openapi/_original/pixc-public-api-swagger-original.json.
  Every operationId below is verified verbatim against the provider-published Swagger.
---

# Receive Pixc order webhooks instead of polling

Pixc says optimized photos arrive "within 24 hours". Polling across that window is wasteful
and Pixc publishes no rate limits to guide you, so register a webhook and let Pixc call you.

## Before you start

- Base URL: `https://dashboard.pixc.com/v1`, `Authorization: Bearer ACCESS_TOKEN`.
- Required scopes: `api:webhook:view`, `api:webhook:create`, `api:webhook:update`,
  `api:webhook:remove`.
- Add `test: true` to rehearse any of these calls without changing the account.

## Event vocabulary

Pixc's `WebhookObject.type` enum, verbatim from the published spec:

| Event | Meaning |
|---|---|
| `order_created` | An order was created |
| `order_status_updated` | An order's status changed |
| `download_generated` | A download is available for an order — the completion signal |

For the optimize-photos flow, `download_generated` is the one that matters.

## Steps

1. **Check what is already registered.** `apiListWebhook` (`GET /api/webhook`, query `limit`,
   `start`). Registering a duplicate is easy because there is no idempotency contract — always
   list first.

2. **Register the endpoint.** `apiAddWebhook` (`POST /api/webhook`). This operation takes
   **form-encoded** parameters, not JSON:
   - `url` — required. The spec says it must start with `http://` or `https://`. Always use
     `https://`; Pixc permits plain http but you should not.
   - `type` — required.
   ```
   curl -X POST \
     --header 'Authorization: Bearer ACCESS_TOKEN' \
     --header 'Content-Type: application/x-www-form-urlencoded' \
     --data 'url=https://your-app.example.com/hooks/pixc' \
     --data 'type=download_generated' \
     'https://dashboard.pixc.com/v1/api/webhook'
   ```
   Read `data.webhookId` off the response.
   **Contract gap to be aware of:** in the published Swagger the `type` form parameter is an
   unconstrained string whose description reads only "url" — the three-value enum is declared
   on the *response* object, not on the create parameter. So the spec alone will not tell you
   which values `type` accepts. Use the enum values above and verify with `apiShowWebhook`.

3. **Verify the registration.** `apiShowWebhook` (`GET /api/webhook/{webhookId}`) and confirm
   the returned `type`, `url` and `active` are what you intended.

4. **Pause or repoint without deleting.** `apiUpdateWebhook` (`PUT /api/webhook/{webhookId}`,
   form-encoded) accepts `url` and `active`. Setting `active=false` is the correct way to stop
   deliveries during a deploy — do not delete and recreate.

5. **Remove when done.** `apiRemoveWebhook` (`DELETE /api/webhook/{webhookId}`).

## Handling the callback

Pixc publishes **no payload schema** for any of the three events, so do not assume a body
shape. Treat the callback as a trigger, not as data:

1. Respond `200` immediately.
2. Call `apiListOrder` or `apiShowOrder` to find the affected order, then
   `apiDownloadListOrder` / `apiImageListOrder` to read the actual results. The REST API is
   the source of truth; the webhook only tells you when to look.

## Rules and gotchas

- **No signature or verification scheme is documented.** There is no HMAC header, no shared
  secret and no source-IP allowlist published. Anyone who learns your callback URL can invoke
  it. Mitigate on your side: use an unguessable path segment, and never trust the request body
  — always re-read state from the API before acting.
- **No retry or delivery guarantee is documented**, and there is no delivery log or replay
  endpoint. Assume at-most-once. Keep a slow reconciliation poll (for example hourly
  `apiListOrder`) as a safety net for missed deliveries.
- **Errors** are the standard `{ "success": false, "code": "...", "message": "..." }` envelope
  with `400` declared; an undocumented `403` is returned for both missing and invalid
  credentials.
