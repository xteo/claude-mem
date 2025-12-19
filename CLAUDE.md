/* To @claude: be vigilant about only leaving evergreen context in this file, claude-mem handles working context separately. */

# Claude-Mem: AI Development Instructions

## What This Project Is

Claude-mem is a Claude Code plugin providing persistent memory across sessions. It captures tool usage, compresses observations using the Claude Agent SDK, and injects relevant context into future sessions.

## Architecture Overview

**5 Lifecycle Hooks**: SessionStart → UserPromptSubmit → PostToolUse → Stop → SessionEnd

**Hooks** (`src/hooks/*.ts`) - TypeScript → ESM, built to `plugin/scripts/*-hook.js`. Pure HTTP clients with no native dependencies.

**Worker Service** (`src/services/worker-service.ts`) - Express API on port 37777, Bun-managed. Slim orchestrator (~150 lines) that coordinates domain services.

**Database** (`src/services/sqlite/`) - SQLite3 at `~/.claude-mem/claude-mem.db` with FTS5 full-text search, 7 migrations.

**MCP Server** (`src/servers/mcp-server.ts`) - Thin HTTP wrapper exposing search tools via Model Context Protocol.

**Search Skill** (`plugin/skills/mem-search/`) - Progressive instruction loading with operations in `operations/*.md`.

**Chroma** (`src/services/sync/ChromaSync.ts`) - Vector embeddings for semantic search.

**Viewer UI** (`src/ui/viewer/`) - React interface at http://localhost:37777, built to `plugin/ui/viewer.html`.

## Source Structure

```
src/
├── hooks/                    # Lifecycle hook implementations
│   ├── context-hook.ts       # SessionStart - injects context
│   ├── new-hook.ts           # UserPromptSubmit - captures prompts
│   ├── save-hook.ts          # PostToolUse - sends observations to worker
│   ├── summary-hook.ts       # Stop - generates session summary
│   ├── cleanup-hook.ts       # SessionEnd - cleanup
│   └── shared/               # Shared hook utilities
├── services/
│   ├── worker-service.ts     # Main Express server (orchestrator)
│   ├── sqlite/               # Database layer
│   │   ├── Database.ts       # SQLite singleton with migrations
│   │   ├── SessionStore.ts   # Session/observation storage
│   │   ├── SessionSearch.ts  # FTS5 search implementation
│   │   └── migrations.ts     # Schema migrations (7 total)
│   ├── worker/               # Worker sub-services
│   │   ├── DatabaseManager.ts
│   │   ├── SessionManager.ts
│   │   ├── SDKAgent.ts       # Claude Agent SDK integration
│   │   ├── SearchManager.ts
│   │   ├── SSEBroadcaster.ts
│   │   └── http/routes/      # HTTP route handlers
│   │       ├── SessionRoutes.ts
│   │       ├── SearchRoutes.ts
│   │       ├── DataRoutes.ts
│   │       ├── SettingsRoutes.ts
│   │       └── ViewerRoutes.ts
│   ├── sync/ChromaSync.ts    # Vector DB sync
│   └── context-generator.ts  # Context injection logic
├── servers/mcp-server.ts     # MCP protocol server
├── ui/viewer/                # React UI (28 components)
├── sdk/                      # Agent SDK integration
├── utils/                    # Shared utilities
└── shared/                   # Edge-processing utilities
```

## Database Schema

**Core Tables** (after 7 migrations):

- `sdk_sessions` - Tracks Claude sessions (claude_session_id, sdk_session_id, project, status)
- `observations` - Extracted insights (type: decision|bugfix|feature|refactor|discovery|change)
- `session_summaries` - Structured summaries (request, investigated, learned, completed, next_steps)
- `user_prompts` - User prompt history per session
- `observations_fts` / `session_summaries_fts` - FTS5 virtual tables for full-text search

**Legacy Tables** (migration 001-002):
- `sessions`, `memories`, `overviews`, `diagnostics`, `transcript_events`

## Privacy Tags

**Dual-Tag System** for meta-observation control:
- `<private>content</private>` - User-level privacy control (manual, prevents storage)
- `<claude-mem-context>content</claude-mem-context>` - System-level tag (auto-injected observations, prevents recursive storage)

**Implementation**: Tag stripping happens at hook layer (edge processing) before data reaches worker/database. See `src/utils/tag-stripping.ts` for shared utilities.

## Build Commands

