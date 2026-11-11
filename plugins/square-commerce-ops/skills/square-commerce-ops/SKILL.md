---
name: square-commerce-ops
description: "Run Square commerce across catalog, inventory, payments, and sales attribution: create, read, and update the catalog and keep stock honest."
version: 0.2.0
license: MIT
author: "the maintainers"
---
# Square Catalog Management

## When this applies

- User needs to create new products in their Square-powered online store
- User needs to update inventory counts for existing products
- User needs to cross-reference manufacturer model numbers with Square SKUs
- User needs to pull **sales totals** — by date range, by location, or by team member
- Stack: Square API (v2) — Catalog, Inventory, Payments, Orders, Reports, Team — proxied through Next.js API routes in the app, deployed on Railway

## Architecture pattern

Square catalog management follows a three-tier architecture:

```
Square API (cloud)
    ↑↓ HTTPS
the app API route (middleware — holds SQUARE_ACCESS_TOKEN)
    ↑↓ HTTPS
catalog admin page (browser — calls the app via proxy)
```

The browser NEVER calls Square directly — all requests go through the the app API route which holds the Square access token in its Railway environment variables.

## Creating new catalog items

### Via the app API (recommended)

If an endpoint exists at the app, POST to it:

```bash
curl -X POST 'https://api.example.com/api/catalog/inventory' \
  -H 'Content-Type: application/json' \
  -d '{
    "products": [
      {
        "title": "Product Name",
        "sku": "SKU123",
        "price": 99.99,
        "description": "Description text"
      }
    ]
  }'
```

If no the app endpoint exists, create one by adding a POST handler to the
existing `app/api/catalog/inventory/route.ts` in the the app project:

```typescript
export async function POST(req: NextRequest) {
  // Validate SQUARE_ACCESS_TOKEN exists
  // Parse body: { products: [{ title, sku, price, description? }] }
  // Build Square catalog objects (see below for format)
  // POST to https://connect.squareup.com/v2/catalog/batch-upsert
  // Refresh CSV after creation
  // Return { ok: true, created: N, square_response: data }
}
```

The handler must:
- Use `idempotency_key` with a timestamp to prevent duplicates
- Build ITEM + ITEM_VARIATION objects for each product
- Auto-refresh the bundled CSV after creation

### Via Square API directly (if you have the token)

POST to `https://connect.squareup.com/v2/catalog/batch-upsert`:

```json
{
  "idempotency_key": "unique-key-here",
  "batches": [{
    "objects": [
      {
        "type": "ITEM",
        "id": "#my-sku",
        "item_data": {
          "name": "Product Name",
          "description": "Description",
          "product_type": "REGULAR",
          "variations": [{
            "type": "ITEM_VARIATION",
            "id": "#my-sku-variation",
            "item_variation_data": {
              "item_id": "#my-sku",
              "name": "Default",
              "sku": "SKU123",
              "pricing_type": "FIXED_PRICING",
              "price_money": {
                "amount": 9999,
                "currency": "USD"
              },
              "track_inventory": true,
              "inventory_alert_type": "LOW_QUANTITY",
              "inventory_alert_threshold": 5
            }
          }]
        }
      }
    ]
  }]
}
```

Key points:
- **`id` fields start with `#`** — these are placeholder IDs that Square resolves within the batch
- **Item ID** is the parent product; **Variation ID** is the child variant
- **`amount` is in cents** — $49.99 = `4999`
- **`track_inventory: true`** enables Square inventory tracking
- **`inventory_alert_type` + `inventory_alert_threshold`** — auto-alerts when stock is low

### Via Square Dashboard (manual alternative)

Square Dashboard → Catalog → Add Item. Use this when only 1-2 new products are needed and there's no API endpoint set up.

## Reading inventory

```bash
curl -s 'https://connect.squareup.com/v2/catalog/list' \
  -H "Authorization: Bearer $SQUARE_ACCESS_TOKEN" \
  -H 'Square-Version: 2026-01-22'
```

Returns all catalog items. Filter by `type === 'ITEM'` && `!is_deleted`.

Inventory counts are fetched separately:

