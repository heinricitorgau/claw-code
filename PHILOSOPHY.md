# Claw Code — 實驗立場

## 研究問題

> **當人類負責設定方向，而多個 agents 負責執行與回饋時，軟體開發流程中的哪些環節會發生結構性改變？哪些不會？**

如果只把這個 repository 當成一個「被 agents 寫出來的 codebase」，你看到的只是表層產物。這裡真正值得研究的是背後那個被反覆驗證的問題：agent-mediated development 到底在改變什麼，改變到什麼程度，以及哪些假設在不同情境下不成立。

這個專案比較合理的定位是：**一個公開、可觀測、可反覆修正的 agent-mediated software development experiment**，而不是一個已完成的自治開發系統。

## 實驗框架

| 層次 | 角色 |
|------|------|
| 程式碼 | 實驗產物（evidence） |
| 協作循環 | 實驗對象（subject） |
| 工作流設計 | 主要實驗變數 |

### 為什麼強調實驗視角

如果一開始就把這類系統描述成成熟產品，很容易忽略三件事：

- 很多行為仍在變動中，尚未穩定
- 很多結果仍需要觀察與對照才能判斷意義
- 很多看起來有效的機制，未必能在不同情境下成立

## 人類的角色

這個專案的核心前提不是讓人類退出開發，而是重新分配責任邊界：

| 責任方 | 主要工作 |
|--------|---------|
| 人類 | 定義方向、設定限制、提供判斷準則 |
| Agents | 拆解、執行、驗證、回報 |
| 系統 | 讓這些循環可觀測、可重試、可比較 |

在這個模型裡，人類最重要的工作不是打字速度，而是定義對的問題、切出正確的任務邊界、判斷哪些 failure 值得 retry 而哪些應該 re-plan，以及分辨短期效率與長期可靠性之間的取捨。

## Discord 的實驗意義

這個專案強調 Discord 不是因為偏好某個工具，而是因為它改變了人類與系統的互動結構：如果人類可以只給一句方向然後離開現場，而系統仍能接收任務、拆解工作、分配角色、回報狀態、從失敗中恢復，那麼被驗證的就不只是工具本身，而是整個工作流是否具備「低同步成本」。

這是一個可測試的假設，不是一個預設的結論。

## 三段式系統的設計選擇

### OmX（oh-my-codex）— 工作流層

[oh-my-codex](https://github.com/Yeachan-Heo/oh-my-codex) 的價值在於：把模糊指令轉成可執行的工作協定，讓實驗變得可重複。這解決的核心問題是：每次都要重新描述細節的摩擦。

**設計假設**：一句話指令可以透過 OmX 轉成可重複執行的工作協定。這個假設目前部分得到驗證，但在指令模糊或任務邊界不清的情況下仍需觀察。

### clawhip — 事件路由層

[clawhip](https://github.com/Yeachan-Heo/clawhip) 把 monitoring 與 delivery 從 agent context 中抽離。

**設計選擇的 rationale**：agents 只專注在實作，不需要格式化 status updates 或路由 notifications。這樣可以更清楚觀察 agent 何時卡住、哪種 failure 最常發生、哪些 recovery 值得自動化。

### OmO（oh-my-openagent）— 多代理協調層

[oh-my-openagent](https://github.com/code-yeongyu/oh-my-openagent) 讓分歧、handoff、review 與 convergence 成為可設計的實驗單位，而不只是「多開幾個 agent」。

## 瓶頸假說

當 agents 可以在短時間內做大量實作時，一個待驗證的觀察是：真正稀缺的資源可能已經從程式碼產出量轉移到以下幾項：

- architectural clarity（架構清晰度）
- judgment（判斷力）
- product taste（品味）
- task decomposition（任務拆解能力）
- operational stability（營運穩定性）
- 對限制條件的敏感度

這個觀察目前主要是定性的，尚缺乏系統性的量化驗證。

## 這個 repository 正在展示什麼

Claw Code 並不嘗試證明「agent 已經可以完全取代開發者」。它正在研究的是：

- 一個 repository 可以被當成實驗場，而不只是交付物
- agent collaboration 可以被公開觀察，而不只是看結果
- planning / execution / review / retry 可以被設計成可比較的循環
- coordination layer 本身可以成為研究主題

程式碼是 evidence。工作流才是實驗主體。

## 開放問題

- 「低同步成本」的工作流設計，是否在任務複雜度上升後仍然成立？
- 三段式系統（OmX / clawhip / OmO）的邊界切分，是否是唯一可行的分解方式？
- 哪類任務在這個 agent 協作模型下仍然需要高頻人工介入，原因是什麼？
- 當 agents 快速大量產出時，人類的架構判斷如何避免成為實際的瓶頸？
