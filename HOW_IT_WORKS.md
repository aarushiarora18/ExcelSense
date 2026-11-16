# How ExcelSense v2 works

ExcelSense answers plain-English questions about five life-insurance CSVs
(25,000 policies). It does **not** let a language model write queries or invent
figures. Instead, every question is matched to one of **103 pre-written analyses**
("problem statements", PS-001 … PS-103). The chosen analysis is run
deterministically with pandas, and only then is the answer written, either by a
template or optionally by Groq (and any model's answer is checked against the
computed numbers).

```
question
  │
  ▼
1. Entry point / UI ──────────── app.py, src/ui/streamlit_app.py, src/ui/cli.py
  │
  ▼
2. Security screening ────────── src/security/guards.py
  │
  ▼
3. Conversation context ──────── src/api/chatbot.py  (Conversation)
  │
  ▼
4. Routing: pick one analysis ── src/router/  (normalizer → entities → matcher → router)
  │        └─ not routed? → clarify or refuse (src/api/chatbot.py `_not_routed`)
  ▼
5. Execution: compute evidence ─ src/ps/executor.py → src/analysis/ → src/data/, src/logic/
  │
  ▼
6. Answer writing ────────────── src/llm/  (summarizer → client / composer → validator)
  │
  ▼
7. Tables, chart, final checks ─ src/evidence/tables.py, src/visualization/charts.py,
  │                              src/api/chatbot.py (`_finish`), src/security/audit.py
  ▼
8. Display ───────────────────── src/ui/streamlit_app.py
```

The whole pipeline is driven by `Chatbot.ask()` in `src/api/chatbot.py`.

---

## Step 0 — Data and configuration (loaded once)

Before any question is asked, the data and the analysis catalogue are loaded.

| What | Files |
|---|---|
| The five source CSVs | `data/csv/Policy_Details.csv`, `Owner_Details.csv`, `Sales_Details.csv`, `Persistency_Details.csv`, `Claims_Details.csv` |
| Data contract: tables, columns, types, joins, sentinels, known data-quality findings | `config/data_contract.json`, read by `src/data/contract.py` |
| Loading and typing the CSVs, reporting every value that fails to parse | `src/data/loaders.py` (`load_dataset`, `Dataset`, `Table`) |
| The 103 analyses: statement, example questions, concepts, tables, execution plan | `config/problem_statements.json` and `data/PS/*.json`, read by `src/ps/registry.py` (`PSRegistry`) |
| Which formulas and business rules each analysis uses | `config/ps_logic_mapping.json`, read by `src/ps/registry.py` |

The analysis catalogue is **generated**, not hand-edited:

- Source of truth: `scripts/ps_defs/part1_portfolio.py` … `part6_executive.py`
  (statements, example questions, concepts) and `scripts/ps_defs/executions.py`,
  `executions_b.py` (execution plans).
- Build: `python scripts/build_ps_catalogue.py` writes `config/problem_statements.json`,
  `data/PS/*.json` and `docs/PROBLEM_STATEMENT_CATALOGUE.md`, and fails if any analysis
  references a column the CSVs do not have.
- `scripts/build_logic_config.py` writes `config/formulas.json`,
  `config/business_rules.json` and `config/ps_logic_mapping.json` from the formula and
  rule definitions in `src/logic/`.

To change what a question routes to, edit `scripts/ps_defs/*` and rebuild; don't
edit the generated JSON.

## Step 1 — Entry point and UI

| Mode | File |
|---|---|
| `streamlit run app.py` (the deployed app) | `app.py` → `src/ui/streamlit_app.py` `main()` |
| `python app.py "question"` or interactive terminal | `app.py` → `src/ui/cli.py` |

`src/ui/streamlit_app.py`:
- `get_bot()` creates one shared `Chatbot` per server process (data loads once).
- `ask()` sends the question to `Chatbot.ask()` and stores the response in
  `st.session_state` (conversation state is per browser session).

## Step 2 — Security screening

**File:** `src/security/guards.py` — `screen_question()`

