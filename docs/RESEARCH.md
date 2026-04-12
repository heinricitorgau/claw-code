# Claw Code 研究報告

**最後更新：** 2026-04-09
**作者：** Heinnrici
**版本：** v1.0

## 摘要

本報告記錄 **Claw Code** 專案的核心設計決策與當前觀察。Claw Code 是一個以 Rust 重寫的 Claude Code，研究目標是探索「clawable」（機器優先）的 coding harness 設計，以及在 AI 輔助開發環境中，透過多代理協調系統能將人工介入壓縮到何種程度。

**核心研究假設**：當 coding intelligence 成本下降，真正的稀缺資源將從程式碼產出量轉移到方向、判斷與架構清晰度。這個假設目前仍是定性觀察，尚缺系統性量化驗證。

## 1. 研究背景與動機

### 1.1 問題陳述

傳統 AI 輔助開發工具（含原始 Claude Code）的設計前提是：主要使用者是坐在 terminal 前的人類開發者。這帶來以下觀測到的限制：

- session 啟動依賴人工操作（trust prompts、terminal 互動）
- 系統狀態散落在 tmux、git、tests 等多層之間，難以機器讀取
- 錯誤分類依賴文字解析（scraping prose），而非結構化 events
- 復原流程需要人工介入（restart worker、重新注入 prompt）
- 設計假設人類在場盯著 terminal，而非讓 agents 自主操作

### 1.2 研究目標

設計並實作一個「clawable」的 coding harness，使其滿足以下條件：

| 條件 | 描述 |
|------|------|
| 可決定性啟動 | 可被機器以可重現方式初始化 |
| 狀態可機器讀取 | failure modes 可被機器正確分類 |
| 不需要人類盯著 terminal 也能恢復 | recovery 流程可自動化 |
| Event-first | 以 typed events 驅動，不以 log scraping 為先 |
| Policy 可執行 | merge、retry、rebase 由機器完成，不只是聊天指令 |

## 2. 系統架構

### 2.1 三層協調架構

```
Discord（人類介面）
        │
        ▼
   OmX（指令轉工作流）
        │
        ▼
  clawhip（事件路由）
        │
        ▼
   OmO（多代理協調）
        │
        ▼
  claw workers（執行層）
        │
        ▼
  git / tests / infra
```

| 層次 | 系統 | 設計選擇的 rationale |
|------|------|---------------------|
| 工作流層 | OmX（oh-my-codex）| 把模糊指令轉成可執行的工作協定，讓實驗變得可重複；消除每次都要重新描述細節的摩擦 |
| 事件路由層 | clawhip | 把 monitoring 與 delivery 留在 coding agent 的 context window 之外；agents 只專注在實作，不需要格式化 status updates 或路由 notifications |
| 多代理協調層 | OmO（oh-my-openagent）| 讓分歧、handoff、review 與 convergence 成為可設計的實驗單位，而不只是「多開幾個 agent」 |

### 2.2 人機介面選擇

**設計決策**：真正的人類介面是 Discord，不是 terminal。

**Rationale**：如果人類可以用手機打一段話然後離開，而 agents 仍能讀取指令、拆解成任務、分配角色、寫程式、跑測試、針對失敗修復、測試通過後 push，那麼被驗證的就不只是工具，而是整個工作流是否具備「低同步成本」。這從「人類 micromanage 每個 terminal 動作」轉移到「人類設定方向、做判斷」，是一個可測試的假設，而非預設結論。

## 3. Rust Port：技術實作

### 3.1 整體規模

| 指標 | 數值 |
|------|------|
| `main` 上的 commits | 292 |
| Rust crates 數量 | 9 |
| 受追蹤 Rust LOC | 48,599 行 |
| 測試 LOC | 2,568 行 |
| 參與作者 | 3 位 |
| 開發時間跨度 | 2026-03-31 → 2026-04-03（約 3 天）|
| Mock parity scenarios | 10 個 |
| 捕捉的 API requests | 19 個 `/v1/messages` requests |

### 3.2 九條功能 Lanes

全部 9 條 lanes 已合併到 `main`：

