# Extension Points Guide

This document maps every mechanism for extending DeerFlow without modifying its core code, ordered from most common to most specialized.

## Overview

```
Extensibility Mechanisms
├── 1. Middleware Chain (Python injection)
├── 2. Config-driven (no code changes)
│   ├── resolve_variable() reflection
│   ├── mcpInterceptors (extensions_config.json)
│   └── httpMiddlewares (extensions_config.json)
├── 3. Pluggable Providers
│   ├── GuardrailProvider
│   ├── SandboxProvider
│   ├── Community tools
│   └── Custom tools/config.yaml
├── 4. IM Channels
├── 5. New API Routes (Gateway)
├── 6. Skills (markdown files)
└── 7. New Subagent Types
```

## 1. Middleware Chain

The primary extension point for cross-cutting concerns (auth, logging, rate limiting, safety, etc.).

### AgentMiddleware Protocol

Every middleware implements `langchain.agents.middleware.AgentMiddleware[AgentState]`:

```python
from langchain.agents.middleware import AgentMiddleware
from langchain_core.messages import BaseMessage

class MyMiddleware(AgentMiddleware):
    """One concern, ~200 lines, single responsibility."""

    def before_agent(self, state: dict) -> dict | None:
        # Pre-agent execution — mutate state or return early
        return state

    def after_agent(self, state: dict, response: dict) -> dict | None:
        # Post-agent execution
        return response

    def before_model(self, messages: list[BaseMessage], state: dict) -> list[BaseMessage] | None:
        # Before LLM invocation — mutate messages
        return messages

    def after_model(self, result: dict, state: dict) -> dict | None:
        # After LLM invocation — mutate result
        return result

    def wrap_model_call(self, handler):
        # Wrap the entire model call (outermost wrapper)
        # Used by InputSanitizationMiddleware, SystemMessageCoalescingMiddleware
        ...

    def wrap_tool_call(self, request, handler):
        # Wrap every tool invocation (for guardrails, auditing, error handling)
        ...
```

### Injection Methods

**A. `extra_middleware` (recommended)**

Injected before ClarificationMiddleware (which must always be last):

```python
from deerflow.agents.middlewares import build_middlewares

middlewares = build_middlewares(
    config,
    model_name=model_name,
    custom_middlewares=[MyAuthMiddleware(), MyLoggingMiddleware()],
)
```

**B. `middleware` (full takeover)**

Replace the entire auto-assembled chain:

```python
from deerflow.agents.factory import create_deerflow_agent

agent = create_deerflow_agent(
    model=model,
    tools=tools,
    middleware=[MyMiddleware1(), MyMiddleware2()],
    # Cannot use with features or extra_middleware
)
```

**C. `features` (declarative)**

Use `RuntimeFeatures` to declaratively enable/disable built-in features:

```python
from deerflow.agents.features import RuntimeFeatures
from deerflow.agents.factory import create_deerflow_agent

agent = create_deerflow_agent(
    model=model,
    tools=tools,
    features=RuntimeFeatures(
        summarization=True,
        auto_title=True,
        memory=True,
        vision=True,
        subagent=True,
        todo=True,
        guardrail=True,
    ),
)
```

### Chain Ordering (Lead Agent)

The complete chain at `backend/AGENTS.md:206`:

| # | Middleware | Hook | Optional? |
|---|---|---|---|
| 1 | InputSanitizationMiddleware | `wrap_model_call` | No |
| 2 | ToolOutputBudgetMiddleware | `after_model` | No |
| 3 | UploadsMiddleware | `before_agent` | No (lead only) |
| 4 | ThreadDataMiddleware | `before_agent` | No |
| 5 | SandboxMiddleware | `before_agent` / `after_agent` | No |
| 6 | DanglingToolCallMiddleware | `wrap_model_call` | No |
| 7 | LLMErrorHandlingMiddleware | `wrap_model_call` | No |
| 8 | **GuardrailMiddleware** | `wrap_tool_call` | Yes (`guardrails.enabled`) |
| 9 | SandboxAuditMiddleware | `wrap_tool_call` | No |
| 10 | ToolErrorHandlingMiddleware | `wrap_tool_call` | No |
| 11 | DynamicContextMiddleware | `before_model` | No |
| 12 | SkillActivationMiddleware | `before_model` | No |
| 13 | **SummarizationMiddleware** | `before_model` | Yes |
| 14 | **TodoListMiddleware** | `before_model` / `after_model` | Yes (`is_plan_mode`) |
| 15 | **TokenUsageMiddleware** | `after_model` | Yes (`token_usage.enabled`) |
| 16 | TitleMiddleware | `after_model` | No |
| 17 | MemoryMiddleware | `after_agent` | No |
| 18 | **ViewImageMiddleware** | `before_model` | Yes (vision model) |
| 19 | DeferredToolFilterMiddleware | `wrap_model_call` | Yes (`tool_search.enabled`) |
| 20 | SystemMessageCoalescingMiddleware | `wrap_model_call` | No |
| 21 | **SubagentLimitMiddleware** | `after_model` | Yes (`subagent_enabled`) |
| 22 | **LoopDetectionMiddleware** | multiple | Yes (`loop_detection.enabled`) |
| 23 | **TokenBudgetMiddleware** | multiple | Yes (`token_budget.enabled`) |
| 24 | **Custom middlewares** | — | Yes (user-injected) |
| 25 | **SafetyFinishReasonMiddleware** | `after_model` | Yes (`safety_finish_reason.enabled`) |
| 26 | ClarificationMiddleware | `wrap_tool_call` | No (must be last) |

