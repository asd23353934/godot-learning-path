# 前端 → Godot 6 月學習路徑

> **背景**：Vue + Angular 前端 / 兼職 10-20 hr/week / 0 Godot 經驗
> **目標**：6 個月內完成 DFS portfolio MVP（[`01-mvp-scope.md`](01-mvp-scope.md)）
> **寫作日期**：2026-05-21

---

## 為什麼前端轉 Godot 比想像中快

你已經會的（**直接 transfer**）：
- JavaScript / TypeScript 語法 → GDScript 幾乎一樣（變數宣告 / function / class / async）
- Vue 組件樹 ≈ Godot scene tree
- Vue emit / Angular EventEmitter ≈ Godot signal
- Vuex / Pinia / NgRx state → Godot autoload (singleton) pattern
- CSS animation → Godot Tween / AnimationPlayer
- React/Vue 條件渲染 → Godot visibility / queue_free
- npm package → Godot AssetLib + community plugins
- Git / branching / PR workflow → 完全一樣
- TypeScript strict mode → GDScript typed mode（你會自然開）

你**還沒接觸過**的（要學）：
- Game loop（`_process(delta)` / `_physics_process(delta)`）
- Resource (.tres) 序列化系統
- Node lifecycle（`_ready()` 不是 `mounted`，差別小）
- 2D 渲染（`draw()` / `CanvasItem`）
- Scene composition vs class inheritance（vs Vue 純組件式）
- AnimationPlayer / AnimatedSprite2D
- Input handling（鍵盤 / 滑鼠 / 觸控 events）

→ 多數 Godot 初學 tutorial **你可以跳過 50%**（變數 / function / class / 條件 / 迴圈）。

---

## 月度切片

### M1（W1-4）：Godot 基礎 + 2 個小遊戲

**目標**：能獨立寫出簡單 2D 遊戲。

