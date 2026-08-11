# GEO Measurement Prototype — Design

**Date:** 2026-08-10
**Status:** Approved (brainstorming complete, pending implementation plan)
**Author:** Mohammad Ali Shehral

---

## 1. Context

The GEO project foundation document proposes three modules: an Analytical Diagnostic Engine (a "GEO Score"), an Automated Data Pipeline (nightly citation tracking), and a Human-AI Interactive Workspace (co-writing with human-preserved voice). Those modules assume that citation of a source by a generative engine can be measured reliably enough to detect the effect of a content change.

The critical survey of the field (Martinez, *Optimizing Visibility in Generative Engines*, arXiv:2607.14035, July 2026, 45 studies) shows that assumption is unproven and probably optimistic:

- Daily source-level Jaccard across four engines over 45 days is approximately **0.34–0.42** (Schulte et al. 2026), with 7–8 repetitions per prompt recommended as a starting point.
- Even at temperature 0, repeated runs change **9–28%** of decisions (Kirsten et al. 2026).
- AI Overview page overlap over two months is **18%**, versus 45% for organic Google.
- **57.8%** of ChatGPT decisions in one study did not activate web search at all.
- The field's confidence in "a white-hat intervention durably improves organic discoverability" is rated **Low**; in "citation scores predict clicks, conversions, or revenue" it is rated **Very low**.

If the measurement floor is above the effect size of realistic content changes, then a scorer, a rewriting workspace, and a citation dashboard are all instruments for reporting noise. This prototype exists to find out, before any of that is built.

## 2. Goal

**Determine whether AI-engine citation can be measured precisely enough, at an affordable cost, to detect a realistic content-intervention effect.**

The deliverable is a reusable measurement instrument plus four concrete outputs (Section 12). It is explicitly a go/no-go artifact, not a product.

### Non-goals

This prototype does not build, and must not be extended to build:

- A GEO Score or any content scorer
- A rewriting or co-writing workspace
- Any frontend, dashboard application, user accounts, or CMS integration
- Competitor feature extraction or dynamic weight retuning
- Google AI Overviews measurement (no sanctioned API; see Section 3.3)

Each is a legitimate part of the eventual project. None helps answer the measurement question, and each would encode assumptions this prototype exists to test.

## 3. Decisions and rationale

### 3.1 Prototype goal: de-risk the measurement

Chosen over "publishable result," "demoable system," and "reusable foundation." Measurement precision is the load-bearing unknown; every other module inherits its error.

### 3.2 Measurement target: observational panel now, controlled site later

The panel tracks which domains get cited across a fixed query set, with no intervention. The schema and analysis are designed so that a controlled site — one whose pages we can modify and republish — can later be added as a treated unit without redesign.

Rationale: an observational panel fully answers "what is the noise floor?" and requires no web property. Causal claims require intervention, so the design must not foreclose it. The `citations.registrable_domain` field plus a `treated` flag on domains (Section 6) is the entire forward-compatibility cost.

### 3.3 Engine surfaces: sanctioned APIs only

Four adapters at launch: Perplexity Sonar, OpenAI web search, Google Gemini with Search grounding, Anthropic Claude web search.

Google AI Overviews is excluded. It has no public API, and scraping SERPs violates Google's Terms of Service — an unacceptable exposure for a university project with a potential industry partner.

**Stated validity limitation, to appear in every output:** the survey notes (§2.4) that "an API receiving URLs, a model equipped with a web tool, and a consumer-facing interface are not the same system." This panel measures API surfaces. Results must not be reported as describing what consumer-UI users see. Characterizing the difference between API and consumer surface stability is itself an open question (see Section 13, Extension B).

### 3.4 Budget: parameterized, with a cost model produced before spend

No budget is committed. `estimate.py` produces a projection from the panel definition so that a concrete number can be taken to whoever funds the work. The collector enforces a hard budget cap.

### 3.5 Scheduling: clustered bursts, not a nightly cron

The foundation document specifies a single nightly 2:00 AM run. At the observed variance, one sample per day per query is indistinguishable from noise. Replaced with repetitions clustered closely in time across multiple daily windows (Section 8).

