# Claw Code Philosophy

## 先把它當成實驗，不要先把它當成產品

如果只把這個 repository 當成一個「被 agents 寫出來的 codebase」，你看到的只會是表層結果。

這裡真正有價值的，不只是程式碼本身，而是背後那個被反覆驗證的問題：

> **當人類負責設定方向，而多個 agents 負責執行與回饋時，軟體開發流程會如何改變？**

所以，這個專案最適合被理解成一個研究場域：

- 程式碼是產物
- 協作循環是實驗對象
- 工作流設計是主要變數

## 為什麼強調實驗視角

如果一開始就把這類系統描述成成熟產品，很容易忽略三件事：

- 很多行為仍在變動
- 很多結果仍需要觀察與對照
- 很多看起來有效的機制，未必能在不同情境下成立

因此，Claw Code 比較合理的定位不是「已完成的自治開發系統」，而是：

**一個公開、可觀測、可反覆修正的 agent-mediated software development experiment。**

## 人類的角色不是消失，而是被重新定義

這個專案的核心前提，不是讓人類退出開發，而是重新分配人類與 agents 的責任：

- 人類定義方向、限制與判準
- agents 負責拆解、執行、驗證與回報
- 系統負責把這些循環變得可觀測、可重試、可比較

在這個模型裡，人類最重要的工作不是打字速度，而是：

- 是否能定義對的問題
- 是否能切出正確的任務邊界
- 是否能判斷哪些 failure 值得 retry，哪些應該 re-plan
- 是否能分辨短期效率與長期可靠性之間的取捨

## Discord 不是噱頭，而是實驗條件之一

這個專案之所以強調 Discord，不是因為它酷，而是因為它改變了人類與系統的互動方式。

如果人類可以只給一句方向，然後離開現場，而系統仍能：

- 接收任務
- 拆解工作
- 分配角色
- 回報狀態
- 從失敗中恢復

那麼被驗證的就不只是工具，而是整個工作流是否具備「低同步成本」。

## 三段式系統的意義

### 1. OmX (`oh-my-codex`)
[oh-my-codex](https://github.com/Yeachan-Heo/oh-my-codex) 是 workflow layer。

它的價值不在於多會下 prompt，而在於它把模糊指令轉成可執行的工作協定，讓實驗變得可重複。

### 2. clawhip
[clawhip](https://github.com/Yeachan-Heo/clawhip) 是 event 與 notification router。

它把 monitoring 與 delivery 從 agent context 中抽離，讓我們可以更清楚觀察：

- agent 何時卡住
- 哪種 failure 最常發生
- 哪些 recovery 值得自動化

### 3. OmO (`oh-my-openagent`)
[oh-my-openagent](https://github.com/code-yeongyu/oh-my-openagent) 是 multi-agent coordination layer。

它的功能不是單純「多開幾個 agent」，而是讓分歧、handoff、review 與 convergence 成為可設計的實驗單位。

## 真正的瓶頸已經改變

當 agents 可以在短時間內做大量實作時，真正稀缺的東西就不再是產出量，而是：

- architectural clarity
- judgment
- product taste
- task decomposition
- operational stability
- 對限制條件的敏感度

所以，這個專案的一個重要研究結論傾向是：

> **agent 不會讓思考不重要；它只會讓清楚思考變得更重要。**

## Claw Code 真正在展示什麼

Claw Code 想展示的不是「agent 已經可以完全取代開發者」。

它真正想展示的是：

- 一個 repository 可以被當成實驗場，而不是只當成交付物
- agent collaboration 可以被公開觀察，而不是只看結果
- planning / execution / review / retry 可以被設計成可比較的循環
- coordination layer 本身可以成為研究主題

程式碼是 evidence。  
工作流才是實驗主體。

## 短版結論

**Claw Code 不是在證明自治萬能。它是在研究：受控的 agent 協作，到底能把開發流程改變到什麼程度。**

如果你要看的是最終產品，這個 repository 可能還太早。  
如果你想看的是實驗材料、工作流假設與系統性觀察，那它正是為了這件事而存在。
