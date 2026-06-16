# HubSpot Integration Plan

How to push the website's three lead forms into HubSpot. Pairs with [`lead-payload.md`](./lead-payload.md), which documents the raw payload shapes.

The integration point is already in place: every form lands server-side in `/api/send` (`src/routes/api/send/+server.js`) with the full payload, so the HubSpot calls happen there, alongside the existing Resend email send.

## 1. Object model

Map to HubSpot's native CRM objects (no custom object needed — those require Enterprise):

| Form | HubSpot result |
|------|----------------|
| `contact` | **Contact** (upsert by email) + message as a **Note** |
| `estimate` | **Contact** + a **Deal** (carries the $ value and pipeline stage) + line items as a Note |
| `design` | **Contact** + color selections as a **Note** (optionally the PNG as a file attachment) |

Why a Deal only for `estimate`: a Deal is HubSpot's money-and-pipeline object, and the estimate is the only payload with a dollar value and real buying intent. `contact` and `design` are top-of-funnel, so Contact + Note is enough. Upserting the Contact by email keeps one record per person even if they design a court and later request an estimate.

## 2. Field mapping

**Contact** (all three) — standard properties plus one custom:

- `email`, `phone` → standard
- `name` → split into `firstname` / `lastname`
- `lifecyclestage` → `lead`
- `az_form_type` (custom dropdown: `contact` / `estimate` / `design`) to segment by entry point

**Deal** (estimate) — a mix of standard, existing, and new `az_*` properties:

| HubSpot deal property | Source | Notes |
|---|---|---|
| `amount` | `estimatedTotal.min` | standard HubSpot field |
| `estimate_amount_website` | midpoint of min/max | **existing** property (hybrid reuse) |
| `lead_source__campaign` | `'Website Estimator'` | **existing** — note the double underscore |
| `dealname` | `"{name} — {courtSize}"` | standard |
| `az_court_size` | `courtSize` | new |
| `az_court_dimensions` | `courtDimensions` | new |
| `az_base_condition` | `baseCondition` | new |
| `az_terrain` | `terrain` | new |
| `az_fencing` / `az_lighting` / `az_net` | `extras.*` | new |
| `az_estimate_min` / `az_estimate_max` | `estimatedTotal.min` / `.max` | new |

`lineItems` and `terrainItem` go into a Note (variable-length breakdown, not worth a property each).

## 3. One-time setup

1. **Private App token** — HubSpot → Settings → Integrations → Private Apps → Create. Scopes: `crm.objects.contacts.write/read`, `crm.objects.deals.write/read`, `crm.objects.notes.write`. Put the `pat-...` token in Vercel env as `HUBSPOT_TOKEN`.
2. **Create the new custom properties** (Settings → Properties, or via the API): `az_form_type` on Contacts; `az_court_size`, `az_court_dimensions`, `az_base_condition`, `az_terrain`, `az_fencing`, `az_lighting`, `az_net`, `az_estimate_min`, `az_estimate_max` on Deals. Two **existing** Deal properties are also written — `estimate_amount_website` and `lead_source__campaign` — which already exist in the account and don't need creating.
3. **Pipeline + stage** are baked in as verified account constants: `PIPELINE_ID = 'default'` and `NEW_LEAD_STAGE_ID = 'appointmentscheduled'`. Change them only if the pipeline or stage id changes (Settings → Objects → Deals → Pipelines).

## 4. Code

Server-only lib (`src/lib/server/...` is never bundled to the client), matching the existing env-import style:

