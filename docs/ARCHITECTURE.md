# Architecture: Eye of the System

Almedia track at HackLab × BRISA (21–22 Nov 2026): **"Eye of the System: Fully Agentic Monitoring"**

The system is the eyes of a mock shopping marketplace. It decides what to watch, notices meaningful changes, investigates them on its own by following a **graph of metrics**, and writes up what it found with evidence. Nobody asks it questions or guides it.

Judging: 40% metric graph and automatic extension · 40% autonomous monitoring and investigation · 20% demo.

The earlier Freecash/postback design is kept in [archive/postback-patrol-v0.md](archive/postback-patrol-v0.md).

---

## 1. Overview

```
 ┌──────────────────────┐   hourly rows    ┌────────────────────────────────────────┐
 │ SIMULATOR            │ ───────────────▶ │ WAREHOUSE (DuckDB)                     │
 │ marketplace model    │                  │ fact tables + events + data load log   │
 │ + fault injector     │                  └───────────────┬────────────────────────┘
 │ + innocent changes   │                                  │ SQL
 └──────────────────────┘                                  ▼
                              ┌─────────────────────────────────────────────────────┐
 new metric (name,            │ METRIC GRAPH (networkx + JSON on disk)              │
 description, SQL) ─────────▶ │ nodes = metrics · edges = FORMULA / BREAKDOWN /     │
   │                          │ FUNNEL / POSSIBLE_CAUSE / EVENT                     │
   ▼                          └───────┬───────────────────────────┬─────────────────┘
 ┌──────────────────────┐             │                           │
 │ EXTENDER             │─ validated ─┘                           │
 │ profile → propose →  │   edges                                 ▼
 │ test → add           │                     ┌───────────────────────────────────┐
 └──────────────────────┘                     │ MONITOR (every simulated hour)    │
                                              │ 1. pick what to watch (priority)  │
                                              │ 2. expected value (seasonal)      │
                                              │ 3. data complete? planned event?  │
                                              │ 4. unusual → open or update case  │
                                              └───────────────┬───────────────────┘
                                                              ▼
                                              ┌───────────────────────────────────┐
                                              │ INVESTIGATOR (best-first search)  │
                                              │ break down → narrow by group →    │
                                              │ test causes → stop rule           │
                                              │ code does the maths; Claude picks │
                                              │ the next step and writes the story│
                                              └───────────────┬───────────────────┘
                                                              ▼
                                              ┌───────────────────────────────────┐
                                              │ INCIDENTS + REPORTER              │
                                              │ fingerprint, dedupe, open/ongoing/│
                                              │ resolved, € impact, trail         │
                                              └───────────────┬───────────────────┘
                                                              ▼
                                              ┌───────────────────────────────────┐
                                              │ UI (Streamlit)                    │
                                              │ graph + tree · live feed ·        │
                                              │ investigation trail · report      │
                                              └───────────────────────────────────┘
```

**The main idea: the deterministic code does all the maths, and Claude does what needs understanding of language.** The maths covers splitting a change into parts, narrowing it to a group, checking formulas and deciding when to stop. Claude chooses between candidate next steps, reads release notes, places new metrics and writes the story. Every step goes into the trail, so every decision can be explained.

---

## 2. Mock marketplace (simulator)

**Scope:** 6 weeks of hourly data, simulated at about 1 hour per 2 seconds in the demo.

| Dimension | Values |
|---|---|
| country | DE, FR, UK |
| platform | ios, android, web |
| category | electronics, fashion, home, beauty, toys |
| advertiser (ads only) | 20 advertisers |

