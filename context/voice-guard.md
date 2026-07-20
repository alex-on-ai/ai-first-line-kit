# Voice guard

Rules both the qualifier and the proposal writer must obey when they produce any
text a human might send. This is what keeps AI output from reading like AI
output. Add your own words.

## Never

- **No em dash** (`—`). It is the number-one "an AI wrote this" tell. Use commas,
  colons, or short sentences.
- **No prices.** Never a number, never a range. The number is fixed in writing
  after a call. (An audit or a first draft is not a price for complex work.)
- **No invented facts.** Company details, proof, and metrics come only from the
  context model. Empty research means an empty summary.
- **No exclamation marks, no emoji** (unless your brand truly uses them).

## Forbidden words (starter list — add yours)

```
leverage        unlock          seamless        world-class
transformative  cutting-edge    passionate      game-changing
revolutionize   synergy         "comprehensive solution"
"delve into"    "in today's ..."  "Whether you're X, Y, or Z" openers
```

Plus: any copy that shrinks your company ("small", "boutique", "just a") unless
that is deliberately your positioning.

## Do

- A number beats an adjective ("shipped in two weeks", not "fast").
- Show, don't claim: proof first, assertion never.
- Write in the **inquiry's language** (reply to a Ukrainian inquiry in
  Ukrainian, etc.), including durations ("два тижні", not "two weeks").
- Phrase "why us" as a benefit to the buyer, not a brag about you.

## Self-check (built into the proposal prompt)

Before returning, the model silently re-reads the draft as the client and asks
"would I book the call?", checking relevance, voice, logic, and clarity, and
revises once if weak. Then it verifies: valid JSON, no em dash, no forbidden
words, no prices, metrics only from the whitelist.
