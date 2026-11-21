# Postback Patrol 🚨: Eye of the System

A fully autonomous monitoring system for a mock shopping marketplace. Built for the **Almedia track** at HackLab × BRISA (21–22 Nov 2026, CIC Berlin): *Eye of the System: Fully Agentic Monitoring*.

## What it does
1. **Metric graph:** about 35 marketplace metrics connected by typed edges: exact formulas, breakdowns by country, platform and category, funnel steps, possible causes and events. A tree view lets people browse it.
2. **Automatic extension:** give it a new metric (name, description, SQL). It works out where the metric fits by testing candidate formulas against the data, and marks ambiguous fits instead of guessing.
3. **Continuous monitoring:** it decides what to watch based on importance, surprise and how recently each metric was checked. Expected values account for time of day and day of week. Late data and planned events are filtered out before anything becomes an alert.
4. **Autonomous investigation:** a best-first search over the graph. It splits a change exactly into its parts, narrows it to the group responsible, then tests possible causes against releases and events. It stops once a supported cause explains at least 80% of the change.
5. **Reports:** € impact, the evidence chain, explanations that were ruled out, what's still uncertain, and the full query trail. Incidents are deduplicated and tracked until they resolve.

The code does the maths. Claude makes the judgement calls and writes the story.

Full design: [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md)

## Stack
Python 3.12 · uv · DuckDB · networkx · Anthropic SDK (Claude Opus 5.5) · Streamlit. Everything except the Claude API is free and runs on a laptop.

## Layout (planned)
```
sim/  warehouse/  graph/  extender/  monitor/  investigate/  incidents/  llm/  app/  eval/
```