### Generative model (per hour × country × platform × category)
```
sessions        = base[country,platform] × hour_of_day[h] × day_of_week[d] × noise + ad_clicks
searches        = sessions × search_rate
result_clicks   = searches × search_ctr              ← search relevance drives this
feed_impr       = sessions × feed_rate
feed_clicks     = feed_impr × feed_ctr
product_views   = result_clicks + feed_clicks
add_to_cart     = product_views × cart_rate × in_stock_rate   ← inventory drives this
checkouts       = add_to_cart × checkout_rate
orders          = checkouts × payment_success_rate   ← payment provider drives this
gmv             = orders × aov
discounts       = gmv × discount_share               (platform-funded)
fees            = gmv × take_rate
ad_revenue      = ad_spend                           (advertisers pay us)
contribution    = fees + ad_revenue − discounts − payment_costs
```
Ads (per advertiser × hour): `budget`, `impressions`, `ad_clicks = impressions × ad_ctr`, `ad_spend = ad_clicks × cpc`, `pacing = cumulative_spend / (budget × hour/24)`.
Inventory (per category): `in_stock_rate`, `purchase_orders`, `supplier_deliveries`.

### Tables (DuckDB)
| Table | Grain | Columns |
|---|---|---|
| `fact_funnel` | hour × country × platform × category | sessions … orders, gmv, discounts, fees |
| `fact_ads` | hour × advertiser × country | budget, impressions, ad_clicks, ad_spend |
| `fact_inventory` | hour × category | in_stock_rate, purchase_orders, deliveries |
| `fact_payments` | hour × country × platform | attempts, successes, provider_errors |
| `fact_refunds` | hour × country × platform × category | refunds (kept back for the live "add a metric" demo) |
| `events` | ts | type (release / campaign / holiday / incident), system, scope (JSON), description |
| `load_log` | hour × table | rows_expected, rows_loaded, loaded_at |

`load_log` is what makes "is this data complete?" answerable.

### Scenarios (injected live from the UI)
| # | Kind | What happens | Correct answer |
|---|---|---|---|
| S1 | Fault | `payment_success_rate` drops 0.97 → 0.70 for android × DE. A release event "payments-sdk 4.2 android" goes in 10 minutes earlier. | Orders ↓ → checkout→order step → android × DE → payment provider errors ↑ → release. **Supported.** |
| S2 | Fault | Search release "ranker v7" for DE: `search_ctr` ↓ 35% for DE only. | Orders ↓ in DE → result_clicks ↓ → search_ctr DE → release timing. **Supported.** |
| S3 | Innocent | Planned "FR Black Week" campaign (in `events`): sessions +40% in FR, discounts ↑. | Expected because of the event. **No alert**, logged as "explained by planned event". |
| S4 | Missing data | UK partitions arrive at 60%, then late. | "Data delay", not an incident. Recheck later and it resolves itself. |
| S5 | Fault (stretch) | Pacing bug: one advertiser spends 100% of budget by 10:00. | Ad spend ↑ → pacing ↑ while impressions are normal → it's the pacing controller, not more traffic. |
| S6 | Cross-system (stretch) | Ads ↑ traffic, feed pushes discounted items, orders ↑ but contribution ↓. | Split contribution into parts: discounts + ad costs explain the drop. |
| S7 | Competing (stretch) | Stockout in electronics and a feed model release at the same hour. | Both tested. Stock explains 80%, the model 20%, so one is supported and one is a hypothesis. |

**Must have for the demo:** S1, S3, S4 and a live new metric. **Nice to have:** S2. **Stretch:** S5–S7.

---

## 3. Metric graph (40% of the score)

### Node = metric
```json
{
  "id": "orders",
  "description": "Completed and paid orders",
  "sql": "SELECT hour, country, platform, category, SUM(orders) AS value FROM fact_funnel GROUP BY ALL",
  "grain": "hour",
  "dimensions": ["country", "platform", "category"],
  "unit": "count",
  "kind": "count | ratio | currency",
  "importance": 0.0,              // calculated, see §5
  "monitor": true,
  "source": "seed | extender"
}
```