```bash
curl -s 'https://connect.squareup.com/v2/inventory/physical-counts' \
  -H "Authorization: Bearer $SQUARE_ACCESS_TOKEN" \
  -H 'Square-Version: 2026-01-22'
```

## Updating stock levels

Use the **Physical Counts API**:

```json
POST https://connect.squareup.com/v2/inventory/physical-counts
{
  "idempotency_key": "...",
  "physical_counts": [{
    "catalog_object_id": "variation_id_from_square",
    "location_id": "your_location_id",
    "quantity": "25",
    "state": "IN_STOCK",
    "occurred_at": "2026-07-28T12:00:00.000Z"
  }]
}
```

## Sales totals & per-employee attribution (READ-ONLY analytics)

This is the most-misunderstood part of Square's API. **Square does not have a single endpoint that answers "what were employee X's total sales in date range Y."** Per-employee attribution lives in different places depending on the question:

### What each Square endpoint actually carries

| Endpoint | Carries employee ID? | What you get |
|---|---|---|
| `GET /v2/payments` | **No** — only `card_details`, `location_id`, `buyer_email` | All completed payments, no ringer attribution |
| `GET /v2/orders/{id}` | **Yes** — `fulfillments[].created_by` carries `team_member_id` | Per-order attribution, BUT only on Square for Retail / Appointments SKU |
| `GET /v2/reports` (custom) | **No** — date/location filtered, not employee filtered | Aggregated reports, not per-employee |
| Square Dashboard → Reports → Team Sales | **Yes** — UI-only, no API equivalent | Per-employee gross/net/refunds/tips over date range |

### Reliable paths, ranked

**1. Square Dashboard Team Sales export (highest trust)** — User logs into `app.squareup.com/dashboard/reports/sales-summary` → set date range → filter by team member → Export CSV. Square is the source of truth here; the CSV columns are typically `Date, Team Member, Gross Sales, Net Sales, Refunds`. Read this CSV in code; never invent columns.

**2. `/v2/orders` attribution (only on Retail/Appointments SKU)** — `fulfillments[].created_by` is the team_member_id. Pull all orders in window and filter. **Will return zero on plain Square POS accounts** — confirm the SKU first.

**3. `/v2/payments` aggregation** — gives you total revenue but cannot be split by employee. Use this for location-level or account-level totals only.

### When you can't answer the question, SAY SO

Per the AGENTS.md / SOUL.md rules: **never fabricate $0 totals to look like a working answer.** If the API returns zero across all date ranges, the token is dead, the account has no completed payments, or the SKU is wrong — diagnose that first via `/v2/locations` (a token-health probe) before assuming the data simply isn't there.

A working per-employee route should look like:

```typescript
// app/api/catalog/analytics/team-sales/route.ts
export async function GET(request: NextRequest) {
  // Path 1: Read user-exported CSV from ~/.openclaw/.../employee-sales.csv
  // Path 2: Call /v2/orders with date filter, filter by team_member_id
  // Path 3: Return explicit "unavailable" instructions if neither works
  // NEVER return $0 with a success envelope.
}
```

The three-tier route is the pattern to copy: fall back to a local export, then the live Orders API, then report honestly that the number is unavailable.

## Manufacturer SKU cross-reference

When the user gets products from a manufacturer, the manufacturer's model numbers rarely match the Square SKUs. Build a mapping:

| MFR Model | User's Product Name | Square SKU |
|-----------|-------------------|-----------|
| B029 | 1x Level Classic | Various (SQ*) |
| B595 | Executive 2x | Various |
| TB049M20 | Apple Watch 3x | TB049M20S/R/G/B |

Store this mapping:
- In the admin inventory page UI (as an "MFR" column)
- In a `docs/manufacturer-cross-reference.md` file in the project
- As a `MFR_MODEL_MAP` constant in the inventory page code

### Manufacturer model mapping (worked example)

Full cross-reference for the manufacturer the product line Jewelry Co. models to catalog product names:

