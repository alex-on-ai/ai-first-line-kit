# Setup (by hand)

Prefer handing this to a coding agent via `AGENTS.md`. If you would rather do it
yourself, here is the whole thing. Budget ~1-2 hours the first time.

## 0. Prerequisites

- An n8n instance with a public HTTPS URL (n8n Cloud, or self-host with Docker on
  a ~$7/mo VPS).
- An LLM API key (OpenAI, Anthropic, or via OpenRouter).
- A Telegram account.
- A Google account (for the proposal Doc). Optional but recommended.
- A website that can POST a form (optional; you can run from the demo page).

## 1. Fill your context (do this first)

Edit the three files in `context/`:

1. `context-model.md` — who you are, what you sell, your ICP, your real proof,
   your voice. Read it back to yourself; if it is vague, the AI will be vague.
2. `icp-rubric.md` — tune the scoring dimensions and the fit cutoff.
3. `voice-guard.md` — add your forbidden words.

## 2. Build the two prompts

Open `prompts/qualify.md` and `prompts/proposal.md`. Replace each `{{CONTEXT}}`
block with the matching parts of your context model, and `{{FORBIDDEN LIST}}`
with your words. Keep every rule below the context block. You will paste these
into two n8n nodes in step 4.

## 3. Import the workflows

In n8n, import all three files from `workflows/`. They come with every secret
replaced by a `REPLACE_ME` / `YOUR_...` placeholder.

## 4. Connect credentials and IDs

- **LLM:** create an OpenAI (or your provider) credential and attach it to the
  "Qualifier model" and "Proposal model" nodes. The nodes are set to a GPT-5.4
  family model; change the model id to yours.
- **Prompts:** paste your assembled prompts into "Qualify against ICP" and "Draft
  proposal" (the system message field).
- **Telegram:** talk to @BotFather, create a bot, copy the token, and add it as a
  Telegram credential on all Telegram nodes. Find your chat id with @get_id_bot
  and put it in the "Owner chat config" node.
- **Google Docs:** add a Google Docs OAuth2 credential and attach it to "Create
  proposal doc" and "Write styled doc". (Self-host: register a Google OAuth
  client and add the n8n callback URL `https://<your-n8n>/rest/oauth2-credential/
  callback` to its authorized redirect URIs.)
- **Data Table:** create a Data Table named `sales_leads` with the columns in
  `docs/ARCHITECTURE.md`, and set its id in every Data-Table node.
- **Automation secret:** generate a random string. Put it in the `x-automation-
  secret` header field of the three "Report ... to site" HTTP nodes, and set the
  same value on your website (step 6). Keep it out of any URL.
- **Error workflow:** in the main workflow's settings, set the Error Workflow to
  `automation-errors`.

### Booking link

The proposal page's "suggested call slots" chips link to a scheduling page
(`https://cal.com/YOUR_HANDLE/YOUR_EVENT` in `proposal-pages.json`, node
"Render proposal page"). Point it at your Cal.com / Calendly event. One nuance
from the course: for COLD outreach, offer two concrete time slots in the
message, not a calendar link; on an inbound proposal page the calendar link is
the right call, and the slot labels double as suggestions.

## 5. Publish

Publish (activate) each of the three workflows. The webhooks go live.

> **Telegram formatting note:** the message nodes use `parse_mode: HTML` and
> HTML-escape their dynamic text. Keep it. Without it, a Google Doc URL with an
> underscore or a `&` in a page URL makes Telegram reject the message.

## 6. Wire your website form (optional)

See `site/estimate-forward.md`. Your form handler POSTs the inquiry to the
`highcraft-lead-intake` webhook, best-effort, with a `callback_url` so the
verdict comes back to an admin view. No website yet? `site/proposal-page.html`
plus the demo form in that doc is enough to run everything from one page.

## 7. Test end to end

1. **Fit inquiry** shaped for your ICP -> Telegram card with score + reason ->
   approve -> proposal page + Google Doc appear.
2. **Not-fit inquiry** -> a reasoned decline draft, not sent.
3. **Fit inquiry from a gmail address** -> `company_summary` comes back empty
   (the anti-hallucination guard), not invented.
4. **Break a credential** -> the error workflow pings your Telegram.

## 8. Go live carefully

- Keep the approve step. Never switch on auto-send to leads.
- Review the first 20-40 outputs closely; then you can sample.
- Add follow-up, a quality-gate LLM, and deeper enrichment later (see the backlog
  in `docs/ARCHITECTURE.md`). Ship the simple version first.