```js
// src/lib/server/hubspot.js
import { HUBSPOT_TOKEN } from '$env/static/private';

const API = 'https://api.hubapi.com';

// Verified account constants
const PIPELINE_ID = 'default';
const NEW_LEAD_STAGE_ID = 'appointmentscheduled';

async function hs(path, body, method = 'POST') {
    const res = await fetch(`${API}${path}`, {
        method,
        headers: { Authorization: `Bearer ${HUBSPOT_TOKEN}`, 'Content-Type': 'application/json' },
        body: body ? JSON.stringify(body) : undefined
    });
    if (!res.ok) throw new Error(`HubSpot ${path} ${res.status}: ${await res.text()}`);
    return res.json();
}

function splitName(full = '') {
    const p = full.trim().split(/\s+/);
    return { firstname: p[0] || '', lastname: p.slice(1).join(' ') || '' };
}

// Upsert by email so design + estimate from the same person merge into one contact
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

// associationTypeId 3 = Deal->Contact, 202 = Note->Contact (HubSpot-defined defaults)
const assoc = (id, typeId) => [{ to: { id }, types: [{ associationCategory: 'HUBSPOT_DEFINED', associationTypeId: typeId }] }];

const createDeal = (contactId, properties) => hs('/crm/v3/objects/deals', { properties, associations: assoc(contactId, 3) });
const addNote = (contactId, body) => hs('/crm/v3/objects/notes', {
    properties: { hs_note_body: body, hs_timestamp: Date.now() },
    associations: assoc(contactId, 202)
});

export async function sendLeadToHubSpot(type, p) {
    const contactId = await upsertContact(p, { az_form_type: type });

    if (type === 'contact' && p.message) {
        await addNote(contactId, p.message);
    }

    if (type === 'estimate') {
        const t = p.estimatedTotal || {};
        const midpoint = Math.round(((t.min ?? 0) + (t.max ?? 0)) / 2);
        await createDeal(contactId, {
            dealname: `${p.name} — ${p.courtSize}`,
            amount: String(t.min ?? ''),
            estimate_amount_website: String(midpoint),  // existing property — hybrid reuse (midpoint)
            lead_source__campaign: 'Website Estimator', // existing property — note the double underscore
            pipeline: PIPELINE_ID,
            dealstage: NEW_LEAD_STAGE_ID,
            az_court_size: p.courtSize,
            az_court_dimensions: p.courtDimensions,
            az_base_condition: p.baseCondition,
            az_terrain: p.terrain,
            az_fencing: p.extras?.fencing,
            az_lighting: p.extras?.lighting,
            az_net: p.extras?.net,
            az_estimate_min: String(t.min ?? ''),
            az_estimate_max: String(t.max ?? '')
        });
        const lines = (p.lineItems || []).map(i => `• ${i.name}: $${i.min}–$${i.max}`).join('\n');
        const terrain = p.terrainItem ? `\n+ ${p.terrainItem.name}: $${p.terrainItem.min}–$${p.terrainItem.max}` : '';
        if (lines) await addNote(contactId, `Estimate breakdown:\n${lines}${terrain}`);
    }

    if (type === 'design') {
        const c = p.colors;
        await addNote(contactId, `Saved a court design:\nKitchen: ${c.kitchen.name}\nInner: ${c.inner.name}\nOuter: ${c.outer.name}`);
    }
}
```

Wire it into `+server.js` so a HubSpot hiccup never breaks the email or the user's submission:

```js
import { sendLeadToHubSpot } from '$lib/server/hubspot';

// inside POST(), right after `const { type, ...payload } = data;`
try {
    await sendLeadToHubSpot(type, payload);
} catch (err) {
    console.error('HubSpot sync failed:', err); // log, don't throw — email + response still succeed
}
```

Keep the `await` (don't fire-and-forget): on Vercel's serverless runtime, work left running after the response can be killed when the function suspends, so let it complete inside the request.

## 5. Decisions baked in & reliability notes

- **Only `estimate` creates a deal.** `contact` and `design` stay Contact + Note.
- **Hybrid properties.** The dollar value also writes to the existing `estimate_amount_website` (midpoint of min/max), and `lead_source__campaign` is set to `'Website Estimator'`. Descriptive fields use the new `az_*` set. Those two are the only pre-existing properties the integration touches; everything `az_*` is new and needs creating.
- **`courtImage` is intentionally dropped in v1.** Saving it needs an extra step: upload to the Files API (`POST /files/v3/files`), then set `hs_attachment_ids` on the note. The color names in the note are usually enough.
- **Isolate failures.** The `try/catch` in `+server.js` means HubSpot being down never 500s the form.
- **Idempotency.** The batch upsert keys on email, so duplicate submits update rather than duplicate the contact. Deals are not deduped — two estimates create two deals, which is usually the intent (two opportunities).
- **Lighter alternative.** If you ever want contacts only and no deals, the HubSpot Forms Submissions API maps fields straight to contact properties and can trigger workflows, but it can't create deals, so you'd lose the pipeline value on the estimate flow.
