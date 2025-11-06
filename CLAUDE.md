# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Discourse MCP** is a Model Context Protocol (MCP) server that exposes Discourse forum capabilities as tools for AI agents. It's published as `@discourse/mcp` with a binary named `discourse-mcp`.

- **Language**: TypeScript (strict mode), compiled to ES2020 with NodeNext modules
- **Runtime**: Node >= 18
- **Package Manager**: pnpm (10.14.0)
- **SDK**: `@modelcontextprotocol/sdk` for MCP server implementation
- **Validation**: Zod for runtime schema validation

## Development Commands

```bash
# Install dependencies
pnpm install

# Type checking (no emit)
pnpm typecheck

# Build TypeScript to dist/
pnpm build

# Run tests (requires build first)
pnpm test

# Run locally with source maps (requires build first)
pnpm dev

# Clean build artifacts
pnpm clean

# Sync test fixtures
pnpm sync:fixtures

# Release management
pnpm release          # Standard version bump
pnpm release:dry      # Dry run
pnpm release:alpha    # Pre-release alpha
pnpm release:beta     # Pre-release beta
```

**Testing**: Uses Node's built-in test runner (`node --test`) against compiled artifacts in `dist/test/**/*.js`. Always run `pnpm build` before `pnpm test` since tests run against compiled code.

## Architecture

### Entry Point & Initialization

**src/index.ts** is the main entry point:

1. **CLI Parsing**: Custom argument parser supporting both `--flag value` and `--flag=value` formats, with kebab-case to snake_case conversion (e.g., `--read-only` → `read_only`)
2. **Configuration**: Merges profile JSON (if provided via `--profile`) with CLI flags (flags take precedence). Validated via Zod `ProfileSchema`.
3. **Server Setup**: Creates MCP server with `@discourse/mcp` name, registers tools based on configuration
4. **Transport**: Supports two modes:
   - `stdio` (default): Standard input/output for typical MCP usage
   - `http`: Streamable HTTP transport for stateless mode with health check (`/health`) and MCP endpoint (`/mcp`)
5. **Site Selection**: If `--site` provided, validates and pre-selects the site; otherwise site selection happens via `discourse_select_site` tool

### Core Patterns

#### SiteState (src/site/state.ts)

Central state manager for dynamic site selection:

- **Per-site HTTP clients**: Cached by normalized base URL
- **Auth resolution**: Matches site to `auth_pairs` configuration, prefers `user_api_key` over `api_key`
- **Lazy initialization**: Clients created on-demand when site is selected
- **Validation**: Calls `/about.json` to validate site before selection

#### HttpClient (src/http/client.ts)

Resilient HTTP client with multiple features:

- **Auth modes**:
  - `none`: No authentication
  - `api_key`: Admin API Key with `Api-Key` and `Api-Username` headers
  - `user_api_key`: User API Key with `User-Api-Key` and `User-Api-Client-Id` headers
- **Retry logic**: 3 attempts with exponential backoff for 429/5xx errors
- **Caching**: In-memory GET cache with TTL (used for `getCached()`)
- **Timeouts**: AbortController-based timeout handling
- **Error logging**: Enhanced diagnostics for network failures (DNS, SSL/TLS, timeouts)

#### Tool Registry (src/tools/registry.ts)

Two types of tools:

1. **Built-in tools** (`src/tools/builtin/*`):
   - Read tools: Always registered (search, read_topic, read_post, list_categories, list_tags, get_user, list_user_posts, filter_topics)
   - Write tools: Only registered when `allowWrites` is true (create_post, create_topic, create_category, create_user)
   - select_site: Hidden when `--site` is provided (tethered mode)

2. **Remote tools** (`src/tools/remote/tool_exec_api.ts`):
   - Discovered via GET `/ai/tools` when `tools_mode` is `auto` or `tool_exec_api`
   - Dynamically registered with JSON Schema → Zod conversion
   - Called via POST `/ai/tools/{name}/call`
   - Names sanitized to MCP-safe format with `remote_` prefix

