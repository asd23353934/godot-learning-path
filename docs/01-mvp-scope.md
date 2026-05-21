# 6 月 Portfolio MVP Scope

> **目標**：6 個月內做出可面試 Godot junior 工程師的作品
> **背景**：兼職 10-20 hr/week / 前端 Vue+Angular 背景 / 0 Godot 經驗 / 0 美術背景
> **預算**：時間 260-520 hr / 金錢盡量 0（必要時 ~5k 美術委外）
> **寫作日期**：2026-05-21
>
> 這份取代 `docs/10-production/mvp-scope.md`（完整版野心，已封存）。
> 完整版 spec 在 `docs/_archive/v1-full-vision-2026-05/`，未來想做 EA 再接回來。

---

## 為什麼 scope 要這麼小

老實算：

```
26 週 × 15 hr/週 (平均) = 390 hr
   - 學 Godot：30-40 hr
   - 美術 AI + 修圖：30-40 hr
   - 履歷 / itch.io / trailer / Live2D 測試：30-45 hr
   = 純開發剩 250-280 hr
```

原 spec 寫到 112 個 doc + 81 卡 + 29 敵人 + 5 stages。**全做要 1500+ hr**。

→ 砍到 **20%** 比較實際。

---

## 求職目標

主投：
- **WorkNite 沃兎奈**（台中，indie VN/H game，Godot junior）
- 廣譜台灣 indie：雷亞、Rayark、Game Forest、樂陞、Funtoro 等

次投：
- 中小型遊戲外包公司
- 任何接受 0 經驗 + 有作品集的 junior 缺

履歷話術定位：
> 「6 個月從前端轉遊戲開發，自學 Godot 完成 deckbuilder + VN + 棋盤 hybrid demo，已上架 itch.io / Steam page。對成人向 / 全年齡 VN + QTE / 卡牌戰鬥都熟悉。」

---

## In scope（一定做）

### 玩法核心
- **1 個棋盤 stage**（~20 格 closed loop，5 種格子類型）
- **1 個流派**（KIT_SWORD 劍流）
- **10-15 張卡**（從 `docs/02-content/cards-pool.md` 既有 29 張挑代表性的）
- **4-5 種敵人 + 1 boss**（DICE_LORD 簡化為 2 phase）
- **5-6 種道具**（從 `docs/02-content/items-pool.md` 挑常用的）
- **Combo Lock universal 機制**（招牌，要 work 完整）
- **戰鬥內 5 波結構**（簡化版，每波 1-2 敵）

### Hub
- **2 NPCs**：訓練師（升級）+ 卡牌商（解卡）
- **VN 風對話 + 分支選項**（展示 VN 系統能力）
- **永久升級**：HP / 能量 上限（2 階各 1）

### 互動角色（**重點 portfolio piece**）
- AnimatedSprite2D + Tween + Area2D
- 5-8 張 Aria 立繪（idle / smile / attack / hurt / victory / thinking / blush / battle_ready）
- 呼吸動畫（Tween loop）
- 點擊不同部位 → 不同反應
- 隨遊戲 state 換 sprite（HP 低 / Combo×1.5 / 戰勝 / 戰敗）

### VN 對話系統（**前端強項展示**）
- Dialog tree（JSON / .tres 配置）
- 分支選項 + state 影響
- 對話中切換 sprite + 表情
- 用於 Hub NPC + 棋盤事件格

### QTE mini-game（**致敬 WorkNite**）
- 1-2 張 QTE 卡（如 CARD_QTE_FOCUS）
- 方向鍵 + 空白鍵反應
- 視玩家反應給 dmg buff
- 直接致敬 Ride Me Taxi Driver 玩法

### 技術 demo
- **Save / load**（XOR 加密 binary，1 slot）
- **i18n**（繁中 + 英文，UI 完整翻）
- **GdUnit4 單元測試**（combat-formulas + Combo Lock）
- **Web export**（HTML5 to itch.io，HR 不用下載就能玩）
- **GitHub repo + commit history**（看得出每天在動）

