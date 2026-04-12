# Parity Status — claw-code Rust Port

Last updated: 2026-04-03

## Mock parity harness — milestone 1

- [x] Deterministic Anthropic-compatible mock service (`rust/crates/mock-anthropic-service`)
- [x] Reproducible clean-environment CLI harness (`rust/crates/rusty-claude-cli/tests/mock_parity_harness.rs`)
- [x] Scripted scenarios: `streaming_text`, `read_file_roundtrip`, `grep_chunk_assembly`, `write_file_allowed`, `write_file_denied`

## Mock parity harness — milestone 2 (behavioral expansion)

- [x] Scripted multi-tool turn coverage: `multi_tool_turn_roundtrip`
- [x] Scripted bash coverage: `bash_stdout_roundtrip`
- [x] Scripted permission prompt coverage: `bash_permission_prompt_approved`, `bash_permission_prompt_denied`
- [x] Scripted plugin-path coverage: `plugin_tool_roundtrip`
- [x] Behavioral diff/checklist runner: `rust/scripts/run_mock_parity_diff.py`

## Harness v2 behavioral checklist

Canonical scenario map: `rust/mock_parity_scenarios.json`

Verified behavioral categories: multi-tool assistant turns, bash flow roundtrips, permission enforcement across tool paths, plugin tool execution path, file tools harness-validated flows.

## Completed behavioral parity work

Hashes from `git log --oneline`. Merge line counts from `git show --stat <merge>`.

| Lane | Status | Feature commit | Merge commit | Diff stat |
|------|--------|----------------|--------------|-----------|
| Bash validation (9 submodules) | complete | `36dac6c` | — (`jobdori/bash-validation-submodules`) | `1005 insertions` |
| CI fix | complete | `89104eb` | `f1969ce` | `22 insertions, 1 deletion` |
| File-tool edge cases | complete | `284163b` | `a98f2b6` | `195 insertions, 1 deletion` |
| TaskRegistry | complete | `5ea138e` | `21a1e1d` | `336 insertions` |
| Task tool wiring | complete | `e8692e4` | `d994be6` | `79 insertions, 35 deletions` |
| Team + cron runtime | complete | `c486ca6` | `49653fe` | `441 insertions, 37 deletions` |
| MCP lifecycle | complete | `730667f` | `cc0f92e` | `491 insertions, 24 deletions` |
| LSP client | complete | `2d66503` | `d7f0dc6` | `461 insertions, 9 deletions` |
| Permission enforcement | complete | `66283f4` | `336f820` | `357 insertions` |

## Tool Surface: 40/40 (spec parity)

### Real implementations (behavioral parity — varying depth)

| Tool | Rust impl | Behavioral notes |
|------|-----------|-----------------|
| **bash** | `runtime::bash` 283 LOC | Subprocess exec, timeout, background, sandbox — strong parity. 9/9 requested validation submodules tracked as complete via `36dac6c`, with on-main sandbox + permission enforcement runtime support |
| **read_file** | `runtime::file_ops` | Offset/limit read — good parity |
| **write_file** | `runtime::file_ops` | File create/overwrite — good parity |
| **edit_file** | `runtime::file_ops` | Old/new string replacement — good parity. `replace_all` was recently added |
| **glob_search** | `runtime::file_ops` | Glob pattern matching — good parity |
| **grep_search** | `runtime::file_ops` | ripgrep-style search — good parity |
| **WebFetch** | `tools` | URL fetch + content extraction — moderate parity (content truncation and redirect handling vs upstream not fully verified) |
| **WebSearch** | `tools` | Search query execution — moderate parity |
| **TodoWrite** | `tools` | Todo/note persistence — moderate parity |
| **Skill** | `tools` | Skill discovery/install — moderate parity |
| **Agent** | `tools` | Agent delegation — moderate parity |
| **TaskCreate** | `runtime::task_registry` + `tools` | In-memory task creation wired into tool dispatch — good parity |
| **TaskGet** | `runtime::task_registry` + `tools` | Task lookup + metadata payload — good parity |
| **TaskList** | `runtime::task_registry` + `tools` | Registry-backed task listing — good parity |
| **TaskStop** | `runtime::task_registry` + `tools` | Terminal-state stop handling — good parity |
| **TaskUpdate** | `runtime::task_registry` + `tools` | Registry-backed message updates — good parity |
| **TaskOutput** | `runtime::task_registry` + `tools` | Output capture retrieval — good parity |
| **TeamCreate** | `runtime::team_cron_registry` + `tools` | Team lifecycle + task assignment — good parity |
| **TeamDelete** | `runtime::team_cron_registry` + `tools` | Team delete lifecycle — good parity |
| **CronCreate** | `runtime::team_cron_registry` + `tools` | Cron entry creation — good parity |
| **CronDelete** | `runtime::team_cron_registry` + `tools` | Cron entry removal — good parity |
| **CronList** | `runtime::team_cron_registry` + `tools` | Registry-backed cron listing — good parity |
| **LSP** | `runtime::lsp_client` + `tools` | Registry + dispatch for diagnostics, hover, definition, references, completion, symbols, formatting — good parity |
| **ListMcpResources** | `runtime::mcp_tool_bridge` + `tools` | Connected-server resource listing — good parity |
| **ReadMcpResource** | `runtime::mcp_tool_bridge` + `tools` | Connected-server resource reads — good parity |
| **MCP** | `runtime::mcp_tool_bridge` + `tools` | Stateful MCP tool invocation bridge — good parity |
| **ToolSearch** | `tools` | Tool discovery — good parity |
| **NotebookEdit** | `tools` | Jupyter notebook cell editing — moderate parity |
| **Sleep** | `tools` | Delay execution — good parity |
| **SendUserMessage/Brief** | `tools` | User-facing message — good parity |
| **Config** | `tools` | Config inspection — moderate parity |
| **EnterPlanMode** | `tools` | Worktree plan mode toggle — good parity |
| **ExitPlanMode** | `tools` | Worktree plan mode restore — good parity |
| **StructuredOutput** | `tools` | Passthrough JSON — good parity |
| **REPL** | `tools` | Subprocess code execution — moderate parity |
| **PowerShell** | `tools` | Windows PowerShell execution — moderate parity |

