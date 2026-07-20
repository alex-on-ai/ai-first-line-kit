# AI First Line — a sales inbox that qualifies and drafts, so you don't have to

An open kit for building an **AI-assisted first line of sales**: a website
inquiry comes in, an AI qualifies it against *your* ideal-client profile, you
approve with one tap, and it drafts a real proposal (a branded web page and a
Google Doc) in your voice. Nothing is ever sent to the lead automatically. The
human stays in the loop; the AI does the reading, scoring, and drafting.

This is the exact system shown in the talk. It is built the way a real one
should be: qualification with a reason, a mini-CRM, error alerts, and a proposal
you would not be embarrassed to send.

> **The one idea:** give the AI your company's context once, automate a whole
> flow (not a single prompt), and keep a human on the approve button. Context
> first, then automation.

## What it does

```
Website form  ─▶  Qualify against your ICP   ─▶  You approve in Telegram  ─▶  Proposal
(estimate)        (score 0-100 + why + signals)   (one tap, nothing auto-sent)   (branded page + Google Doc)
                        │                                                              │
                        ▼                                                              ▼
                  mini-CRM log                                                  back to your site
                  (every lead, every verdict)                                  (admin panel)
```

- **Qualifies** each inquiry against a scoring rubric you define (problem fit,
  ICP fit, commercial signal, urgency), with a written reason and named buying
  signals. Not-a-fit gets a polite, reasoned decline draft.
- **Researches** the lead lightly (reads their website by email domain).
- **Drafts a proposal** in your voice: what you heard, a two-week first step, a
  timeline, one relevant case, and a next step. No prices, no invented facts.
- **Two artifacts**: a token-gated branded web page and a styled Google Doc.
- **Human-in-the-loop**: approve or reject in Telegram. Nothing reaches the lead
  without you.
- **Reliable**: retries, timeouts, and a Telegram alert if anything breaks.

## The stack (all swappable)

| Piece | Used here | Swap for |
|-------|-----------|----------|
| Orchestration | [n8n](https://n8n.io) (self-hosted, ~$7/mo) | Make, Zapier |
| LLM | OpenAI GPT-5.4 | Claude, or any model + an OpenRouter fallback |
| Data / mini-CRM | n8n Data Table | Supabase / Postgres |
| Proposal doc | Google Docs API | Notion, PandaDoc |
| Proposal page | n8n webhook serving HTML | Cloudflare Pages, Vercel |
| Approval | Telegram | Slack |
| Website form | Next.js `/estimate` route | any form that can POST JSON |

## How to build it

**You are meant to hand this repo to a coding agent** (Claude Code, Codex, or
similar). Point it at [`AGENTS.md`](AGENTS.md) and answer its questions about
your company. It will fill the context model, adapt the prompts, and walk you
through importing the workflow and connecting credentials.

Prefer to do it by hand? Follow [`docs/SETUP.md`](docs/SETUP.md).

## Repo map

```
AGENTS.md                 agent instructions to set the whole thing up (start here)
docs/SETUP.md             step-by-step for a human
docs/ARCHITECTURE.md      how the flow works, node by node
context/
  context-model.md        WHO YOU ARE — fill this first, everything reads it
  icp-rubric.md           the 0-100 scoring rubric
  voice-guard.md          forbidden words, no em dash, no prices
prompts/
  qualify.md              the qualifier system prompt (uses your context)
  proposal.md             the proposal system prompt (uses your context)
workflows/
  sales-first-line.json   the main n8n workflow (import this)
  proposal-pages.json     serves the branded proposal page
  automation-errors.json  Telegram alert on any failure
site/
  estimate-forward.md     how to wire a website form to the webhook
  proposal-page.html      standalone proposal-page template (reference)
```

## Principles this kit is built on

Drawn from the practitioners the talk cites (Volodymyr Kuts / Profigent, Illia
Azovtsev / GrowthBand, and others):

1. **Automation does not make decisions. It prepares them.** Keep a human on the
   approve button for anything client-facing.
2. **Context before cleverness.** A shared context model beats a clever prompt.
3. **Don't give an agent what an `if` can solve.** n8n does the branching and
   state; the LLM only does bounded judgment.
4. **Score with a reason.** A number the AI can't explain is a number you can't
   trust. Always ask *why*, including why a lead does *not* fit.
5. **Verify, don't invent.** Proof and metrics come from a whitelist. Empty
   research means an empty summary, never a fabricated one.
6. **Quality comes from architecture, not the most expensive model.** Retries,
   guards, and a self-check beat a bigger model with no scaffolding.

## License

MIT. Use it, change it, ship it.
