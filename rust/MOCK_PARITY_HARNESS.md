# Mock LLM parity harness

This milestone adds a deterministic Anthropic-compatible mock service and a reproducible CLI harness for the Rust `claw` binary.

## Artifacts

| Artifact | Description |
|----------|-------------|
| `crates/mock-anthropic-service/` | Mock `/v1/messages` service |
| `crates/rusty-claude-cli/tests/mock_parity_harness.rs` | End-to-end clean-environment harness |
| `scripts/run_mock_parity_harness.sh` | Convenience wrapper |

## Scenarios

The harness runs these scripted scenarios against a fresh workspace and isolated environment variables:

| # | Scenario | Verification target |
|---|----------|---------------------|
| 1 | `streaming_text` | SSE streaming response handling |
| 2 | `read_file_roundtrip` | Read path execution + synthesis |
| 3 | `grep_chunk_assembly` | Chunked grep output handling |
| 4 | `write_file_allowed` | Write success path |
| 5 | `write_file_denied` | Permission denial path |
| 6 | `multi_tool_turn_roundtrip` | Multi-tool assistant turns |
| 7 | `bash_stdout_roundtrip` | Bash execution flow |
| 8 | `bash_permission_prompt_approved` | Permission prompt — approved |
| 9 | `bash_permission_prompt_denied` | Permission prompt — denied |
| 10 | `plugin_tool_roundtrip` | Plugin tool execution path |

## Run

```bash
cd rust/
./scripts/run_mock_parity_harness.sh
```

Behavioral checklist / parity diff:

```bash
cd rust/
python3 scripts/run_mock_parity_diff.py
```

Scenario-to-PARITY mappings: `mock_parity_scenarios.json`.

## Manual mock server

```bash
cd rust/
cargo run -p mock-anthropic-service -- --bind 127.0.0.1:0
```

The server prints `MOCK_ANTHROPIC_BASE_URL=...`; point `ANTHROPIC_BASE_URL` at that URL and use any non-empty `ANTHROPIC_API_KEY`.
