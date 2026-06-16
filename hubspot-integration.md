# HubSpot Integration Plan

How to push the website's three lead forms into HubSpot. Pairs with [`lead-payload.md`](./lead-payload.md), which documents the raw payload shapes.

The integration lives in `src/lib/server/hubspot.js` and is called from `/api/send` (`src/routes/api/send/+server.js`) **after** the Resend email, wrapped in `try/catch` so a HubSpot failure never breaks the email or the response.

> **v1 has no Notes.** This account's Service Key has no notes scope, so the integration does not call the Notes API. The variable-length detail that would have been a note (line items, contact message, design colors) is stored in **text properties** instead — nothing important is lost, and the deal still lands on the New Lead board with all properties intact.

## 1. Object model

| Form | HubSpot result |
|------|----------------|
| `contact` | **Contact** (upsert by email); `message` stored on the contact (`az_lead_detail`) |
| `estimate` | **Contact** + a **Deal** ($ value, pipeline stage, and the breakdown in `az_breakdown`) |
| `design` | **Contact** (upsert); color selections stored on the contact (`az_lead_detail`) |

Only `estimate` creates a Deal, so only estimates appear on the New Lead board. Upserting the Contact by email keeps one record per person even if they design a court and later request an estimate.

## 2. Field mapping

**Contact** (all three):

| HubSpot contact property | Source | Notes |
|---|---|---|
| `email`, `phone` | payload | standard; `phone` omitted if absent |
| `firstname` / `lastname` | split from `name` | standard |
| `lifecyclestage` | `'lead'` | standard |
| `az_form_type` | the form type | new — single-line text (or dropdown with exact values `contact`/`estimate`/`design`) |
| `az_lead_detail` | `message` (contact) or formatted colors (design) | new — **multi-line text** |

**Deal** (estimate only) — standard, existing, and new `az_*` properties:

| HubSpot deal property | Source | Notes |
|---|---|---|
| `amount` | `estimatedTotal.min` | standard — conservative low end |
| `estimate_amount_website` | midpoint of min/max | **existing** property (hybrid reuse) |
| `lead_source__campaign` | `'Website Estimator'` | **existing** dropdown — note the double underscore; the option must exist (see §5) |
| `dealname` | `"{name} — {courtSize}"` | standard |
| `az_court_size` | `courtSize` | new |
| `az_court_dimensions` | `courtDimensions` | new |
| `az_base_condition` | `baseCondition` | new |
| `az_terrain` | `terrain` | new |
| `az_fencing` / `az_lighting` / `az_net` | `extras.*` | new |
| `az_estimate_min` / `az_estimate_max` | `estimatedTotal.min` / `.max` | new |
| `az_breakdown` | `lineItems` + `terrainItem`, folded into text | new — **multi-line text** |

## 3. One-time setup

1. **Service Key** — create a Service Key in HubSpot with four scopes: Contacts read/write and Deals read/write. **No notes scope needed.** Put the token in Vercel env as `HUBSPOT_TOKEN`.
2. **Create the new custom properties:**
   - **Contacts:** `az_form_type`, `az_lead_detail` (multi-line text)
   - **Deals:** `az_court_size`, `az_court_dimensions`, `az_base_condition`, `az_terrain`, `az_fencing`, `az_lighting`, `az_net`, `az_estimate_min` (number), `az_estimate_max` (number), `az_breakdown` (multi-line text)
   - **Property types:** make the two estimate amounts **number** and everything else **single-line text**, except `az_lead_detail` and `az_breakdown` which are **multi-line text**. Do **not** create `az_fencing` / `az_lighting` / `az_net` (or `az_form_type`) as dropdowns unless their option internal values exactly match the payload enums — see §5.
   - Two **existing** Deal properties are also written — `estimate_amount_website` and `lead_source__campaign` — which already exist in the account and don't need creating.
3. **Pipeline + stage** are baked in as verified account constants: `PIPELINE_ID = 'default'` and `NEW_LEAD_STAGE_ID = 'appointmentscheduled'`. Change them only if the pipeline or stage id changes.

## 4. Code

The shipped module (`src/lib/server/hubspot.js`):