| Lane | 功能 | 規模 | 核心模組 |
|------|------|------|---------|
| 1 | Bash validation | +1004 LOC | `bash_validation.rs` — readOnlyValidation、destructiveCommandWarning、modeValidation、sedValidation、pathValidation、commandSemantics |
| 2 | CI fix | +22/-1 | `sandbox.rs` — 用真實 capability probe 取代 binary 存在假設 |
| 3 | File-tool | 744 LOC | `file_ops.rs` — binary detection、size limits、workspace boundary、symlink escape |
| 4 | TaskRegistry | 335 LOC | `task_registry.rs` — thread-safe in-memory task lifecycle（create/get/list/stop/update/output）|
| 5 | Task wiring | 79/-35 | `tools/lib.rs` — TaskRegistry 接到 6 個 task tools |
| 6 | Team+Cron | 363 LOC | `team_cron_registry.rs` — TeamRegistry + CronRegistry |
| 7 | MCP lifecycle | 406 LOC | `mcp_tool_bridge.rs` — MCP lifecycle bridge（connection status、resource listing、tool dispatch）|
| 8 | LSP client | 438 LOC | `lsp_client.rs` — diagnostics、hover、definition、references、completion、symbols |
| 9 | Permission enforcement | 340 LOC | `permission_enforcer.rs` — tool gating、file write boundary、bash read-only heuristics |

### 3.3 Mock Parity Harness

Mock parity harness 的設計原則：可決定性（deterministic）的 scripted scenarios、乾淨環境（clean-env）的 test runner、與 Anthropic API 相容的 mock service。

| Scenario | 驗證內容 |
|----------|---------|
| `streaming_text` | streaming response handling |
| `read_file_roundtrip` | read path execution + synthesis |
| `grep_chunk_assembly` | chunked grep output handling |
| `write_file_allowed` | write success path |
| `write_file_denied` | permission denial path |
| `multi_tool_turn_roundtrip` | multi-tool assistant turns |
| `bash_stdout_roundtrip` | bash execution flow |
| `bash_permission_prompt_approved` | permission prompt handling（approved）|
| `bash_permission_prompt_denied` | permission prompt handling（denied）|
| `plugin_tool_roundtrip` | plugin tool execution path |

**Rationale**：在沒有真實 API key 的情況下驗證行為；可決定性地重現特定 scenarios；防止行為 regression 進入 main。

## 4. 「Clawable」設計原則

### 4.1 七個設計原則

| 原則 | 說明 | Rationale |
|------|------|-----------|
| State machine first | 每個 worker 有明確 lifecycle states（spawning → trust_required → ready_for_prompt → running → blocked → finished/failed）| 隱性狀態使 recovery 不可靠 |
| Events over scraped prose | Channel output 由 typed events 推導，不靠文字解析 | 文字解析在格式變動時脆弱且不可測試 |
| Recovery before escalation | 已知 failure modes 先自動修復，再決定是否求助 | 降低人工介入頻率，使 agent 可真正自主運行 |
| Branch freshness before blame | 先檢查 stale branch，再把紅測試當成新 regression | 避免浪費 recovery 資源在錯誤分類的 failures 上 |
| Partial success is first-class | MCP startup 可以部分成功、部分失敗，需有 degraded-mode reporting | 全有全無的語義使 partial-success 狀態隱形 |
| Terminal is transport, not truth | tmux/TUI 是實作細節，orchestration state 活在更高一層 | 狀態若只存在 terminal 輸出中，就無法被機器查詢 |
| Policy is executable | merge、retry、rebase、stale cleanup 由機器執行，不只是聊天指令 | 只存在聊天中的 policy 不是真正的系統約束 |

### 4.2 Failure Taxonomy

明確分類所有可能的 failure，使自動化 retry policy 可依 failure type 分流：

| Failure type | Recovery 方向 |
|-------------|--------------|
| `prompt_delivery` | 重新送 prompt |
| `trust_gate` | auto-trust 或 escalate |
| `branch_divergence` | rebase/stale cleanup |
| `compile` | 修編譯錯誤 |
| `test` | 修測試 |
| `plugin_startup` | 重試 plugin 初始化 |
| `mcp_startup` | 重試 MCP 連線 |
| `mcp_handshake` | 協議層修復 |
| `gateway_routing` | 路由修復 |
| `tool_runtime` | tool 執行層修復 |
| `infra` | 基礎設施 escalate |

## 5. 觀察與當前發現

### 5.1 瓶頸假說

當 agent systems 可以在幾小時內重建一個 codebase，一個待驗證的觀察是：稀缺資源可能已從打字速度轉移到 architectural clarity、task decomposition、judgment、taste、對什麼值得建構的 conviction，以及知道哪些部分可並行、哪些必須受限。

**當前限制**：這個觀察主要是定性的，且主要基於 Claw Code 這個單一案例。普遍性尚待在不同專案規模與類型下驗證。

### 5.2 機器可讀性作為一等設計屬性

傳統 coding 工具設計給人類閱讀（log 文字、terminal output）。在 multi-agent 系統中，機器可讀性需要與人類可讀性同等對待。關鍵設計決策：typed events 取代 log scraping、structured state machines 取代隱性狀態、explicit failure taxonomy 取代「看 log 猜原因」。

### 5.3 人機介面的調整