```bash
npm run build-and-sync        # Build, sync to marketplace, restart worker (most common)
npm run build                 # Compile TypeScript only
npm run sync-marketplace      # Copy to ~/.claude/plugins only
npm run worker:restart        # Restart worker service only
npm run worker:status         # Check worker status
npm run worker:logs           # View worker logs
npm run test                  # Run vitest tests
npm run bug-report            # Generate bug report for troubleshooting
```

**Viewer UI**: http://localhost:37777

## Testing

```
tests/
├── happy-paths/              # Core functionality tests
├── error-handling/           # Error scenarios
├── integration/              # Full lifecycle tests
├── security/                 # Security validation (command injection, etc.)
└── services/                 # Service-specific tests
```

Run tests: `npm test` (vitest) or `npm run test:parser` for SDK parser tests.

## Configuration

Settings are managed in `~/.claude-mem/settings.json`. Auto-created with defaults on first run.

**Core Settings:**
- `CLAUDE_MEM_MODEL` - Model for observations/summaries (default: claude-sonnet-4-5)
- `CLAUDE_MEM_CONTEXT_OBSERVATIONS` - Observations injected at SessionStart
- `CLAUDE_MEM_WORKER_PORT` - Worker service port (default: 37777)
- `CLAUDE_MEM_WORKER_HOST` - Worker bind address (default: 127.0.0.1, use 0.0.0.0 for remote access)

**System Configuration:**
- `CLAUDE_MEM_DATA_DIR` - Data directory location (default: ~/.claude-mem)
- `CLAUDE_MEM_LOG_LEVEL` - Log verbosity: DEBUG, INFO, WARN, ERROR, SILENT (default: INFO)

## File Locations

- **Source**: `<project-root>/src/`
- **Built Plugin**: `<project-root>/plugin/`
- **Installed Plugin**: `~/.claude/plugins/marketplaces/thedotmack/`
- **Database**: `~/.claude-mem/claude-mem.db`
- **Chroma**: `~/.claude-mem/chroma/`
- **Logs**: `~/.claude-mem/logs/worker-YYYY-MM-DD.log`

## Requirements

- **Bun** (all platforms - auto-installed if missing)
- **uv** (all platforms - auto-installed if missing, provides Python for Chroma)
- Node.js (build tools only)

## Plugin Structure

```
plugin/
├── .claude-plugin/
│   ├── plugin.json           # Plugin metadata
│   └── .mcp.json             # MCP server registration
├── scripts/                  # Compiled JavaScript
│   ├── *-hook.js             # Lifecycle hooks
│   ├── worker-service.cjs    # Express server (bundled)
│   ├── mcp-server.cjs        # MCP server (bundled)
│   └── smart-install.js      # Installation helper
├── hooks/hooks.json          # Hook lifecycle definitions
├── skills/
│   ├── mem-search/           # Search skill with operations/
│   └── troubleshoot/         # Troubleshooting skill
└── ui/viewer.html            # Built React UI
```

## MCP Tools

The MCP server exposes these tools via `plugin/.mcp.json`:
- `search` - Full-text search with filtering (query, type, concepts, files, project)
- `timeline` - Timeline context with anchor point
- `get_recent_context` - Recent observations
- `get_context_timeline` - Timeline from observation ID
- `help` - Tool help/instructions

## Documentation

**Public Docs**: https://docs.claude-mem.ai (Mintlify)
**Source**: `docs/public/` - MDX files, edit `docs.json` for navigation
**Deploy**: Auto-deploys from GitHub on push to main

**DO NOT** put planning docs or internal notes in `docs/public/`.

## Pro Features Architecture

Claude-mem is designed with a clean separation between open-source core functionality and optional Pro features.

**Open-Source Core** (this repository):
- All worker API endpoints on localhost:37777 remain fully open and accessible
- Pro features are headless - no proprietary UI elements in this codebase
- Pro integration points are minimal: settings for license keys, tunnel provisioning logic
- The architecture ensures Pro features extend rather than replace core functionality

**Pro Features** (coming soon, external):
- Enhanced UI (Memory Stream) connects to the same localhost:37777 endpoints
- Additional features like advanced filtering, timeline scrubbing
- Access gated by license validation, not by modifying core endpoints

## Development Workflow

1. Make changes in `src/`
2. Run `npm run build-and-sync` to build and deploy
3. Worker auto-restarts with new code
4. Test via Claude Code or http://localhost:37777

For hook debugging, check `~/.claude-mem/logs/worker-$(date +%Y-%m-%d).log`

## Important Notes

- No need to edit the changelog - it's generated automatically from GitHub releases
- Version is tracked in `package.json` and `plugin/.claude-plugin/plugin.json` (keep in sync)
- Use `npm run bug-report` when troubleshooting issues
