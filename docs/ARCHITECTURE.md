# Almedia track: "Offer Health Agent"
HackLab × BRISA, 21–22 Nov 2026 · Track brief: *Agentic Monitoring for an Ads Marketplace App: build AI agents that keep a real marketplace app running smoothly.*

---

## 1. How Almedia works

**Business:** Freecash (plus Prizeplay and zave.it cashback). Users earn money by installing and playing apps or games, doing surveys and signing up for trials. Advertisers (game studios, apps, fintech) pay Almedia for each install or in-app action. Almedia keeps the difference between what the advertiser pays and what the user gets.
- ~$1B revenue run rate, profitable, no outside funding. 9M+ monthly earning users, 100M+ events a day, 300TB of behavioural data. $500M+ paid out to users.
- In 2026 Freecash was removed from the App Store and Google Play over data-harvesting allegations. It came back (Google Play in May, iOS in June). Trust and compliance are sensitive right now.

**How a reward gets credited** (from Freecash's own support docs):
```
user clicks offer ──► installs / plays ──► game's SDK ──► MMP (AppsFlyer/Adjust…) ──► POSTBACK ──► Freecash credits reward
```
If any step fails, Freecash never gets the postback and the user isn't paid.

**Engineering stack (from job postings):** TypeScript/NestJS microservices. MySQL, MongoDB, Redis, BigQuery. Pub/Sub and RabbitMQ. Docker/Kubernetes, GitLab CI, Datadog. Python on the data/AI side (LangGraph, RAG, evals).

## 2. Pain points they've admitted in public
From Freecash's "Answer to recent allegations" post and its tracking help page:
1. **Tracking failures.** "A small share of offers fails to track". They say they have "historically been unable to stop this". Breaks can be **limited to one OS, country or task** within an offer.
2. **They notice problems late.** They detect them from **spikes in support tickets** and an "offer health" metric (the share of users who open a ticket for an offer). Tickets come in **hours after** the break.
3. **Support "has not kept pace with our growth".** Agents **manually review** missing-reward tickets. Account managers check the MMP and escalate to advertisers, which is slow.
4. **Fraud and restricted accounts.** Reviews need to be "faster and more accurate". They hire several fraud analysts and investigators, so it's still largely manual.
5. **Margin and config errors:** reward set wrong, payout provider down. These follow from running thousands of offers.

**Our pitch:** *Find a broken offer from the data within minutes, not from tickets hours later. Tell a tracking break apart from fraud. Act safely. Do the support and advertiser paperwork automatically.*

## 3. What the system does
```
                 ┌────────────────────────── DATA PLANE ───────────────────────────┐
 Simulator  ───► │ clicks · postbacks · events · payouts · withdrawals · tickets · │
 (or their feed) │ changes (deploys/config) · service_health      [DuckDB ≈ BigQuery]│
 + Fault injector└──────────────┬──────────────────────────────────────────────────┘
                                │ every 60 s
                       ┌────────▼─────────┐   cheap SQL, no LLM
                       │  1. DETECTORS    │   z-scores per offer × OS × country
                       └────────┬─────────┘
                                │ alert (only anomalies reach the LLM)
                       ┌────────▼─────────────────────────────┐
                       │  2. INVESTIGATOR AGENT (Claude)      │
                       │  hypothesise → query → check → narrow│
                       │  read-only tools                     │
                       └────────┬─────────────────────────────┘
                                │ submit_diagnosis (strict JSON)
                       ┌────────▼─────────┐
                       │  3. POLICY GATE  │  auto / needs-approval / never
                       └───┬──────────┬───┘
                 auto      │          │ approve (Slack-style button)
                  ┌────────▼───┐  ┌───▼──────────────┐
                  │ 4. ACTIONS │  │ human on-call    │
                  └────────┬───┘  └───┬──────────────┘
                           └────┬─────┘
                       ┌────────▼──────────────────────────────┐
                       │ 5. OUTPUTS: incident report ·          │
                       │ advertiser escalation · support macro │
                       │ · auto-credit list · audit log        │
                       └───────────────────────────────────────┘
                       6. EVAL HARNESS replays scripted scenarios with
                          known answers → scoreboard
```

### 3.1 Data model (DuckDB, BigQuery-style SQL)
| Table | Key columns |
|---|---|
| `offers` | offer_id, advertiser, mmp, os, countries, reward_eur (to user), payout_eur (from advertiser), events[] |
| `clicks` | ts, user_id, offer_id, os, country, device_id, ip_asn |
| `postbacks` | ts, offer_id, user_id, type (install/event), event_name, mmp, lag_s, mapped (bool) |
| `rewards` | ts, user_id, offer_id, amount_eur, status |
| `withdrawals` | ts, user_id, method (PayPal/crypto/gift card), amount, status, provider_code |
| `tickets` | ts, user_id, offer_id, category, text |
| `changes` | ts, kind (deploy/offer_config/sdk_update), target, diff |
| `service_health` | ts, service, error_rate, p95_ms |
| `users` | user_id, created_at, device_id, ip_asn, country, risk_flags |

### 3.2 Detectors (plain Python + SQL, no LLM)
They run every minute for each `offer × os × country`, using a rolling 30 minutes compared with the same hour over the last 7 days:
| Signal | Fires when | Usually means |
|---|---|---|
| install rate = installs / clicks | z < −4 (≥50 clicks) | tracking break |
| install rate | z > +4, or time-to-complete far too fast | fraud |
| unmapped event postbacks | sudden rise | advertiser renamed an event |
| postback lag p95 | > 3× baseline | MMP delay |
| reward / payout ratio | > 1 or > 2× config | config error (margin leak) |
| withdrawal failure rate | > 5% for one method | payout provider down |
| ticket rate per offer | z > +3 | Freecash's current signal (we should beat it) |

Detectors only *flag*. Several alerts in the same window are grouped into one investigation.

### 3.3 Investigator agent
- **Model:** Claude Opus 5.5 (`claude-opus-5-5`), effort `high` (the default is medium, so set it explicitly). Server-side `fallbacks: "default"` handles rare safety refusals.
- **Loop:** Claude Managed Agents. Anthropic hosts the agent loop. Our tools are custom tools answered by our backend. The Console session viewer gives us a clean trace to show judges. It's in beta and bills session runtime on top of tokens.
  **Fallback:** the SDK Tool Runner (`client.beta.messages.tool_runner`) if we want everything on the laptop.
- **Read-only tools** (all capped at about 200 rows with a timeout, so a bad query can't flood the context):
  - `run_sql(query)`: read-only SELECT against the views
  - `segment_breakdown(metric, offer_id, by=[os,country,app_version,mmp])`
  - `postback_log(offer_id, segment, since)`: includes unmapped event names
  - `recent_changes(target, since)`: deploys, offer edits, SDK updates. **Linking a problem to a recent change is the key move.**
  - `user_cluster(offer_id, since)`: groups users by device, IP range/ASN, account age and time-to-complete
  - `search_tickets(offer_id | text)`
  - `service_health(service)`
- **Final output:** a `submit_diagnosis` tool with `strict: true`. The prompt tells Claude to end with it. Forced tool choice isn't available on Opus 5.5, so the prompt and the strict schema do the job.
```json
{
  "category": "TRACKING_BREAK | FRAUD | CONFIG_ERROR | MMP_DELAY | INFRA | NO_ACTION",
  "scope": {"offer_id": "...", "os": "ios", "country": "DE", "event": null},
  "root_cause": "AppsFlyer SDK update in game v2.3 stopped install postbacks on iOS",
  "evidence": [{"query": "...", "finding": "installs/clicks 0.31 → 0.02 since 14:05; v2.3 released 14:00"}],
  "confidence": 0.86,
  "impact": {"users_affected": 412, "user_rewards_at_risk_eur": 3100, "revenue_at_risk_eur": 5200},
  "proposed_actions": ["pause_offer_segment", "post_status_banner", "queue_auto_credit", "escalate_advertiser"]
}
```

### 3.4 Policy gate and actions
| Tier | Actions | Why |
|---|---|---|
| **Auto** | open incident · post an in-app "tracking issue, don't start this offer" banner for that segment · draft the support reply · draft the advertiser email | Reversible, and protects users straight away |
| **Needs approval** (one-click) | pause offer for the segment · hold payouts for a suspicious cluster · bulk-credit affected users | Moves money or affects revenue |
| **Never** | ban users · change rewards · delete data | Too risky; a human decides |

Every action is written to an audit log with the evidence behind it. A dry-run mode is on by default.

### 3.5 Outputs (where it saves support time)
- **Incident report:** timeline, scope, root cause, impact.
- **Advertiser escalation:** "Install postbacks for iOS DE stopped at 14:05, after your v2.3 release…"
- **Support macro + auto-credit list:** users who clicked and opened the app in the broken window get credited without a ticket. This cuts the manual reviews they say they can't keep up with.

## 4. Fault scenarios (the demo and the eval)
| # | Inject | Correct answer |
|---|---|---|
| 1 | iOS-DE install postbacks stop after an SDK update | TRACKING_BREAK, scope ios/DE, linked to the change |
| 2 | Game renames `level_10` → `lvl10` | TRACKING_BREAK (unmapped event), propose mapping fix |
| 3 | 300 new accounts, same ASN/device pattern, finishing in 4 min | FRAUD, hold payouts for the cluster |
| 4 | Reward set to 25.00 instead of 2.50 | CONFIG_ERROR, pause offer |
| 5 | PayPal withdrawals return 5xx | INFRA, banner + incident, offers untouched |
| 6 | MMP lag goes up but postbacks still arrive | MMP_DELAY, wait, no pause |
| 7 | Normal Sunday-night drop across all offers | **NO_ACTION** (false-positive test) |

The answers differ by **direction and scope**: conversion **down** in one segment → tracking break. Conversion **up**, too fast, clustered users → fraud. Everything down at once → infra, or nothing wrong.

## 5. Eval harness (their job ad asks for evals)
Run each scenario 5× with different random seeds. Report:
- detection rate and **time to detect vs. the ticket-spike baseline** (simulated tickets come in 2–6 h after the break)
- category accuracy · scope accuracy (exact segment)
- **wrong action rate on #6/#7** (pausing a healthy offer costs revenue)
- users protected and € saved compared with waiting for tickets
- cost and latency per investigation

Rough cost per investigation: 10 tool calls, about 60k tokens in and 5k out, so **about $0.35**. Less with prompt caching of the system prompt and schema docs. The LLM runs only on alerts, never on the stream itself.

## 6. Built on Almedia's own stack
From their live job board (20 Nov 2026, 28 open roles):

| Their stack (named in postings) | What we use |
|---|---|
| Python + FastAPI (Product Engineer), Python services for rewards/offers/payments (Junior Engineer) | **Python + FastAPI** backend |
| BigQuery, dbt, Hex, PostgreSQL, MongoDB, MySQL, Redis | **DuckDB with BigQuery-style SQL**. Optionally define the tables as **dbt models (dbt-duckdb)**, with the detectors as dbt tests / data-quality checks |
| Google Pub/Sub, RabbitMQ, Kafka | Simulator publishes to an in-process queue shaped like Pub/Sub messages |
| Datadog | Alerts follow a Datadog-monitor-style JSON shape, so they could plug in later |
| GCP, Docker, Kubernetes, Terraform, GitLab CI / GitHub Actions | `docker compose up` for the demo |
| "Claude, Cursor, Copilot" as everyday tools | Built with Claude Code; the agent runs on Claude |

**What they say they want from AI work** (AI Full Stack Engineer posting): *"Architect durable AI workflows that adapt as the LLM landscape evolves, not brittle prompt chains"*, *"complex agentic systems"*, *"work across multiple systems and data sources, making sense of fragmented data"*, and work with marketing/UA/operations teams.
**Our answer:** a graph with durable checkpoints, one swappable model adapter, and an eval harness, so a model change is measured rather than guessed.

**Their data team's own goal** (Junior Software Engineer – Data): *"Build tests, monitoring, and data quality checks that catch problems before a stakeholder has to tell you."* That's exactly what Postback Patrol does.

**Agent framework:** no current posting names one. (An earlier "AI Engineer – Data" posting listed LangChain/LangGraph, but it's no longer up.) We use **LangGraph** for the parts it is good at: durable checkpoints, `interrupt()` for human approval, and streaming each step to the UI. The investigator node calls Claude through a thin adapter, so we can swap the model and compare it in the eval.

## 7. Tech choices
- Python 3.11. `duckdb` for the data, `langgraph` + `langchain-anthropic` (Claude Opus 5.5) for the agent, FastAPI backend, Streamlit dashboard (fastest to build).
- Simulator: users, offers and traffic with a daily pattern, plus a fault-injector panel.
- If Almedia gives us **real or sandbox data or their app**, swap the simulator for their feed. The detectors and tools only see SQL views, so nothing else changes.

## 8. Demo script (3 min)
1. Dashboard is calm. *"Freecash today finds a broken offer when tickets pile up, hours later."*
2. Inject #1 live. The detector fires within about a minute. The agent's queries stream on screen: breakdown → only iOS DE → recent changes → SDK v2.3.
3. Diagnosis card appears. Banner goes up automatically. Click **Approve pause**. Show the advertiser email, the auto-credit list and the support macro.
4. Inject #3 (fraud). Same alert shape, **opposite** diagnosis, payouts held for the cluster.
5. Inject #7. The agent says *no action*.
6. Scoreboard: detection, accuracy, false actions, time saved vs. tickets, € protected, cost per run.

## 9. Build plan (2 people)
| When | Person A (agent) | Person B (data/UI) |
|---|---|---|
| Sat 10–11 | Talk to Almedia (questions below) | Same |
| Sat 11–14 | Tools + agent loop + diagnosis schema | Simulator + DuckDB tables + fault injector |
| Sat 14–18 | Investigation on scenarios #1, #3 | Detectors + dashboard |
| Sat 18–22 | Policy gate + actions + outputs | Agent timeline view + approve button |
| Sun AM | Eval harness, all 7 scenarios, fix misses | Scoreboard, polish |
| Sun PM | Rehearse demo ×3, record backup video | Slides (problem → demo → numbers) |

## 10. Ask Almedia on Saturday morning
1. Is there a **real app, sandbox or dataset** for the track, or do we simulate?
2. How do you spot a broken offer today? Is it really ticket spikes, and what's your "offer health" metric?
3. Your top 3 incident types from the last quarter?
4. What can run **automatically**, and what needs a human (pausing offers, holding payouts)?
5. Which fraud signals matter most (device, ASN, time-to-complete, emulators)?
6. Is Datadog/BigQuery the right picture, and should we match their schema?
