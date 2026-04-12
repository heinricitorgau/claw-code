# Experimental Checkpoint — claw-code Rust Port

最後更新：2026-04-03

## 這份文件的角色

- 這份最上層的 `PARITY.md` 是 `rust/scripts/run_mock_parity_diff.py` 所消費的 canonical 文件。
- 用途：記錄一個**可檢查的實驗 checkpoint**，而非功能完整性宣告。
- 要求的 9-lane checkpoint：**9 條 lanes 全部已合併到 `main`。**

## 當前狀態快照

| 指標 | 數值 |
|------|------|
| `main` HEAD | `ee31e00`（stub implementations 已被真實的 AskUserQuestion + RemoteTrigger 取代）|
| `main` 上的 commits | 292 |
| 所有 branches 合計 commits | 293 |
| Rust crates | 9 |
| 受追蹤 Rust LOC | 48,599 行 |
| 測試 LOC | 2,568 行 |
| 參與作者 | 3 位 |
| 日期範圍 | 2026-03-31 → 2026-04-03 |
| Mock parity scenarios | 10 個 |
| 捕捉的 `/v1/messages` requests | 19 個 |

## 這個 checkpoint 想回答什麼

- Rust port 目前對齊到哪個程度
- 哪些能力有可驗證證據，哪些只是表面接線
- 哪些地方仍需保持保守，不應過度宣稱 parity

## Mock parity harness — 里程碑 1

- [x] 可決定性的、相容 Anthropic 的 mock service（`rust/crates/mock-anthropic-service`）
- [x] 可重現、乾淨環境的 CLI harness（`rust/crates/rusty-claude-cli/tests/mock_parity_harness.rs`）
- [x] Scripted scenarios：`streaming_text`、`read_file_roundtrip`、`grep_chunk_assembly`、`write_file_allowed`、`write_file_denied`

## Mock parity harness — 里程碑 2（行為擴充）

- [x] Scripted multi-tool turn coverage：`multi_tool_turn_roundtrip`
- [x] Scripted bash coverage：`bash_stdout_roundtrip`
- [x] Scripted permission prompt coverage：`bash_permission_prompt_approved`、`bash_permission_prompt_denied`
- [x] Scripted plugin-path coverage：`plugin_tool_roundtrip`
- [x] Behavioral diff/checklist runner：`rust/scripts/run_mock_parity_diff.py`

## Harness v2 行為檢查清單

Canonical scenario map：`rust/mock_parity_scenarios.json`

已驗證的行為類別：multi-tool assistant turns、bash flow roundtrips、permission enforcement across tool paths、plugin tool execution path、file tools harness-validated flows、streaming response support。

## 9-lane checkpoint（實驗里程碑）

| Lane | 狀態 | Feature commit | Merge commit | 證據 |
|------|------|----------------|--------------|------|
| 1. Bash validation | merged | `36dac6c` | `1cfd78a` | `jobdori/bash-validation-submodules`, `rust/crates/runtime/src/bash_validation.rs`（`+1004`）|
| 2. CI fix | merged | `89104eb` | `f1969ce` | `rust/crates/runtime/src/sandbox.rs`（`+22/-1`）|
| 3. File-tool | merged | `284163b` | `a98f2b6` | `rust/crates/runtime/src/file_ops.rs`（`+195/-1`）|
| 4. TaskRegistry | merged | `5ea138e` | `21a1e1d` | `rust/crates/runtime/src/task_registry.rs`（`+336`）|
| 5. Task wiring | merged | `e8692e4` | `d994be6` | `rust/crates/tools/src/lib.rs`（`+79/-35`）|
| 6. Team+Cron | merged | `c486ca6` | `49653fe` | `rust/crates/runtime/src/team_cron_registry.rs`, `rust/crates/tools/src/lib.rs`（`+441/-37`）|
| 7. MCP lifecycle | merged | `730667f` | `cc0f92e` | `rust/crates/runtime/src/mcp_tool_bridge.rs`, `rust/crates/tools/src/lib.rs`（`+491/-24`）|
| 8. LSP client | merged | `2d66503` | `d7f0dc6` | `rust/crates/runtime/src/lsp_client.rs`, `rust/crates/tools/src/lib.rs`（`+461/-9`）|
| 9. Permission enforcement | merged | `66283f4` | `336f820` | `rust/crates/runtime/src/permission_enforcer.rs`, `rust/crates/tools/src/lib.rs`（`+357`）|

## Lane 細節與目前觀察

### Lane 1 — Bash validation

- **狀態：** 已合併到 `main`。
- **Feature commit：** `36dac6c` — `feat: add bash validation submodules — readOnlyValidation, destructiveCommandWarning, modeValidation, sedValidation, pathValidation, commandSemantics`
- **證據：** branch-only diff 新增 `rust/crates/runtime/src/bash_validation.rs` 與一個 `runtime::lib` export（2 個檔案共 `+1005`）。
- **`main` branch 的實際情況：** `rust/crates/runtime/src/bash.rs` 仍是 `main` 上啟用中的實作，長度 **283 LOC**，含 timeout/background/sandbox execution。`PermissionEnforcer::check_bash()` 在 `main` 上加入了 read-only gating，但專用 validation module 尚未落地。

