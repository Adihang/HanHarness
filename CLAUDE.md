# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

# Project Instructions

- This is **HanPlanet CLI** — a fork of [OpenHarness](https://github.com/HKUDS/OpenHarness) customized for the Hanplanet service.
- Use OpenHarness tools deliberately.
- Keep changes minimal and verify with tests when possible.

# Commands

```bash
# Install (development)
uv sync --extra dev
cd frontend/terminal && npm ci

# Run
uv run hanplanet                 # Interactive TUI (preferred entry point)
uv run oh -p "prompt"            # Non-interactive

# Test
uv run pytest -q                 # All tests
uv run pytest tests/test_engine/ # Single module
uv run pytest -k "test_name"     # Single test
uv run pytest --cov=src/openharness

# Lint / type-check
uv run ruff check src tests scripts
uv run mypy src/openharness       # Not enforced in CI but useful

# Frontend typecheck
cd frontend/terminal && npx tsc --noEmit

# Standalone packaging (for distribution without Python/Node)
uv sync --extra dev --extra standalone
uv run python scripts/build_standalone.py --clean   # macOS/Linux
uv run python scripts/package_hanplanet_cli_hdd.py  # HDD bundle
```

CLI entry points: `hanplanet`, `oh`, `openh` → `openharness.cli:app`; `ohmo` → `ohmo.cli:app`.

# Architecture

**HanPlanet CLI / OpenHarness** is a Python agent harness — infrastructure wrapping an LLM with tool execution, permissions, hooks, and multi-agent coordination.

## Request Flow

```
User Input → CLI / React TUI
  → QueryEngine (engine/query_engine.py)
  → query loop (engine/query.py) — streaming tool-call cycle
  → PermissionChecker (permissions/checker.py)
  → Hooks: PreToolUse → execute → PostToolUse
  → Tool execution (tools/)
  → Results feed back into the loop
```

## Key Subsystems

| Subsystem | Path | Role |
|-----------|------|------|
| Engine | `src/openharness/engine/` | `QueryEngine` owns conversation history; `query.py` is the core streaming tool-call loop |
| Tools | `src/openharness/tools/` | 44+ tools (file I/O, shell, web, search, MCP, agents, tasks, scheduling). All inherit `BaseTool`, registered in `ToolRegistry` |
| Permissions | `src/openharness/permissions/` | Three modes: DEFAULT (ask), AUTO (allow), PLAN (block writes). Checks paths, denied commands, tool allow/block lists; hardcoded `SENSITIVE_PATH_PATTERNS` in `checker.py` |
| Hooks | `src/openharness/hooks/` | PreToolUse/PostToolUse lifecycle events; supports hot-reload. Four handler types: shell command, HTTP, prompt, agent |
| Skills | `src/openharness/skills/` | On-demand `.md` knowledge injected into system prompt at runtime |
| Plugins | `src/openharness/plugins/` | Loadable extensions; user dir at `~/.openharness/plugins/`, project dir at `.openharness/plugins/` |
| MCP | `src/openharness/mcp/` | Model Context Protocol — wraps external MCP servers as native tools |
| Commands | `src/openharness/commands/registry.py` | 54+ slash commands (large single file) |
| Config | `src/openharness/config/settings.py` | Pydantic settings schema; runtime config at `~/.openharness/settings.json` |
| UI | `src/openharness/ui/` | React+Ink TUI (primary) + Textual fallback; WebSocket protocol in `protocol.py` |
| Swarm | `src/openharness/swarm/` + `coordinator/` | Multi-agent spawning and team management |
| Auth | `src/openharness/auth/` | Multi-provider OAuth and credential flows |
| Memory | `src/openharness/memory/` | Persistent agent memory across sessions |
| Sandbox | `src/openharness/sandbox/` | OS-level process isolation for tool execution |
| ohmo | `ohmo/` | Separate personal-agent CLI with gateway and channel (Slack, Telegram, Discord, Lark) support |

## API Clients

Provider abstraction lives in `src/openharness/api/`. All clients implement `SupportsStreamingMessages`. Supported backends: Claude (Anthropic), OpenAI, Ollama, Kimi, GLM, MiniMax, GitHub Copilot, any OpenAI-compatible endpoint.

## Configuration

Resolution order: CLI args → env vars → config file → defaults. Key env vars: `ANTHROPIC_API_KEY`, `ANTHROPIC_BASE_URL`, `ANTHROPIC_MODEL`.

## Adding a Tool

1. Create a file in `src/openharness/tools/` inheriting `BaseTool`
2. Define a Pydantic input model
3. Implement `async def execute(self, arguments, context) -> ToolResult`
4. Register in `src/openharness/tools/__init__.py`
