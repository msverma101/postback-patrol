# Postback Patrol 🚨

*Nobody should do the task and not get paid.*

An AI agent that keeps a rewards marketplace healthy. Built for the **Almedia track** at HackLab × BRISA (21–22 Nov 2026, CIC Berlin): *Agentic Monitoring for an Ads Marketplace App*.

## The problem
On offerwall apps like Freecash, a reward is credited only if a postback travels **user → game → MMP → app**. When one link breaks (often for just one OS or country), users do the task and don't get paid. Today it's usually noticed from support ticket spikes, hours later, and fixed by hand.

## What it does
1. **Detectors** watch clicks, postbacks, payouts and tickets every minute, per offer × OS × country.
2. A **Claude investigator** digs into each alert with read-only tools and decides: tracking break, fraud, config error, MMP delay, infra, or nothing wrong.
3. A **policy gate** decides what runs automatically (banner, incident) and what needs one-click approval (pause offer, hold payouts, credit users).
4. **Outputs:** incident report, advertiser escalation, support macro, audit log.
5. An **eval harness** replays 7 fault scenarios with known answers and scores detection, accuracy, false actions and time saved compared with waiting for tickets.

Full design: [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md)

## Stack
Python 3.11 · DuckDB · FastAPI · Streamlit · Anthropic SDK (Claude Opus 5.5)

## Layout (planned)
```
sim/        traffic simulator + fault injector
detectors/  anomaly detectors (SQL)
agent/      investigator agent, tools, diagnosis schema
actions/    policy gate + actions + outputs
eval/       scenarios + scoreboard
app/        dashboard
```