## 4. Architecture

This is a measurement instrument. The design priority is provenance and reproducibility, not throughput or features. Every reported number must be traceable to a raw response retained on disk.

```
panel.yaml ──> estimate.py ──> cost projection ($0 spent)
    │
    └──> collect.py ──> raw/<sweep_id>/<call_id>.json  (immutable)
                    └──> calls
                             │
                       normalize.py
                             │
                    ┌────────┴────────┐
              observations         citations
                    └────────┬────────┘
                        analyze.py
                             │
                        report HTML
```

A **sweep** is one complete pass over the panel at one point in time. Sweeps are the unit of scheduling, idempotency, and analysis.

### Stack

- Python 3.12, dependency management via `uv`
- `httpx` (async) for API calls
- SQLite for structured data; JSON files on disk for raw responses
- `pytest` for tests
- `statsmodels` / `pymer4`-style mixed-effects fitting, with simulation-based power analysis
- Report generation follows the `build_report.py` pattern from `nn-infinite-gen`

SQLite is chosen deliberately over Postgres. The panel produces thousands of rows per day, not millions per hour. A single portable file that can be attached to a paper's artifact package is worth more than a server that has to be operated.

## 5. Components

| Component | Responsibility | Interface |
|---|---|---|
| `panel.yaml` | Experiment definition: queries, paraphrases, engines, repetitions, windows, budget cap | Declarative config, versioned in git |
| `pricing.yaml` | Per-engine per-call and per-token prices, with a `verified_on` date | User-maintained input |
| `estimate.py` | Projects calls/day, cost/day, cost/month per engine from panel + pricing | `estimate(panel, pricing) -> CostModel` |
| `adapters/<engine>.py` | One file per engine: issue request, return raw response; parse raw into observation | `query(text) -> RawResponse`<br>`parse(raw) -> Observation` |
| `collect.py` | Execute a sweep: async, rate-limited, budget-capped, idempotent per `sweep_id` | `run_sweep(panel, sweep_id)` |
| `normalize.py` | Activation detection, citation extraction, URL canonicalization | `normalize(raw) -> Observation` |
| `analyze.py` | Decomposition, variance components, Jaccard, MDE via simulation | `analyze(db) -> Results` |
| `report.py` | Self-contained HTML report | `build(results) -> path` |

`EngineAdapter` is the key isolation boundary. Adding a fifth engine means writing one file plus a fixture test, touching nothing else. Adapters must not perform analysis, and `analyze.py` must never call an adapter.

### `pricing.yaml` as an input, not a constant

Vendor prices change and are not knowable at design time. `pricing.yaml` is a first-class user-maintained input carrying a `verified_on` date. `estimate.py` emits a warning if `verified_on` is more than 30 days old. Prices are never hardcoded in source.

## 6. Data model

