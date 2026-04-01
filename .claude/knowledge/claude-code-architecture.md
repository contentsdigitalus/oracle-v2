# Claude Code Architecture - Knowledge Reference

> Source: Analysis of `ceokornten/claude-code` repository (snapshot from source map exposure, March 31, 2026)
> Analyzed: 2026-04-01

---

## Overview

Claude Code CLI เป็น AI coding assistant ที่ทำงานใน terminal สร้างด้วย **TypeScript + Bun** ขนาด ~1,900 files, 512,000+ lines

## Tech Stack

| Component | Technology |
|---|---|
| Runtime | Bun |
| Language | TypeScript (strict mode) |
| Terminal UI | React + Ink |
| CLI Parser | Commander.js |
| Schema Validation | Zod v4 |
| API | Anthropic SDK |
| Code Search | ripgrep |
| Feature Flags | GrowthBook (compile-time via Bun bundler) |
| Telemetry | OpenTelemetry + gRPC |
| Analytics | Statsig |
| Auth | OAuth 2.0, JWT, macOS Keychain |
| Protocols | MCP SDK, LSP |

## Core Architecture

### Directory Structure (`src/`)

```
src/
├── main.tsx              # CLI entrypoint (~4,684 lines), Commander.js parser
├── QueryEngine.ts        # LLM interaction core (~46K lines)
├── Tool.ts               # Base tool definitions (~29K lines)
├── commands.ts           # Command registration (~25K lines)
├── tools.ts              # Tool registry
├── context.ts            # Context gathering (CLAUDE.md, git, memory)
├── cost-tracker.ts       # Token/USD budget enforcement
│
├── tools/                # 43 tool implementations
├── commands/             # ~100+ slash commands
├── components/           # ~140 React+Ink UI components
├── hooks/                # ~85 React hooks
├── services/             # API, MCP, OAuth, LSP, analytics
├── bridge/               # IDE integration (VS Code, JetBrains)
├── coordinator/          # Multi-agent orchestration
├── skills/               # Bundled skills system (17 skills)
├── plugins/              # Plugin architecture
├── state/                # App state management
├── memdir/               # Memory system (8 files)
├── buddy/                # Companion/pet feature
├── voice/                # Voice mode
├── vim/                  # Vim keybinding mode
├── remote/               # Remote session management
├── constants/            # 21 config files
├── schemas/              # Schema definitions
├── types/                # Type definitions
├── utils/                # Utilities
└── ... (35 directories total)
```

### Key Files

| File | Size | Purpose |
|---|---|---|
| `QueryEngine.ts` | ~46K lines | API calls, streaming, tool loops, retries, token counting |
| `Tool.ts` | ~29K lines | Tool type definitions, permission model, input schemas |
| `commands.ts` | ~25K lines | Command registration, lazy loading, env-specific filtering |
| `main.tsx` | ~4.7K lines | CLI entrypoint with parallel startup optimization |

## Tool System (43 Tools)

| Group | Tools |
|---|---|
| **File** | FileReadTool, FileEditTool, FileWriteTool, NotebookEditTool |
| **Search** | GlobTool, GrepTool (ripgrep-based) |
| **Execute** | BashTool, PowerShellTool, REPLTool |
| **Web** | WebFetchTool, WebSearchTool |
| **Agent** | AgentTool (sub-agent spawning), SkillTool, ToolSearchTool |
| **Task** | TaskCreate/Get/Update/List/Stop/Output, TeamCreate/Delete, TodoWriteTool |
| **MCP** | MCPTool, ListMcpResourcesTool, ReadMcpResourceTool, McpAuthTool |
| **Planning** | EnterPlanModeTool, ExitPlanModeTool, EnterWorktreeTool, ExitWorktreeTool |
| **Other** | SleepTool, BriefTool, ConfigTool, AskUserQuestionTool, SendMessageTool, RemoteTriggerTool, ScheduleCronTool, LSPTool |

Tools are built via `buildTool()` factory with: `isEnabled`, `isConcurrencySafe`, `isReadOnly`, permission checking.

**Simple mode** (`CLAUDE_CODE_SIMPLE` env var) restricts to: Bash, FileRead, FileEdit only.

## Command System (~100+ Commands)

| Category | Commands |
|---|---|
| **Git** | `/commit`, `/review`, `/diff`, `/branch` |
| **Config** | `/config`, `/theme`, `/keybindings`, `/model` |
| **Context** | `/memory`, `/skills`, `/tasks`, `/plan` |
| **Session** | `/vim`, `/voice`, `/cost`, `/compact`, `/resume`, `/share` |
| **Extension** | `/mcp`, `/plugin`, `/hooks` |
| **Auth** | `/login`, `/logout`, `/oauth-refresh`, `/doctor` |

Commands filtered by: auth type (claude-ai vs console), execution context (remote-safe, bridge-safe).

## Bundled Skills (17)