Subagents use a shorter chain (items 1-2, 4-10, skipping lead-only middlewares).

## 2. Config-Driven Extension (No Code Changes)

Tools, models, sandbox providers, guardrail providers, and interceptors are all loaded at runtime via `resolve_variable()` — no DeerFlow source modification needed.

### resolve_variable() Reflection

Located at `backend/packages/harness/deerflow/reflection/`:

```python
from deerflow.reflection import resolve_variable, resolve_class

# Load any Python variable by import path
my_tool = resolve_variable("my_package.tools:my_tool_function")

# Load and validate a class against a base class
my_provider = resolve_class(
    "my_package.providers:MyProvider",
    base_class=MyProviderBase,
)
```

### Adding a New Tool

```yaml
# config.yaml
tools:
  - name: my_custom_tool
    use: my_package.tools:my_custom_function
    description: "My custom tool"
    group: my_group
```

The function must return a `BaseTool` instance or a callable that LangChain can wrap.

### Adding a New Model

```yaml
# config.yaml
models:
  - name: my-model
    display_name: My Custom Model
    use: langchain_my_provider:ChatMyModel
    model: my-model
    api_key: $MY_API_KEY
    max_tokens: 4096
    supports_thinking: false
    supports_vision: false
```

### Adding a New Sandbox Provider

```yaml
# config.yaml
sandbox:
  use: my_package.sandbox:MySandboxProvider
```

Must implement the `SandboxProvider` protocol (`acquire`, `get`, `release`) and `Sandbox` interface (`execute_command`, `read_file`, `write_file`, `list_dir`).

### Adding a New Guardrail Provider

```yaml
# config.yaml
guardrails:
  enabled: true
  provider: my_package.guardrails:MyGuardrailProvider
```

Must implement structural protocol with `evaluate(request)` and `aevaluate(request)` methods (no base class required).

### mcpInterceptors (extensions_config.json)

Register custom interceptors that run before every MCP tool call:

```json
{
  "mcpInterceptors": [
    "my_package.mcp.auth:build_auth_interceptor",
    "my_package.mcp.logging:build_logging_interceptor"
  ]
}
```

Each interceptor function receives the MCP tool request and can modify headers, inject auth tokens, log calls, or abort. Added via PR #2451 (upstream `bytedance/deer-flow`).

### httpMiddlewares (extensions_config.json)

Register ASGI middlewares on the Gateway FastAPI app:

```json
{
  "httpMiddlewares": [
    "my_package.http:MyHTTPMiddleware"
  ]
}
```