| MFR Prefix | catalog Name | Example Colors |
|-----------|--------------|----------------|
| B029 | 1x Level Classic | Black, Silver, Gold, Rose Gold, Silver/Gold, Silver/Black, Silver/Rose Gold |
| B041 | 1x Level Heart | Black, Silver, Gold, Rose Gold, Silver/Gold |
| B007 | 2x Level Infinity | Black, Gold, Silver, Rose, Silver/Gold, Black/Gold, Rose/Silver |
| B065 | 2x Level Butterfly | Black, Gold, Silver, Rose Gold, Silver/Gold, Silver/Black, Silver/Rose Gold, Black/Silver, Gold/Silver |
| B595 | Executive 2x ($349.99) | Gold, Gold/Silver, Silver |
| BG108 | 1x Brace ($249.99) | Silver (10 crystal), Gold (10 crystal), Rose Gold (10 crystal) |
| B452 | 2x Level Diamond | Black, Gold, Rose Gold, Silver, Silver/Gold |
| B001 | 2x Level Prestige | Black, Black/Gold, Black Matte, Copper, Gold, Rose Gold, Silver, Silver/Gold |
| B049 | 3x Level Classic | Black, Copper, Gold, Rose Gold, Silver, Silver/Gold, Silver/Black, Black/Gold |
| B049 (Titanium) | 3x Level Titanium | Gold |
| B587 | 6x Level Classic | Black, Gold, Silver, Black/Gold, Silver/Gold, Silver/Black |
| TB049M20 | Apple Watch 3x ($499.99) | Silver, Rose Gold, Gold, Black |

## Common pitfalls

- **Square API version matters** — always use the version from the existing code (`2026-01-22` in the the app)
- **Idempotency keys prevent duplicates** — use a unique string per batch (`catalog-create-${Date.now()}`)
- **Square has rate limits** — 50 requests per second for catalog queries, much lower for writes
- **Category IDs** — products need a category to appear in the right section of the Square store. If omitted, they go to "Uncategorized"
- **Location ID** — required for inventory tracking. The user's location ID is typically in the existing code
- **CSV sync** — after creating items via API, the bundled CSV may need refresh. Some systems cache the CSV
- **Product images** — Square catalog items support images via the Catalog Image API. Items created without images show as placeholder
- **`/v2/payments` does NOT carry the ringer's team_member_id** — do not promise per-employee attribution from this endpoint
- **Per-employee totals require either a Dashboard CSV export or the Retail/Appointments SKU** — pick one, don't infer
- **A live endpoint returning zero across all date ranges means the token is dead or the account state changed** — diagnose via `/v2/locations` before writing more code
- **Don't fabricate $0 totals** — return an explicit "unavailable" with instructions instead

## Real session: catalog + Square catalog (Jul 2026)

Created 7 new products in Square via the app proxy:
- **1x Brace (BG108)**: Silver, Gold, Rose Gold — $249.99
- **Apple Watch 3x (TB049M20)**: Silver, Rose Gold, Gold, Black — $499.99

Square API endpoint pattern used: POST `/api/catalog/inventory` with `{ products: [...] }` body.

**Key learning (Jul 2026):** DNS resolution for external APIs (`supabase.co`, `connect.squareup.com`) may fail from the agent environment. If a POST to create catalogs times out, it's likely not a code issue — retry from the user's machine or check Railway deploy status. The the app POST endpoint pattern is correct; deployment timing and DNS are the usual failure points.

## Example: per-employee sales attribution

Use a synthetic employee and date range when demonstrating sales reports.

1. Verify whether the Payments API includes employee attribution before choosing it as a source.
2. Use an authorized Team Sales CSV export or supported Orders API data, and document unavailable fields.
3. Treat empty responses as inconclusive until credentials, permissions, locations and date filters are checked. Never invent zero totals.
4. Select the merchant's authorized sales locations explicitly; exclude supplier and fulfillment locations.


## Full pricing reference

The catalogue lives in the merchant's own dashboard. Do not copy pricing, supplier, or banking details into a skill, a reference, or a prompt.


## Related skills

- `ecommerce-payment-readiness-audit` — broader audit of Square/Payment setup
- `pdf-data-extraction` — extracting manufacturer data from PDFs before creating Square items
- `deploy` — Railway deployment for the API route
- `railway-deploy-verify-loop` — Railway commit → poll → live-verify loop
