# 學習進度（週級）

> 起始日：2026-05-22
> 配合 [`docs/02-learning-path.md`](docs/02-learning-path.md) 月度切片
> 細項任務見 [`docs/05-tasks.md`](docs/05-tasks.md)

---

## Status Legend

```
[ ]   Todo
[~]   In Progress
[x]   Done
[!]   Blocked
[s]   Skipped
```

---

## 進度總覽

```
M1 (W1-W4)   ░░░░░░░░░░  0%   Godot 基礎 + 2 tutorial
M2 (W5-W8)   ░░░░░░░░░░  0%   DFS Phase 0-2（戰鬥 prototype）
M3 (W9-W13)  ░░░░░░░░░░  0%   DFS Phase 3-4（戰鬥成熟 + 棋盤）
M4 (W14-W17) ░░░░░░░░░░  0%   DFS Phase 5-6（整合 + Hub Meta）
M5 (W18-W22) ░░░░░░░░░░  0%   DFS Phase 7-8（內容 + 美術）
M6 (W23-W26) ░░░░░░░░░░  0%   收尾 + Live2D + 履歷
```

完成週數：**0 / 26**

---

## M1（W1-W4）：Godot 基礎 + 2 tutorial

### W1（2026-05-22 ~ 05-28）：安裝 + GDScript 起步

- 目標時數：10-15 hr
- [ ] 裝 Godot 4.6.2 Standard + VSCode + Godot Tools 擴充
- [ ] 看完 Godot Docs - Getting Started（philosophy / nodes / signals）
- [ ] Hello world：按鈕 → console print
- [ ] GDScript 練 100 行（fizzbuzz / 排序 / class）
- [ ] 週末 retro：3 行心得寫在下方

週記：

```
（每週做完寫 1-3 行：學到什麼、卡哪裡、下週調整）
```

---

### W2：Dodge the Creeps tutorial