```sql
CREATE TABLE sweeps (
    sweep_id           TEXT PRIMARY KEY,
    started_at         TEXT NOT NULL,
    completed_at       TEXT,
    status             TEXT NOT NULL,      -- running | complete | incomplete | aborted_budget
    panel_config_hash  TEXT NOT NULL,
    pricing_hash       TEXT NOT NULL,
    git_sha            TEXT NOT NULL,
    window_label       TEXT NOT NULL,      -- e.g. "morning"
    budget_spent_usd   REAL NOT NULL DEFAULT 0
);

CREATE TABLE calls (
    call_id        TEXT PRIMARY KEY,
    sweep_id       TEXT NOT NULL REFERENCES sweeps(sweep_id),
    query_id       TEXT NOT NULL,
    paraphrase_id  TEXT NOT NULL,
    engine         TEXT NOT NULL,
    model_snapshot TEXT,                   -- as reported by the API response
    rep_index      INTEGER NOT NULL,
    requested_at   TEXT NOT NULL,
    latency_ms     INTEGER,
    http_status    INTEGER,
    outcome        TEXT NOT NULL,          -- ok | transport_error | engine_refusal
    error_class    TEXT,
    retry_count    INTEGER NOT NULL DEFAULT 0,
    backoff_ms     INTEGER NOT NULL DEFAULT 0,
    raw_path       TEXT,
    content_hash   TEXT,
    cost_usd       REAL,
    UNIQUE (sweep_id, query_id, paraphrase_id, engine, rep_index)
);

CREATE TABLE observations (
    call_id     TEXT PRIMARY KEY REFERENCES calls(call_id),
    activated   INTEGER NOT NULL,          -- 0 = search never activated (a real outcome)
    n_sources   INTEGER NOT NULL DEFAULT 0,
    answer_len  INTEGER
);

CREATE TABLE citations (
    call_id            TEXT NOT NULL REFERENCES calls(call_id),
    position           INTEGER NOT NULL,   -- order of appearance
    raw_url            TEXT NOT NULL,
    final_url          TEXT,
    registrable_domain TEXT,
    char_offset        INTEGER,
    PRIMARY KEY (call_id, position)
);

CREATE TABLE domains (
    registrable_domain TEXT PRIMARY KEY,
    treated            INTEGER NOT NULL DEFAULT 0,  -- forward-compat: controlled site
    notes              TEXT
);

-- The per-query candidate risk set (see 9.2). Frozen after burn-in; the
-- analysis dataset is the cross product of activated calls with the rows
-- of this table for the call's query_id.
CREATE TABLE query_domains (
    query_id           TEXT NOT NULL,
    registrable_domain TEXT NOT NULL REFERENCES domains(registrable_domain),
    k_threshold        INTEGER NOT NULL,   -- min burn-in citation count for membership
    added_in_sweep     TEXT NOT NULL REFERENCES sweeps(sweep_id),
    frozen_at          TEXT NOT NULL,
    PRIMARY KEY (query_id, registrable_domain, k_threshold)
);

-- Domains first cited after the freeze. Never silently enter the model;
-- reported as the novel-domain arrival diagnostic (9.2.1) and consumed
-- directly by Extension B.
CREATE TABLE post_freeze_domains (
    query_id           TEXT NOT NULL,
    registrable_domain TEXT NOT NULL,
    first_seen_sweep   TEXT NOT NULL REFERENCES sweeps(sweep_id),
    PRIMARY KEY (query_id, registrable_domain)
);
```

Three properties this schema enforces:

**Null outcomes are retained.** The survey states plainly: "Outputs without search, without citations, or with errors are outcomes, not data to be discarded." A call where search never activated is a row with `activated = 0`, not an absent row. This is what makes the denominator correct.

**Model snapshot is captured per call**, not per config. When a vendor silently rolls a model, the data shows it instead of hiding it inside an unexplained variance jump.

**URL provenance is three columns.** The survey warns that parameters, redirects, AMP versions, translated pages, and aggregators artificially fragment a domain. Raw URL, resolved final URL, and registrable domain are all retained, and the canonicalization rules ship as documented, tested code.

The `UNIQUE` constraint on `calls` is what makes sweeps idempotent: re-running a `sweep_id` resumes rather than duplicating.

## 7. Panel configuration

```yaml
panel_id: noise-floor-v1
budget_cap_usd: 250.00        # cumulative across the whole panel, not per sweep
burn_in_days: 3               # sweeps used only to construct the risk set (9.2.1)
risk_set_k: [1, 2, 5]         # membership thresholds; all reported side by side

engines:
  - id: perplexity
    model: sonar-pro
  - id: openai
    model: gpt-5.1
    tool: web_search
  - id: gemini
    model: gemini-3-pro
    tool: google_search_grounding
  - id: anthropic
    model: claude-sonnet-5
    tool: web_search

repetitions: 8
windows: ["09:00", "15:00", "21:00"]
locale: en-US

queries:
  - id: q001
    intent: informational
    domain: personal-finance
    paraphrases:
      - "what is a roth ira contribution limit"
      - "How much can I put into a Roth IRA this year?"
      - "roth ira annual limit 2026"
```

Queries must span multiple intents and content domains. Paraphrases must include at least one question-phrased variant, because AI Overview activation is 13.7% overall but 64.7% for question-phrased queries (Xu et al. 2026) — query form is a first-order factor, not a nuisance.

## 8. Scheduling

