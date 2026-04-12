# Claw Code Usage

This guide covers the Rust workspace under `rust/` and the `claw` CLI binary. Run `/doctor` as the first command after build to verify your setup.

## Preflight check

```bash
cd rust
cargo build --workspace
./target/debug/claw
# first command inside the REPL
/doctor
```

`/doctor` is the built-in setup and preflight diagnostic. With a saved session: `./target/debug/claw --resume latest /doctor`.

## Prerequisites

- Rust toolchain with `cargo`
- One of: `ANTHROPIC_API_KEY` (direct API access) or `claw login` (OAuth-based auth)
- Optional: `ANTHROPIC_BASE_URL` when targeting a proxy or local service

## Build

```bash
cd rust
cargo build --workspace
```

The CLI binary is at `rust/target/debug/claw` after a debug build.

## Invocation modes

### Interactive REPL

```bash
cd rust
./target/debug/claw
```

### One-shot prompt

```bash
cd rust
./target/debug/claw prompt "summarize this repository"
```

### Shorthand prompt

```bash
cd rust
./target/debug/claw "explain rust/crates/runtime/src/lib.rs"
```

### JSON output

```bash
cd rust
./target/debug/claw --output-format json prompt "status"
```

## Model and permission controls

```bash
cd rust
./target/debug/claw --model sonnet prompt "review this diff"
./target/debug/claw --permission-mode read-only prompt "summarize Cargo.toml"
./target/debug/claw --permission-mode workspace-write prompt "update README.md"
./target/debug/claw --allowedTools read,glob "inspect the runtime crate"
```

**Permission modes**

| Mode | Effect |
|------|--------|
| `read-only` | File reads permitted; writes denied |
| `workspace-write` | Writes within workspace boundary permitted |
| `danger-full-access` | No restrictions |

**Model aliases**

| Alias | Resolves to |
|-------|-------------|
| `opus` | `claude-opus-4-6` |
| `sonnet` | `claude-sonnet-4-6` |
| `haiku` | `claude-haiku-4-5-20251213` |

## Authentication

### API key

```bash
export ANTHROPIC_API_KEY="sk-ant-..."
```

### OAuth

```bash
cd rust
./target/debug/claw login
./target/debug/claw logout
```

## Local and third-party model endpoints

`claw` supports Anthropic-compatible and OpenAI-compatible endpoints via environment variables.

**Rationale**: separating endpoint selection (via `*_BASE_URL`) from auth selection (via `*_API_KEY`) allows the same binary to target local dev servers, proxies, or alternative providers without recompilation.

### Anthropic-compatible endpoint

```bash
export ANTHROPIC_BASE_URL="http://127.0.0.1:8080"
export ANTHROPIC_AUTH_TOKEN="local-dev-token"

cd rust
./target/debug/claw --model "claude-sonnet-4-6" prompt "reply with the word ready"
```

### OpenAI-compatible endpoint

```bash
export OPENAI_BASE_URL="http://127.0.0.1:8000/v1"
export OPENAI_API_KEY="local-dev-token"

cd rust
./target/debug/claw --model "qwen2.5-coder" prompt "reply with the word ready"
```

### Ollama

```bash
export OPENAI_BASE_URL="http://127.0.0.1:11434/v1"
unset OPENAI_API_KEY

cd rust
./target/debug/claw --model "llama3.2" prompt "summarize this repository in one sentence"
```

### OpenRouter

```bash
export OPENAI_BASE_URL="https://openrouter.ai/api/v1"
export OPENAI_API_KEY="sk-or-v1-..."

cd rust
./target/debug/claw --model "openai/gpt-4.1-mini" prompt "summarize this repository in one sentence"
```

## Provider matrix

| Provider | Protocol | Auth env var(s) | Base URL env var | Default base URL |
|----------|----------|-----------------|------------------|------------------|
| Anthropic (direct) | Anthropic Messages API | `ANTHROPIC_API_KEY` or `ANTHROPIC_AUTH_TOKEN` or OAuth | `ANTHROPIC_BASE_URL` | `https://api.anthropic.com` |
| xAI | OpenAI-compatible | `XAI_API_KEY` | `XAI_BASE_URL` | `https://api.x.ai/v1` |
| OpenAI-compatible | OpenAI Chat Completions | `OPENAI_API_KEY` | `OPENAI_BASE_URL` | `https://api.openai.com/v1` |

The OpenAI-compatible backend serves as the gateway for OpenRouter, Ollama, and any service using the OpenAI `/v1/chat/completions` wire format.

## Tested model aliases

| Alias | Resolved model name | Provider | Max output tokens | Context window |
|-------|---------------------|----------|-------------------|----------------|
| `opus` | `claude-opus-4-6` | Anthropic | 32 000 | 200 000 |
| `sonnet` | `claude-sonnet-4-6` | Anthropic | 64 000 | 200 000 |
| `haiku` | `claude-haiku-4-5-20251213` | Anthropic | 64 000 | 200 000 |
| `grok` / `grok-3` | `grok-3` | xAI | 64 000 | 131 072 |
| `grok-mini` / `grok-3-mini` | `grok-3-mini` | xAI | 64 000 | 131 072 |
| `grok-2` | `grok-2` | xAI | — | — |