`batch`, `claudeApi`, `claudeApiContent`, `claudeInChrome`, `debug`, `keybindings`, `loop`, `loremIpsum`, `remember`, `scheduleRemoteAgents`, `simplify`, `skillify`, `stuck`, `updateConfig`, `verify`, `verifyContent`

## MCP (Model Context Protocol)

### Configuration Hierarchy
```
enterprise (highest) > local > project > user > plugins
```

### Supported Transports
- **stdio** - Local process spawning (default)
- **sse** - Server-Sent Events for remote servers
- **http** - Streamable HTTP
- **ws** - WebSocket
- **sdk** - In-process for VS Code extension
- **claudeai-proxy** - Routes through Claude.ai

Features: env var expansion (`${VAR}`), policy controls (allowlists/denylists), OAuth token refresh, 60s timeouts, auto-reconnection.

## Permission Model

- **Modes**: `default`, `bypass`, `auto`
- **Rules**: `alwaysAllow`, `alwaysDeny`, `alwaysAsk`
- Deeply immutable `ToolPermissionContext`
- `bypass` only allowed in sandboxed/Docker environments

## System Prompt Architecture

- **Static cacheable sections** + **dynamic per-turn sections**
- `systemPromptSection()` creates memoized sections (cached until `/clear` or `/compact`)
- `DANGEROUS_uncachedSystemPromptSection()` for volatile sections recomputed every turn
- Sections: cyber risk, tool usage, agent behavior, skill discovery, memory/context, language prefs, output style, MCP instructions, token budget

## Feature Flags (Compile-time Dead Code Elimination)

Bun bundler eliminates dead code at build time:

| Flag | Purpose |
|---|---|
| `BUDDY` | Companion pet feature |
| `PROACTIVE` | Proactive assistant mode |
| `KAIROS` | In-process team spawning |
| `COORDINATOR_MODE` | Multi-agent coordination |
| `VOICE_MODE` | Voice input support |
| `DAEMON` | Background daemon mode |
| `DIRECT_CONNECT` | Direct API connection |
| `SSH_REMOTE` | SSH remote sessions |
| `LODESTONE` | Unknown feature |
| `TRANSCRIPT_CLASSIFIER` | Auto-mode state management |

## Buddy/Companion System

Hidden feature-gated feature (`BUDDY` flag):

- **18 species**: duck, goose, blob, cat, dragon, octopus, owl, penguin, turtle, snail, ghost, axolotl, capybara, cactus, robot, rabbit, mushroom, chonk
- **5 rarities**: common (weight:60), uncommon (25), rare (10), epic (4), legendary (1)
- **5 stats**: DEBUGGING, PATIENCE, CHAOS, WISDOM, SNARK
- **6 eye styles**: `·` `✦` `×` `◉` `@` `°`
- **8 hats**: none, crown, tophat, propeller, halo, wizard, beanie, tinyduck
- **1% shiny chance**
- Generated deterministically from userId via seeded PRNG (Mulberry32)
- Bones (visual) = computed from hash, never persisted
- Soul (name, personality) = LLM-generated, stored in config
- 3-frame ASCII animation at 500ms tick rate
- Speech bubble with 10s display + fade
- Pet interaction with heart animation (2.5s burst)

## Design Patterns

1. **Tool-based Architecture** - Each capability is a self-contained tool with schema
2. **Parallel Prefetching** - Startup loads keychain, MDM, model capabilities concurrently
3. **Lazy Loading** - Heavy modules loaded on demand
4. **Multi-Agent Coordination** - Sub-agents with worktree isolation
5. **Bridge Pattern** - IDE integration via JWT-authenticated Unix Domain Socket IPC
6. **Memoized System Prompts** - Static sections cached, dynamic sections recomputed per turn
7. **Feature Flag Elimination** - Compile-time code removal via Bun bundler

## Startup Optimization

1. **Parallel prefetch** (non-blocking): keychain, MDM reads, model capabilities
2. **Blocking prefetch**: trust dialog, auth validation, settings
3. **Deferred prefetch** (after first render): user context, git info, tips, cloud credentials, change detection

## Bridge System (IDE Integration)

- 31 files implementing bidirectional communication
- Supports VS Code and JetBrains
- JWT-authenticated messaging
- Session management and REPL sessions
- Unix Domain Socket IPC
- Trusted device management

## Memory System (`memdir/`)

8 files: `findRelevantMemories.ts`, `memoryScan.ts`, `memoryAge.ts`, `memoryTypes.ts`, `paths.ts`, `teamMemPaths.ts`, `teamMemPrompts.ts`

## Context Gathering

- `getSystemContext()` - git status, branch info, recent commits
- `getUserContext()` - CLAUDE.md files via `getClaudeMds()`, memory files, current date
- Git status truncated at 2,000 chars
- `--bare` mode skips auto-discovery

---

*Last updated: 2026-04-01*