Three windows per day. Within a window, all 8 repetitions of a given (query, paraphrase, engine) are issued closely together.

```
window "morning"   09:00 -> sweep_id = noise-floor-v1_2026-08-11_morning
window "afternoon" 15:00 -> ...
window "evening"   21:00 -> ...
```

This design separates three sources of variation that a nightly cron confounds:

- **Run-to-run** (within window, same inputs) — generation stochasticity
- **Within-day across windows** — time-of-day and index-refresh effects
- **Across days** — drift

Local execution via `launchd` or `cron` invoking `collect.py`; no server required.

## 9. Analysis

### 9.1 Decomposition

Citation probability is never reported as a bare share. It is always decomposed:

```
Pr(s cited) = Pr(A = 1) × Pr(s ∈ R | A = 1) × Pr(s cited | s ∈ R, A = 1)
```

Because the panel is observational and we do not see the retrieval set `R`, the middle term is not directly identified. The prototype therefore reports the two terms it *can* identify — activation rate and citation-given-activation — and states the third as unobserved rather than silently folding it in. This is precisely the error the survey attributes to commercial dashboards.

### 9.2 Variance components

#### 9.2.1 The candidate risk set

The outcome is binary — "domain *d* cited in call *c*" — so the analysis dataset is not the `citations` table. It is the cross product of calls with a set of **candidate domains at risk of citation**, including the zero rows for candidates that were not cited. That candidate set must be defined explicitly, because the `citations` table contains only the ones, and any set derived from observed citations is endogenous: a domain enters the universe *because* it was cited, which biases Pr(C = 1) upward and truncates the estimated domain-level variance at the low end.

**Rule.** The risk set is per-query, frozen after a burn-in window:

1. Designate the first `burn_in_days` of sweeps (a `panel.yaml` field) as burn-in.
2. For query *q*, the risk set at threshold *K* is every registrable domain cited in at least *K* burn-in calls for *q*, across all engines, paraphrases, and windows.
3. Write the resulting membership to `query_domains` with `frozen_at` set, and hash it into the analysis manifest. It does not change thereafter.
4. Domains first cited for *q* after the freeze are recorded in `post_freeze_domains` and **excluded from the model**, so the denominator cannot grow with sample size.

Burn-in sweeps are used only to construct the risk set and are excluded from the variance and MDE estimates, so membership and outcome are never estimated from the same data.

**Sensitivity is a deliverable, not a footnote.** The whole point of the prototype is to characterize measurement behavior, so the analysis is run at *K* ∈ {1, 2, 5} and all three variance decompositions are reported side by side. If the noise floor is stable across *K*, the choice is immaterial and can be fixed later; if it is not, that instability is itself a primary finding and must be reported as one.

**Novel-domain arrival is a reported diagnostic.** The rate at which `post_freeze_domains` accumulates is plotted per query over the panel. Excluding these domains from the model does not discard them: this arrival curve is the input Extension B needs, so the freeze and the drift study are complementary rather than in tension.

#### 9.2.2 Model

Fit on **activated calls only** (`activated = 1`). Activation is modeled separately as a call-level binary outcome, consistent with §9.1's refusal to collapse activation into citation. Mixing them would load activation-rate differences into the engine and query random effects and misattribute the noise floor.

```
logit Pr(C_{cdi} = 1) = β₀
                      + b_query + b_paraphrase + b_engine
                      + b_date + b_window + b_domain
                      + b_call
```

`b_call` is an explicit call-level random intercept capturing run-to-run stochasticity. It is named rather than left as "residual" because in a Bernoulli GLMM the residual variance is fixed at π²/3 by the link function and cannot absorb it — without `b_call`, run-to-run variance is silently misallocated to the date and window terms.

Report, at each *K*:

- The variance attributable to each grouping factor
- The **base citation rate** over the risk set, since every effect size in §9.4 is expressed relative to it
- The risk-set size per query

This is the noise floor expressed per factor, and it says directly which factor is worth spending budget on.

### 9.3 Stability

"Source set" is ambiguous across 3 paraphrases × 8 repetitions × 3 windows, and the aggregation choice changes the number substantially — a union over many calls trends toward 1.0 and overstates stability. Two distinct quantities are therefore reported, never conflated:

