---
name: Manage Pixc photo-standard templates
description: >-
  Create, inspect, update and remove the Pixc templates that encode a store's photo standard —
  background, border, shadow, output file types and aspect sizes.
api: openapi/pixc-templates-api-openapi.yml
operations: [apiListTemplate, apiAddTemplate, apiShowTemplate, apiUpdateTemplate, apiRemoveTemplate]
generated: '2026-08-13'
method: generated
source: >-
  https://pixc.com/api/ plus openapi/_original/pixc-public-api-swagger-original.json.
  Every operationId and every enum value below is verified verbatim against the
  provider-published Swagger.
---

# Manage Pixc photo-standard templates

A template is the reusable definition of what "on-brand" means for a store's product photos.
Every order references one, so getting templates right is what makes a catalog consistent.

## Before you start

- Base URL: `https://dashboard.pixc.com/v1`, `Authorization: Bearer ACCESS_TOKEN`.
- Required scopes: `api:template:view`, `api:template:create`, `api:template:update`,
  `api:template:remove`.
- Template operations do not consume image credits, but rehearse with `test: true` anyway —
  it validates the body without writing.

## The template fields, verbatim from the spec

| Field | Type | Notes |
|---|---|---|
| `name` | string | **Required on create.** Not accepted on update. |
| `group` | enum | Only value: `autobg`. Attaches a fully automated background handler to every order using this template. |
| `background` | string | Hex colour, e.g. `#ffffff`. |
| `border` | string | Hex colour, e.g. `#ffffff`. |
| `borderWidth` | string | px or %. Note the response schema types this as integer-or-string. |
| `shadow` | string | Accepted on create and update; the response `TemplateObject` does not return it. |
| `types[]` | array | Each `{ "type": ... }`, one of `jpg`, `png`, `psd`, `original`. |
| `sizes[]` | array | Each `{ "aspect": ..., "width": n, "height": n }`. Aspect is one of `square`, `twothree`, `threetwo`, `fourthree`, `custom`, `ratio`. |
| `instructions` | string | Free-text editing instructions. Accepted on create and update; not returned by `TemplateObject`. |

## Steps

1. **List before creating.** `apiListTemplate` (`GET /api/template`, query `limit`, `start`).
   Templates accumulate and there is no uniqueness constraint on `name`, so check first.

2. **Create.** `apiAddTemplate` (`POST /api/template`, JSON body). Minimum viable body is
   `{ "name": "..." }`. A realistic body:
   ```json
   {
     "name": "Storefront white background",
     "group": "autobg",
     "background": "#ffffff",
     "types": [{ "type": "jpg" }, { "type": "png" }],
     "sizes": [
       { "aspect": "square", "width": 2048, "height": 2048 },
       { "aspect": "fourthree" }
     ],
     "instructions": "Centre the product with 10% padding. Remove reflections."
   }
   ```
   Read `data.templateId` off the response.

3. **Inspect.** `apiShowTemplate` (`GET /api/template/{templateId}`).

4. **Update.** `apiUpdateTemplate` (`PUT /api/template/{templateId}`, JSON body). `name` is
   **not** in the update body schema — you can create a template with a name but not rename it
   through the API. `types` and `sizes` are sent as whole arrays; treat the update as a replace
   of those collections, not a merge, and always read the current template with
   `apiShowTemplate` first so you do not drop entries.

5. **Remove.** `apiRemoveTemplate` (`DELETE /api/template/{templateId}`). Returns the bare
   `{ "success": true }` envelope. Pixc does not document what happens to in-flight orders that
   reference a deleted template — check for open orders with `apiListOrder` before deleting.

## Rules and gotchas

- **Choose `group: "autobg"` deliberately.** It is the only group value and it switches the
  order to fully automated processing. Omit it when you want Pixc's human editing pass.
- **`shadow` and `instructions` are write-only in practice** — both are accepted in the create
  and update bodies but neither appears on the `TemplateObject` returned by
  `apiShowTemplate`, so you cannot read back what you set. Keep your own copy of the intended
  values.
- **Not idempotent.** No `Idempotency-Key`. A retried `apiAddTemplate` creates a duplicate.
- **Errors** are `{ "success": false, "code": "...", "message": "..." }` with `400` declared;
  a missing required `name` surfaces there. An undocumented `403` is returned for both missing
  and invalid credentials.
