# Challenge 3 — Multi-Agent System (Root → Weather + Search)

**Date:** 2026-06-26
**Notebook:** `challenge3.ipynb`

## Goal

Demonstrate a multi-agent system in Google ADK with three agents:

1. A **root / coordinator** agent that receives user requests and delegates.
2. A **weather** agent (the existing "Pat" from challenge 2).
3. A **search** agent using ADK's built-in `google_search` tool.

The root delegates each request to the appropriate sub-agent, and the notebook
outputs events that demonstrate the sub-agents being used.

## Key constraint that drives the design

ADK's built-in tools (`google_search`, code execution, Vertex AI Search) have a
hard limitation: a built-in tool must be the **only** tool in its agent, and
*"Built-in tools cannot be used within a sub-agent"*
(<https://adk.dev/tools/limitations/>). The partial exception for search via
`sub_agents=` is still buggy — routing succeeds but execution fails
(<https://github.com/google/adk-python/issues/4449>).

**Decision:** wire the specialists to the root via `AgentTool` (agent-as-a-tool),
not `sub_agents=`. This is the documented, robust workaround and still satisfies
"root delegates to weather and search sub-agents."

## Architecture

```
                 ┌─────────────────────────┐
   user ───────► │  Max  (root/coordinator) │
                 │  gemini-2.5-flash        │
                 │  tools=[AgentTool(Pat),  │
                 │         AgentTool(Sage)] │
                 └───────┬─────────────┬────┘
              weather    │             │   general knowledge /
              questions  ▼             ▼   current events
        ┌──────────────────────┐  ┌──────────────────────┐
        │  Pat  (weather)      │  │  Sage  (search)      │
        │  get_lat_lon +       │  │  tools=[google_search]│
        │  get_extended_...    │  │  (built-in, alone)    │
        └──────────────────────┘  └──────────────────────┘
```

## Components

- **Pat (weather sub-agent)** — reused unchanged from challenge 2. Custom function
  tools `get_lat_lon` + `get_extended_weather_forecast`, plus the existing logging
  and US-location/malicious-input validation callbacks. No changes required.

- **Sage (search sub-agent)** — new. `Agent(name="Sage", model="gemini-2.5-flash",
  tools=[google_search])`. Gets the reused *logging* callbacks
  (`log_user_prompt_callback`, `log_model_response_callback`) so its activity is
  visible, but deliberately **not** the weather-specific US-location validation
  callback (that would wrongly block general search queries).

- **Max (root/coordinator)** — new. `tools=[AgentTool(agent=weather_agent),
  AgentTool(agent=search_agent)]`. Instruction routes weather/forecast/temperature
  questions about US locations to `Pat`, and everything else (general knowledge,
  current events, facts) to `Sage`; may call both when a request needs both. Gets
  the logging callbacks so its routing decision is logged.

## Data flow

1. User prompt → `Max` (root) via `InMemoryRunner.run_async`.
2. `Max` emits a `function_call` to the chosen `AgentTool` (`Pat` or `Sage`).
3. The sub-agent runs its own tool loop (Pat: geocode → NWS; Sage: google_search)
   and its `CALLBACK/...` log lines fire.
4. The sub-agent's answer returns to `Max` as a `function_response`.
5. `Max` relays a final text answer to the user.

## Demonstrating sub-agent usage (events)

A new `run_and_show_events(prompt)` helper streams `run_async` and prints, per
event: the **author**, any **tool call** (`Max → Pat(...)`, `Max → Sage(...)`),
any **tool result**, and the **final response**. Combined with the sub-agents'
existing `CALLBACK/...` log lines, this makes delegation unmistakable even if the
inner sub-agent events do not propagate to the outer event stream.

### Test prompts

| Prompt | Expected delegation |
|--------|--------------------|
| "What's the weather in Chicago, IL?" | → **Pat** (weather) |
| "Who is the CEO of Google and what year was it founded?" | → **Sage** (search) |
| "What's the weather in Austin, TX, and what is Austin famous for?" | → **both** Pat and Sage |

## Scope / non-goals

- Existing challenge 2 cells (auth, tools, callbacks, Pat, the Groq agent, the
  challenge-2 tests) are left **untouched**. New cells are appended.
- The Groq/Llama agent remains as-is (it errors harmlessly without an API key);
  it is not part of the new multi-agent system.
- All agents run locally via `InMemoryRunner` (no Vertex Agent Engine), reusing
  the notebook's existing auth and runtime setup.

## Deliverable

`challenge3.ipynb` with the appended cells, committed and pushed to the GitHub
repository for grading.