### Stubs only (surface parity, no behavioral depth)

| Tool | Status | Notes |
|------|--------|-------|
| **AskUserQuestion** | stub | Needs live user I/O integration |
| **McpAuth** | stub | Needs full auth UX beyond the MCP lifecycle bridge |
| **RemoteTrigger** | stub | Needs HTTP client |
| **TestingPermission** | stub | Test-only, low priority |

## Slash commands: 67/141 upstream entries

- 27 original specs (pre-today) — all with real handlers
- 40 new specs — parse + stub handler ("not yet implemented")
- Remaining ~74 upstream entries are internal modules/dialogs/steps, not user `/commands`

### Behavioral feature checkpoints

**Bash tool — 9/9 requested validation submodules complete:**

- [x] `sedValidation` — validate sed commands before execution
- [x] `pathValidation` — validate file paths in commands
- [x] `readOnlyValidation` — block writes in read-only mode
- [x] `destructiveCommandWarning` — warn on rm -rf, etc.
- [x] `commandSemantics` — classify command intent
- [x] `bashPermissions` — permission gating per command type
- [x] `bashSecurity` — security checks
- [x] `modeValidation` — validate against current permission mode
- [x] `shouldUseSandbox` — sandbox decision logic

Harness note: milestone 2 validates bash success plus workspace-write escalation approve/deny flows; dedicated validation submodules landed in `36dac6c`; on-main runtime carries sandbox + permission enforcement.

**File tools — completed checkpoint:**

- [x] Path traversal prevention (symlink following, `../` escapes)
- [x] Size limits on read/write
- [x] Binary file detection
- [x] Permission mode enforcement (read-only vs workspace-write)

Harness note: read_file, grep_search, write_file allow/deny, and multi-tool same-turn assembly are covered by the mock parity harness; file edge cases + permission enforcement landed in `a98f2b6` and `336f820`.

**Config/Plugin/MCP flows:**

- [x] Full MCP server lifecycle (connect, list tools, call tool, disconnect)
- [ ] Plugin install/enable/disable/uninstall full flow
- [ ] Config merge precedence (user > project > local)

Harness note: external plugin discovery + execution covered via `plugin_tool_roundtrip`; MCP lifecycle landed in `cc0f92e`; plugin lifecycle + config merge precedence remain open.

## Runtime behavioral gaps

| Gap | Status |
|-----|--------|
| Permission enforcement across all tools (read-only, workspace-write, danger-full-access) | complete |
| Output truncation (large stdout/file content) | open |
| Session compaction behavior matching | open |
| Token counting / cost tracking accuracy | open |
| Streaming response support validated by mock parity harness | complete |

Harness note: current coverage includes write-file denial, bash escalation approve/deny, and plugin workspace-write execution paths; permission enforcement landed in `336f820`.

## Migration readiness

| Condition | Status |
|-----------|--------|
| `PARITY.md` maintained and honest | complete |
| No `#[ignore]` tests hiding failures (only 1 allowed: `live_stream_smoke_test`) | open |
| CI green on every commit | open |
| Codebase shape clean for handoff | open |
