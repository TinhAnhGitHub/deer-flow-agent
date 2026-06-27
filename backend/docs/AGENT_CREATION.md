# Custom Agent Creation Guide

This document explains how custom agents work in DeerFlow, how to create them, and the design patterns used.

## Architecture Overview

DeerFlow follows a **one graph, many profiles** pattern — every custom agent runs the identical LangGraph graph (`make_lead_agent`) but with different configuration, personality, tool access, and skill scoping. There is no need to compile a new graph per agent.

```
┌──────────────────────────────────────────────────────────────────────────┐
│                         Custom Agent Architecture                        │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  users/{user_id}/agents/{agent_name}/                                    │
│  ├── config.yaml         # AgentConfig (model, tool_groups, skills)      │
│  ├── SOUL.md             # Personality injected into system prompt       │
│  └── memory.json         # Per-agent memory (isolated from other agents)  │
│                                                                          │
│  Agent lifecycle:                                                        │
│  POST /api/agents → files created → run with agent_name → self-update   │
└──────────────────────────────────────────────────────────────────────────┘
```

## AgentConfig Schema

Each custom agent has a `config.yaml` parsed into `AgentConfig` (`backend/packages/harness/deerflow/config/agents_config.py:38`):

```yaml
name: my-research-agent        # Hyphen-case identifier (required)
description: "..."             # Human-readable description
model: gpt-4                   # Optional model override (null = default)
tool_groups:                   # Optional tool group whitelist
  - web_search
  - code_execution
skills:                        # Optional skill whitelist
  - researcher                 # None = all enabled skills
  - data-analysis              # [] = no skills
```

| Field | Type | Meaning |
|---|---|---|
| `name` | `str` | Unique identifier, must match `^[A-Za-z0-9-]+$` |
| `description` | `str` | Human-readable description |
| `model` | `str \| None` | Override the default model for this agent |
| `tool_groups` | `list[str] \| None` | Whitelist tool groups; `None` = all tools |
| `skills` | `list[str] \| None` | Whitelist skills; `None` = all enabled, `[]` = none |

## SOUL.md Personality System

Every custom agent can have a `SOUL.md` file that defines its personality, behavioral guardrails, and specialized instructions. The content is injected into the system prompt at runtime.

```
# SOUL.md

You are a meticulous research assistant specializing in academic literature.
Always cite sources, prefer peer-reviewed references, and organize findings
by relevance. When summarizing papers, highlight methodology, limitations,
and reproducibility concerns.
```

**How it works**: `apply_prompt_template()` at `backend/packages/harness/deerflow/agents/lead_agent/prompt.py:779` calls `get_agent_soul(agent_name)` which reads the SOUL.md file, then formats it into `SYSTEM_PROMPT_TEMPLATE` as the `{soul}` placeholder.

The soul is **not** part of the static system prompt prefix — it is per-agent content loaded at graph construction time.

## REST API

Full CRUD at `/api/agents` (requires `agents_api.enabled=true` in config):

| Method | Endpoint | Purpose |
|---|---|---|
| `POST` | `/api/agents` | Create agent (name, description, model, tool_groups, skills, soul) |
| `GET` | `/api/agents` | List all agents |
| `GET` | `/api/agents/{name}` | Get agent details + SOUL.md |
| `PUT` | `/api/agents/{name}` | Update agent config and/or SOUL.md |
| `DELETE` | `/api/agents/{name}` | Delete agent and all files |
| `GET` | `/api/agents/check?name=` | Validate name and check availability |

Also available at `/api/user-profile` for the global USER.md injected into all agents.

### Creating an Agent

```bash
curl -X POST /api/agents \
  -H "Content-Type: application/json" \
  -d '{
    "name": "my-research-agent",
    "description": "Expert research agent with web access",
    "model": "gpt-4",
    "tool_groups": ["web_search", "code_execution"],
    "skills": ["researcher"],
    "soul": "You are a meticulous research assistant..."
  }'
```

### Using an Agent

Pass the `agent_name` in the run configurable:

```json
{
  "configurable": {
    "agent_name": "my-research-agent",
    "thread_id": "thread-123"
  }
}
```

## How Instantiation Works

When a run request includes `agent_name`, the Gateway resolves it through this pipeline:

```
Gateway parse request → build_run_config()
  ↓
_make_lead_agent(config, app_config)
  ├─ cfg = _get_runtime_config(config) → extracts agent_name, is_bootstrap, etc.
  ├─ agent_config = load_agent_config(agent_name)
  │    ├─ resolve_agent_dir(name, user_id) → per-user path
  │    ├─ reads config.yaml → AgentConfig
  │    └─ validates name, returns None if name is None (default agent)
  │
  ├─ available_skills = _available_skill_names(agent_config, is_bootstrap)
  │    ├─ if agent_config.skills is None → all enabled skills
  │    ├─ if agent_config.skills is [] → no skills
  │    └─ if agent_config.skills is [...] → only listed skills
  │
  ├─ model_name = _resolve_model_name(requested or agent_config.model)
  │
  ├─ final_tools = get_available_tools(groups=agent_config.tool_groups)
  │    └─ filter_tools_by_skill_allowed_tools(...) → skill policy filter
  │
  ├─ middlewares = build_middlewares(config, agent_name=agent_name, ...)
  │    └─ MemoryMiddleware(agent_name) → per-agent memory isolation
  │
  ├─ system_prompt = apply_prompt_template(agent_name, soul, skills, ...)
  │    └─ SYSTEM_PROMPT_TEMPLATE.format(soul=get_agent_soul(agent_name))
  │
  └─ create_agent(model, tools, middlewares, system_prompt, state_schema)
```

