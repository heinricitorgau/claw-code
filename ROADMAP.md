# ROADMAP.md

# Experimental Roadmap for Claw Code

## 這份路線圖的定位

這不是產品發售路線圖，也不是功能清單。

它更像是一份實驗計畫，描述 claw-code 接下來要驗證哪些問題、補強哪些觀測能力，以及哪些假設還沒有被證明。

核心方向是：

> **讓 agent-mediated development 從「能跑」變成「可觀測、可評估、可恢復」。**

## 實驗主軸

目前路線圖圍繞四個主軸：

1. **可控性**：系統是否能在不依賴人工盯場的前提下穩定運作
2. **可觀測性**：失敗、阻塞、進度與狀態是否能被機器與人類正確理解
3. **可恢復性**：常見 failure modes 是否能被自動修復，而不是只會報錯
4. **可對齊性**：Rust port 是否逐步接近可驗證的行為對齊，而不是停留在表面功能存在

## 現在已知的問題

### 1. 啟動仍然太脆弱

- trust prompts 可能阻塞啟動
- prompt 可能送錯地方
- session 存在不等於 worker 已可用

### 2. 狀態仍然太分散

- tmux
- clawhip event stream
- git/worktree 狀態
- test 狀態
- plugin/MCP runtime 狀態

### 3. 太多判斷仍依賴人類推理

- 失敗分類不夠結構化
- event 太像 log，而不是 state
- recovery 還沒有系統性收斂

### 4. 對齊驗證還不夠完整

- 有些能力只在 registry 層接上
- 有些行為只是 stub replacement，不是真正 end-to-end parity
- 有些 success 還沒有被納入可重現的 checkpoint

## 實驗原則

1. **先讓狀態可被觀測，再追求更高自動化。**
2. **先讓 failure 可被分類，再追求更快 recovery。**
3. **先讓流程可被驗證，再追求功能面更完整。**
4. **先讓 parity checkpoint 誠實，再談更大的能力宣稱。**

## Phase 1 — 讓 worker lifecycle 可被可靠觀測

這一階段要回答的問題是：

> 我們能不能明確知道 worker 是否 ready、blocked、finished，還是其實只是 quiet？

重點工作：

- worker readiness handshake
- trust resolution flow
- prompt misdelivery detection
- structured session control API

成功標誌：

- prompt 不再在 worker 未 ready 時送出
- trust gate 變成可檢測、可解除的 state
- shell 誤投遞成為正式 failure 類型，而不是只能靠人猜

## Phase 2 — 讓 event 成為一級資料，而不是 log 副產品

這一階段要回答的問題是：

> 我們能不能用結構化事件，而不是 noisy logs，來描述 lane 的真實狀態？

重點工作：

- canonical lane event schema
- failure taxonomy
- summary compression

成功標誌：

- clawhip 直接消費 typed events
- Discord summaries 可以從 structured state 渲染
- 人類與 agents 都不再需要從 build spam 猜進度

## Phase 3 — 讓 recovery 變成系統能力

這一階段要回答的問題是：

> 常見失敗能不能先自動處理一次，再決定是否需要人工介入？

重點工作：

- stale-branch detection
- common recovery recipes
- green-ness contract

成功標誌：

- stale branch 先被辨識，再跑廣域測試
- 已知 failure 有標準 recovery 路徑
- 「綠燈」不再是模糊說法，而是有層級定義

## Phase 4 — 讓 task execution 真正以 claws 為中心

這一階段要回答的問題是：

> 任務是否能以結構化封包派送，而不是只能依賴長篇 prompt？

重點工作：

- typed task packet format
- policy engine
- machine-readable lane board

成功標誌：

- 任務可記錄、可重放、可重試
- merge / retry / escalation 規則變成 executable policy
- downstream tooling 可直接查詢狀態，而不是抓取畫面文字

## Phase 5 — 讓 plugin / MCP lifecycle 進入可驗證範圍

這一階段要回答的問題是：

> plugin 與 MCP failures 能不能被完整描述，而不是只呈現成「看起來壞掉了」？

重點工作：

- lifecycle contract
- startup / handshake / degraded-mode reporting
- end-to-end MCP parity coverage

成功標誌：

- partial startup 可以被正確回報
- healthy 與 degraded 狀態可被區分
- parity harness 能驗證 lifecycle 的主要路徑

## 當前 backlog 要怎麼看

目前 backlog 不應被視為「還沒做完的產品功能」，而應被視為：

- 哪些實驗條件還不穩定
- 哪些狀態還不夠可觀測
- 哪些失敗還不能被機器正確理解
- 哪些 parity claims 還缺少證據

因此，像是：

- `doctor` / preflight diagnostics
- JSON output contract
- degraded MCP reporting
- task packets
- branch freshness detection
- worker state persistence

這些工作不是週邊優化，而是讓實驗成立的必要基礎。

## 如何判斷這份實驗是否在進步

我們不應只問功能是不是變多，而應問：

- 啟動是否更穩
- 狀態是否更清楚
- recovery 是否更少依賴人工
- parity 報告是否更誠實
- agent 協作是否更容易被觀察與重現

如果這些指標沒有變好，就算功能變多，也不代表實驗真的成功。

## 短版結論

claw-code 的路線圖，不是在追求「更像一個炫的 agent 產品」。

它真正要做的是：

- 把 agent-mediated development 變成可研究的對象
- 把 failure 變成可分類的資料
- 把 recovery 變成可驗證的能力
- 把 parity 變成可持續追蹤的 checkpoint