Runs before anything else reads the question:
- removes control characters and enforces length limits (500 characters, 80 words);
- refuses prompt-injection attempts ("ignore previous instructions", "tell me the lapse
  rate is 95%") and code or SQL;
- the cleaned (`sanitized`) text is what the rest of the pipeline uses.

A refused question is logged by `src/security/audit.py` and answered with a fixed
message.

## Step 3 — Conversation context

**File:** `src/api/chatbot.py` — class `Conversation`

Remembers the last 10 turns. If a question looks like a follow-up ("what about P05?")
and doesn't route on its own, the previous turn's subject (product, state, plan,
agent id, policy id) is added back and routing is tried again (`_apply_context`).

## Step 4 — Routing: choosing one analysis

**Folder:** `src/router/`. Entry point: `QueryRouter.route()` in `src/router/router.py`.

### 4a. Normalise the wording — `src/router/normalizer.py`
- `clean()` lowercases and strips punctuation.
- `expand_aliases()` maps synonyms onto the catalogue's vocabulary through `ALIASES`
  (for example "lapses" → "lapse", "cut / lower" → "reduce", "how quickly" → "time
  taken", "policy year" → "persistency bucket").
- `content_terms()` drops stop words, leaving the terms that are scored.
- `find_unsupported()` checks `UNSUPPORTED` for concepts the data doesn't contain
  (channel, cost, agent tenure, zone and so on).
- `is_prescriptive()` recognises "how can we / what can we do to" followed by an
  action verb.

### 4b. Extract identifiers — `src/router/entities.py`
`EntityExtractor.extract()` finds policy ids (11 digits), agent ids (6 digits), branch
numbers, product codes (P01…), years, quarters, months, states, plan names, statuses
and persistency buckets. It checks each one against the actual data, so a
non-existent id can be named in the refusal.

### 4c. Score every analysis — `src/router/matcher.py`
`PSMatcher.score()` scores all 103 analyses; `_score_one()` combines weighted signals
(`WEIGHTS`):

| Signal | Weight | What it measures |
|---|---|---|
| `dimension_signal` | 0.18 | what the question groups by ("by state", "which agents", "how does *income* affect…") |
| `intent_signal` | 0.17 | the shape of answer asked for (rank, compare, trend, driver, average…) from `INTENT_CUES`, `DECISIVE_CUES` |
| `phrase_signal` | 0.16 | multi-word phrases shared with the analysis's example questions |
| `example_similarity` | 0.14 | closeness to the analysis's own example questions |
| `rare_term_signal` | 0.10 | shared words weighted by rarity (IDF) |
| `concept_overlap` | 0.07 | declared concepts the question mentions |
| `statement_similarity` | 0.06 | closeness to the analysis's statement |
| `domain_signal` | 0.06 | vocabulary tied to the analysis's tables and columns |
| `tier_signal` | 0.05 | descriptive / analytical / diagnostic / executive (`TIER_CUES`) |
| `entity_fit` | 0.01 | identifiers the question supplies that the analysis accepts |

`PSMatcher.uncovered()` is the **coverage check**: every measure the question names
(`MEASURE_FAMILIES`: claims, conduct, APE/premium, lapse, surrender, persistency) and
every grouping it asks for must be something the analysis actually covers.

### 4d. Decide — `src/router/router.py` `route()`
Applied in order (thresholds at the top of the file):

1. **Unsupported data** → refuse, naming what's missing.
2. **Unknown identifier** (policy id, agent id, product, year not in the data) →
   refuse, naming it.
3. **Best score below `CONFIDENCE_FLOOR` (0.32)** → refuse rather than guess.
4. **A state is named** but the best analysis can't report by state → use the best
   state-level analysis, or ask.
5. **Coverage:** if the best match leaves part of the question uncovered, use a
   runner-up within `COVERAGE_MARGIN` (0.10) that covers everything, or ask.
6. **Too close to call:** if a runner-up that covers the whole question is within
   `AMBIGUITY_MARGIN` (0.020) → ask, offering both.
7. Otherwise → **routed**, with any parameters (id, product, year) taken from the
   entities.

If the question isn't routed, `Chatbot._not_routed()` in `src/api/chatbot.py`
turns the decision into a clarification ("Did you mean: …?") or a refusal.

## Step 5 — Execution: computing the evidence

**File:** `src/ps/executor.py` — `PSExecutor.execute()`

1. Validates the parameters against the ones the analysis declares
   (`src/ps/registry.py` `validate_parameters`).
2. Checks the result cache (`src/ps/cache.py`). Results are keyed by analysis,
   parameters and a fingerprint of the data, so a cached figure is never stale.
3. Fills `$placeholders` in the analysis's execution plan and runs its **pattern**.

**Patterns** (`src/analysis/patterns.py` `PATTERNS`; three more in
`src/analysis/advanced_patterns.py`):

| Pattern | Used for |
|---|---|
| `aggregate` | a single figure (overall lapse rate, total APE) |
| `group_metrics` | a figure per group (lapse rate by state) |
| `top_n`, `concentration`, `outliers` | rankings, dependence on top agents/customers, unusual values |
| `distribution`, `timeseries` | spreads and trends |
| `cohort_compare`, `driver_scan` | comparing groups, what is associated with lapse |
| `record_lookup` | one policy or agent |
| `reconcile`, `scope_check`, `quality_report` | data integrity checks |
| `executive` | the overview / scorecard |
| `composite_score`, `two_stage_group`, `rule_breach` | agent quality score, customer segments, unusual combinations |

Patterns use:
- `src/data/query_engine.py`: `Query` (filter, join, derive, group_by, rank…), which
  records every step and the rows it dropped as **provenance**;
- `src/logic/formula_engine.py`: every metric (lapse rate, settlement ratio…) is
  computed by a named formula, so each figure carries its formula id and inputs;
- `src/logic/rule_engine.py`: business rules (for example, removing personal fields,
  minimum sample sizes) applied and recorded.

The result is an **evidence package** (`src/ps/evidence.py` `EvidencePackage`): the
figures, the result table, rules and formulas used, data-quality notes, limitations
and provenance. If anything fails, it is marked *insufficient* rather than guessed.
`_finalise()` attaches the analysis's declared limitations and, for diagnostic
analyses, a note that the data can't establish causes.

## Step 6 — Writing the answer

**Folder:** `src/llm/`. Entry point: `Summarizer.summarize()` in `src/llm/summarizer.py`.

| Job | File |
|---|---|
| Template answer from the evidence (no model) | `src/llm/composer.py` `compose_answer()` |
| Model selection: Groq if `GROQ_API_KEY` is set, otherwise the deterministic client | `src/llm/client.py` `default_client()` |
| The prompt and the evidence sent to the model | `src/llm/contract.py` (`SYSTEM_PROMPT`, `build_messages`) |
| Checking a model's answer | `src/llm/validator.py` `validate_answer()` |

The validator rejects a model answer that:
- contains a number not in the evidence;
- names an entity not in the evidence;
- makes a causal claim the data can't support;
- leaks internal names;
- is too long;
- ignores insufficient evidence.

A rejected or failed model answer falls back to the template answer. With no API key
(the current deployment), the template answer is always used.

`Chatbot.ask()` then adds two lead sentences where they apply:
- **a named state:** that state's own figure first (`src/llm/composer.py`
  `focus_sentence`);
- **a prescriptive question** ("how can we reduce lapses?"): the `PRESCRIPTIVE_LEAD`
  sentence saying the data shows where to act, not which action will work.

## Step 7 — Tables, chart and final checks

| Job | File |
|---|---|
| Readable tables with explicit units (%, ₹, counts) | `src/evidence/tables.py` `build_tables()` |
| Chart specification, only from values already in the evidence | `src/visualization/charts.py` `build_chart()` |
| Last check that no personal field (name, postal code) reaches the response | `src/api/chatbot.py` `_assert_no_personal_data()`, using `src/security/guards.py` `find_personal_fields()` |
| Audit log of the decision, without the raw question or identifiers | `src/security/audit.py` |

The response is a `ChatResponse` with outcome `answered`, `needs_clarification`,
`refused` or `could_not_complete`.

## Step 8 — Display

**File:** `src/ui/streamlit_app.py`
- `render_turn()` shows the answer, suggestions or refusal.
- `render_chart()` and `_category_bars()` draw charts (Altair bars in evidence
  order, starting at zero).
- `render_evidence()` shows the tables, formulas, rules and provenance in an
  expander.
- `render_sidebar()` shows example questions and session controls.

---

## Worked example: "How does occupation relate to lapses?"

| Step | What happens | Where |
|---|---|---|
| 2 | Passes screening | `guards.py` |
| 4a | "lapses" → "lapse lapsed"; terms: occupation, relate, lapse | `normalizer.py` |
| 4c | "relate" is a relationship word, so "occupation" is treated as the grouping; PS-037 (lapse rate by occupation) scores highest | `matcher.py` |
| 4d | Above the floor, covers lapse and occupation, no close rival → routed to PS-037 | `router.py` |
| 5 | `group_metrics` pattern: join policies to owners, flag lapsed policies, group by occupation, lapse rate = lapsed ÷ policies, ranked against the portfolio rate | `executor.py`, `patterns.py`, `query_engine.py`, `formula_engine.py` |
| 6 | Template answer names the highest and lowest occupations against the portfolio rate | `composer.py` |
| 7–8 | Table and bar chart of lapse rate by occupation | `tables.py`, `charts.py`, `streamlit_app.py` |

## Tests

| Area | Tests |
|---|---|
| Data loading and query engine | `tests/data/` |
| Formulas and business rules | `tests/logic/` |
| Patterns and executor | `tests/analysis/`, `tests/ps/` |
| Routing | `tests/router/` and `tests/regression/test_routing_regression.py` (533 catalogue questions against `tests/regression/baseline.json`) |
| Chatbot end to end, including the fixes in `UPDATES.md` | `tests/api/test_chatbot.py`, `tests/e2e/` |
| Answer writing and validation | `tests/llm/` |
| Security | `tests/security/` |
| Tables, charts, UI | `tests/evidence/`, `tests/visualization/`, `tests/ui/` |
| Benchmark (writes `tests/benchmark/results.json`) | `tests/benchmark/` |

Run everything with `pytest`.
