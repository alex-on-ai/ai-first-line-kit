# ICP scoring rubric

The qualifier scores every inquiry 0-100 as the **sum of four dimensions**, then
decides fit. Tune the weights and the cutoff to your business. Higher-resolution
scoring (0-100, not a 1-5 star) makes the model's judgment more consistent and
easier to explain on screen.

## Dimensions (must sum to 100)

| Dimension | Max | What earns points |
|-----------|-----|-------------------|
| `problem_fit` | 40 | A real operational problem you actually solve. The more it matches a pain in the context model, the higher. |
| `icp_fit` | 25 | Role, company size, region, industry match. |
| `commercial_signal` | 20 | Budget and timeline are plausible for your work and are stated at all. |
| `urgency_authority` | 15 | A decision-maker is writing, and there is a named trigger (deadline, lost revenue, compliance date, growth). |

Adjust the maxima if your business weights things differently (e.g. an agency
that lives on referrals might cut `commercial_signal` and raise `problem_fit`).
Keep them summing to 100.

## The rules the qualifier must follow

- `score` **must equal** `problem_fit + icp_fit + commercial_signal +
  urgency_authority`, and the four numbers must appear in the `why`.
- `fit = true` only when `score >= 55` **AND** no disqualifier from the context
  model applies. Tune the cutoff to how selective you are.
- Judge the **message content first**. Website research and the email domain are
  secondary evidence.
- If research is empty, do not award points for things the message does not
  state, and return an empty `company_summary`. Never invent.
- Always produce a `why`, including **why a lead does not fit**. A score with no
  reason is a black box.

## Signal taxonomy

`signals[]` should prefer these labels where they truly apply, plus short
free-text only when none fit:

`hiring` · `funding` · `tech_change` · `new_role` · `team_growth` ·
`stack_match` · `budget_stated` · `urgency`

## Timing

`timing` is `hot` / `warm` / `cold` — how ready this lead is to move, based on
stated urgency, authority, and a concrete trigger.