**Run-to-run Jaccard (the noise floor).** The unit is the set of registrable domains cited in a *single call*. Compute mean pairwise Jaccard over the 8 repetitions within a fixed (query, paraphrase, engine, window). This is the quantity the MDE depends on.

**Day-union Jaccard (benchmark-comparable).** The unit is the union of registrable domains cited across all calls for a (query, engine) on a given day. Compute between consecutive days. This matches the construction used by Schulte et al. and is the only one comparable to their published 0.34–0.42.

Both are also computed between windows within a day, which separates time-of-day effects from day-scale drift.

The comparison against the consumer-surface benchmark is reported with the §3.3 limitation attached: prior work measured consumer surfaces, this measures API surfaces, and a divergence may reflect the surface difference rather than a difference in underlying stability. Quantifying which is Extension B, not a claim this prototype can make.

### 9.4 Minimum detectable effect

Simulation-based, not closed-form. Closed-form power for a binary outcome under crossed random effects is unreliable. Procedure:

1. Fit the null model (§9.2.2) to collected data
2. Parametric bootstrap: simulate datasets with an injected treatment effect of size δ, at a given panel size
3. Refit and test; repeat to estimate power at δ
4. Sweep δ and panel size to produce an MDE surface

δ is expressed in percentage points **relative to the base citation rate over the risk set** reported in §9.2.2, and is therefore only interpretable alongside it. The MDE surface is produced at each *K* ∈ {1, 2, 5}; if the surfaces diverge materially, the risk-set definition is reported as a limiting factor on the go/no-go rather than silently resolved by picking one.

Output: "to detect a δ percentage-point citation lift, against a base rate of *p*, at 80% power, you need N queries × K reps × D days, costing $X."

## 10. Error handling

For a measurement instrument, error handling *is* part of the measurement.

- **Errors are data.** HTTP status, error class, retry count, and total backoff are recorded and retained on the `calls` row.
- **Three non-citation outcomes are never collapsed:**
  - `transport_error` — network or 5xx failure; excluded from denominators, retained in DB
  - `engine_refusal` — the model declined to answer; excluded from denominators, retained
  - `activated = 0` — search never activated; **included** in denominators as a legitimate outcome
- **Backoff is logged**, since request timing can shift results.
- **Partial sweeps are marked `incomplete`** and excluded from analysis by default. Analysis never silently treats a partial sweep as whole. A sweep is `complete` only when every planned call has a terminal outcome.
- **Budget cap aborts.** On reaching `budget_cap_usd`, the collector stops, marks the sweep `aborted_budget`, and exits non-zero.
- **Idempotent resume.** Re-running a `sweep_id` issues only the calls missing from the `UNIQUE` key set.

## 11. Testing

- **Fixture-based adapter tests.** Real captured API responses replayed offline. No live API calls in the test suite, so the suite is free and deterministic.
- **URL canonicalization table tests.** Table-driven cases covering AMP variants, tracking parameters, redirect chains, trailing slashes, `www`, IDN, and aggregator wrappers. This is where silent data corruption lives, and a canonicalization bug would invalidate every downstream number.
- **Denominator tests.** Synthetic sweeps with known activation and citation rates; assert the decomposition returns the exact known values, including when null outcomes are present.
- **Risk-set expansion tests.** On a synthetic panel with a known domain universe: assert the (call × candidate-domain) expansion produces the correct row count; assert zero rows exist for eligible-but-uncited domains; assert `activated = 0` calls are excluded from the §9.2.2 model but retained for the activation model; assert post-freeze domains land in `post_freeze_domains` and never in the analysis dataset; assert the *K* ∈ {1, 2, 5} sets are properly nested.
- **Power-calculation tests.** Simulated data with an injected known effect; assert the MDE procedure recovers it within tolerance.
- **Idempotency test.** Run a sweep, kill it mid-way, resume; assert no duplicate calls and no lost rows.
- **Budget-cap test.** Assert the collector aborts at the cap and marks the sweep correctly.
- **`--dry-run` mode.** Exercises the full pipeline against a mock engine at $0 cost. Doubles as the demo path.