### Edge types (each supports a different kind of reasoning)
| Type | Meaning | Example | Validation | How it's used in reasoning |
|---|---|---|---|---|
| `FORMULA_MUL` | parent = Π children (or a ratio) | orders = checkouts × payment_success_rate | holds on history to within 0.5% error | Exact split of a change by log change |
| `FORMULA_ADD` | parent = Σ ± children | contribution = fees + ad_revenue − discounts − payment_costs | to within 0.5% | Exact split by difference |
| `BREAKDOWN` | parent = Σ over a dimension | orders → orders by country | the groups add up to the parent | Narrowing to a group |
| `FUNNEL` | step i+1 = step i × rate | searches → result_clicks → … → orders | it's a chain of formula edges | Finds the step where people drop out |
| `POSSIBLE_CAUSE` | A may drive B, not exactly | in_stock_rate → add_to_cart; ad_spend → sessions | correlation over time with a lag, confidence 0–1 | Hypotheses only, **never "supported" on its own** |
| `EVENT_LINK` | event can affect metric | release `payments-sdk` → payment_success_rate | matches the system/scope tags | Explains changes; recognises planned changes |

Each edge stores `{type, formula?, dimension?, lag?, confidence, status: validated | ambiguous | rejected, evidence: {...}, source: seed | extender}`.

### The tree for people
- The root is **contribution** (the top-level business metric).
- Children follow `FORMULA` edges, so the tree reads contribution → fees → gmv → orders → checkouts → …
- Each node can expand into its `BREAKDOWN` groups.
- `POSSIBLE_CAUSE` and `EVENT` edges appear only in the graph view, as dashed lines.

### Seed graph
About 35 metrics are written by hand in `graph/seed.yaml`. `refund_rate`, `avg_delivery_days` and one more are kept back so they can be added live in the demo.

---

## 4. Automatic extension: adding a new metric

**Input:** `{name, description, sql}` only. The contributor doesn't say where it fits.

```
1. PROFILE   run the SQL → grain, dimension columns, value range, unit guess, kind (count/ratio/€)
             reject if the SQL fails or returns no rows (error back to the contributor)

2. PROPOSE   a) numeric search over the existing metrics (same grain, same dimensions in common):
                new ≈ A/B, A×B, A+B, A−B, Σ_dim A          (pairs limited to the top 40 by name/description similarity)
             b) Claude reads name + description + SQL + list of nearby metrics
                → proposes edges with a type and a reason (structured JSON)
             c) correlation over time with a lag (0–6 h) against every monitored metric → POSSIBLE_CAUSE candidates

3. VALIDATE  FORMULA candidates: error over the last 14 days < 0.5% AND it holds within each group of every dimension in common
             BREAKDOWN: the groups add up to the total, within 0.1%
             POSSIBLE_CAUSE: |corr| > 0.5 at the best lag, AND Claude's proposed direction is plausible
             Decide:
               exactly one formula fits        → validated
               more than one fits              → ambiguous (keep all of them, show "needs review", use the simplest one)
               Claude proposed it, the numbers fail → rejected (kept with its evidence)

4. INTEGRATE add the node and edges, compute importance, build the baseline from history, set monitor=true
             write an "extension report": what was found, what was rejected, and why
```

**Why it's robust:** Claude proposes and the numbers decide. A formula never becomes "validated" just because the LLM said so.

**Live demo metric:** `refund_rate` with the description "share of orders refunded" and SQL over `fact_refunds`.
- **Expected result:** `refunds = orders × refund_rate` is validated, plus breakdowns by country and platform.
- **Possible cause:** `in_stock_rate` is flagged as a weak POSSIBLE_CAUSE, or rejected.

---

## 5. Monitoring: what to watch, and when it's worth an alert

### What gets checked each cycle (attention budget: about 60 series per hour)
```
importance(m)  = share of the root that m is responsible for, following FORMULA edges
                 (e.g. orders ≈ 0.9, beauty-only metric ≈ 0.1)
surprise(m,g)  = |robust z| from the last check
staleness(m,g) = hours since last checked / maximum interval allowed

priority = importance × (1 + surprise) × staleness
```
- **Always checked:** the root and the first level of the graph.
- **Breakdowns:** only checked when the parent's z-score is above 2, or when their staleness is too high (rotating spot checks).
- **This is the "decides what to watch" part.** It's shown live in the UI as a "watching now" list.