### Key Files

- **src/index.ts**: CLI entry point, server initialization, transport setup
- **src/site/state.ts**: SiteState class for site selection and client management
- **src/http/client.ts**: HttpClient with auth, retries, caching
- **src/http/cache.ts**: In-memory caching implementation
- **src/tools/registry.ts**: Tool registration orchestration
- **src/tools/builtin/**: Individual built-in tool implementations
- **src/tools/remote/tool_exec_api.ts**: Remote tool discovery and registration
- **src/tools/types.ts**: Shared tool types
- **src/util/logger.ts**: Logging with levels (silent, error, info, debug)
- **src/util/redact.ts**: Secret redaction for logs
- **src/user-api-key-generator.ts**: Interactive User API Key generation

## Authentication

Two authentication modes with different use cases:

1. **Admin API Keys** (`api_key` + `api_username`):
   - Require admin/moderator permissions
   - Full access including user/category creation
   - Headers: `Api-Key`, `Api-Username`

2. **User API Keys** (`user_api_key` + optional `user_api_client_id`):
   - Can be generated by any user (no admin required)
   - User-specific permissions and rate limits
   - Auto-expire after 180 days of inactivity
   - Headers: `User-Api-Key`, `User-Api-Client-Id`
   - Preferred when both are provided for same site

**Important**: User API Key takes precedence over Admin API Key in `auth_pairs` for the same site.

## Write Safety

Write operations are protected by multiple gates:

1. Must have `--allow_writes` flag
2. Must NOT have `--read_only=true` (default is true)
3. Must have at least one entry in `--auth_pairs`
4. Write tools enforce ~1 req/sec rate limit

Only when ALL conditions are met will write tools be registered.

## Configuration

Supports both CLI flags and profile JSON:

- **Flag formats**: `--flag value`, `--flag=value`, `--kebab-case` → `snake_case`
- **Profile loading**: `--profile path/to/profile.json` loads base config
- **Override precedence**: CLI flags > profile values > defaults
- **Validation**: All config validated via Zod schemas

Key configuration options:
- `site`: Tether to single site (hides select_site tool)
- `auth_pairs`: Array of per-site auth configs
- `tools_mode`: Control remote tool discovery (`auto`, `discourse_api_only`, `tool_exec_api`)
- `default_search`: Prefix added to every search query
- `max_read_length`: Character limit for post content (default 50000)
- `log_level`: Logging verbosity (`silent`, `error`, `info`, `debug`)

## Cursor Rules

The `.cursor/rules/ai-agents-always.mdc` file indicates that `AGENTS.md` should always be included in context when working with AI agents. This file contains detailed guidance on:

- Tool behavior and output formats
- Remote Tool Execution API integration
- Query language syntax for `discourse_filter_topics`
- CLI configuration details
- Networking resilience features

## Testing Strategy

- Tests live in `src/test/`
- Run against compiled code in `dist/test/`
- Use Node's built-in test runner
- Must rebuild before testing if source changes
- Fixtures can be synced with `pnpm sync:fixtures`

## Development Tips

- **Always rebuild**: After source changes, run `pnpm build` before testing or running locally
- **Debug logging**: Use `--log_level debug` to see HTTP requests, responses, and detailed errors
- **Tethered mode**: Use `--site` flag to skip site selection step during development
- **Profile files**: Keep secrets in profile JSON instead of command line for security
- **Remote tools**: Set `--tools_mode=discourse_api_only` to disable remote tool discovery during testing
- **Transport modes**: Use `--transport http --port 3000` for stateless HTTP mode instead of stdio

## Publishing

- Package published as `@discourse/mcp`
- Binary name: `discourse-mcp`
- Preferred usage: `npx -y @discourse/mcp@latest`
- Version management: standard-version (semver)
- Build required before publish: `prepublishOnly` script runs `pnpm build`