### 美術 / 音樂
- **風格**：Slay the Spire 厚塗 + 強描邊 + 限制色板（3-4 主色）
- **卡牌 illustration**：AI 生 + Krita 統一濾鏡
- **UI**：Figma 自做（不用 AI）
- **Aria 立繪 5-8 張**：AI + 後製 + 統一風格
- **BGM**：免費資源（如 Pixabay / Free Music Archive，OpenGameArt）
- **SFX**：免費 SFX 包

---

## Out of scope（一定不做，全在 `_archive/`）

- 其他 5 流派（TIME / ORACLE / MEMORY / BREW / DICE）
- Stage 2-5
- Mirror of Fate 成長樹
- Ascension 1-20
- Daily / Weekly Challenge
- Cosmetics 系統
- Codex / Lore
- Trinket Evolution
- Synergy 1+1>2
- Board Buildings
- 4 種資源經濟（只用 Gold）
- 進階 Recall Pool（保留基本 8 卡 idea 在 doc 但不實作）
- Cutscene Act 1 / Act 2 真結局
- ASTRA + 4 後續 boss
- 完整 NPC 對話池
- 5 表情變體（5-8 張立繪就好）
- 配音

---

## Deliverables（要交什麼給 HR）

| 項目 | 形式 | 用途 |
|------|------|------|
| **可玩 demo** | itch.io HTML5 | HR 點連結直接玩 |
| **GitHub repo** | 公開 | 看 code quality + commit history |
| **README.md** | repo 首頁 | 截圖 5-8 張 + 玩法 + 技術 highlights + 學到什麼 |
| **60-90 秒 trailer** | YouTube / itch.io | 行銷面，秀「整體感」 |
| **設計文件範例** | repo 內 docs/ | 給 1-2 份你引以為傲的 spec（如 combat-formulas.md），展示溝通能力 |
| **Live2D 整合測試** | 另一個小 repo | 證明會 Live2D（補 WorkNite gap）|
| **履歷 PDF** | 1111 / 104 / yourator | 簡潔版 + 連結到上述 |

---

## 6 個月 milestone 切片

詳細週時程見 `docs/_portfolio/02-learning-path.md`。

| Month | 重點 |
|-------|------|
| **M1**（W1-4）| Godot 基礎 + 完成 2 個 tutorial 小作品 |
| **M2**（W5-8）| DFS Phase 0-2：walking skeleton + 戰鬥 prototype |
| **M3**（W9-13）| DFS Phase 3-4：戰鬥 mature + 棋盤 prototype |
| **M4**（W14-17）| DFS Phase 5-6：整合 + Hub + Save |
| **M5**（W18-22）| DFS Phase 7-8：內容填充 + 美術整合 + VN + QTE + Aria 互動 |
| **M6**（W23-26）| 收尾：polish + Live2D test + trailer + 履歷 + 應徵 |

---

## 風險 / mitigations

| 風險 | 機率 | mitigation |
|------|------|----------|
| Godot 學太慢，M2 還沒進 DFS | 中 | M1 卡關就找 Discord / 朋友求救，不要硬撐 |
| 兼職時間被工作壓縮 | 高 | 每週固定時段（如週末早上）保護 |
| 美術 AI 出圖風格不統一 | 高 | M1 就決定 prompt template + 濾鏡 action，不要拖到 M5 |
| 想做太多功能 | 極高 | 凡是不在「In scope」list 的全砍，**已存 _archive 不會丟** |
| WorkNite 6 月不開職缺 | 高 | 廣譜投，不要只投 1 家 |
| 美術自評過不去 | 中 | 接受 demo「設計優先，美術後補」邏輯，正式上架後再委外 |

---

## 不投 H 內容的決定

- WorkNite 是 H 工作室，但他們願景是「**成人遊戲不只 H，是好玩**」
- 你 demo 做**全年齡版**，展示玩法本身
- 面試時直接說「demo 為求廣投投全年齡，可做 H 方向，已熟悉 DLsite / 18+ 內容業界規範」
- 避免「demo 給其他公司面試時尷尬」+「美術成本爆增」

---

## 我會繼續用的既有 doc（不在 _archive）

這些 doc 留在 `docs/` 根目錄，portfolio 開發會繼續引用：

