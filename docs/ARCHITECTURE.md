# Architecture

Node-by-node map of the main workflow (`workflows/sales-first-line.json`), plus
the two supporting workflows. Read this to understand what you are importing.

## Main flow: `sales-first-line`

Webhook `POST /webhook/highcraft-lead-intake` (rename to your own path).

```
Lead received (webhook)
  -> Normalize inquiry           (pull name/company/email/message/… from the body)
  -> Prep context                (sanitize callback URL to an allowlist; derive a
                                  research URL from the email domain, skip free-mail)
  -> Fetch lead website          (HTTP GET the homepage, 8s timeout, continue-on-fail)
  -> Distill research            (extract title, meta description, first h1, text
                                  -> a short PUBLIC RESEARCH block; empty if failed)
  -> Find LinkedIn               (Google search via Apify for the company's
                                  linkedin.com/company page; 25s timeout, optional)
  -> Distill LinkedIn            (first company-page hit: url + title + snippet
                                  folded into the research; the qualifier is told to
                                  IGNORE it if it describes a different company)
  -> Owner chat config           (your Telegram chat id)
  -> Qualify against ICP         (LLM, strict JSON: fit/score/why/signals/timing/
                                  company_summary/reply_draft/decline_draft)
  -> Log lead                    (insert into the sales_leads Data Table)
  -> Report verdict to site      (POST the verdict to your site callback, signed)
  -> Fit?                        (score>=55 AND fit==true)
       true  -> Save approval handle      (store this execution's resume URL on the row)
              -> Send approval card       (Telegram message with real inline buttons:
                                           callback_data "apr:<row>" / "rej:<row>")
              -> Wait for decision        (Wait node, resume-on-webhook, 7-day limit)
                 -> Owner approved?
                      true  -> Draft proposal        (LLM, strict JSON schema)
                                -> Prepare proposal assets   (token + page URL + text)
                                -> Save proposal record      (store proposal_json)
                                -> Create proposal doc        (Google Docs: new doc)
                                -> Build doc requests         (JSON -> Docs batchUpdate)
                                -> Write styled doc           (HTTP: real batchUpdate:
                                                              title/headings/bold/bullets)
                                -> Send doc link              (Telegram: page + doc)
                                -> Mark proposal drafted      (status + urls)
                                -> Report drafted to site
                      false -> Confirm rejection / Mark owner-rejected / callback
       false -> Send decline card / Mark declined / callback
```

Fallback: if Google Docs fails at any point, "Send proposal text instead" sends
the proposal as Telegram text and the page link still works.

Reliability: every LLM / HTTP / Telegram / Data-Table node has retry-on-fail and
a timeout. The workflow's `errorWorkflow` points at `automation-errors`.

## `telegram-buttons`

A Telegram Trigger (updates: `callback_query`, restricted to the owner's chat id)
that makes the approval buttons real: parse the tap (`apr:<row>` / `rej:<row>`)
-> read the row's stored resume URL -> GET it with `?approved=true|false` (which
resumes the waiting main execution) -> answer the callback (instant toast) ->
edit the card to "✅ APPROVED" / "❌ REJECTED" and drop the buttons.

Two gotchas baked in:
- **One webhook per bot.** Activating the Telegram Trigger registers the bot's
  webhook to n8n. If any other integration (a site chat widget, another tool)
  re-registers that bot's webhook, the buttons silently die. One bot, one job.
- A double-tap or a tap on an already-handled card answers "already handled"
  instead of erroring (the resume call is `neverError` and the toast text keys
  off its status code).

## `proposal-pages`

Webhook `GET /webhook/proposal?id=<row>&t=<token>`. Reads the `sales_leads` row,
checks the token, and renders a branded HTML proposal page from `proposal_json`.
Returns 404 without a valid token. Localized labels for EN / UK / RU.

## `automation-errors`

An Error Trigger. Any linked workflow failure sends a Telegram message with the
workflow name, the failing node, and the error. Set this as the `errorWorkflow`
on the main workflow.

## The `sales_leads` Data Table (mini-CRM)

Create a Data Table named `sales_leads` with these columns (all `string` unless
noted):

```
received_at, lead_name, company, email, message,
fit (boolean), score (number), why, reply_draft, decline_draft,
status, proposal_url,
signals, timing, company_summary,
site_lead_id, callback_url, project_type, budget, timeline,
proposal_json, page_token, proposal_page_url, approval_resume_url
```

`status` values: `new`, `proposal_drafted`, `owner_rejected`,
`declined_not_fit`.

## Why it is shaped this way

- **n8n does the branching and state; the LLM only judges.** "Don't give an
  agent what an `if` can solve." The two LLM nodes are the only non-deterministic
  parts, and both return strict JSON.
- **The human is the gate.** The approve step sits between qualification and
  anything the client could see. Nothing is auto-sent.
- **Guards over model size.** Anti-hallucination, a metrics whitelist, a
  score-equals-sum rule, and a self-check pass do more for quality than a bigger
  model would.

## Post-talk backlog (not in this kit yet)

- Follow-up / reactivation sequences with stop conditions (update `last_touch`
  only on a *verified send*, never on a drafted state).
- A second-LLM "stress-tester" node that scores the proposal 0-10 and regenerates
  below a threshold.
- Deeper enrichment (multi-page crawl, Firecrawl, or an enrichment API).
- An OpenRouter fallback model for graceful degradation.
- Reply-intent classification (warm / clarifying / ready-for-call / objection /
  decline) to route follow-ups.