### Expected behaviour
- **Baseline:** the median of the same hour and same weekday over the last 4 weeks, scaled by the overall level of the last 24 hours.
- **Spread:** MAD (median absolute deviation) over the same window. The score is `z = (actual − expected) / (1.4826 × MAD)`.
- **Ratios** (rates) use counts that are large enough, so small groups don't trigger alerts. They're only scored when the denominator is above 200.

### Filters before raising an alert (in order)
1. **Data complete?** If `load_log.rows_loaded / rows_expected < 0.95` for any table the metric depends on, it becomes a `DATA_DELAY` notice, not an incident. It's checked again next hour.
2. **Planned event?** If an `events` row of type campaign/holiday has a scope that overlaps and an expected direction that matches, it becomes `EXPLAINED_BY_EVENT`. It's logged but not raised.
3. **Lasted long enough?** It needs |z| > 3 for one hour, or |z| > 2 for 2 hours in a row.
4. **Already known?** If it matches an open incident's fingerprint, the system updates that incident and doesn't create a new one.

---

## 6. Investigation algorithm (40% of the score)

The investigation is a **best-first search over the graph**. Each step is one query. The limit is 15 queries.

```
state: frontier (priority queue of (metric, group)), evidence[], hypotheses[], explained_share

start: frontier ← (anomalous metric, all groups)

loop:
  pick the best (metric, group) from frontier
  1. DATA CHECK   are the tables behind it complete? if not → stop, mark as DATA_DELAY

  2. BREAK DOWN   if the metric has FORMULA children:
                    MUL: share_i = Δlog(child_i) / Δlog(parent)
                    ADD: share_i = Δchild_i / Δparent
                    add each child to frontier, scored by share_i × |z_child|

  3. NARROW       for each BREAKDOWN dimension:
                    for each group g: explanatory power EP(g) = Δ_g / Δ_total
                                      surprise S(g) = JS divergence of g's share, actual vs expected
                    pick the smallest set of groups with ΣEP ≥ 0.8 and a high S (Adtributor-style)
                    try a 2-dimension combination (country × platform) if 1 dimension doesn't explain enough

  4. TEST CAUSES  at a leaf (no FORMULA children left, or children explain < 20%):
                    for each POSSIBLE_CAUSE parent and each EVENT_LINK within ±3 h:
                       timing:  does the cause's change start at or before the effect's? (change-point)
                       scope:   does the cause's group match the narrowed group?
                       size:    does the size of the cause's change predict the effect? (regression on the group)
                    each hypothesis gets {for: [...], against: [...], score}

  5. CLAUDE STEP  given the trail so far + top 5 frontier candidates + open hypotheses:
                    pick next (or STOP), reason in one line
                    may read event descriptions / release notes to propose a cause the graph doesn't have yet

  stop when:  a hypothesis is SUPPORTED and explained_share ≥ 0.8
          or  15 queries used          → report with "partially explained"
          or  frontier empty           → report "unexplained change"
```

### Labels on the result
- **SUPPORTED:** the breakdown into parts is exact, the timing matches, the group matches, and it explains at least 60% of the change.
- **HYPOTHESIS:** some evidence matches, but not all.
- **RULED OUT:** tested, and the evidence is against it (shown in the report).

### Symptom or first cause?
- A metric whose change is ≥ 90% explained by its children is a **symptom**.
- The **first cause** is the earliest change point along the explained path. Start times are compared by onset hour.

### Claude's role in the investigation
- Claude never does the arithmetic.
- The candidate scores are already calculated when Claude sees them, so its choice is between options the maths has ranked. It's still allowed to override the ranking, with a reason.
- That makes the trail easy to read: *"Chose payment_success_rate[android×DE] (share 0.91, z −11) over search_ctr (share 0.04)."*