```js
import { env } from '$env/dynamic/private';

const API = 'https://api.hubapi.com';

// ---- verified account constants ----
const PIPELINE_ID = 'default';
const NEW_LEAD_STAGE_ID = 'appointmentscheduled';

// existing properties (already in the account):
const ESTIMATE_AMOUNT_PROP = 'estimate_amount_website'; // number
const LEAD_SOURCE_PROP = 'lead_source__campaign';       // enumeration (double underscore!)

async function hs(path, body, method = 'POST') {
    const res = await fetch(`${API}${path}`, {
        method,
        headers: { Authorization: `Bearer ${env.HUBSPOT_TOKEN}`, 'Content-Type': 'application/json' },
        body: body ? JSON.stringify(body) : undefined
    });
    if (!res.ok) throw new Error(`HubSpot ${path} ${res.status}: ${await res.text()}`);
    return res.json();
}

function splitName(full = '') {
    const p = full.trim().split(/\s+/);
    return { firstname: p[0] || '', lastname: p.slice(1).join(' ') || '' };
}

// Upsert by email so design + estimate from the same person merge into one contact.
async function upsertContact({ name, email, phone }, extra = {}) {
    const { firstname, lastname } = splitName(name);
    const r = await hs('/crm/v3/objects/contacts/batch/upsert', {
        inputs: [{
            idProperty: 'email',
            id: email,
            properties: { email, firstname, lastname, ...(phone ? { phone } : {}), lifecyclestage: 'lead', ...extra }
        }]
    });
    return r.results[0].id;
}

// associationTypeId 3 = Deal->Contact (HubSpot-defined default)
const assoc = (id, typeId) => [{ to: { id }, types: [{ associationCategory: 'HUBSPOT_DEFINED', associationTypeId: typeId }] }];
const createDeal = (contactId, properties) =>
    hs('/crm/v3/objects/deals', { properties, associations: assoc(contactId, 3) });

export async function sendLeadToHubSpot(type, p) {
    if (!env.HUBSPOT_TOKEN) return; // dormant until the token is configured

    const contactExtra = { az_form_type: type };

    if (type === 'contact' && p.message) {
        contactExtra.az_lead_detail = p.message;
    }
    if (type === 'design' && p.colors) {
        const c = p.colors;
        const line = (z) => c[z] ? `${z[0].toUpperCase() + z.slice(1)}: ${c[z].name} (${c[z].hex})` : '';
        contactExtra.az_lead_detail = ['Court design:', line('kitchen'), line('inner'), line('outer')]
            .filter(Boolean).join(' | ');
    }

    const contactId = await upsertContact(p, contactExtra);

    if (type === 'estimate') {
        const t = p.estimatedTotal || {};
        const mid = (t.min != null && t.max != null)
            ? Math.round((Number(t.min) + Number(t.max)) / 2)
            : (t.min ?? '');

        const lines = (p.lineItems || []).map(i => `• ${i.name}: $${i.min}–$${i.max}`).join('\n');
        const terrain = p.terrainItem ? `\n+ ${p.terrainItem.name}: $${p.terrainItem.min}–$${p.terrainItem.max}` : '';
        const breakdown = lines ? `Estimate breakdown:\n${lines}${terrain}` : '';

        await createDeal(contactId, {
            dealname: `${p.name} — ${p.courtSize || 'Court'}`,
            amount: String(t.min ?? ''),                 // standard field, conservative low end
            pipeline: PIPELINE_ID,
            dealstage: NEW_LEAD_STAGE_ID,

            [ESTIMATE_AMOUNT_PROP]: String(mid),         // existing property (midpoint)
            [LEAD_SOURCE_PROP]: 'Website Estimator',     // existing — must exist as a dropdown option

            az_court_size: p.courtSize,
            az_court_dimensions: p.courtDimensions,
            az_base_condition: p.baseCondition,
            az_terrain: p.terrain,
            az_fencing: p.extras?.fencing,
            az_lighting: p.extras?.lighting,
            az_net: p.extras?.net,
            az_estimate_min: String(t.min ?? ''),
            az_estimate_max: String(t.max ?? ''),
            ...(breakdown ? { az_breakdown: breakdown } : {})  // multi-line text property
        });
    }
}
```

Wired into `+server.js` after the email send:

```js
import { sendLeadToHubSpot } from '$lib/server/hubspot';

// near the end of POST(), after the email branches, before the final return:
try {
    await sendLeadToHubSpot(type, payload);
} catch (err) {
    console.error('HubSpot sync failed:', err); // log, don't throw — email + response still succeed
}
```

**Note on `$env/dynamic/private`:** the token is read at runtime, not inlined at build time via `$env/static/private`. This keeps the build green even when `HUBSPOT_TOKEN` is absent (preview deploys, fresh checkouts) and lets the integration sit dormant — the `if (!env.HUBSPOT_TOKEN) return;` guard short-circuits until the token exists.

## 5. Decisions baked in & gotchas

- **No Notes in v1.** No notes scope on this tier, so the breakdown/message/colors live in text properties (`az_breakdown` on the deal, `az_lead_detail` on the contact) instead of a note. If a notes scope is added later, that detail can move back to a Note.
- **Only `estimate` creates a deal.** `contact` and `design` stay Contact-only.
- **`lead_source__campaign` is a dropdown.** The code writes `'Website Estimator'` to it, so that exact option must exist on the property — otherwise HubSpot rejects the write. (Or point it at a text field.)
- **The two money fields intentionally differ.** Standard `amount` is the conservative low end (`estimatedTotal.min`) while `estimate_amount_website` is the midpoint. Not a bug.
- **Dropdown validation foot-gun.** `az_fencing` / `az_lighting` / `az_net` receive raw payload enums (`none/standard/premium`, `none/2pole/4pole`, `none/standalone/permanent`), and `az_form_type` receives `contact/estimate/design`. If any are created as dropdowns, their option internal values must exactly equal those strings or the write is rejected. Safest: single-line text.
- **`courtImage` is intentionally dropped in v1.** Saving the design render needs the Files API (`POST /files/v3/files`) plus an attachment reference; deferred.
- **Isolate failures.** The `try/catch` in `+server.js` means HubSpot being down never 500s the form.
- **Idempotency.** Contact upsert keys on email, so repeat submits update rather than duplicate. Deals are not deduped — two estimates create two deals (two opportunities).

## 6. Validation notes

Shapes were checked against HubSpot's API docs:

- `POST /crm/v3/objects/contacts/batch/upsert` with `inputs: [{ idProperty: 'email', id, properties }]` — confirmed. Email must be a unique identifier (it is by default for contacts); single-input upserts are the supported path.
- Deal→Contact `associationTypeId: 3` and the inline `associations` shape — confirmed as HubSpot-defined defaults. These can vary by portal, so confirm against the account once via `GET /crm/v4/associations/deals/contacts/labels` if a deal ever fails to associate.
- The upsert response returns `results[0].id`, which the code reads to attach the deal — standard batch-response shape; confirm on the first live submission.
- `lifecyclestage: 'lead'` is set on every upsert. If a repeat visitor is already a customer, this can move their stage backward unless the portal blocks it. Set it on create-only if that becomes an issue.
