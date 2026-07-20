# Wiring a website form to the first line

Your form handler forwards each inquiry to the n8n webhook. Two rules:

1. **Best-effort.** An automation outage must never break the visitor's
   submission. Forward in a `try/catch`, short timeout, ignore failures.
2. **Send a `callback_url`.** It lets the workflow report the verdict and the
   proposal links back to your admin view (see the callback endpoint below).

## The forward (framework-agnostic)

After you have saved/emailed the lead as you normally would, POST to the webhook:

```js
async function forwardToFirstLine(lead, callbackUrl) {
  const url = process.env.N8N_LEAD_INTAKE_URL; // https://<your-n8n>/webhook/highcraft-lead-intake
  if (!url) return false;
  try {
    const res = await fetch(url, {
      method: 'POST',
      headers: { 'content-type': 'application/json' },
      body: JSON.stringify({
        source: 'website-estimate',
        lead_id: lead.id,          // your DB row id, so the callback can find it
        callback_url: callbackUrl, // https://<your-site>/api/estimate/qualification
        name: lead.name,
        company: lead.company || '',
        email: lead.email,
        message: lead.detail,
        project_type: lead.type || '',
        budget: lead.budget || '',
        timeline: lead.timeline || '',
      }),
      signal: AbortSignal.timeout(5000),
    });
    return res.ok;
  } catch {
    return false; // never throw; the inbox/DB is the record of truth
  }
}
```

Build the `callback_url` from your own host, and keep an allowlist on the n8n
side (the "Prep context" node only accepts callbacks to your domains).

## The callback endpoint (receives the verdict)

A machine-to-machine route, authenticated by the shared secret header (never a
user session). It writes the AI verdict and proposal links onto the lead row.

```js
// POST /api/estimate/qualification
export async function POST(request) {
  const provided = request.headers.get('x-automation-secret');
  if (!timingSafeEqualStrings(provided, process.env.ESTIMATE_AUTOMATION_SECRET)) {
    return json({ error: 'Unauthorized' }, 401);
  }
  const body = await request.json(); // { lead_id, ai_status, fit, score, why,
                                      //   reply_draft, decline_draft,
                                      //   company_summary, proposal_doc_url,
                                      //   proposal_page_url }
  await updateLead(body.lead_id, body); // write ai_* columns
  return json({ ok: true });
}
```

`ai_status` values you will receive: `qualified`, `declined_not_fit`,
`proposal_drafted`, `owner_rejected`. Store them and show them in your admin
panel with the score, the reason, and the proposal links.

Suggested columns to add to your leads table: `ai_status`, `ai_fit` (bool),
`ai_score` (int), `ai_why`, `ai_reply_draft`, `ai_decline_draft`,
`ai_company_summary`, `proposal_doc_url`, `proposal_page_url`, `ai_updated_at`.

## No website yet?

Use `proposal-page.html` as a reference for the output, and run the whole system
from a single static form that POSTs `{name, company, email, message}` to the
webhook with `mode: 'no-cors'`. That is exactly what the live demo does.