- 目標時數：10-15 hr
- [ ] 跟著 [Your First 2D Game](https://docs.godotengine.org/en/stable/getting_started/first_2d_game/index.html) 做完
- [ ] 改成自己版本（如 mob 改成卡牌掉下來閃避）
- [ ] 學到 scene 組合 / signal 連接 / Timer / 隨機 spawn / HUD
- [ ] 週末 retro

週記：

```

```

---

### W3：Custom Resource (.tres) + Autoload

- 目標時數：10-15 hr
- [ ] 看 Resources 文件
- [ ] 寫 `CardData.gd`（extends Resource）+ 建 3 張測試卡 .tres
- [ ] 寫 `GameState.gd` autoload（player_hp / energy）
- [ ] 寫 `EventBus.gd` autoload（5-10 個 signal）
- [ ] 週末 retro

週記：

```

```

---

### W4：第 2 個 tutorial + 卡牌 prototype

- 目標時數：10-15 hr
- [ ] 跟一個 YouTube「Godot 4 Deckbuilder Tutorial」
- [ ] 試做：手牌顯示 + 拖卡到敵人 + 扣 HP
- [ ] 月底 retro：M1 哪裡超時 / 哪裡輕鬆

週記：

```

```

**M1 完成驗收**：能獨立寫出簡單 2D 遊戲。

---

## M2（W5-W8）：DFS Phase 0-2

### W5：Phase 0 setup

- 目標時數：10-15 hr
- [ ] 建 DFS 專案結構（5-6 個 autoload）
- [ ] 設 Godot project settings（render / display / input）
- [ ] Git repo + commit pattern 開始

週記：

```

```

---

### W6：Phase 1 walking skeleton

- 目標時數：10-15 hr
- [ ] 主選單 → 棋盤 → 戰鬥 → 結算 → 回主選單（全空殼）
- [ ] 場景切換 + 全局 state 串起來

週記：

```

```

---

### W7：Phase 2 戰鬥 prototype (上)

- 目標時數：10-15 hr
- [ ] 戰鬥場景：玩家 + 1 敵人
- [ ] 手牌 UI（hardcode 5 張卡）
- [ ] 拖卡釋放 → 扣敵人 HP

週記：

```

```

---

### W8：Phase 2 戰鬥 prototype (下)

- 目標時數：10-15 hr
- [ ] 敵人回合（簡單 attack）
- [ ] 5 波結構基礎
- [ ] 月底 retro

週記：

```

```

---

## M3（W9-W13）：DFS Phase 3-4

### W9：Phase 3 戰鬥成熟 (1/3) — 卡牌資料化

- 目標時數：10-15 hr
- [ ] hardcode 卡改成從 .tres 讀
- [ ] 卡牌池基本建立

週記：

```

```

---

### W10：Phase 3 戰鬥成熟 (2/3) — Status + Power

- 目標時數：10-15 hr
- [ ] 6 種 status（poison / weak / vulnerable / strength / dex / shield）
- [ ] Power 卡實作

週記：

```

```

---

### W11：Phase 3 戰鬥成熟 (3/3) — Combo Lock + 測試

- 目標時數：10-15 hr
- [ ] Combo Lock universal 機制（招牌）
- [ ] GdUnit4 寫 5-10 個 combat-formulas 測試

週記：

```

```

---

### W12：Phase 4 棋盤 prototype (1/2)

- 目標時數：10-15 hr
- [ ] 簡化版棋盤（20 格 closed loop）
- [ ] 5 種格子（gold / item / event / combat / shop）
- [ ] 玩家擲骰移動

週記：

```

```

---

### W13：Phase 4 棋盤 prototype (2/2)

- 目標時數：10-15 hr
- [ ] 棋盤敵人移動（簡單 random AI）
- [ ] Path encounter（移動經過觸發戰鬥）
- [ ] 月底 retro

週記：

```

```

---

## M4（W14-W17）：DFS Phase 5-6 整合 + Hub

### W14：Phase 5 整合 (1/2)

- 目標時數：10-15 hr
- [ ] 完整 1 個 run loop：選 kit → 進棋盤 → 戰鬥 → 結算 → 再來
- [ ] 三選一獎勵 UI

週記：

```

```

---

### W15：Phase 5 整合 (2/2) — Boss

- 目標時數：10-15 hr
- [ ] Boss 戰（DICE_LORD 簡化 2 phase）

週記：

```

```

---

### W16：Phase 6 Hub + VN

- 目標時數：10-15 hr
- [ ] Hub 場景（簡單 2D）+ 2 NPCs（訓練師 + 卡牌商）
- [ ] VN 風對話系統（dialog tree + 分支選項）

週記：

```

```

---

### W17：Phase 6 Save / Settings

- 目標時數：10-15 hr
- [ ] 永久升級（HP 2 階 / 能量 1 階）
- [ ] Save / load（XOR 加密 binary，1 slot）
- [ ] Settings menu（解析度 / 音量 / 語言）
- [ ] 月底 retro

週記：

```

```

---

## M5（W18-W22）：Phase 7-8 內容 + 美術

### W18：Phase 7 內容填充 (1/2)

- 目標時數：10-15 hr
- [ ] 補滿 10-15 張卡 .tres
- [ ] 補滿 4-5 敵人 + DICE_LORD

週記：

```

```

---

### W19：Phase 7 內容填充 (2/2)

- 目標時數：10-15 hr
- [ ] 補滿 5-6 道具
- [ ] 棋盤事件 3-5 種（VN 系統 + 分支）
- [ ] QTE 卡 1-2 張（致敬 WorkNite）

週記：

```

```

---

### W20：Phase 8 美術 (1/2) — Aria 立繪

- 目標時數：10-15 hr
- [ ] AI 生 5-8 張 Aria 立繪
- [ ] 10-15 張卡 illustration（AI + Krita 修）

週記：

```

```

---

### W21：Phase 8 美術 (2/2) — UI + 互動

- 目標時數：10-15 hr
- [ ] 卡牌邊框 + UI（Figma）
- [ ] AnimatedSprite2D + Tween + Area2D 互動角色
- [ ] BGM 3-5 首 + SFX 20-30 個

週記：

```

```

---

### W22：Phase 9 polish 開始

- 目標時數：10-15 hr
- [ ] 戰鬥 hit pause / screen shake / 飄字
- [ ] Tutorial 5 popup
- [ ] i18n 補完（繁中 + 英文）
- [ ] 月底 retro

週記：

```

```

---

## M6（W23-W26）：收尾 + Live2D + 履歷

### W23：Polish 收尾

- 目標時數：10-15 hr
- [ ] Bug fix（連跑 5 場 run 找 bug）
- [ ] 平衡（auto-play 100 場驗）
- [ ] 音樂 / SFX 最終整合

週記：

```

```

---

### W24：Web export + itch.io + trailer

- 目標時數：10-15 hr
- [ ] HTML5 export 測試 + 上 itch.io
- [ ] 60-90 秒 trailer（OBS 錄 + DaVinci / CapCut 剪）
- [ ] 截圖 5-8 張

週記：

```

```

---

### W25：Live2D 整合測試

- 目標時數：10-15 hr
- [ ] 下載 Live2D 官方免費模型
- [ ] 裝 GDCubism plugin
- [ ] 30 秒 demo（hover / 點擊 / lip sync）
- [ ] 開新 GitHub repo（小 + 獨立 + 1 README）

週記：

```

```

---

### W26：履歷 + 應徵

- 目標時數：10-15 hr
- [ ] 1111 / 104 / yourator 履歷 + 連結
- [ ] 履歷 PDF
- [ ] 內推（Twitter / Discord / FB 社團找）
- [ ] 投 WorkNite + 廣譜 10-20 家

週記：

```

```

**M6 完成驗收**：履歷投出 + 至少 1 個面試機會。

---

## 整體 retro 記錄

完成每個 M 後在這邊記 3-5 行整體心得：

### M1 retro
```

```

### M2 retro
```

```

### M3 retro
```

```

### M4 retro
```

```

### M5 retro
```

```

### M6 retro
```

```