**Bash tool — upstream 有 18 個 submodules，Rust 目前有 1 個。** Harness coverage 已證明 bash execution 與 prompt escalation flows，但尚未覆蓋完整 upstream validation matrix。

### Lane 2 — CI fix

- **狀態：** 已合併到 `main`。
- **Feature commit：** `89104eb` — `fix(sandbox): probe unshare capability instead of binary existence`
- **Merge commit：** `f1969ce`
- **證據：** `rust/crates/runtime/src/sandbox.rs` 長度為 **385 LOC**，現在根據真實 `unshare` capability 與 container signals 判斷 sandbox support，而不是只因為 binary 存在就假設支援。
- **重要性：** `.github/workflows/rust-ci.yml` 執行 `cargo fmt --all --check` 與 `cargo test -p rusty-claude-cli`；這條 lane 消除了 runtime behavior 中只在 CI 出現的 sandbox 假設。

### Lane 3 — File-tool

- **狀態：** 已合併到 `main`。
- **Feature commit：** `284163b` — `feat(file_ops): add edge-case guards — binary detection, size limits, workspace boundary, symlink escape`
- **Merge commit：** `a98f2b6`
- **證據：** `rust/crates/runtime/src/file_ops.rs` 長度為 **744 LOC**，含 `MAX_READ_SIZE`、`MAX_WRITE_SIZE`、NUL-byte binary detection，以及 canonical workspace-boundary validation。
- **Harness coverage：** `read_file_roundtrip`、`grep_chunk_assembly`、`write_file_allowed` 與 `write_file_denied` 均由 clean-env harness 實際執行。

### Lane 4 — TaskRegistry

- **狀態：** 已合併到 `main`。
- **Feature commit：** `5ea138e` — `feat(runtime): add TaskRegistry — in-memory task lifecycle management`
- **Merge commit：** `21a1e1d`
- **證據：** `rust/crates/runtime/src/task_registry.rs` 長度為 **335 LOC**，提供 thread-safe in-memory registry，具備 `create`、`get`、`list`、`stop`、`update`、`output`、`append_output`、`set_status` 與 `assign_team`。
- **範圍限制：** 以真實 runtime-backed task records 取代固定 payload 的 stub state，但尚未加入 external subprocess execution。

### Lane 5 — Task wiring

- **狀態：** 已合併到 `main`。
- **Feature commit：** `e8692e4` — `feat(tools): wire TaskRegistry into task tool dispatch`
- **Merge commit：** `d994be6`
- **證據：** `rust/crates/tools/src/lib.rs` 透過 `execute_tool()` 與具體的 `run_task_*` handlers，分派 `TaskCreate`、`TaskGet`、`TaskList`、`TaskStop`、`TaskUpdate` 與 `TaskOutput`。task tools 現在透過 `global_task_registry()` 在 `main` 上暴露真實的 registry state。

### Lane 6 — Team+Cron

- **狀態：** 已合併到 `main`。
- **Feature commit：** `c486ca6` — `feat(runtime+tools): TeamRegistry and CronRegistry — replace team/cron stubs`
- **Merge commit：** `49653fe`
- **證據：** `rust/crates/runtime/src/team_cron_registry.rs` 長度為 **363 LOC**，新增 thread-safe 的 `TeamRegistry` 與 `CronRegistry`；`rust/crates/tools/src/lib.rs` 將 `TeamCreate`、`TeamDelete`、`CronCreate`、`CronDelete` 與 `CronList` 接到這些 registries。
- **範圍限制：** team/cron tools 具備 in-memory lifecycle behavior；尚未進到真實 background scheduler 或 worker fleet 層級。

### Lane 7 — MCP lifecycle

- **狀態：** 已合併到 `main`。
- **Feature commit：** `730667f` — `feat(runtime+tools): McpToolRegistry — MCP lifecycle bridge for tool surface`
- **Merge commit：** `cc0f92e`
- **證據：** `rust/crates/runtime/src/mcp_tool_bridge.rs` 長度為 **406 LOC**，可追蹤 server connection status、resource listing、resource reads、tool listing、tool dispatch acknowledgements、auth state 與 disconnects。`rust/crates/tools/src/lib.rs` 將 `ListMcpResources`、`ReadMcpResource`、`McpAuth` 與 `MCP` 路由到 `global_mcp_registry()` handlers。
- **範圍限制：** 以 registry bridge 取代了純 stub responses；end-to-end MCP connection population 與更完整的 transport/runtime 深度，仍取決於更完整的 MCP runtime（`mcp_stdio.rs`、`mcp_client.rs`、`mcp.rs`）。

### Lane 8 — LSP client

