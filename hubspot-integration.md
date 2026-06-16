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

**Deal** (estimate) — `amount` = `estimatedTotal.min`, plus custom properties:

| Payload field | HubSpot deal property | Type |
|---------------|----------------------|------|
| `courtSize` | `az_court_size` | single-line text |
| `courtDimensions` | `az_court_dimensions` | single-line text |
| `baseCondition` | `az_base_condition` | single-line text |
| `terrain` | `az_terrain` | single-line text |
| `extras.fencing` / `extras.lighting` / `extras.net` | `az_fencing`, `az_lighting`, `az_net` | dropdown |
| `estimatedTotal.min` / `estimatedTotal.max` | `az_estimate_min`, `az_estimate_max` | number |

`lineItems` and `terrainItem` go into a Note (variable-length breakdown, not worth a property each).

## 3. One-time setup

1. **Private App token** — HubSpot → Settings → Integrations → Private Apps → Create. Scopes: `crm.objects.contacts.write/read`, `crm.objects.deals.write/read`, `crm.objects.notes.write`. Put the `pat-...` token in Vercel env as `HUBSPOT_TOKEN`.
2. **Create the custom properties** above (Settings → Properties, or via the API). Contacts get `az_form_type`; Deals get the `az_*` set.
3. **Grab the pipeline + stage ids** — the default pipeline is `default`; you need the internal id of your "New lead" stage (e.g. `appointmentscheduled` on the default pipeline). Settings → Objects → Deals → Pipelines.

## 4. Code

Server-only lib (`src/lib/server/...` is never bundled to the client), matching the existing env-import style:

```js
// src/lib/server/hubspot.js
import { HUBSPOT_TOKEN } from '$env/static/private';

const API = 'https://api.hubapi.com';

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
        await createDeal(contactId, {
            dealname: `${p.name} — ${p.courtSize}`,
            amount: String(t.min ?? ''),
            pipeline: 'default',
            dealstage: 'appointmentscheduled', // <-- your "New lead" stage id
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

## 5. Reliability notes

- **Isolate failures.** The `try/catch` above means HubSpot being down never 500s the form.
- **Idempotency.** The batch upsert keys on email, so duplicate submits update rather than duplicate the contact. Deals are not deduped — two estimates create two deals, which is usually the intent (two opportunities).
- **The design PNG** as a real attachment needs an extra step: upload `courtImage` to the Files API (`POST /files/v3/files`), then set `hs_attachment_ids` on the note. Skippable for v1 — the color names in the note are usually enough.
- **Lighter alternative.** If you ever want contacts only and no deals, the HubSpot Forms Submissions API maps fields straight to contact properties and can trigger workflows, but it can't create deals, so you'd lose the pipeline value on the estimate flow.
