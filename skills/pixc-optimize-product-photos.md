---
name: Optimize a batch of product photos with Pixc
description: >-
  Create a photo-standard template, submit a batch of product image URLs as a Pixc order,
  and collect the optimized results — the core Pixc Public API flow.
api: openapi/pixc-orders-api-openapi.yml
operations: [apiAddTemplate, apiListTemplate, apiAddOrder, apiListOrder, apiShowOrder, apiCancelOrder, apiImageListOrder, apiDownloadListOrder]
generated: '2026-08-13'
method: generated
source: >-
  https://pixc.com/api/ plus openapi/_original/pixc-public-api-swagger-original.json.
  Every operationId below is verified verbatim against the provider-published Swagger.
---

# Optimize a batch of product photos with Pixc

Use this when a store or marketplace has raw product photos at public URLs and needs them
returned to one consistent standard — background removed, cropped, resized, converted, with
optional shadow and border.

## Before you start

- Base URL: `https://dashboard.pixc.com/v1`
- Every request needs `Authorization: Bearer ACCESS_TOKEN`. The token comes from
  Account Settings > API Access in the Pixc Dashboard (https://pixc.com/dashboard/account/api).
- **Orders spend prepaid image credits.** Run the whole flow with the header `test: true`
  first. In test mode Pixc still validates your payload and your credentials but changes
  nothing and returns mock data. Drop the header only when the payload is confirmed good.
- Required scopes: `api:template:create`, `api:template:view`, `api:order:create`,
  `api:order:view`.

## Steps

1. **Reuse a template if one exists.** Call `apiListTemplate`
   (`GET /api/template`, query `limit`, `start`). A template is the photo standard, and
   reusing one is what keeps a catalog consistent. If a suitable template already exists,
   note its `templateId` and skip to step 3.

2. **Otherwise create the template** with `apiAddTemplate` (`POST /api/template`, JSON body).
   `name` is required. Optional fields, all from the provider's schema:
   - `group` — only accepts `"autobg"`, which attaches a fully automated background handler
     to every order using this template. Set it when no human retouching pass is wanted.
   - `background` and `border` — hex colours, e.g. `#ffffff`.
   - `borderWidth` — string, px or %.
   - `shadow` — string.
   - `types[]` — each `{ "type": ... }` where type is one of `jpg`, `png`, `psd`, `original`.
   - `sizes[]` — each `{ "aspect": ..., "width": ..., "height": ... }` where aspect is one of
     `square`, `twothree`, `threetwo`, `fourthree`, `custom`, `ratio`.
   - `instructions` — free-text editing instructions.
   Read `data.templateId` off the response.

3. **Submit the order** with `apiAddOrder` (`POST /api/order`, JSON body):
   ```json
   {
     "templateId": 123,
     "files": [
       { "url": "https://example.com/sku-001.jpg", "name": "sku-001", "mid": "SKU-001" }
     ]
   }
   ```
   `files` is the only required field. Set `mid` to your own catalog id on every file — it is
   the ONLY correlation key Pixc echoes back, and without it you cannot reliably match an
   output image to the SKU you sent. Read `data.orderId` off the response.

4. **Wait.** Pixc states results arrive within 24 hours. Prefer a webhook over polling — see
   the `pixc-receive-order-webhooks` skill and register `download_generated`. If you must
   poll, call `apiShowOrder` (`GET /api/order/{orderId}`) on a slow interval, e.g. every
   15 minutes. Do not poll tightly: Pixc publishes no rate limits and no rate-limit headers,
   so there is no signal telling you when you are being aggressive.

5. **Collect the results.** Two views of the same order:
   - `apiImageListOrder` (`GET /api/order/{orderId}/image`) — per-image records carrying
     `name`, `mid`, `urlOriginal`, `urlEdited` and a `results[]` array with one entry per
     template type/size combination (`url`, `size`, `background`, `type`, `border`). Use this
     when writing images back into a product catalog.
   - `apiDownloadListOrder` (`GET /api/order/{orderId}/download`) — packaged downloads
     (`downloadId`, `url`). Use this when a human wants a zip.
   Both take `limit` and `start`.

6. **Cancel if needed.** `apiCancelOrder` (`DELETE /api/order/{orderId}`) while the order is
   still open.

## Rules and gotchas

- **Not idempotent.** There is no `Idempotency-Key` and no retry-safety contract. A repeated
  `apiAddOrder` creates a SECOND order and spends credits twice. Record the `orderId` before
  any retry, and on a network timeout call `apiListOrder` to check whether the order landed
  rather than resubmitting.
- **Pagination is offset only** — `limit` and `start`, with no total count and no cursor. You
  cannot tell from a response whether more records exist; page until a short array comes back.
- **Errors** use `{ "success": false, "code": "...", "message": "..." }`, not RFC 9457
  problem+json. Pixc declares only `200` and `400`. In practice the API also returns an
  undocumented `403` with the body `{"message":"API Error","success":false}` for BOTH a
  missing and an invalid token, with no `code` — so on a 403 you cannot distinguish the two.
  Re-check the Authorization header and the token's scopes.
- **`status` on an order is an untyped integer** whose value vocabulary Pixc does not publish.
  Do not branch on specific numbers; use the webhook or the presence of downloads/images as
  the completion signal.