- **狀態：** 已合併到 `main`。
- **Feature commit：** `2d66503` — `feat(runtime+tools): LspRegistry — LSP client dispatch for tool surface`
- **Merge commit：** `d7f0dc6`
- **證據：** `rust/crates/runtime/src/lsp_client.rs` 長度為 **438 LOC**，可在 stateful registry 中建模 diagnostics、hover、definition、references、completion、symbols 與 formatting。`rust/crates/tools/src/lib.rs` 暴露的 `LSP` tool schema 已列出 `symbols`、`references`、`diagnostics`、`definition` 與 `hover`，並將 requests 透過 `registry.dispatch(action, path, line, character, query)` 路由。
- **範圍限制：** 目前 parity 停留在 registry/dispatch 層級；completion/format 存在於 registry model 中，但尚未清楚地暴露在 tool schema 邊界；真實外部 language-server process orchestration 仍是另一條獨立工作。

### Lane 9 — Permission enforcement

- **狀態：** 已合併到 `main`。
- **Feature commit：** `66283f4` — `feat(runtime+tools): PermissionEnforcer — permission mode enforcement layer`
- **Merge commit：** `336f820`
- **證據：** `rust/crates/runtime/src/permission_enforcer.rs` 長度為 **340 LOC**，在 `rust/crates/runtime/src/permissions.rs` 之上新增 tool gating、file write boundary checks，以及 bash read-only heuristics。`rust/crates/tools/src/lib.rs` 已暴露 `enforce_permission_check()`，並在 tool specs 中攜帶每個 tool 的 `required_permission`。

跨 tool paths 的 permission enforcement：Harness scenarios 已驗證 `write_file_denied`、`bash_permission_prompt_approved` 與 `bash_permission_prompt_denied`。`PermissionEnforcer::check()` 委派給 `PermissionPolicy::authorize()`，回傳結構化的 allow/deny 結果。

## Tool Surface：`main` 上暴露 40 個 tool specs

- `mvp_tool_specs()` 在 `rust/crates/tools/src/lib.rs` 中暴露 **40** 個 tool specs。
- 核心 execution 已存在於 `bash`、`read_file`、`write_file`、`edit_file`、`glob_search` 與 `grep_search`。
- `mvp_tool_specs()` 中的 product tools 包含 `WebFetch`、`WebSearch`、`TodoWrite`、`Skill`、`Agent`、`ToolSearch`、`NotebookEdit`、`Sleep`、`SendUserMessage`、`Config`、`EnterPlanMode`、`ExitPlanMode`、`StructuredOutput`、`REPL` 與 `PowerShell`。
- 這次 9-lane 推進，以 registry-backed handlers 取代 `Task*`、`Team*`、`Cron*`、`LSP` 與 MCP tools 原本的 fixed-payload stubs。
- `Brief` 在 `execute_tool()` 中作為 execution alias 處理，不是 `mvp_tool_specs()` 中獨立暴露的 tool spec。

### 仍然受限或刻意保持淺層的部分

| Tool / 能力 | 目前狀態 |
|------------|---------|
| `AskUserQuestion` | 仍只回傳 pending response payload，非真實 interactive UI wiring |
| `RemoteTrigger` | stub response |
| `TestingPermission` | 僅用於測試 |
| Task/team/cron/MCP/LSP | 已不再是 fixed-payload stubs，但仍是 registry-backed approximations，非完整 external-runtime integrations |
| Bash 深層 validation | 在 `36dac6c` 合併前仍只存在 branch 中 |

## 與舊版 PARITY 檢查清單的對齊結果

- [x] Path traversal prevention（含 symlink following 與 `../` escapes）
- [x] Size limits on read/write
- [x] Binary file detection
- [x] Permission mode enforcement（read-only vs workspace-write）
- [x] Config merge precedence（user > project > local）— `ConfigLoader::discover()` 按 user → project → local 載入；`loads_and_merges_claude_code_config_files_by_precedence()` 已驗證 merge order
- [x] Plugin install/enable/disable/uninstall flow — `rust/crates/commands/src/lib.rs` 的 `/plugin` slash handling 委派到 `rust/crates/plugins/src/lib.rs` 的 `PluginManager::{install, enable, disable, uninstall}`
- [x] 沒有 `#[ignore]` tests 把 failures 藏起來 — 對 `rust/**/*.rs` 做 `grep` 後找到 0 個 ignored tests

## 仍待完成的實驗缺口

| 缺口 | 狀態 |
|------|------|
| End-to-end MCP runtime lifecycle（超越目前 registry bridge）| 待完成 |
| Output truncation（large stdout/file content）| 已完成 |
| Session compaction behavior matching | 待完成 |
| Token counting / cost tracking accuracy | 待完成 |
| Bash validation lane 合併到 `main` | 已完成 |
| 每個 commit 的 CI 都是綠的 | 待完成 |

## Migration Readiness

| 條件 | 狀態 |
|------|------|
| `PARITY.md` 維護中且內容誠實 | 完成 |
| 9 條要求的 lanes 都有 commit hashes 與 current status 文件化 | 完成 |
| 9 條要求的 lanes 都已落地到 `main` | 完成 |
| 沒有 `#[ignore]` tests 藏 failure | 完成 |
| 每個 commit 的 CI 都是綠的 | 待完成 |
| Codebase shape 足夠乾淨，可交付 handoff documentation | 完成 |
