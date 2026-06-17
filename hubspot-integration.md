# HubSpot Integration Plan

How to push the website's three lead forms into HubSpot. Pairs with [`lead-payload.md`](./lead-payload.md), which documents the raw payload shapes.

The integration lives in `src/lib/server/hubspot.js` and is called from `/api/send` (`src/routes/api/send/+server.js`) **after** the Resend email, wrapped in `try/catch` so a HubSpot failure never breaks the email or the response.

> **v1 has no Notes.** This account's Service Key has no notes scope, so the variable-length detail that would have been a note (line items, contact message, design colors) is stored in **text properties** instead — `az_lead_detail` on the contact and `az_breakdown` on the deal.

## 1. Object model

**Every form type creates a Deal** so every lead lands on the New Lead board. They're told apart by `lead_source__campaign` and the dealname.

| Form | Contact | Deal | Source label |
|------|---------|------|--------------|
| `contact` | upsert by email; message in `az_lead_detail` | minimal; message in `az_breakdown` | `Website Contact Form` |
| `estimate` | upsert by email | rich: `$`, `az_*` fields, breakdown in `az_breakdown` | `Website Estimator` |
| `design` | upsert by email; colors in `az_lead_detail` | minimal; colors in `az_breakdown` | `Website Court Builder` |

Upserting the Contact by email keeps one record per person. Note: deals are **not** deduped, so a person who uses the designer and then the estimator creates two deals.

## 2. Field mapping

**Contact** (all three):

| Property | Source | Notes |
|---|---|---|
| `email`, `phone` | payload | standard; `phone` omitted if absent |
| `firstname` / `lastname` | split from `name` | standard |
| `lifecyclestage` | `'lead'` | standard |
| `az_form_type` | the form type | `contact`/`estimate`/`design` — set on **both** the contact and the deal (text field on deals) |
| `az_lead_detail` | `message` (contact) or colors (design) | multi-line text |

**Deal** (all three):

| Property | Source | Notes |
|---|---|---|
| `dealname` | `"{name} — {label}"` | `Estimate` uses court size; others use `Contact form` / `Court Builder` |
| `pipeline` / `dealstage` | `default` / `appointmentscheduled` | constants |
| `lead_source__campaign` | per-type source label | **existing dropdown** — the value must exist as an option (see §3) |

**Deal** (estimate only, additional):

| Property | Source | Notes |
|---|---|---|
| `amount` | `estimatedTotal.min` | standard, conservative low end |
| `estimate_amount_website` | midpoint of min/max | existing number property |
| `az_court_size`, `az_court_dimensions`, `az_base_condition`, `az_terrain` | payload | text |
| `az_fencing` / `az_lighting` / `az_net` | `extras.*` | text |
| `az_estimate_min` / `az_estimate_max` | `estimatedTotal.min` / `.max` | number |
| `az_breakdown` | `lineItems` + `terrainItem` | multi-line text |

For `contact` / `design`, `az_breakdown` holds the message / color summary instead.

## 3. One-time setup

1. **Service Key** — create a Service Key with four scopes: Contacts read/write and Deals read/write. No notes scope. Put the token in Vercel env as `HUBSPOT_TOKEN`.
2. **Create the custom properties:**
   - **Contacts:** `az_form_type`, `az_lead_detail` (multi-line text)
   - **Deals:** `az_court_size`, `az_court_dimensions`, `az_base_condition`, `az_terrain`, `az_fencing`, `az_lighting`, `az_net`, `az_estimate_min`/`az_estimate_max` (number), `az_breakdown` (multi-line text)
   - Types: amounts **number**, everything else **single-line text**, except `az_lead_detail` / `az_breakdown` which are **multi-line text**.
3. **Add the two new `lead_source__campaign` dropdown options** (it already has `Website Estimator`):
   - `Website Contact Form`
   - `Website Court Builder`

   Without these, the contact/design deal writes are rejected (enumeration only accepts defined options). Alternatively, switch `lead_source__campaign` to a plain text field and any value flows through.
4. **Pipeline + stage** are baked in as verified constants: `PIPELINE_ID = 'default'`, `NEW_LEAD_STAGE_ID = 'appointmentscheduled'`.

## 4. Code

The shipped module (`src/lib/server/hubspot.js`):