All at `backend/packages/harness/deerflow/agents/lead_agent/agent.py:423`.

## Bootstrap Flow

When `is_bootstrap=True`, the agent enters a special bootstrap mode:

```
is_bootstrap=True
  ├─ available_skills restricted to {"bootstrap"} only
  ├─ tools include setup_agent (not update_agent)
  ├─ minimal system prompt for agent creation
  └─ purpose: let the user describe what agent they want,
       then setup_agent persists the new agent's SOUL.md + config.yaml
```

The `setup_agent` tool is a **bootstrap-only** tool that creates a brand-new custom agent directory. After bootstrap completes, subsequent runs use the created `agent_name` with the normal pipeline.

## Self-Updating Pattern

Custom agents get the `update_agent` tool (bound only when `agent_name` is set):

```python
# _make_lead_agent() line ~541
extra_tools = [update_agent] if agent_name else []
```

The `update_agent` tool allows the agent to:
- Overwrite its own `SOUL.md` (personality evolution)
- Update `config.yaml` fields (model, tool_groups, skills)

This enables **agent self-improvement** — the agent can reflect on its performance and persist configuration changes mid-conversation. The tool performs atomic writes (write to `.tmp` then rename) to prevent corruption.

## Per-User Isolation

Custom agents are stored per-user:

```
{base_dir}/users/{user_id}/agents/{agent_name}/
  ├── config.yaml
  ├── SOUL.md
  └── memory.json
```

- The same `agent_name` by different users is completely isolated
- `user_id` is resolved via `get_effective_user_id()` (defaults to `"default"` in no-auth mode)
- MemoryMiddleware stores per-agent memory at `users/{user_id}/agents/{agent_name}/memory.json`

## Tool and Skill Scoping

Custom agents can restrict their available tools and skills:

### Tool Groups

```yaml
# agent config.yaml
tool_groups: ["web_search", "code_execution"]
```

Passed to `get_available_tools(groups=...)` at `backend/packages/harness/deerflow/tools/__init__.py`. When `None`, all tools are available.

### Skill Whitelist

```yaml
# agent config.yaml
skills: ["researcher", "data-analysis"]
# None → all enabled skills
# [] → no skills
# ["a","b"] → only "a" and "b"
```

The whitelist is applied in `_available_skill_names()` and `filter_tools_by_skill_allowed_tools()` at `backend/packages/harness/deerflow/skills/tool_policy.py`.

## Creating a Completely New Agent Type (Different Graph)

Besides the custom agent profile pattern, you can create an entirely new LangGraph agent with a different graph structure by using the factory directly:

```python
from deerflow.agents.factory import create_deerflow_agent
from deerflow.agents.features import RuntimeFeatures
from langchain_core.messages import BaseMessage

class MyCustomState(BaseMessage):
    my_field: str = ""
    # ... your own state schema

agent = create_deerflow_agent(
    model=my_model,
    tools=[tool1, tool2],
    middleware=None,                    # None = auto-assembled
    extra_middleware=[MyCustomMiddleware()],
    features=RuntimeFeatures(...),      # or use declarative features
    state_schema=MyCustomState,         # or extend ThreadState
    checkpointer=checkpointer,
)
```

The factory at `backend/packages/harness/deerflow/agents/factory.py:61` accepts:

| Parameter | Purpose |
|---|---|
| `model` | Any `BaseChatModel` instance |
| `tools` | List of `BaseTool` instances |
| `middleware` | Full takeover — replaces entire chain |
| `features` | Declarative feature flags (mutually exclusive with `middleware`) |
| `extra_middleware` | Additional middlewares injected before ClarificationMiddleware |
| `plan_mode` | Enable TodoMiddleware |
| `state_schema` | Custom LangGraph state type |
| `checkpointer` | Persistence backend |

## Testing Custom Agents

Tests live in `backend/tests/test_custom_agent.py`. Key test patterns:

```python
# Create agent via API
response = await client.post("/api/agents", json={...})
assert response.status_code == 201

# Run with agent_name
result = await client.post(f"/api/threads/{tid}/runs/stream", json={
    "input": {"messages": [...]},
    "config": {"configurable": {"agent_name": "my-agent"}},
})

# Verify agent-specific behavior
assert "soul content" in result["messages"][0]["content"]
```

## Key Files Reference

| File | Purpose |
|---|---|
| `backend/app/gateway/routers/agents.py` | REST API CRUD for custom agents |
| `backend/packages/harness/deerflow/config/agents_config.py` | `AgentConfig` model, `load_agent_config()`, `validate_agent_name()` |
| `backend/packages/harness/deerflow/agents/lead_agent/agent.py` | `_make_lead_agent()` — resolves agent config into the graph |
| `backend/packages/harness/deerflow/agents/lead_agent/prompt.py` | `apply_prompt_template()`, `get_agent_soul()`, SYSTEM_PROMPT_TEMPLATE |
| `backend/packages/harness/deerflow/tools/builtins/setup_agent.py` | Bootstrap agent creation tool |
| `backend/packages/harness/deerflow/tools/builtins/update_agent.py` | Self-update tool for custom agents |
| `backend/packages/harness/deerflow/agents/factory.py` | `create_deerflow_agent()` universal factory |
| `backend/packages/harness/deerflow/config/paths.py` | Path resolution: `user_agent_dir()`, `agent_dir()` |
| `backend/tests/test_custom_agent.py` | Test suite |
