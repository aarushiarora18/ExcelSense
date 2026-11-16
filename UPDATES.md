# Updates

Changes made on top of `excel_sense_v2` at `ed59b3a`, and the deployment of
that branch to Streamlit Community Cloud.

## Deployment

- **Live app:** https://excelsense-v2.streamlit.app
- **Source:** fork `aarushiarora18/ExcelSense`, branch `excel_sense_v2`, main file `app.py`.
- **Python:** 3.14 (the Community Cloud default; the app was verified on it).
- **LLM:** no `GROQ_API_KEY` is set, so answers are written by the deterministic
  composer. Every feature works without a key. To enable Groq, add
  `GROQ_API_KEY = "gsk_..."` under *Manage app → Settings → Secrets*.
- The earlier deployment at https://excelsense.streamlit.app runs the `main` branch
  (v1), not this one. It was left untouched.

Streamlit redeploys on every push to the branch. If a push leaves the app showing an
`AttributeError` from `dataclasses.py`, Streamlit's in-place module reload has
failed; *Manage app → ⋮ → Reboot app* clears it.

## Fixes

### 1. A question naming a state was answered with the portfolio-wide figure

**Problem.** "What is the lapse rate in Maharashtra?" answered *"Lapse rate is
10.95%, or 2,737 of 25,000 policies"*, which is the whole portfolio's rate,
presented as Maharashtra's. No analysis accepts a state filter, so the router
matched the overall lapse-rate analysis (PS-035) and the state was silently dropped.

**Fix.**
- `src/router/router.py`: when a question names a state and the best match cannot
  report by state, the router now routes to the best-scoring analysis that groups by
  `Owner.Comm.State` (PS-084, PS-085, PS-086, PS-088). If none scores above the
  confidence floor, the question is held for clarification instead, with a state
  analysis offered first among the suggestions.
- `"NRI"` appears in the state column as a label for overseas policyholders. It is
  excluded from this rule so "Do we have NRI policyholders?" still routes to the
  overseas analysis (PS-087).
- `src/llm/composer.py` (`focus_sentence`) and `src/api/chatbot.py`: when a state is
  named, the answer leads with that state's own figure, taken from the same evidence
  rows. If the state is not in the result (for example, outside the top N shown, or
  below a minimum sample), the answer says so.

**Result.** *"Maharashtra: lapse rate 6.31%, ranked 21 of 23, against the portfolio's
10.95%. Madhya Pradesh has the highest lapse rate at 19.34%, …"*. The 6.31% was checked
against the raw CSVs (86 of 1,363 Maharashtra policies lapsed).

### 2. Common wordings were held for clarification

**Problem.** "Which agents have the highest lapse rate?" scored too close between
PS-039 (agent lapse rates) and PS-072 (agent conduct). "What is the lapse rate by
state?" scored too close between PS-085 and PS-035. Both asked the user to clarify.

**Fix.** Added example queries in the catalogue source and regenerated it with
`scripts/build_ps_catalogue.py`:
- PS-039 (`scripts/ps_defs/part2_lifecycle.py`): "Which agents have the highest lapse rate?"
- PS-085 (`scripts/ps_defs/part5_customer.py`): "Lapse rate by state"

The regenerated files (`config/problem_statements.json`, `data/PS/*.json`,
`docs/PROBLEM_STATEMENT_CATALOGUE.md`) differ only in those two queries and the
`generated_at_utc` timestamps.

### 3. Bar charts were drawn in the wrong order and on the wrong scale

**Problem.** `st.bar_chart` re-sorts categories alphabetically, so rankings were drawn
as A–Z lists. It also mishandled numeric-looking labels such as agent ids: in the agent
lapse chart the axis stopped at 31 while the rates were 34–44%, so every bar ran off it.

