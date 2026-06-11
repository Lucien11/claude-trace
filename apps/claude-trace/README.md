# claude-trace

Record all your interactions with Claude Code as you develop your projects. See everything Claude hides: system prompts, tool outputs, and raw API data in an intuitive web interface.

> **Fork notice.** This is a fork of [`@mariozechner/claude-trace`](https://github.com/mariozechner/claude-trace) by Mario Zechner. It adds two things on top of the original:
>
> - **Native-binary support** — works with Claude Code's native (Bun-compiled) installer, not just the Node.js install. Native binaries are detected automatically and traced through a local reverse proxy (originally contributed by Richard-Weiss).
> - **Configurable upstream** — the proxy forwards to whatever `ANTHROPIC_BASE_URL` you use (a relay, gateway, or self-hosted endpoint), instead of always going to `api.anthropic.com`. When unset, it falls back to the official API, so existing setups are unaffected.
>
> See [New in this fork](#new-in-this-fork) for details. All original functionality is unchanged.

## Install

```bash
npm install -g @lucien_lc/claude-trace
```

## Usage

```bash
# Start Claude Code with logging
claude-trace

# Include all API requests (by default, only substantial conversations are logged)
claude-trace --all   # or the long form: claude-trace --include-all-requests

# Run Claude with specific arguments
claude-trace --run-with chat --model sonnet-3.5

# Show help
claude-trace --help

# Extract OAuth token
claude-trace --extract-token

# Generate HTML report manually from previously logged .jsonl
claude-trace --generate-html logs.jsonl report.html

# Generate HTML including all requests (not just substantial conversations)
claude-trace --generate-html logs.jsonl --include-all-requests

# Generate conversation summaries and searchable index
claude-trace --index
```

Logs are saved to `.claude-trace/log-YYYY-MM-DD-HH-MM-SS.{jsonl,html}` in your current directory. The HTML file is self-contained and opens in any browser without needing a server.

## Request Filtering

By default, claude-trace filters logs to focus on substantial conversations:

- **Default behavior**: Only logs requests to `/v1/messages` with more than 2 messages in the conversation
- **With `--include-all-requests`**: Logs all requests made to `api.anthropic.com` including single-message requests and other endpoints

This filtering reduces log file size and focuses on meaningful development sessions, while still allowing you to capture everything when needed for debugging.

## Conversation Indexing

Generate AI-powered summaries of your coding sessions:

```bash
claude-trace --index
```

This feature:

- Scans all `.jsonl` log files in `.claude-trace/` directory
- Filters meaningful conversations (more than 2 messages, non-compacted)
- Uses Claude CLI to generate titles and summaries for each conversation
- Creates `summary-YYYY-MM-DD-HH-MM-SS.json` files with conversation metadata
- Generates a master `index.html` with chronological listing of all sessions
- Links directly to individual conversation HTML files

**Note:** Indexing will incur additional API token usage as it calls Claude to summarize conversations.

## What you'll see

- **System prompts** - The hidden instructions Claude receives
- **Tool definitions** - Available tool descriptions and parameters
- **Tool outputs** - Raw data from file reads, searches, API calls
- **Thinking blocks** - Claude's internal reasoning process
- **Token usage** - Detailed breakdown including cache hits
- **Raw JSONL logs** - Complete request/response pairs for analysis
- **Interactive HTML viewer** - Browse conversations with model filtering
- **Debug views** - Raw calls shows all HTTP requests without filtering; JSON debug shows processed API data
- **Conversation indexing** - AI-generated summaries and searchable index of all sessions

## New in this fork

These capabilities are additions on top of the upstream `@mariozechner/claude-trace`.

### Native-binary support

Recent Claude Code releases ship as a self-contained native binary (a Bun-compiled
bundle) rather than a Node.js script. The original `--require` interceptor only works
with the Node.js build, so tracing silently failed on native installs.

This fork detects native binaries by their magic bytes (Mach-O, ELF, PE) and, when one
is found, traces it through a local HTTP reverse proxy instead of the interceptor. No
configuration is needed — it is automatic:

```bash
claude-trace            # works whether claude is a Node.js script or a native binary
```

### Configurable upstream (follows `ANTHROPIC_BASE_URL`)

The reverse proxy no longer hard-codes `https://api.anthropic.com`. It forwards to the
base URL you actually use, resolved in this order:

1. `ANTHROPIC_BASE_URL` from the environment
2. `env.ANTHROPIC_BASE_URL` in `~/.claude/settings.json`
3. `https://api.anthropic.com` (default — unchanged behavior when nothing is set)

This makes claude-trace work when your traffic goes through a relay, gateway, or
self-hosted endpoint:

```bash
# Trace traffic that goes through a local gateway
ANTHROPIC_BASE_URL=http://127.0.0.1:8080 claude-trace
```

Because `settings.json` takes precedence over inherited environment variables in Claude
Code, the proxy also overlays its own address via `--settings` when launching Claude —
overriding only `ANTHROPIC_BASE_URL` while preserving every other setting (model
mappings, tokens, etc.). The result is the link `claude → proxy → your upstream`, with
the full conversation captured along the way.

Supported upstream protocols: `http` and `https` (with automatic default ports 80/443).

## Requirements

- Node.js 16+
- Claude Code CLI installed

## Development

### Running in dev mode

```bash
# Install dependencies
npm install

# Start dev mode
npm run dev
```

Dev mode compiles both the main app (`src/`) and frontend (`frontend/src/`) with file watching. For frontend development, open `http://localhost:8080/test` to see live updates as you modify frontend code.

### Testing the CLI

```bash
# Test compiled version
node --no-deprecation dist/cli.js

# Test TypeScript source directly
npx tsx --no-deprecation src/cli.ts
```

### Building

```bash
# Build everything
npm run build

# Or build specific parts
npm run build:backend  # CLI and interceptor
npm run build:frontend # Web interface
```

**Generated artifacts:**

- `dist/` - Compiled CLI and interceptor JavaScript
- `frontend/dist/` - Bundled web interface (CSS + JS)
- Built HTML generator that embeds the web interface

The built artifacts are ready for npm publishing and include:

- Self-contained HTML reports with embedded CSS/JS
- Node.js CLI with mitmproxy integration
- TypeScript definitions

### Architecture

**Two-part system:**

1. **Backend** (`src/`)

   - **CLI** (`cli.ts`) - Command-line interface and argument parsing. Launches Claude Code and injects interceptors
   - **Interceptor** (`interceptor.ts`) - injects itself into Claude Code, intercepts calls to fetch(), and logs them to JSONL files in .claude-trace/ in the current working dir.
   - **HTML Generator** (`html-generator.ts`) - Embeds frontend into self-contained HTML reports
   - **Index Generator** (`index-generator.ts`) - Creates AI-powered conversation summaries and searchable index
   - **Shared Conversation Processor** (`shared-conversation-processor.ts`) - Core conversation processing logic shared between frontend and backend
   - **Token Extractor** (`token-extractor.js`) - A simpler interceptor that extracts Claude Code OAuth tokens

2. **Frontend** (`frontend/src/`)
   - **`app.ts`** - Main ClaudeApp component, handles data processing and view switching
   - **`index.ts`** - Application entry point, injects CSS and initializes app
   - **`types/claude-data.ts`** - TypeScript interfaces for API data structures
   - **`utils/data.ts`** - Processes raw HTTP pairs, reconstructs SSE messages
   - **`utils/markdown.ts`** - Markdown to HTML conversion utilities
   - **`components/simple-conversation-view.ts`** - Main conversation display with tool visualization
   - **`components/raw-pairs-view.ts`** - Raw HTTP traffic viewer
   - **`components/json-view.ts`** - JSON debug data viewer
   - **`styles.css`** - Tailwind CSS with VS Code theme variables