---

## 7. Incidents and reporting

**Fingerprint** = (first-cause metric, narrowed group, direction).

| Situation | What happens |
|---|---|
| New fingerprint | Open an incident and send an alert |
| Same fingerprint, still unusual | Status `ongoing`. Update the numbers and the duration; no new alert |
| Back within normal range for 3 hours | Status `resolved`. Send a resolution note with the total impact |
| Different metric but same cause | Linked to the existing incident as a symptom |

**Report sections:**
1. **Headline:** *"Orders −18% in DE since 14:00 — payment failures on Android after payments-sdk 4.2"*
2. **Business impact:**
   - Lost GMV and contribution in €, so far and per hour
   - Orders affected
3. **Evidence chain:** each step with its numbers:
   ```
   orders ↓ → checkout→order step (91%) → android×DE (88%) → provider_errors ×6 → release at 13:50
   ```
4. **Competing explanations:** the ones that were tested and why each was ruled out
5. **Uncertain:** what's still unclear (*"iOS DE slightly ↓ too, could be spill-over, n is small"*)
6. **Trail:** every query, plus Claude's one-line reason for each step
7. **Next check:** when it's looked at again

---

## 8. Using Claude

- **Model:** `claude-opus-5-5`, called through the **Anthropic Python SDK directly**. LangChain isn't needed.
- **Settings:**
  - `thinking: {type: "adaptive"}`
  - No `temperature` parameter
  - Structured JSON output via `output_config.format`, or tools with `strict: true` and `tool_choice: auto`. Forcing a tool call returns an error on Opus 5.5.
- **Three calls:**
  1. `extender.propose(metric, neighbours)` → candidate edges, as JSON
  2. `investigator.next_step(trail, candidates, hypotheses)` → `{choice, reason}` or `STOP`
  3. `reporter.write(incident)` → the report text, written only from numbers the code supplies
- **Cost:**
  - About 2–4k input tokens per step and about 15 steps per investigation, so roughly **$0.25–0.40 per investigation**
  - Prompt caching on the system prompt and the graph summary
  - Haiku 5.5 or Sonnet 5.5 while developing, Opus for the demo and the evals
- **Fallback:** if the API is down, `next_step` uses the highest-scored candidate. The system still runs end to end, which is a good thing to say in the pitch.

---

## 9. UI (Streamlit)

| Panel | Shows |
|---|---|
| **Graph** | The metric graph (pyvis / streamlit-agraph). Edge colour by type, dashed for POSSIBLE_CAUSE, orange for ambiguous. Nodes being investigated pulse. |
| **Tree** | Browsable tree from contribution down: expand a node to see its groups and sparklines |
| **Live feed** | Simulated clock, "watching now" list, notices (data delay, explained by event), incidents |
| **Investigation** | Step-by-step trail: query → numbers → decision. The share of the change explained fills up as it goes |
| **Report** | The final incident report |
| **Control** | Buttons to trigger S1–S7, a speed slider, and an "Add metric" form (name / description / SQL) |

---

## 10. Evaluation (shown in the demo)

`eval/run.py` runs each scenario over 3 random seeds with the simulated clock sped up, and compares against the known correct answer.

| Metric | Meaning |
|---|---|
| Detection | Fault found within N simulated hours |
| Cause correct | First-cause metric and group match the correct answer |
| False alerts | Alerts during innocent changes (S3) or missing data (S4) |
| Queries used | How efficient the investigation is |
| Extension accuracy | Edges for the 3 held-back metrics vs the correct edges (precision/recall) |
| Time to cause | Simulated minutes from the fault to the report |

A table of these numbers goes on one slide.

---

## 11. Repo layout