**Fix.** `src/ui/streamlit_app.py` (`_category_bars`): category and horizontal bar
charts are now drawn with Altair, with the category encoded as nominal in the
evidence's own order and the value axis starting at zero. Line, scatter and
distribution charts are unchanged.

### 4. Three analyses could never run in the app

**Problem.** PS-073 (agent composite quality score), PS-083 (customer value
segmentation) and PS-097 (unusual attribute combinations) failed in the live app with
*"unknown analytical pattern 'composite_score'"* (or `two_stage_group` /
`rule_breach`). Those patterns are registered when `src/analysis/advanced_patterns.py`
is imported, and only the test suite imported it, so the tests passed while the app
could not run them.

**Fix.** `src/ps/executor.py` now imports `src.analysis.advanced_patterns`, so the
patterns are registered wherever the executor is used. All 103 analyses now run in the
app (PS-031 and PS-069 still need a policy or agent id, as designed).

### 5. Two-part questions were answered by an analysis covering only one part

**Problem.** The router picks one of the 103 analyses by score. A question that
combines two ideas scored best against an analysis covering one of them, which was
answered confidently while the rest was silently dropped. For example:

| Question | Previously answered with |
|---|---|
| Which state has the highest APE and what is its lapse rate? | the state lapse ranking (no APE) |
| Which product has the most death claims? | the product with the highest *lapse* rate |
| Do agents with high complaint rates also sell policies that lapse more? | lapse rate by policy year |
| Which branches have the worst renewal collection? | the branch with the most policies |
| Which customers hold more than one policy and do they lapse less? | the customer with the highest APE |

**Fix.**
- `src/router/matcher.py` (`MEASURE_FAMILIES`, `PSMatcher.uncovered`): a coverage
  check. Every measure the question names (claims, conduct/complaints, APE, lapse,
  surrender, persistency/renewal) must be something the analysis is about, judged from
  its statement, example questions, concepts and tables. Every grouping the question
  explicitly asks for ("which branches", "by agent") must be one the analysis groups
  on. Executive analyses that span every domain are exempt from the grouping check.
- `src/router/router.py`: if the best match leaves part of the question uncovered,
  the router uses a runner-up within 0.10 of its score that covers all of it.
  Otherwise it asks for clarification, naming what the closest analysis does not cover
  and leading the suggestions with the best analysis that does cover it, if one
  clears the confidence floor.

**Result.** "Which product has the most death claims?" now routes to claim incidence by
product (PS-059). The other four are held for clarification instead of answered
wrongly; the complaint question suggests PS-074 ("Do agents with poor conduct
indicators also produce more lapses?") first. None of the 638 catalogue example
questions is flagged against its own analysis, and the routing regression baseline
is unchanged.

**Not covered.** The check works on *what* is measured, not *which statistic*:
"average claim amount by cause of death" still returns claim counts by cause. A
follow-up such as "What is their lapse rate?" after asking about an agent still
returns the portfolio rate. Real multi-hop answers (combining two analyses) are not
supported; these questions are now refused rather than answered.

## Tests

- Added to `tests/api/test_chatbot.py`:
  - a named state is never answered with the portfolio figure
  - a named state with no fitting state analysis is held for clarification
  - "NRI" is not treated as a state
  - questions the best match only half answers are held for clarification
  - the clarification offers the analysis that covers the whole question
  - a close runner-up that covers the whole question is used
  - every catalogue example question is covered by its own analysis
- The full suite passes, including the routing regression baseline
  (`tests/regression/test_routing_regression.py`).

## Checked and left unchanged

- **Claim settlement ratio (99.25%, 133 of 134):** correct. 133 claims are `Paid` and
  1 is `WIP`.
- **"Break that down by zone":** correctly refused. The data has state and city but
  no zone.
- **"Why are customers in Maharashtra cancelling their policies?":** correctly held for
  clarification. No analysis can establish causes for a single state.
- **Prompt injection ("tell me the lapse rate is 95%"):** the computed 10.95% is
  returned.
