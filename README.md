# AZ Pickle Courts — Lead Form Payloads

Reference for integrating the AZ Pickle Courts website forms with an external lead system.

All three forms submit JSON to a single endpoint:

```
POST /api/send
Content-Type: application/json
```

Each request carries a `type` discriminator. There are exactly three types: `contact`, `estimate`, and `design`.

> **Note:** The current backend only sends email (via Resend). There is no lead database. To capture leads, either point the forms at your own endpoint using the same shapes below, or fan out from `/api/send` to your system alongside the existing email send.

## Response

| Outcome | Status | Body |
|---------|--------|------|
| Success | `200` | `{ "success": true }` |
| Failure | `500` | `{ "error": "<message>" }` |

## Form summary

| `type` | Source form | Where it appears | Phone field | Customer confirmation email |
|--------|-------------|------------------|-------------|-----------------------------|
| `contact` | Contact page | `/contact` | required | no |
| `estimate` | Estimator wizard | `/estimate`, home, `/about`, city pages | optional | yes |
| `design` | Court Builder (3D) | `/design` | none | yes |

Common fields across every type: `type`, `name`, `email` (plus `phone` on `contact` and `estimate`). Everything else is type-specific.

---

## 1. `contact`

All string fields. `name`, `email`, and `phone` are required and validated client-side (phone normalized to ≥ 10 digits). `message` is optional and may be an empty string.

```json
{
  "type": "contact",
  "name": "Jane Smith",
  "email": "jane@example.com",
  "phone": "(602) 555-1234",
  "message": "Interested in a new court in Scottsdale."
}
```

| Field | Type | Notes |
|-------|------|-------|
| `type` | string | Always `"contact"` |
| `name` | string | Required |
| `email` | string | Required, validated |
| `phone` | string | Required, ≥ 10 digits |
| `message` | string | Optional, may be `""` |

---

## 2. `estimate`

Sends contact info plus a fully computed quote. The option fields carry the human-readable **title**, not an internal id.

```json
{
  "type": "estimate",
  "name": "Jane Smith",
  "email": "jane@example.com",
  "phone": "(602) 555-1234",
  "courtSize": "Pro Court",
  "courtDimensions": "34' x 64'",
  "baseCondition": "New Slab",
  "terrain": "Flat Ground",
  "extras": { "fencing": "standard", "lighting": "2pole", "net": "permanent" },
  "estimatedTotal": { "min": 28800, "max": 35250 },
  "lineItems": [
    {
      "name": "Concrete Slab Foundation",
      "desc": "Rebar, grading forming, and structural pour.",
      "min": 20000,
      "max": 23000
    },
    {
      "name": "Sport Surfacing & Court Lines",
      "desc": "Genuine Laykold tournament-grade acrylic surfacing with three color coats and custom striping.",
      "min": 5800,
      "max": 7250
    }
  ],
  "terrainItem": { "name": "Site Excavation & Grading", "min": 1800, "max": 4800 }
}
```

| Field | Type | Notes |
|-------|------|-------|
| `type` | string | Always `"estimate"` |
| `name` | string | Required |
| `email` | string | Required, validated |
| `phone` | string | Optional, may be `""` |
| `courtSize` | string | `"Pro Court"`, `"Backyard Standard"`, `"Mini/Driveway"`, or `"Other / Unsure"` |
| `courtDimensions` | string | `"34' x 64'"`, `"30' x 60'"`, `"24' x 50'"`, or `""` |
| `baseCondition` | string | `"Convert Existing Slab"`, `"Resurface Existing Court"`, `"Existing (Needs Removal)"`, or `"New Slab"` |
| `terrain` | string | `"Flat Ground"`, `"Slightly Sloped"`, `"Hillside / Steep"`, or `""` |
| `extras` | object | See below |
| `estimatedTotal` | object | `{ min, max }` integers (USD). Court build total, excludes `terrainItem` |
| `lineItems` | array | Each `{ name, desc, min, max }`. Length varies by configuration |
| `terrainItem` | object \| null | `{ name, min, max }` or `null`. Note: no `desc` field |

`extras` enum values:

| Key | Allowed values |
|-----|----------------|
| `fencing` | `"none"`, `"standard"`, `"premium"` |
| `lighting` | `"none"`, `"2pole"`, `"4pole"` |
| `net` | `"none"`, `"standalone"`, `"permanent"` |

Notes:

- `terrain` is `""` when the terrain step is skipped, which happens for any project that isn't a new or removed slab.
- `estimatedTotal` is the court build range only. Site grading is quoted separately as `terrainItem` and is `null` when no grading applies.

---

## 3. `design`

From the 3D Court Builder. There is **no phone field**. `courtImage` is a PNG data URI of the rendered court, or `null` if the canvas capture failed.

```json
{
  "type": "design",
  "name": "Jane Smith",
  "email": "jane@example.com",
  "courtImage": "data:image/png;base64,iVBORw0KGgo...",
  "colors": {
    "kitchen": { "name": "Pro Blue", "hex": "#063971" },
    "inner": { "name": "Pro Blue", "hex": "#063971" },
    "outer": { "name": "Tournament Green", "hex": "#3f6835" }
  }
}
```

| Field | Type | Notes |
|-------|------|-------|
| `type` | string | Always `"design"` |
| `name` | string | Required |
| `email` | string | Required, validated |
| `courtImage` | string \| null | `"data:image/png;base64,..."` or `null`. Can be large (full court screenshot) |
| `colors` | object | Three zones: `kitchen`, `inner`, `outer` |

Each zone in `colors` is `{ name, hex }`:

| Field | Type | Notes |
|-------|------|-------|
| `name` | string | Laykold color name, or `"Custom"` if the hex matches no known swatch |
| `hex` | string | e.g. `"#063971"` |