#### W1：GDScript + Godot Editor 適應期（10-15 hr）
- [ ] 裝 Godot 4.6.2（不要 .NET 版）+ VSCode + Godot Tools 擴充
- [ ] 看 [Godot Docs - Getting Started](https://docs.godotengine.org/en/stable/getting_started/introduction/index.html) 全部
- [ ] **跳過**：Step by Step / Scripting languages（你會了）
- [ ] **看**：Godot's design philosophy / Project organization / Scenes and nodes / Signals
- [ ] 寫 hello world：場景內按按鈕 → console print
- [ ] 練 GDScript：用 GDScript 寫 100 行小腳本（你 TypeScript 的 fizzbuzz / 排序 / class 範例都可以重寫）

#### W2：Dodge the Creeps tutorial（10-15 hr）
- [ ] 跟著 [Your First 2D Game](https://docs.godotengine.org/en/stable/getting_started/first_2d_game/index.html) 做完
- [ ] **重點學到**：
  - Scene 組合（Player / Mob / HUD 各自 scene 然後組合）
  - Signal 連接（Player hit → game over）
  - Timer node
  - 隨機生成 mob（spawn pattern）
  - HUD（Score / Game Over UI）
- [ ] 不照抄，改你自己版本（如 mob 改成卡牌掉下來閃避）

#### W3：Custom Resource (.tres) + Autoload（10-15 hr）
- [ ] 看 [Resources 文件](https://docs.godotengine.org/en/stable/tutorials/scripting/resources.html)
- [ ] 寫 `CardData.gd`（extends Resource）
- [ ] Inspector 編輯 .tres 檔案：建 3 張測試卡（STRIKE / DEFEND / FOCUS）
- [ ] 寫 `GameState.gd` autoload，存當前 player_hp / energy
- [ ] 寫 `EventBus.gd` autoload，定義 5-10 個遊戲 signal
- [ ] **重點學到**：data-driven design + 全局 state 管理

#### W4：第 2 個 tutorial + 設計卡牌 prototype（10-15 hr）
- [ ] 跟著一個 deckbuilder tutorial（YouTube：「Godot 4 Deckbuilder Tutorial」搜）
- [ ] 試做：手牌顯示 + 拖卡到敵人 + 扣 HP
- [ ] 不求美觀，求機制能動

**M1 完成 = 你已經會 Godot 8 成基礎。剩下都是 DFS 細節。**

---

### M2（W5-8）：DFS Phase 0-2

詳細任務看 `docs/08-implementation/steps/phase-0-setup.md` 到 `phase-2-combat-prototype.md`。

#### W5-6：Phase 0-1（10-15 hr × 2）
- 建 DFS 專案結構（5-6 個 autoload）
- 設 Godot project settings（render / display / input）
- Walking skeleton：主選單 → 棋盤 → 戰鬥 → 結算 → 回主選單（全空殼）
- Git repo + commit pattern 開始

#### W7-8：Phase 2 戰鬥 prototype（10-15 hr × 2）
- 戰鬥場景：玩家 + 1 敵人
- 手牌 UI（hardcode 5 張卡）
- 拖卡釋放 → 扣敵人 HP
- 敵人回合（簡單 attack）
- 5 波結構基礎

---

### M3（W9-13）：DFS Phase 3-4

#### W9-11：Phase 3 戰鬥成熟（10-15 hr × 3）
- 把 hardcode 卡改成從 .tres 讀
- 加 6 種 status（poison / weak / vulnerable / strength / dex / shield）
- Power 卡實作
- **加 Combo Lock universal 機制**（招牌）
- GdUnit4 寫 5-10 個 combat-formulas 測試

#### W12-13：Phase 4 棋盤 prototype（10-15 hr × 2）
- 簡化版棋盤（20 格 closed loop）
- 5 種格子（gold / item / event / combat / shop）
- 玩家擲骰移動
- 棋盤敵人移動（簡單 random AI）
- Path encounter（移動經過觸發戰鬥）

---

### M4（W14-17）：Phase 5-6 整合 + Hub

#### W14-15：Phase 5 整合（10-15 hr × 2）
- 完整 1 個 run loop：選 kit → 進棋盤 → 戰鬥 → 結算 → 再來
- 三選一獎勵 UI
- Boss 戰（DICE_LORD 簡化 2 phase）

#### W16-17：Phase 6 Hub + Meta（10-15 hr × 2）
- Hub 場景（簡單 2D）
- 2 NPCs（訓練師 + 卡牌商）
- **VN 風對話系統**（dialog tree + 分支選項）
- 永久升級（HP 2 階 / 能量 1 階）
- Save / load（XOR 加密 binary，1 slot）
- Settings menu（解析度 / 音量 / 語言）

---

### M5（W18-22）：Phase 7-8 內容 + 美術 + 互動角色

#### W18-19：Phase 7 內容填充（10-15 hr × 2）
- 補滿 10-15 張卡 .tres（依 `docs/02-content/cards-pool.md`）
- 補滿 4-5 敵人 + DICE_LORD
- 補滿 5-6 道具
- 棋盤事件 3-5 種（**展示 VN 系統 + 分支選擇**）
- **QTE 卡 1-2 張**（致敬 WorkNite）

#### W20-21：Phase 8 美術整合 + Aria 互動（10-15 hr × 2）
- AI 生 5-8 張 Aria 立繪（依 `03-pipeline.md` prompt template）
- 10-15 張卡 illustration（AI + Krita 修）
- 卡牌邊框 + UI（Figma）
- **AnimatedSprite2D + Tween + Area2D 互動角色**（依 `03-pipeline.md` 範例 code）
- BGM 3-5 首（免費資源）+ SFX 20-30 個

#### W22：Phase 9 polish 開始（10-15 hr）
- 戰鬥 hit pause / screen shake / 飄字
- Tutorial 5 popup
- i18n 補完（繁中 + 英文）

---

### M6（W23-26）：收尾 + Live2D + 履歷 + 應徵

#### W23：Polish 收尾（10-15 hr）
- Bug fix（連跑 5 場 run 找 bug）
- 平衡（auto-play 100 場驗）
- 音樂 / SFX 最終整合

#### W24：Web export + itch.io + trailer（10-15 hr）
- HTML5 export 測試 + 上 itch.io
- 60-90 秒 trailer（用 OBS 錄 + DaVinci Resolve / CapCut 剪）
- 截圖 5-8 張

#### W25：Live2D 整合測試（10-15 hr）
- 下載 Live2D 官方免費模型（Hiyori / Mark / Mao）
- 裝 GDCubism plugin
- 寫 30 秒 demo（hover / 點擊 / lip sync）
- 開新 GitHub repo（小 + 獨立 + 1 README）

#### W26：履歷 + 應徵（10-15 hr）
- 1111 / 104 / yourator 履歷 + 連結
- 履歷 PDF
- 內推（Twitter / Discord / FB 社團找）
- 投 WorkNite + 廣譜 10-20 家

---

## 每週固定時段建議

兼職最大敵人是「**沒有固定時間就會被各種事擠掉**」。

建議排表：
- **週六上午 9:00-13:00**（4 hr，最重要，做核心開發）
- **週日上午 9:00-13:00**（4 hr，續做或學習）
- **週二 / 週四晚 20:00-22:00**（4 hr，bug fix / 學東西）
- **總**：12-14 hr/week 穩定產出

要求自己**手機放遠**。

---

## 容易卡關的點 + 解法

| 卡關 | 解法 |
|------|------|
| GDScript 不熟 | 用 ChatGPT / Claude 翻譯 TypeScript → GDScript |
| Godot Editor 找不到功能 | 看 [Godot Docs](https://docs.godotengine.org/) + Discord 中文台問 |
| 不會做戰鬥動畫 | YouTube 「Godot card game tutorial」直接抄 |
| 美術出不出統一感 | 用同一 prompt + LoRA + Krita 同 action |
| 沒人 playtest | 找 1-2 個朋友每月跑 1 次 demo + 錄影 |
| 想開新功能 | 看 `01-mvp-scope.md` 「Out of scope」list，全砍 |

---

## 學習資源

### 必看

- **[Godot 官方 Docs](https://docs.godotengine.org/en/stable/)** — 4.x 中文有部分
- **[Godot 4 入門系列 - GDQuest](https://www.gdquest.com/tutorial/godot/learning-paths/learn-godot-4-from-zero/)** — 收費但品質好
- **YouTube「Godot 4 Deckbuilder」、「Godot 4 RPG」、「Godot 4 Card Game」** — 抄就對了

### 社群

- **Godot Discord**（[discord.gg/4JBkykG](https://discord.gg/4JBkykG)） — 24/7 有人回問題
- **GodotShaders.com** — 免費 shader（粒子 / 後製效果）
- **[r/godot](https://reddit.com/r/godot)** — 看別人怎麼做

### 工具

- **Godot 4.6.2 Standard**（[godotengine.org](https://godotengine.org)）
- **VSCode + Godot Tools 擴充**
- **GdUnit4**（單元測試）
- **Aseprite**（$20 美術 / pixel art / 動畫 sprite sheet，可選）
- **Krita**（免費繪圖 + AI 修圖）
- **Figma**（免費 UI 設計）
- **DaVinci Resolve / CapCut**（免費剪 trailer）
- **OBS Studio**（免費螢幕錄影）

### 美術資源

- **Kenney.nl** — 免費 asset pack（CC0）
- **OpenGameArt.org** — 免費 game asset
- **itch.io free assets** — 篩選 free + commercial OK
- **思源黑體 / 思源宋體** — 免費商用中文字型
- **Google Fonts** — 免費英文字型
- **NovelAI**（$10-25/月） — anime AI 美術
- **Stable Diffusion local** — 免費（需 GPU）

### 音樂 SFX

- **Pixabay Music** — CC0
- **Free Music Archive** — CC 各種
- **freesound.org** — CC 各種
- **Suno**（$10/月） — AI BGM 生成

---

## 如果你已經卡 1 週還沒進展

說真的：來找我問。Discord 開頻道、寫 issue、什麼都好，**不要硬撐**。

兼職 6 月時程很緊。1 週卡關 = 4 週 buffer 用掉 25%。

---

## 相關文件

- 6 月 portfolio scope：[`01-mvp-scope.md`](01-mvp-scope.md)
- 美術 + 互動 + VN pipeline：[`03-pipeline.md`](03-pipeline.md)
- 既有 phase 切片：[`docs/08-implementation/steps/`](https://github.com/asd23353934/dice-fate-survivor/tree/master/docs/08-implementation/steps)（原 DFS repo）
- Godot 4 stack：[`docs/07-tech/stack.md`](https://github.com/asd23353934/dice-fate-survivor/blob/master/docs/07-tech/stack.md)（原 DFS repo）