## 12. Success criteria

The prototype is done when the following four outputs exist and are reproducible from retained raw data:

1. **Variance decomposition** — citation variability attributed to call (run-to-run), paraphrase, window, day, engine, and domain, reported at each *K* ∈ {1, 2, 5}, together with the base citation rate and risk-set size that make the numbers interpretable
2. **Stability estimate** — run-to-run Jaccard and day-union Jaccard for API surfaces, window-over-window and day-over-day, with the day-union figure compared against the consumer-surface benchmark under the §3.3 limitation
3. **Cost model** — budget in, panel size and MDE out
4. **Go/no-go** — a defensible statement of whether an effect of realistic size is detectable at an affordable budget

Note that outputs 1–3 are informative regardless of whether the answer to 4 is yes. A well-characterized negative result ("citation cannot be measured precisely enough at this budget") is a genuine contribution to a field the survey describes as having Low confidence in exactly this question.

## 13. Extensions (deliberately deferred)

**Extension A — controlled site.** Once a modifiable indexed domain is available, flag it `treated` in `domains`, apply interventions, and analyze with difference-in-differences. The comparison pool is defined as the untreated members of the same frozen per-query risk set (§9.2.1) — not all observed domains — so treated and control units are drawn from the same universe under the same membership rule. The treated domain must satisfy the risk-set rule at the chosen *K* before intervention, or the parallel-trends assumption has no support. Requires the noise floor from this prototype to size the intervention.

**Extension B — drift as an object of study.** Extend the panel to 30–60 days, pin and log model snapshots, and characterize drift regimes and changepoints (survey §13.5). The `post_freeze_domains` arrival curve from §9.2.1 is a direct input: novel-domain arrival is one of the drift signals, so the freeze that keeps the §9.2.2 denominator stable also generates this extension's primary data. Publishable as a descriptive study, since all existing stability work measures consumer surfaces rather than API surfaces.

**Extension C — predictive validity probe.** Re-analyze collected data to test whether observable page features predict which domains get cited. Cheap once the panel data exists; falsifies the scorer premise if features do not predict.

## 14. Risks

| Risk | Mitigation |
|---|---|
| Noise floor exceeds realistic effect sizes; project's premise fails | This is the point of the prototype. A negative result is reported, not worked around. |
| Vendor changes API or pricing mid-panel | `model_snapshot` per call and `pricing.yaml` with `verified_on` make the change visible rather than silent |
| API surfaces prove unrepresentative of consumer surfaces | Stated as a limitation in every output; quantifying the gap becomes Extension B |
| Budget exhausted before enough days collected | Hard cap plus `estimate.py` projection before any spend; panel size is config, not code |
| Canonicalization bug silently merges or splits domains | Dedicated table-driven test suite; raw and final URL both retained so the data can be re-normalized |
| Noise floor turns out to be an artifact of the risk-set definition | Analysis runs at *K* ∈ {1, 2, 5} and reports all three; divergence across *K* is reported as a primary finding rather than resolved by picking one |
| Burn-in consumes budget without producing analyzable data | Burn-in sweeps are ordinary sweeps and remain in the DB; only their use in estimation is restricted, so they still contribute to stability diagnostics and to the cost model |

## 15. References

- Martinez, O. *Optimizing Visibility in Generative Engines: A Critical Survey of Generative Engine Optimization (2023–2026).* arXiv:2607.14035, July 2026. Local copy: `../../arxiv_2607.14035_GEO_critical_survey.pdf`
- Aggarwal, P. et al. *GEO: Generative Engine Optimization.* KDD 2024. arXiv:2311.09735
- Schulte, J. et al. *Don't measure once: Measuring visibility in AI search (GEO).* arXiv:2604.07585, 2026
- Kirsten, E. et al. *Characterizing web search in the age of generative AI.* Findings of ACL 2026
- Kim, S. et al. *SAGEO Arena.* arXiv:2602.12187, 2026
- Xu, H. et al. *Measuring Google AI Overviews.* arXiv:2605.14021, 2026
- Project foundation document: `../../GEO project_Foundation Document.pdf`