```
sim/         generator.py (marketplace model), scenarios.py (S1–S7), clock.py
warehouse/   db.py (DuckDB setup), views.sql
graph/       seed.yaml, model.py (networkx + JSON), tree.py
extender/    profile.py, propose.py (formula search + Claude), validate.py
monitor/     baseline.py, attention.py (priority), filters.py (data complete, events)
investigate/ decompose.py, drilldown.py (Adtributor), causes.py, search.py (loop + stop rule)
incidents/   store.py (fingerprint, lifecycle), report.py
llm/         client.py (Anthropic SDK), prompts/
app/         streamlit_app.py
eval/        run.py, truth.yaml
```

**Libraries:** python 3.12, uv, duckdb, pandas, numpy, networkx, ruptures (change points; optional), anthropic, streamlit, streamlit-agraph, pyyaml. All free.

---

## 12. Build plan (2 people)

Hacking runs from Sat 11:00 to the **Sun 16:30 submission deadline**. Submit a 3-minute video, a public GitHub repo and slides. Pitches are at 18:00.

| When | Person A (you) | Person B (friend) |
|---|---|---|
| Sat 11:00–14:00 | Simulator + DuckDB tables + S1/S3/S4 | Seed graph YAML + graph model + tree |
| Sat 14:00–19:00 | Baseline + attention + filters (monitor loop) | Breaking a change into parts + narrowing by group |
| Sat night | Extender (formula search, validation, Claude propose) | Testing causes + search loop + Claude next_step |
| Sun 09:00–12:30 | Incidents + report + eval | Streamlit UI (graph, trail, controls) |
| Sun 12:30–14:30 | **Code freeze 14:30.** Fix bugs, run eval | Slides (5 min) |
| Sun 14:30–16:30 | Record the 3-minute video (one of you on camera + live demo) | README with how to run it, make the repo **public**, submit on Tally |

**Point where the whole thing first works end to end (Sat ~19:00):** S1 runs through detection → investigation → a text report, shown in the terminal and without the UI.

## 13. Demo script (5 minutes)

1. **(30 s)** Show the graph and tree: *"This is the system's picture of how the business works."*
2. **(60 s)** Start the clock. The "watching now" list rotates. Trigger **S3** (FR Black Week): orders rise, and the notice says "explained by planned event". No alert.
3. **(90 s)** Trigger **S1**. The alert fires, and the graph lights up along the path the investigation takes. The trail fills in, and the share of the change explained reaches 0.88. Then the report: € impact, the evidence chain, what was ruled out, what's uncertain.
4. **(60 s)** **Add `refund_rate` live**:
   - The extender finds the formula and validates it
   - It shows one ambiguous candidate and one rejected
   - The new node appears in the graph and gets monitored
5. **(30 s)** Trigger **S4** (UK data late). It shows "data delay, not an incident", and the notice clears when the data arrives.
6. **(30 s)** Eval slide + *"code does the maths, Claude makes the judgement calls, every step can be explained."*

---

## 14. Services (hackathon credits)
| Service | Use? | For |
|---|---|---|
| **Anthropic Claude API credits** | ✅ Yes | All LLM calls. Put the credits org's key in `ANTHROPIC_API_KEY` |
| Prior Labs TabPFN (free) | Optional | Alternative way to predict expected values (TabPFN regression on hour, weekday and lag features), compared against the median baseline in the eval |
| Cognition Devin ($200) | Optional | Hand it isolated, well-specified tasks (e.g. the Streamlit UI skeleton) so it works alongside us |
| Linkup search | ❌ No | We use mock data, so there's no web lookup to do |
| NVIDIA Brev GPU | ❌ No | No model training |
| Nebius Token Factory | ❌ No | Needs a bank card, and Claude covers everything |

## 15. Open questions for Almedia
1. Can we speed up simulated time in the demo, or does it need to run in real time?
2. Is flagging an ambiguous edge for a human OK, or must the system decide everything itself?
3. Will you give us a metric to add during the demo, or do we choose one?
4. Is 30–50 metrics with 3 dimensions the right size?
5. What false alarm do you see most often in real life? We'll simulate that one.
