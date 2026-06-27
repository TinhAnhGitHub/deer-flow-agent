# Documentation

This directory contains detailed documentation for the DeerFlow backend.

## Quick Links

| Document | Description |
|----------|-------------|
| [ARCHITECTURE.md](ARCHITECTURE.md) | System architecture overview |
| [API.md](API.md) | Complete API reference |
| [AGENT_CREATION.md](AGENT_CREATION.md) | Custom agent creation, SOUL.md, bootstrap flow |
| [EXTENSION_POINTS.md](EXTENSION_POINTS.md) | All extension mechanisms: middleware, config-driven, providers |
| [AUTH_DESIGN.md](AUTH_DESIGN.md) | User authentication, CSRF, and per-user isolation design |
| [CONFIGURATION.md](CONFIGURATION.md) | Configuration options |
| [SETUP.md](SETUP.md) | Quick setup guide |
| [GUARDRAILS.md](GUARDRAILS.md) | Guardrail middleware and pluggable provider protocol |
| [IM_CHANNEL_CONNECTIONS.md](IM_CHANNEL_CONNECTIONS.md) | User-owned IM channel connections |
| [MCP_SERVER.md](MCP_SERVER.md) | MCP server setup (stdio/SSE/HTTP), OAuth |
| [BLOCKING_IO_DETECTION.md](BLOCKING_IO_DETECTION.md) | Blockbuster runtime gate and async IO offloading |
| [SSO.md](SSO.md) | Single sign-on configuration |
| [TUI.md](TUI.md) | Terminal UI architecture |

## Feature Documentation

| Document | Description |
|----------|-------------|
| [STREAMING.md](STREAMING.md) | Token-level streaming design: Gateway vs DeerFlowClient paths, `stream_mode` semantics, per-id dedup |
| [middleware-execution-flow.md](middleware-execution-flow.md) | Middleware execution order and hook matrix |
| [FILE_UPLOAD.md](FILE_UPLOAD.md) | File upload functionality |
| [PATH_EXAMPLES.md](PATH_EXAMPLES.md) | Path types and usage examples |
| [SANDBOX_MEMORY_PROFILING.md](SANDBOX_MEMORY_PROFILING.md) | Sandbox memory baseline and runtime comparison guide |
| [summarization.md](summarization.md) | Context summarization feature |
| [plan_mode_usage.md](plan_mode_usage.md) | Plan mode with TodoList |
| [AUTO_TITLE_GENERATION.md](AUTO_TITLE_GENERATION.md) | Automatic title generation |
| [TITLE_GENERATION_IMPLEMENTATION.md](TITLE_GENERATION_IMPLEMENTATION.md) | Title implementation details |
| [MEMORY_IMPROVEMENTS.md](MEMORY_IMPROVEMENTS.md) | Memory system design and improvements |
| [MEMORY_SETTINGS_REVIEW.md](MEMORY_SETTINGS_REVIEW.md) | Memory settings review |
| [task_tool_improvements.md](task_tool_improvements.md) | Subagent task tool improvements |

## Development

| Document | Description |
|----------|-------------|
| [TODO.md](TODO.md) | Planned features and known issues |
| [REPLAY_E2E.md](REPLAY_E2E.md) | E2E replay testing framework |
| [rfc-create-deerflow-agent.md](rfc-create-deerflow-agent.md) | `create_deerflow_agent` factory RFC |
| [rfc-extract-shared-modules.md](rfc-extract-shared-modules.md) | Shared module extraction RFC |

## Getting Started

1. **New to DeerFlow?** Start with [SETUP.md](SETUP.md) for quick installation
2. **Configuring the system?** See [CONFIGURATION.md](CONFIGURATION.md)
3. **Understanding the architecture?** Read [ARCHITECTURE.md](ARCHITECTURE.md)
4. **Building integrations?** Check [API.md](API.md) for API reference

## Document Organization

```
docs/
├── README.md                       # This file
├── ARCHITECTURE.md                 # System architecture
├── API.md                          # API reference
├── AGENT_CREATION.md               # Custom agent creation guide
├── EXTENSION_POINTS.md             # Extension mechanisms guide
├── AUTH_DESIGN.md                  # User authentication and isolation design
├── AUTH_UPGRADE.md                 # Auth design upgrade notes
├── AUTH_TEST_PLAN.md               # Auth test plan
├── AUTH_TEST_DOCKER_GAP.md         # Auth Docker gap analysis
├── CONFIGURATION.md                # Configuration guide
├── SETUP.md                        # Setup instructions
├── GUARDRAILS.md                   # Guardrail middleware
├── IM_CHANNEL_CONNECTIONS.md       # IM channel connections
├── MCP_SERVER.md                   # MCP server setup
├── BLOCKING_IO_DETECTION.md        # Blocking IO detection
├── SSO.md                          # SSO configuration
├── TUI.md                          # Terminal UI
├── FILE_UPLOAD.md                  # File upload feature
├── PATH_EXAMPLES.md                # Path usage examples
├── summarization.md                # Summarization feature
├── plan_mode_usage.md              # Plan mode feature
├── STREAMING.md                    # Token-level streaming design
├── middleware-execution-flow.md    # Middleware execution order
├── AUTO_TITLE_GENERATION.md        # Title generation
├── TITLE_GENERATION_IMPLEMENTATION.md  # Title implementation details
├── MEMORY_IMPROVEMENTS.md          # Memory system design
├── MEMORY_IMPROVEMENTS_SUMMARY.md  # Memory improvements summary
├── MEMORY_SETTINGS_REVIEW.md       # Memory settings review
├── SANDBOX_MEMORY_PROFILING.md     # Sandbox memory profiling
├── task_tool_improvements.md       # Task tool improvements
├── TODO.md                         # Roadmap and issues
├── REPLAY_E2E.md                   # E2E replay testing
├── rfc-create-deerflow-agent.md    # Agent factory RFC
├── rfc-extract-shared-modules.md   # Shared modules RFC
├── rfc-grep-glob-tools.md          # Grep/glob tools RFC
└── APPLE_CONTAINER.md              # Apple container notes
```
