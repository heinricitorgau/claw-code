# Claw Code — Rust Implementation

A Rust rewrite of the Claw Code CLI agent harness. For task-oriented usage with copy/paste examples, see [`../USAGE.md`](../USAGE.md).

## Quick start

```bash
# Inspect available commands
cd rust/
cargo run -p rusty-claude-cli -- --help

# Build the workspace
cargo build --workspace

# Run the interactive REPL
cargo run -p rusty-claude-cli -- --model claude-opus-4-6

# One-shot prompt
cargo run -p rusty-claude-cli -- prompt "explain this codebase"

# JSON output for automation
cargo run -p rusty-claude-cli -- --output-format json prompt "summarize src/main.rs"
```

## Configuration

```bash
export ANTHROPIC_API_KEY="sk-ant-..."
# Or use a proxy
export ANTHROPIC_BASE_URL="https://your-proxy.com"
```

OAuth-based auth:

```bash
cargo run -p rusty-claude-cli -- login
```

## Mock parity harness

The workspace includes a deterministic Anthropic-compatible mock service and a clean-environment CLI harness for end-to-end parity verification.

```bash
cd rust/

# Run the scripted clean-environment harness
./scripts/run_mock_parity_harness.sh

# Start the mock service manually for ad hoc CLI runs
cargo run -p mock-anthropic-service -- --bind 127.0.0.1:0
```

**Harness coverage**

| Scenario | Verification target |
|----------|---------------------|
| `streaming_text` | SSE streaming response handling |
| `read_file_roundtrip` | Read path execution + synthesis |
| `grep_chunk_assembly` | Chunked grep output handling |
| `write_file_allowed` | Write success path |
| `write_file_denied` | Permission denial path |
| `multi_tool_turn_roundtrip` | Multi-tool assistant turns |
| `bash_stdout_roundtrip` | Bash execution flow |
| `bash_permission_prompt_approved` | Permission prompt — approved |
| `bash_permission_prompt_denied` | Permission prompt — denied |
| `plugin_tool_roundtrip` | Plugin tool execution path |

Primary artifacts:

- `crates/mock-anthropic-service/` — reusable mock Anthropic-compatible service
- `crates/rusty-claude-cli/tests/mock_parity_harness.rs` — clean-env CLI harness
- `scripts/run_mock_parity_harness.sh` — reproducible wrapper
- `scripts/run_mock_parity_diff.py` — scenario checklist and PARITY mapping runner
- `mock_parity_scenarios.json` — scenario-to-PARITY manifest

## Feature status

| Feature | Status |
|---------|--------|
| Anthropic / OpenAI-compatible provider flows + streaming | complete |
| OAuth login/logout | complete |
| Interactive REPL (rustyline) | complete |
| Tool system (bash, read, write, edit, grep, glob) | complete |
| Web tools (search, fetch) | complete |
| Sub-agent / agent surfaces | complete |
| Todo tracking | complete |
| Notebook editing | complete |
| CLAUDE.md / project memory | complete |
| Config file hierarchy (`.claw.json` + merged config sections) | complete |
| Permission system | complete |
| MCP server lifecycle + inspection | complete |
| Session persistence + resume | complete |
| Cost / usage / stats surfaces | complete |
| Git integration | complete |
| Markdown terminal rendering (ANSI) | complete |
| Model aliases (opus/sonnet/haiku) | complete |
| Direct CLI subcommands (`status`, `sandbox`, `agents`, `mcp`, `skills`, `doctor`) | complete |
| Slash commands (including `/skills`, `/agents`, `/mcp`, `/doctor`, `/plugin`, `/subagent`) | complete |
| Hooks (`/hooks`, config-backed lifecycle hooks) | complete |
| Plugin management surfaces | complete |
| Skills inventory / install surfaces | complete |
| Machine-readable JSON output across core CLI surfaces | complete |

## Model aliases

| Alias | Resolves to |
|-------|-------------|
| `opus` | `claude-opus-4-6` |
| `sonnet` | `claude-sonnet-4-6` |
| `haiku` | `claude-haiku-4-5-20251213` |

## CLI flags and commands