當系統設計成讓人類透過 Discord 給方向、agents 自主完成工作，觀察到的效果包括：人類可以在任何設備工作、agents 的 context window 不被 UI 狀態管理佔用。**待驗證**：這個模式是否在更高複雜度或更長時間跨度的任務下仍然穩定。

### 5.4 Parity Testing 的必要性

在重寫專案中，行為一致性比程式碼一致性更重要。Mock parity harness 的價值：在沒有真實 API key 的情況下驗證行為、可決定性地重現特定 scenarios、防止行為 regression 進入 main。

### 5.5 並行 Lane 開發的觀察

9 個 feature lanes 在約 3 天內完成（2026-03-31 → 2026-04-03），達到 48,599 行 Rust LOC。這是在 typed interfaces between layers 的前提下實現的——lanes 互相不阻塞。**開放問題**：這個速度是否可維持，還是前期 typed interface 設計已預先吸收了後期的協調成本？

## 6. 當前限制與未解問題

### 6.1 已知技術限制

| 限制 | 說明 |
|------|------|
| Bash validation 不完整 | Upstream 有 18 個 submodules，Rust 目前只有 1 個 |
| Team/Cron 未接到真實 scheduler | 目前只有 in-memory registry，未連接真實 background scheduler |
| MCP end-to-end 連線 | Registry bridge 已到位，但完整 transport/runtime 深度仍待完成 |
| LSP completion/format | Registry model 存在，但 external language-server process orchestration 仍是獨立工作 |

### 6.2 架構未解問題

- **Session 啟動穩定性**：trust prompts 可能阻塞 TUI 啟動；「session exists」不等於「session is ready」；shell 誤投遞仍可能發生。
- **事件系統完整性**：agents 目前仍必須從 noisy text 中推斷部分狀態；關鍵狀態尚未全部正規化為 machine-readable events。
- **Recovery loop 的自動化程度**：部分 recovery 仍需人工介入，特別是 infra-level failures 的自動分類仍不足。

## 7. 路線圖摘要

| Phase | 研究問題 | 主要工作 |
|-------|---------|---------|
| Phase 1 | Worker 是否 ready、blocked 或 quiet？| Readiness handshake、trust resolution flow、structured session control API |
| Phase 2 | 能否用結構化事件而非 noisy logs 描述 lane 狀態？| Canonical lane event schema、failure taxonomy、summary compression |
| Phase 3 | 常見失敗能否先自動處理再人工介入？| Stale-branch detection、automated recovery、green-ness contract |

## 8. 結論

Claw Code 展示了一個待研究的設計方向：從「人類使用 AI 工具」轉向「AI agents 使用人類方向」。在這個模型中，程式碼是產物，系統可觀測性是設計屬性，人類的核心貢獻是 conviction（相信什麼值得建構）、direction（往哪裡走）、judgment（哪些決策不該自動化）。

這套設計方向目前在 Claw Code 這個案例上呈現出初步的可行性，但其普遍性與限制條件仍需要在更多情境下持續驗證。

## 開放問題

- 「低同步成本」的 Discord 介面假設，是否在任務涉及高度模糊性或快速反饋需求時仍然成立？
- Failure taxonomy 的 11 種分類是否足夠，還是存在常見的 failure 模式無法被當前分類準確捕捉？
- 三層協調架構（OmX / clawhip / OmO）是否是必要的最小分解，還是存在更簡單的等效設計？
- 9 條 lanes 在約 3 天內完成：這個速度的可持續性與品質下限如何確定？

## 附錄

### A. 關鍵檔案索引

| 檔案 | 說明 |
|------|------|
| `PHILOSOPHY.md` | 系統哲學與設計立場 |
| `ROADMAP.md` | 詳細路線圖（Phase 1-5）|
| `PARITY.md` | 9-lane checkpoint 狀態與 mock parity harness |
| `USAGE.md` | Rust workspace 使用說明 |
| `docs/container.md` | Container-first 工作流（Docker/Podman）|
| `rust/crates/runtime/` | 核心 runtime（sandbox、file_ops、task_registry、lsp_client 等）|
| `rust/crates/tools/` | Tool dispatch layer |
| `rust/crates/rusty-claude-cli/` | CLI binary 與 mock parity harness |

### B. 相關外部資源

- [oh-my-codex (OmX)](https://github.com/Yeachan-Heo/oh-my-codex) — workflow layer
- [clawhip](https://github.com/Yeachan-Heo/clawhip) — event/notification router
- [oh-my-openagent (OmO)](https://github.com/code-yeongyu/oh-my-openagent) — multi-agent coordination
- [設計哲學延伸說明](https://x.com/realsigridjin/status/2039472968624185713)

*本文件基於 claw-code repository 截至 2026-04-09 的狀態。*