```js
import { env } from '$env/dynamic/private';

const API = 'https://api.hubapi.com';
const PIPELINE_ID = 'default';
const NEW_LEAD_STAGE_ID = 'appointmentscheduled';
const ESTIMATE_AMOUNT_PROP = 'estimate_amount_website';
const LEAD_SOURCE_PROP = 'lead_source__campaign';

const SOURCE = {
    estimate: 'Website Estimator',
    contact: 'Website Contact Form',
    design: 'Website Court Builder'
};
const DEAL_SUFFIX = { estimate: 'Estimate', contact: 'Contact form', design: 'Court Builder' };

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

async function upsertContact({ name, email, phone }, extra = {}) {
    const { firstname, lastname } = splitName(name);
    const r = await hs('/crm/v3/objects/contacts/batch/upsert', {
        inputs: [{ idProperty: 'email', id: email, properties: { email, firstname, lastname, ...(phone ? { phone } : {}), lifecyclestage: 'lead', ...extra } }]
    });
    return r.results[0].id;
}

// associationTypeId 3 = Deal->Contact (HubSpot-defined default)
const assoc = (id, typeId) => [{ to: { id }, types: [{ associationCategory: 'HUBSPOT_DEFINED', associationTypeId: typeId }] }];
const createDeal = (contactId, properties) => hs('/crm/v3/objects/deals', { properties, associations: assoc(contactId, 3) });

export async function sendLeadToHubSpot(type, p) {
    if (!env.HUBSPOT_TOKEN) return;

    let detail = '';
    const contactExtra = { az_form_type: type };

    if (type === 'contact' && p.message) {
        detail = p.message;
        contactExtra.az_lead_detail = detail;
    }
    if (type === 'design' && p.colors) {
        const c = p.colors;
        const line = (z) => c[z] ? `${z[0].toUpperCase() + z.slice(1)}: ${c[z].name} (${c[z].hex})` : '';
        detail = ['Court design:', line('kitchen'), line('inner'), line('outer')].filter(Boolean).join(' | ');
        contactExtra.az_lead_detail = detail;
    }

    const contactId = await upsertContact(p, contactExtra);

    // Every form type creates a Deal so it lands on the New Lead board.
    const dealProps = {
        dealname: `${p.name || 'New lead'} — ${DEAL_SUFFIX[type] || 'Lead'}`,
        pipeline: PIPELINE_ID,
        dealstage: NEW_LEAD_STAGE_ID,
        [LEAD_SOURCE_PROP]: SOURCE[type] || 'Other'
    };

    if (type === 'estimate') {
        const t = p.estimatedTotal || {};
        const mid = (t.min != null && t.max != null) ? Math.round((Number(t.min) + Number(t.max)) / 2) : (t.min ?? '');
        const lines = (p.lineItems || []).map(i => `• ${i.name}: $${i.min}–$${i.max}`).join('\n');
        const terrain = p.terrainItem ? `\n+ ${p.terrainItem.name}: $${p.terrainItem.min}–$${p.terrainItem.max}` : '';
        const breakdown = lines ? `Estimate breakdown:\n${lines}${terrain}` : '';

        Object.assign(dealProps, {
            dealname: `${p.name} — ${p.courtSize || 'Court'}`,
            amount: String(t.min ?? ''),
            [ESTIMATE_AMOUNT_PROP]: String(mid),
            az_court_size: p.courtSize,
            az_court_dimensions: p.courtDimensions,
            az_base_condition: p.baseCondition,
            az_terrain: p.terrain,
            az_fencing: p.extras?.fencing,
            az_lighting: p.extras?.lighting,
            az_net: p.extras?.net,
            az_estimate_min: String(t.min ?? ''),
            az_estimate_max: String(t.max ?? ''),
            ...(breakdown ? { az_breakdown: breakdown } : {})
        });
    } else if (detail) {
        dealProps.az_breakdown = detail;
    }

    await createDeal(contactId, dealProps);
}
```

Wired into `+server.js` after the email send:

```js
try {
    await sendLeadToHubSpot(type, payload);
} catch (err) {
    console.error('HubSpot sync failed:', err);
}
```

## 5. Decisions & gotchas

- **All three create deals** (changed from estimate-only). Distinguished by `lead_source__campaign` + dealname; estimates also carry a dollar `amount` and the `az_*` fields.
- **The two new source values are a dropdown prerequisite.** `Website Contact Form` and `Website Court Builder` must exist as `lead_source__campaign` options or those deal writes are rejected (caught/logged — the contact still upserts, just no deal). Estimate is unaffected.
- **`az_form_type` tags both** the contact and the deal. On contacts it's a dropdown with lowercase values; on deals it's a **text** field (so it accepts the same lowercase `contact`/`estimate`/`design` with no option-matching).
- **No dedup on deals** — design-then-estimate by the same person creates two deals. Low volume, usually fine.
- **The two money fields differ on purpose:** `amount` = `estimatedTotal.min`; `estimate_amount_website` = midpoint.
- **`courtImage` is dropped in v1** — saving the render needs the Files API.
- **Isolated failures:** the `try/catch` in `+server.js` means HubSpot being down never 500s the form.

## 6. Validation notes

Verified live against the account (read-only probe):

- All `az_*` + `az_lead_detail` + `az_breakdown` properties exist with compatible types; `estimate_amount_website` and `lead_source__campaign` exist.
- `lead_source__campaign` currently has: Website Estimator, Referral, Repeat Customer, Google/Organic Search, Social Media, Phone Call, Other — so the two new options still need adding.
- Pipeline `default` + stage `appointmentscheduled` exist; Deal→Contact association `typeId 3` exists.
- The contacts batch upsert keys on email and returns `results[0].id`, which the code uses to attach the deal.
