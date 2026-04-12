# Experimental Roadmap — Claw Code

## 這份路線圖的定位

這不是產品發售路線圖，也不是功能清單。它是一份實驗計畫，描述 claw-code 接下來要驗證哪些問題、補強哪些觀測能力，以及哪些假設還沒有被證明。

核心方向：**讓 agent-mediated development 從「能跑」變成「可觀測、可評估、可恢復」。**

## 四個實驗主軸

| 主軸 | 研究問題 |
|------|---------|
| 可控性 | 系統是否能在不依賴人工盯場的前提下穩定運作？ |
| 可觀測性 | 失敗、阻塞、進度與狀態是否能被機器與人類正確理解？ |
| 可恢復性 | 常見 failure modes 是否能被自動修復，而不是只會報錯？ |
| 可對齊性 | Rust port 是否逐步接近可驗證的行為對齊，而不是停留在表面功能存在？ |

## 現在已知的問題

### 1. 啟動仍然太脆弱

trust prompts 可能阻塞啟動；prompt 可能送錯地方；session 存在不等於 worker 已可用。

### 2. 狀態仍然太分散

tmux、clawhip event stream、git/worktree 狀態、test 狀態、plugin/MCP runtime 狀態各自獨立，缺乏統一可查詢的視圖。

### 3. 太多判斷仍依賴人類推理

失敗分類不夠結構化；event 太像 log，而不是 state；recovery 還沒有系統性收斂。

### 4. 對齊驗證還不夠完整

有些能力只在 registry 層接上；有些行為只是 stub replacement，不是真正 end-to-end parity；有些 success 還沒有被納入可重現的 checkpoint。

## 實驗原則

| 原則 | Rationale |
|------|-----------|
| 先讓狀態可被觀測，再追求更高自動化 | 不可觀測的系統無法判斷自動化是否正確運作 |
| 先讓 failure 可被分類，再追求更快 recovery | 分類不清的失敗會導致 recovery 策略的盲目嘗試 |
| 先讓流程可被驗證，再追求功能面更完整 | 未驗證的功能會累積技術債與假設漂移 |
| 先讓 parity checkpoint 誠實，再談更大的能力宣稱 | 過度宣稱會破壞 checkpoint 作為研究基準的可信度 |

## Phase 1 — 讓 worker lifecycle 可被可靠觀測

**研究問題**：我們能不能明確知道 worker 是否 ready、blocked、finished，還是只是 quiet？

| 工作 | 預期效果 |
|------|---------|
| Worker readiness handshake | Prompt 不再在 worker 未 ready 時送出 |
| Trust resolution flow | Trust gate 變成可檢測、可解除的 state |
| Prompt misdelivery detection | Shell 誤投遞成為正式 failure 類型，而不是只能靠人猜 |
| Structured session control API | — |

## Phase 2 — 讓 event 成為一級資料，而不是 log 副產品

**研究問題**：我們能不能用結構化事件，而不是 noisy logs，來描述 lane 的真實狀態？

| 工作 | 預期效果 |
|------|---------|
| Canonical lane event schema | clawhip 直接消費 typed events |
| Failure taxonomy | Discord summaries 可以從 structured state 渲染 |
| Summary compression | 人類與 agents 都不再需要從 build spam 猜進度 |

## Phase 3 — 讓 recovery 變成系統能力

**研究問題**：常見失敗能不能先自動處理一次，再決定是否需要人工介入？

| 工作 | 預期效果 |
|------|---------|
| Stale-branch detection | Stale branch 先被辨識，再跑廣域測試 |
| Common recovery recipes | 已知 failure 有標準 recovery 路徑 |
| Green-ness contract | 「綠燈」不再是模糊說法，而是有層級定義 |

## Phase 4 — 讓 task execution 真正以 claws 為中心

**研究問題**：任務是否能以結構化封包派送，而不是只能依賴長篇 prompt？

| 工作 | 預期效果 |
|------|---------|
| Typed task packet format | 任務可記錄、可重放、可重試 |
| Policy engine | Merge / retry / escalation 規則變成 executable policy |
| Machine-readable lane board | Downstream tooling 可直接查詢狀態，而不是抓取畫面文字 |

## Phase 5 — 讓 plugin / MCP lifecycle 進入可驗證範圍

**研究問題**：plugin 與 MCP failures 能不能被完整描述，而不是只呈現成「看起來壞掉了」？

| 工作 | 預期效果 |
|------|---------|
| Lifecycle contract | Partial startup 可以被正確回報 |
| Startup / handshake / degraded-mode reporting | Healthy 與 degraded 狀態可被區分 |
| End-to-end MCP parity coverage | Parity harness 能驗證 lifecycle 的主要路徑 |

## 如何看待目前的 backlog

目前 backlog 不應被視為「還沒做完的產品功能」，而應被視為：哪些實驗條件還不穩定、哪些狀態還不夠可觀測、哪些失敗還不能被機器正確理解、哪些 parity claims 還缺少證據。

因此，`doctor` / preflight diagnostics、JSON output contract、degraded MCP reporting、task packets、branch freshness detection、worker state persistence 這些工作不是週邊優化，而是讓實驗成立的必要基礎。

## 進步的判斷標準

不應只問功能是不是變多，而應問：

- 啟動是否更穩
- 狀態是否更清楚
- recovery 是否更少依賴人工
- parity 報告是否更誠實
- agent 協作是否更容易被觀察與重現

## 開放問題

- Phase 1-5 的順序是否最優？是否有某些 phases 應該並行？
- Failure taxonomy 的分類粒度如何確定——太細會導致維護成本，太粗會失去 recovery 的精準性？
- 「green-ness contract」的定義應該由誰來決定，以及它應該在多個層次（unit test / integration / parity harness）之間如何分配？
