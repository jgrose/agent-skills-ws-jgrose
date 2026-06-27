# Challenge 6 — "Project ReadyNow!" FEMA Emergency-Preparedness Agent — Design

**Date:** 2026-06-26
**Status:** Approved (design) — pending build
**File to produce:** `agent-skills-ws-jgrose/challenge6.ipynb`

## Goal

Capstone: a complex ADK multi-agent system for FEMA "ReadyNow!" that gives people
real-time disaster guidance — weather + news alerts, evacuation routes to safety,
and preparedness answers — validated and refined before it reaches the user, with
full interaction logging, input validation, and deployment to Vertex AI Agent Engine.

## Requirements mapping (from the case study)

| # | Requirement | How this design meets it |
|---|-------------|--------------------------|
| 1 | Root agent describing capabilities + coordinating sub-agents | `Master Chief` root coordinator |
| 2 | Sub-agents: weather, internet search, routes via Maps, Q&A | `Raiden`, `Cypher`, `Sonic`, `Navi` |
| 3 | Sequential workflow that validates + refines responses | `SequentialAgent(Cortana → GLaDOS → Link)` |
| 4 | Callbacks logging ALL user-agent interactions | reuse Ch2 callbacks, extended to log tool calls |
| 5 | User input validation (refuse off-mission) | adapted Ch2 validator: FEMA mission-scope, fail-closed |
| 6 | Deploy to Agent Platform (Agent Engine) | `AdkApp` → `agent_engines.create` (version-pinned) |
| 7 | Test code demonstrating functionality | local tests (saved) + remote `stream_query` tests |
| + | Architecture diagram (instruction #1) | rendered/described in notebook |

## Architecture

```
Master Chief (root coordinator)            ← states mission; refuses off-mission input (fail-closed)
  └─ ReadyNow_Team (SequentialAgent)       ← the validate-and-refine workflow
       ├─ 1. Cortana (Dispatcher, LlmAgent)  → state['draft_answer']
       │       AgentTool specialists:
       │         • Raiden — weather forecast + ACTIVE ALERTS (NWS, no key)
       │         • Cypher — real-time news / internet search (google_search)
       │         • Sonic  — evacuation routes (Routes API via ADC + fallback)
       │         • Navi   — preparedness Q&A (google_search / Ready.gov)
       ├─ 2. GLaDOS (Critique)  reads {draft_answer}         → state['review_notes']
       └─ 3. Link   (Refine)    reads {draft_answer}+{review_notes} → state['final_answer']
```

Pattern: **router-inside-Sequential** (the dispatcher is step 1 of the SequentialAgent),
so every request takes one traceable path and is always critiqued + refined before return.
Specialists are wired as `AgentTool` (so the built-in `google_search` works and the
dispatcher can call several in one turn). Workflow steps are wired as `sub_agents` of
the SequentialAgent, handing off via session-state keys.

## Components — reuse vs. new

Reused from jgrose Challenge 2–4 (adapted, original wording):
- Auth cell (ADC, Vertex, `GOOGLE_GENAI_USE_VERTEXAI=TRUE`).
- `get_lat_lon` — Geocoding API v4 via **ADC bearer token, no API key**.
- `get_extended_weather_forecast` — NWS.
- Logging callbacks (`log_user_prompt_callback`, `log_model_response_callback`) + helpers.
- Coordinator pattern (Max → renamed Master Chief), AgentTool specialists.
- Critique→refine agents (Vera/Remy → GLaDOS/Link).

New:
- `get_active_weather_alerts(lat, lon)` — NWS `/alerts/active?point=lat,lon` (no key).
  Covers the "weather **and alerts**" requirement.
- `get_evacuation_route(origin, destination)` — Google **Routes API**
  (`routes.googleapis.com …:computeRoutes`) authenticated with the **same ADC bearer
  token** as geocoding (no hardcoded Maps key), with a **haversine simulation fallback**
  if the Routes API is unavailable/restricted, so the demo always produces a route.
- Mission-scope input validation: adapt the Ch2 validator to refuse requests outside the
  FEMA emergency-preparedness mission (keyword + short LLM classifier), **fail-closed**.
- Tool-call logging: extend `after_tool` / `before_tool` so search and route calls are
  logged too ("log ALL interactions").
- Deployment: `AdkApp(agent=root)` → `agent_engines.create(...)` with `importlib.metadata`
  version pinning; Maps not needed in env_vars because routing uses ADC. Clearly-marked
  cleanup cell to delete the engine afterward.
- Remote test: `agent_engines.get(...).stream_query(...)` against the deployed engine.

## Key design decisions

- **No hardcoded Maps API key.** Both geocoding and routing use the ADC bearer-token
  pattern already in jgrose's `get_lat_lon`. Eliminates the committed-secret flaw seen
  in the reference notebooks; routing degrades gracefully to simulation if Routes API
  is off.
- **Fail-closed validation.** For a public emergency endpoint, an off-mission or unsafe
  request is refused; if the classifier errors, default to refuse (safer than fail-open).
- **Tiered models (optional):** `gemini-2.5-flash` for specialists; may use a stronger
  model for critique/refine if budget allows. Default all flash for cost/latency.
- **Deploy is build-ready, user-run.** All deploy + remote-test cells are authored and
  validated, but executed by the user in a live Colab Enterprise lab (auto-auth).

## Originality

All agent names (Master Chief, Cortana, Raiden, Cypher, Sonic, Navi, GLaDOS, Link),
instructions, comments, and markdown are written fresh. Same ADK concepts and jgrose
code lineage; no lines copied verbatim from the jimmy-go / jimpson / justin notebooks.

## Testing

- **Local (saved outputs):** disaster scenarios — (a) hurricane weather+alerts for a US
  city, (b) evacuation route to safety, (c) preparedness go-bag question, (d) an
  off-mission request that is refused. Event stream shows dispatcher → specialists →
  critique → refine.
- **Remote:** the same prompts run against the deployed Agent Engine via `stream_query`.

## Practical gotchas

- Run in Colab Enterprise (auto-auth). Enable Geocoding + Routes APIs.
- Deploy takes minutes and persists (cost) until the cleanup cell is run.
- NWS is US-only (consistent with FEMA scope).