**Design**:
- `01-design/combat-system.md` — 戰鬥 + Combo Lock 規範（前面寫的還能用）
- `01-design/combat-formulas.md` — 戰鬥公式（**§1.5 Combo Lock 是亮點**，§11-12 ORACLE/MEMORY 開發不到可忽略）
- `01-design/status-effects.md` — 基本 status（KIT_TIME/MEMORY 專屬可忽略）
- `01-design/card-system.md`、`board-system.md`、`enemy-system.md`、`item-system.md`
- `01-design/tutorial.md` — 用 popup 1-13（**忽略 EA E1-E8 + Combo Lock 14-17 進階**）
- `01-design/gem-system.md` — 12 寶石（簡化用，不一定全做）
- `01-design/trinket-system.md` — 10 MVP trinket（忽略 EA 15 + starter trinkets）
- `01-design/economy.md` — 看 「Run 內商店價格」「Hub 商店價格」section，**忽略 4 資源那段**
- `01-design/pillars.md`、`pacing.md`、`edge-cases.md`、`progression.md`、`progression-curve.md`

**Content**:
- `02-content/cards-pool.md` — 29 卡完整，挑 10-15 張做
- `02-content/enemies-pool.md` — Stage 1 完整 10 種，挑 4-5 做
- `02-content/items-pool.md` — 11 道具，挑 5-6
- `02-content/starting-kits.md` — **只看 KIT_SWORD section**，忽略其他流派
- `02-content/stages.md` — **只看 Stage 1 完整 45 格 layout**，簡化為 ~20 格
- `02-content/bosses.md` — **只看 BOSS_DICE_LORD section**，忽略 4 後續 boss
- `02-content/achievements.md` — 用基本 25，**忽略 advanced 70**

**Narrative**（簡化）:
- `03-narrative/main-character.md` — 用 Aria 基本設定（**忽略 KIT_MEMORY 進階 narrative 章節**）
- `03-narrative/npcs.md` — 用訓練師 + 卡牌商基本對白
- `03-narrative/world.md` — 世界觀概念

**Tech**（全用）:
- `07-tech/stack.md`、`architecture.md`（autoload 11 → 6 個就夠）
- `07-tech/data-schemas.md`、`save-format.md`、`i18n.md`、`coding-conventions.md`
- `07-tech/error-handling.md`、`logging.md`
- `07-tech/performance-budget.md`、`debug-tools.md`

**Implementation**:
- `08-implementation/steps/phase-0` 到 `phase-9`（全用，但每 phase 砍 scope）
- `08-implementation/tools.md`、`workflow.md`、`milestones.md`、`dependencies-graph.md`

**Art / Audio / UX**:
- `04-art/style-guide.md`、`game-feel.md`、`card-frame.md`、`ui-style.md`
- `04-art/asset-list.md`、`asset-pipeline.md`、`character-design.md`
- `05-audio/audio-pipeline.md`、`music-design.md`（基本）、`sfx-list.md`（基本）
- `06-ux/screens.md`、`ui-numbers.md`、`navigation-flow.md`、`settings.md`、`accessibility.md`

**Testing**（部分）:
- `09-testing/strategy.md`、`unit-tests.md`、`e2e-testing.md`
- `09-testing/balance-testing.md` — **忽略 retention 系統校準章節**，看基本 auto-play 即可
- `09-testing/playtest-plan.md`、`bug-tracking.md`

---

## 相關文件

- 學習路徑：[`02-learning-path.md`](02-learning-path.md)
- 美術 + 互動 + VN pipeline：[`03-pipeline.md`](03-pipeline.md)
- 完整野心版 spec（封存）：[`docs/_archive/v1-full-vision-2026-05/`](https://github.com/asd23353934/dice-fate-survivor/tree/master/docs/_archive/v1-full-vision-2026-05)（原 DFS repo）
- 既有 phase 切片：[`docs/08-implementation/steps/`](https://github.com/asd23353934/dice-fate-survivor/tree/master/docs/08-implementation/steps)（原 DFS repo）
- 既有戰鬥公式：[`docs/01-design/combat-formulas.md`](https://github.com/asd23353934/dice-fate-survivor/blob/master/docs/01-design/combat-formulas.md)（原 DFS repo）