Unrecognized model names are passed through verbatim (for OpenRouter slugs, Ollama tags, or full Anthropic model IDs).

## User-defined aliases

```json
{
  "aliases": {
    "fast": "claude-haiku-4-5-20251213",
    "smart": "claude-opus-4-6",
    "cheap": "grok-3-mini"
  }
}
```

Settings files: `~/.claw/settings.json`, `.claw/settings.json`, or `.claw/settings.local.json`. Local project settings override user-level settings.

## Provider detection order

1. Model name starts with `claude` → Anthropic.
2. Model name starts with `grok` → xAI.
3. Otherwise: checks `ANTHROPIC_API_KEY`/`ANTHROPIC_AUTH_TOKEN`, then `OPENAI_API_KEY`, then `XAI_API_KEY`.
4. If nothing matches: defaults to Anthropic.

## Terminology note: "codex" in this project

The term "codex" in this codebase does not refer to OpenAI Codex. References are:

- **`oh-my-codex` (OmX)**: workflow and plugin layer above `claw`; provides planning modes, parallel multi-agent execution, notification routing, and other automation features. See [PHILOSOPHY.md](./PHILOSOPHY.md) and the [oh-my-codex repo](https://github.com/Yeachan-Heo/oh-my-codex).
- **`.codex/` directories** (e.g. `.codex/skills`, `.codex/agents`, `.codex/commands`): legacy lookup paths still scanned alongside `.claw/` directories.
- **`CODEX_HOME`**: optional environment variable for a custom root for user-level skill and command lookups.

`claw` does not support OpenAI Codex sessions, the Codex CLI, or Codex session import/export. To use OpenAI models (e.g. GPT-4.1), configure the OpenAI-compatible provider as shown above.

## HTTP proxy

`claw` respects `HTTP_PROXY`, `HTTPS_PROXY`, and `NO_PROXY` (both upper- and lower-case) when issuing outbound requests.

```bash
export HTTPS_PROXY="http://proxy.corp.example:3128"
export HTTP_PROXY="http://proxy.corp.example:3128"
export NO_PROXY="localhost,127.0.0.1,.corp.example"
```

Alternatively, use `proxy_url` in `ProxyConfig` as a single catch-all proxy for both HTTP and HTTPS traffic:

```rust
use api::{build_http_client_with, ProxyConfig};

let config = ProxyConfig::from_proxy_url("http://proxy.corp.example:3128");
let client = build_http_client_with(&config).expect("proxy client");
```

When `proxy_url` is set it takes precedence over separate `http_proxy` and `https_proxy` fields. If a proxy URL cannot be parsed, `claw` falls back to a direct (no-proxy) client.

## Operational commands

```bash
cd rust
./target/debug/claw status
./target/debug/claw sandbox
./target/debug/claw agents
./target/debug/claw mcp
./target/debug/claw skills
./target/debug/claw system-prompt --cwd .. --date 2026-04-04
```

## Session management

REPL turns are persisted under `.claw/sessions/` in the current workspace.

```bash
cd rust
./target/debug/claw --resume latest
./target/debug/claw --resume latest /status /diff
```

Useful REPL commands: `/help`, `/status`, `/cost`, `/config`, `/session`, `/model`, `/permissions`, `/export`.

## Config file resolution order

| Priority | Path |
|----------|------|
| 1 (lowest) | `~/.claw.json` |
| 2 | `~/.config/claw/settings.json` |
| 3 | `<repo>/.claw.json` |
| 4 | `<repo>/.claw/settings.json` |
| 5 (highest) | `<repo>/.claw/settings.local.json` |

## Mock parity harness

```bash
cd rust
./scripts/run_mock_parity_harness.sh
```

Manual mock service startup:

```bash
cd rust
cargo run -p mock-anthropic-service -- --bind 127.0.0.1:0
```

The server prints `MOCK_ANTHROPIC_BASE_URL=...`; point `ANTHROPIC_BASE_URL` at that URL.

## Verification

```bash
cd rust
cargo test --workspace
```

## Workspace crates

| Crate | Role |
|-------|------|
| `api` | Provider clients, SSE streaming, auth |
| `commands` | Slash command metadata, parsing, help rendering |
| `compat-harness` | TS manifest extraction harness |
| `mock-anthropic-service` | Deterministic Anthropic-compatible mock |
| `plugins` | Plugin metadata, install/enable/disable surfaces |
| `runtime` | Session, config, permissions, MCP, auth/runtime loop |
| `rusty-claude-cli` | Main CLI binary (`claw`) |
| `telemetry` | Session tracing and usage telemetry types |
| `tools` | Built-in tool specs and execution |
