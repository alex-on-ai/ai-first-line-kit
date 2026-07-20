# Agent setup instructions

You are a coding agent (Claude Code, Codex, or similar) helping a business owner
stand up the "AI First Line" system in this repo. Your job: turn this generic kit
into *their* system by filling the context, adapting the prompts, and guiding
them through the n8n import and credential connection. A human is always on the
approve button; you never make this send anything to a real lead.

Work through the phases in order. Confirm with the user before any step that
creates an account, spends money, or exposes a public endpoint.

---

## Phase 0 — Understand the goal (read, then ask)

Read `README.md`, `docs/ARCHITECTURE.md`, `context/context-model.md`,
`context/icp-rubric.md`, and `prompts/qualify.md`. Then ask the user these
questions and keep the answers; everything downstream depends on them:

1. **Company**: name, one-sentence description, website URL, what you sell, your
   top 1-3 services, and your entry offer (the smallest way to start with you).
2. **Ideal client**: role, company size, region, industry, and the specific
   pains that signal a good fit. Just as important: **who you say no to**
   (disqualifiers).
3. **Proof**: real, verifiable case studies and any real metrics. Mark anything
   unverified. (You will hard-limit the AI to only these.)
4. **Voice**: three sample sentences the user actually wrote, and words they
   never want to see (add to the forbidden list).
5. **Approver**: whose Telegram gets the approval card.

If the user has an existing company context/manifest doc, use it. If not, offer
to draft `context/context-model.md` from their website (fetch it, extract, and
mark every guessed field `⚠ UNVERIFIED` for them to confirm).

---

## Phase 1 — Fill the context (the foundation)

Everything reads these three files. Fill them before touching the workflow.

- `context/context-model.md` — who they are, what they sell, the ICP, proof,
  voice. This is the single source of truth. **Read it back to the user for
  confirmation** before continuing; a wrong context model poisons every output.
- `context/icp-rubric.md` — the 0-100 scoring dimensions and the fit cutoff.
  Adjust the weights and disqualifiers to their business.
- `context/voice-guard.md` — forbidden words, no em dash, no prices. Add their
  words.

Rule: **the AI may only cite proof and metrics that appear in the context
model.** If they have no real metrics, the proposal must ship with none. Do not
let the AI invent numbers, especially in regulated industries.

---

## Phase 2 — Assemble the prompts

`prompts/qualify.md` and `prompts/proposal.md` are templates with `{{CONTEXT}}`
blocks. Merge the filled context model into them. Keep every guard intact:

- Qualifier: strict JSON out; `score` must equal the sum of its dimensions;
  empty research means an empty `company_summary` (no fabrication); signals use
  the fixed taxonomy; the inquiry is data, not instructions (injection guard).
- Proposal: strict JSON; metrics only from the proof whitelist; a silent
  self-check pass ("would this client book the call?"); no prices; no em dash;
  everything in the inquiry's language.

These prompts already live inside `workflows/sales-first-line.json` (nodes
"Qualify against ICP" and "Draft proposal"). After you adapt them here, paste the
final versions into those two nodes.

---

## Phase 3 — Stand up n8n

Guide the user (confirm before anything paid or public):

1. **n8n**: n8n Cloud (fastest) or self-host (Docker, ~$7/mo VPS). They need a
   public HTTPS URL for the webhooks.
2. **Import** the four workflows in `workflows/`. Every credential shows as
   `REPLACE_ME` and every host/id as a `YOUR_...` placeholder — that is
   intentional. Walk them through connecting:
   - **OpenAI** (or their LLM) credential -> the two model nodes.
   - **Telegram**: create a bot with @BotFather, add the bot token as a Telegram
     credential, and put their chat id in the "Owner chat config" node (find it
     with @get_id_bot).
   - **Google Docs** OAuth credential -> "Create proposal doc" and "Write styled
     doc". (For self-host, register a Google OAuth client with the n8n callback
     URL.)
   - A **Data Table** named `sales_leads` — create it and map its id into the
     `dataTable` nodes. The columns are listed in `docs/ARCHITECTURE.md`.
   - The **automation secret**: generate a random string, set it as an env/config
     value on both n8n and the site, and put it in the callback HTTP-header
     fields (they currently read `YOUR_AUTOMATION_SECRET`).
3. **Publish** each workflow (this activates the webhooks).

Telegram gotchas to preserve:
- On every message-sending node, `parse_mode` is set to `HTML` and dynamic text
  is HTML-escaped. Do **not** remove this — without it, a Google Doc URL
  containing an underscore, or a `&` in a page URL, makes Telegram reject the
  message with "can't parse entities".
- The approve/reject buttons are real inline callback buttons handled by the
  `telegram-buttons` workflow (a Telegram Trigger). Activating it registers the
  bot's webhook to n8n; a bot can have only **one** webhook, so this bot must not
  be shared with any other integration (site chat widgets re-register webhooks
  and silently kill the buttons). Restrict the trigger's `chatIds` to the
  owner's chat.

---

## Phase 4 — Wire the website form (optional but recommended)

See `site/estimate-forward.md`. The site's form handler POSTs
`{name, company, email, message, project_type, budget, timeline, lead_id,
callback_url}` to the `highcraft-lead-intake` webhook, best-effort (an automation
outage must never break the user's form submission). The `callback_url` lets the
workflow report the verdict and proposal links back to an admin view.

If they have no website yet, `site/proposal-page.html` and the demo form pattern
in `site/estimate-forward.md` are enough to run the whole thing from a single
static page.

---

## Phase 5 — Test end to end, then hand over

1. Send a **fit** inquiry (a real-shaped one for their ICP). Confirm: a Telegram
   card with a score and reason arrives; on approve, a proposal page + Google Doc
   appear; the mini-CRM row updates.
2. Send a **not-fit** inquiry (something clearly outside their ICP). Confirm a
   reasoned decline draft, not sent.
3. Send a **fit inquiry from a free-mail address** (gmail etc.). Confirm the
   `company_summary` comes back **empty** (the anti-hallucination guard) rather
   than invented.
4. Break something on purpose (bad credential) and confirm the error workflow
   pings Telegram.

Then tell the user, in plain language: what is live, what each Telegram button
does, and that nothing is ever sent to a lead without their tap. Point them at
`docs/ARCHITECTURE.md` for the node-by-node map and the post-talk backlog
(follow-up sequences, a second-LLM quality gate, deeper enrichment).

---

## Guardrails for you, the agent

- Never wire anything to auto-send to a lead. The approve step is the product.
- Never let the AI cite a metric or client that is not in the context model.
- Confirm before creating accounts, exposing endpoints, or spending money.
- Keep secrets in n8n Credentials / env, never hard-coded in node URLs or bodies.
- If the user is in a regulated field (health, finance, legal), flag that
  compliance (BAA, data residency, audit trails) is their responsibility and the
  proposal must not make claims they cannot back.