Combined with automatic `request.state.run_metadata → config["metadata"]` forwarding (PR #3135), this enables per-request context injection (tenant IDs, tracing headers, etc.) without patching Gateway source.

## 3. Pluggable Providers

### Community Tools

Located at `backend/packages/harness/deerflow/community/`. Each tool is a self-contained subpackage:

```
community/
├── tavily/           # Web search + fetch
├── jina_ai/          # Jina reader API
├── firecrawl/        # Web scraping
├── image_search/     # Image search via DuckDuckGo
├── aio_sandbox/      # Docker sandbox provider
├── brave/            # Brave search
├── browserless/      # Browser automation
├── ddg_search/       # DuckDuckGo search
├── exa/              # Exa search
├── fastcrw/          # Fast crawl
├── groundroute/      # Ground route
├── infoquest/        # Info quest
└── searxng/          # SearXNG search
```

To add a new community tool:
1. Create subpackage `community/my_provider/`
2. Implement the tool function
3. Add it to `config.yaml[tools]` via `resolve_variable()`

### Custom Tools via config.yaml

Beyond community tools, any custom tool can be registered:

```yaml
tools:
  - name: my_api_tool
    use: my_package.tools:my_api_tool
    description: "Query my internal API"
    group: internal
```

Tool groups are referenced by `AgentConfig.tool_groups` for per-agent scoping.

## 4. IM Channels

Add a new messaging platform by subclassing `Channel` at `backend/app/channels/base.py`:

```python
from app.channels.base import Channel

class MyPlatformChannel(Channel):
    async def start(self):
        # Connect to platform, start polling/webhook
        ...

    async def stop(self):
        # Disconnect, clean up
        ...

    async def send(self, message: OutboundMessage) -> None:
        # Send message to platform
        ...
```

Then register it in `app/channels/service.py` and configure in `config.yaml`:

```yaml
channels:
  my_platform:
    enabled: true
    api_key: $MY_API_KEY
```

Existing implementations: `slack.py`, `feishu.py`, `telegram.py`, `discord.py`, `dingtalk.py`, `wechat.py`.

## 5. New API Routes (Gateway)

Add a new FastAPI router to `backend/app/gateway/routers/`:

```python
from fastapi import APIRouter

router = APIRouter(prefix="/api", tags=["my_feature"])

@router.get("/my-feature")
async def my_endpoint():
    return {"status": "ok"}
```

Then register it in `create_app()` at `backend/app/gateway/app.py`:

```python
from app.gateway.routers import my_feature

app.include_router(my_feature.router)
```

## 6. Skills (Markdown Files)

Skills are the simplest extension — just a `SKILL.md` file with YAML frontmatter:

```markdown
---
name: PDF Processing
description: Handle PDF documents efficiently
license: MIT
allowed-tools:
  - read_file
  - write_file
  - bash
---

# Skill Instructions

Content injected into system prompt when the skill is enabled...
```

Place in `skills/public/` (committed) or `skills/custom/` (gitignored). Enable/disable via `extensions_config.json`:

```json
{
  "skills": {
    "pdf-processing": {"enabled": true}
  }
}
```

Slash activation: `/skill-name task` loads the SKILL.md for the current turn only.

Install via `POST /api/skills/install` with a `.skill` ZIP archive.

## 7. New Subagent Types

Add a new built-in subagent at `backend/packages/harness/deerflow/subagents/builtins/`:

```python
from deerflow.subagents.executor import SubagentConfig

my_agent = SubagentConfig(
    name="my-specialist",
    description="Specialist for X tasks",
    tools=["tool1", "tool2"],  # explicit tool list
    system_prompt="You are a specialist...",
)
```

Register in `backend/packages/harness/deerflow/subagents/registry.py`.

## Extension Point Decision Tree

```
What do you want to add?
│
├─ Cross-cutting behavior (auth, logging, safety)?
│   └─ Write an AgentMiddleware → inject via extra_middleware
│
├─ A new tool/capability (web search, DB query)?
│   └─ Write tool function → add to config.yaml[tools]
│
├─ A new model provider?
│   └─ Write LangChain integration → add to config.yaml[models]
│
├─ A new sandbox (Docker, K8s, remote)?
│   └─ Implement Sandbox + SandboxProvider → set config.yaml[sandbox.use]
│
├─ Pre-tool authorization?
│   └─ Implement GuardrailProvider → set config.yaml[guardrails]
│
├─ Intercept MCP tool calls (auth headers, logging)?
│   └─ Write interceptor → add to extensions_config.json[mcpInterceptors]
│
├─ Gateway-level middleware (per-request context)?
│   └─ Write ASGI middleware → add to extensions_config.json[httpMiddlewares]
│
├─ New REST endpoint?
│   └─ Create FastAPI router → register in create_app()
│
├─ New IM platform?
│   └─ Subclass Channel → register in service.py
│
├─ Domain-specific instruction set?
│   └─ Write SKILL.md → place in skills/{public,custom}/
│
└─ New subagent type?
    └─ Create SubagentConfig → register in registry.py
```

## Key Files Reference

| File | Purpose |
|---|---|
| `backend/packages/harness/deerflow/agents/middlewares/` | All middleware implementations (26 files) |
| `backend/packages/harness/deerflow/agents/factory.py` | `create_deerflow_agent()` with middleware/features/extra_middleware API |
| `backend/packages/harness/deerflow/agents/lead_agent/agent.py` | `build_middlewares()` — chain assembly |
| `backend/packages/harness/deerflow/reflection/__init__.py` | `resolve_variable()` / `resolve_class()` |
| `backend/packages/harness/deerflow/config/` | `AppConfig`, `ToolConfig`, `ModelConfig`, sandbox config |
| `backend/packages/harness/deerflow/sandbox/sandbox.py` | Abstract `Sandbox` interface |
| `backend/packages/harness/deerflow/skills/` | `Skill`, `SkillStorage`, tool policy filtering |
| `backend/packages/harness/deerflow/community/` | Community tool implementations |
| `backend/packages/harness/deerflow/tools/__init__.py` | `get_available_tools()` assembly |
| `backend/packages/harness/deerflow/subagents/` | Subagent registry and executor |
| `backend/app/channels/base.py` | Abstract `Channel` base class |
| `backend/app/gateway/app.py` | `create_app()` — FastAPI app with router registration |
| `config.yaml` | Main config — tools, models, sandbox, providers |
| `extensions_config.json` | MCP servers, skills, interceptors, HTTP middlewares |