```text
claw [OPTIONS] [COMMAND]

Flags:
  --model MODEL
  --output-format text|json
  --permission-mode MODE
  --dangerously-skip-permissions
  --allowedTools TOOLS
  --resume [SESSION.jsonl|session-id|latest]
  --version, -V

Top-level commands:
  prompt <text>
  help
  version
  status
  sandbox
  dump-manifests
  bootstrap-plan
  agents
  mcp
  skills
  system-prompt
  login
  logout
  init
```

For the canonical live help text: `cargo run -p rusty-claude-cli -- --help`

## Slash commands (REPL)

Tab completion expands slash commands, model aliases, permission modes, and recent session IDs.

- Session / visibility: `/help`, `/status`, `/sandbox`, `/cost`, `/resume`, `/session`, `/version`, `/usage`, `/stats`
- Workspace / git: `/compact`, `/clear`, `/config`, `/memory`, `/init`, `/diff`, `/commit`, `/pr`, `/issue`, `/export`, `/hooks`, `/files`, `/branch`, `/release-notes`, `/add-dir`
- Discovery / debugging: `/mcp`, `/agents`, `/skills`, `/doctor`, `/tasks`, `/context`, `/desktop`, `/ide`
- Automation / analysis: `/review`, `/advisor`, `/insights`, `/security-review`, `/subagent`, `/team`, `/telemetry`, `/providers`, `/cron`, and more
- Plugin management: `/plugin` (with aliases `/plugins`, `/marketplace`)

Notable slash commands: `/skills [list|install <path>|help]`, `/agents [list|help]`, `/mcp [list|show <server>|help]`, `/doctor`, `/plugin [list|install <path>|enable <name>|disable <name>|uninstall <id>|update <id>]`, `/subagent [list|steer <target> <msg>|kill <id>]`.

See [`../USAGE.md`](../USAGE.md) for usage examples.

## Workspace layout

```text
rust/
├── Cargo.toml              # Workspace root
├── Cargo.lock
└── crates/
    ├── api/                # Provider clients + streaming + request preflight
    ├── commands/           # Shared slash-command registry + help rendering
    ├── compat-harness/     # TS manifest extraction harness
    ├── mock-anthropic-service/ # Deterministic local Anthropic-compatible mock
    ├── plugins/            # Plugin metadata, manager, install/enable/disable surfaces
    ├── runtime/            # Session, config, permissions, MCP, prompts, auth/runtime loop
    ├── rusty-claude-cli/   # Main CLI binary (`claw`)
    ├── telemetry/          # Session tracing and usage telemetry types
    └── tools/              # Built-in tools, skill resolution, tool search, agent runtime surfaces
```

| Crate | Responsibilities |
|-------|-----------------|
| `api` | Provider clients, SSE streaming, request/response types, auth (API key + OAuth bearer), request-size/context-window preflight |
| `commands` | Slash command definitions, parsing, help text generation, JSON/text command rendering |
| `compat-harness` | Extracts tool/prompt manifests from upstream TS source |
| `mock-anthropic-service` | Deterministic `/v1/messages` mock for CLI parity tests and local harness runs |
| `plugins` | Plugin metadata, install/enable/disable/update flows, plugin tool definitions, hook integration surfaces |
| `runtime` | `ConversationRuntime`, config loading, session persistence, permission policy, MCP client lifecycle, system prompt assembly, usage tracking |
| `rusty-claude-cli` | REPL, one-shot prompt, direct CLI subcommands, streaming display, tool call rendering, CLI argument parsing |
| `telemetry` | Session trace events and supporting telemetry payloads |
| `tools` | Tool specs + execution: Bash, ReadFile, WriteFile, EditFile, GlobSearch, GrepSearch, WebSearch, WebFetch, Agent, TodoWrite, NotebookEdit, Skill, ToolSearch, and runtime-facing tool discovery |

## Repository stats

| Metric | Value |
|--------|-------|
| Rust LOC | ~20K lines |
| Crates | 9 |
| Binary name | `claw` |
| Default model | `claude-opus-4-6` |
| Default permissions | `danger-full-access` |

## License

See repository root.
