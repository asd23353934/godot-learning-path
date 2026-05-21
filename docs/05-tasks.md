# 6 月 Portfolio 任務清單（含驗收）

> 完成 task → 回報我 → 我驗收 → 標 `[x]`
> 寫作日期：2026-05-21
> 配合 [`01-mvp-scope.md`](01-mvp-scope.md) / [`02-learning-path.md`](02-learning-path.md)

---

## 怎麼用

1. 從 **進度總覽** 看當前在哪
2. 從 **當週 task** 挑 1-2 個開始
3. 完成 → 跟我說「**T005 完成**」+ 提供驗收憑證
4. 我驗 OK → 我直接 edit 此 doc 標 `[x]` + commit
5. 我同時告訴你下個 3 個優先 task

---

## 驗收 workflow

### 你要提供的「驗收憑證」（依任務類型）

| Task 類型 | 驗收憑證 |
|---------|--------|
| 裝軟體 | 終端機指令輸出 / 截圖描述 |
| 看 doc / tutorial | 答 2-3 個我的問題 |
| 寫 code | git commit hash + 我去讀 |
| 寫 test | 同上 + 跑 test 結果 |
| 上 itch.io / GitHub | URL 給我，我 WebFetch |
| Design 決定 | 跟我說你的選擇 + 理由 |

### 不通過怎麼辦

- 我會說「這條沒過，原因 X，建議 Y」
- 你修 → 再回報
- 不會卡你太久，多數 fail 是命名 / typed 之類小事

---

## Status legend

```
[ ]   Todo
[~]   In Progress（你開始做了）
[x]   Done（我驗過）
[!]   Blocked（卡住，需查資料 / 求救）
[s]   Skipped（決定不做）
```

---

## 進度總覽

```
M1 (W1-4)  ░░░░░░░░░░ 0%   Godot 基礎 + 2 tutorial
M2 (W5-8)  ░░░░░░░░░░ 0%   DFS Phase 0-2
M3 (W9-13) ░░░░░░░░░░ 0%   DFS Phase 3-4
M4 (W14-17)░░░░░░░░░░ 0%   DFS Phase 5-6
M5 (W18-22)░░░░░░░░░░ 0%   DFS Phase 7-8
M6 (W23-26)░░░░░░░░░░ 0%   收尾 + Live2D + 履歷

總任務數：90+
完成：0
```

每次我標 done 時也會更新此進度條。

---

# M1（W1-W4）：Godot 基礎

## W1：安裝 + GDScript 起步

### 環境設定

- [ ] **T001** 裝 Godot 4.6.2 Standard
  - 驗收：跟我說「裝好了」+ 在終端跑 `godot --version` 貼結果
  - 估時：30 min
  - 連結：[godotengine.org/download](https://godotengine.org/download)

- [ ] **T002** 裝 VSCode + Godot Tools 擴充
  - 驗收：開 VSCode 看到 Godot Tools icon + 開 .gd 檔有語法高亮
  - 估時：30 min
  - 連結：[Godot Tools VSCode](https://marketplace.visualstudio.com/items?itemName=geequlim.godot-tools)

- [ ] **T003** 設 VSCode tab 縮排
  - 驗收：建 `.vscode/settings.json`，內容是 `04-coding-standards.md` 第 5 章那段
  - 估時：5 min

- [ ] **T004** 裝 gdformat CLI
  - 驗收：`gdformat --version` 跑出來
  - 估時：15 min
  - 指令：`pip install "gdtoolkit==4.*"`

- [ ] **T005** 在 dice-fate-survivor repo 建 Godot project
  - 驗收：給我 commit hash，我去 `git log` 確認檔案有 `project.godot`
  - 估時：30 min
  - 設定：Renderer = Forward+ / Compatibility（看你機器）
  - Commit message: `chore: init Godot 4.6.2 project`

- [ ] **T006** 加 `.gitignore` Godot 版
  - 驗收：commit hash，我去 cat `.gitignore` 看
  - 估時：5 min
  - 內容：抄 `04-coding-standards.md` 第 16 章

### 學 GDScript 基礎

- [ ] **T007** 看完官方 Godot Docs 4 章節
  - [Project organization](https://docs.godotengine.org/en/stable/tutorials/best_practices/project_organization.html)
  - [Scenes and nodes](https://docs.godotengine.org/en/stable/tutorials/scripting/scenes_and_nodes/index.html)
  - [Signals](https://docs.godotengine.org/en/stable/getting_started/step_by_step/signals.html)
  - [Custom Resources](https://docs.godotengine.org/en/stable/tutorials/scripting/resources.html)
  - 驗收：我問你 2 問題（如「signal 跟 method call 差在哪」「為什麼用 Resource 不用 Dictionary」）
  - 估時：3-5 hr

- [ ] **T008** 寫 100 行 GDScript 練習
  - 練：fizzbuzz / 排序 / 簡單 class（不寫 game，純語法）
  - 驗收：commit `tests/practice/learn_gdscript.gd`，我讀過 OK
  - 估時：2 hr

---

## W2：Dodge the Creeps Tutorial

- [ ] **T009** 跟完 [Your First 2D Game](https://docs.godotengine.org/en/stable/getting_started/first_2d_game/index.html) tutorial
  - 驗收：commit 完成版到 `tutorials/dodge_the_creeps/` 資料夾，我讀過
  - 估時：6-10 hr

- [ ] **T010** 改一個變化版（如：mob 改卡牌掉下來閃避）
  - 驗收：commit + 截圖 / 影片描述（我會看 code）
  - 估時：2-3 hr

- [ ] **T011** 寫一段 README 講你學到什麼
  - 驗收：`tutorials/dodge_the_creeps/README.md`，至少 5 點 bullet
  - 估時：30 min

---

## W3：Custom Resource + Autoload + 開 DFS infrastructure

- [ ] **T012** 寫 `scripts/core/card_data.gd`（Resource class）
  - Schema：`card_id / name_key / energy_cost / damage / card_type (enum) / description_key`
  - 驗收：commit + 我讀，檢查 typed + 命名 + class_name
  - 估時：1 hr

- [ ] **T013** 建 3 張測試卡 .tres
  - `data/cards/card_strike.tres` (Attack/1/6)
  - `data/cards/card_defend.tres` (Skill/1/block 5)
  - `data/cards/card_focus.tres` (Skill/1/draw 2)
  - 驗收：commit + 我讀 .tres 內容
  - 估時：30 min

- [ ] **T014** 寫 `scripts/autoload/event_bus.gd`
  - 至少 5 個 signal：`card_played(card)` / `combat_started` / `combat_ended(result)` / `player_hp_changed(hp, max_hp)` / `card_drawn(card)`
  - 註冊到 Project Settings → Autoload
  - 驗收：commit + 我讀
  - 估時：1 hr

- [ ] **T015** 寫 `scripts/autoload/game_state.gd`
  - 變數：`player_hp / player_max_hp / energy / current_run`
  - 函數：`reset_for_new_run()` / `take_damage(amount)`
  - 驗收：commit + 我讀
  - 估時：1.5 hr

- [ ] **T016** 寫單元測試驗證 game_state
  - 在 `tests/test_game_state.gd` 寫 3-5 個 GdUnit4 測試
  - 驗收：跑 test 結果（pass / fail）截圖描述給我
  - 估時：1.5 hr
  - 連結：[GdUnit4 docs](https://github.com/MikeSchulze/gdUnit4)

- [ ] **T017** Commit 後寫 weekly devlog #1
  - 公開（Twitter / 巴哈 / blog 二選一）
  - 內容：這週做了什麼 / 卡住什麼 / 下週目標
  - 驗收：URL 給我，我 WebFetch 看
  - 估時：30 min

---

## W4：Scene Composition + UI + DFS skeleton

- [ ] **T018** 看完 Godot Docs Control nodes 章節
  - 驗收：我問 2 問題
  - 估時：1-2 hr

- [ ] **T019** 建 `scenes/main/main_menu.tscn`
  - 元素：Title / Start Button / Settings Button / Exit Button
  - 用 Control + VBoxContainer
  - 驗收：commit + 我讀 .tscn
  - 估時：1 hr

- [ ] **T020** 建 `scenes/combat/card_widget.tscn`
  - 元素：邊框 + 能量數字 + 名稱 + 效果文字 + illustration placeholder
  - Script 接收 CardData，UI 自動 update
  - 驗收：commit + 我讀 + 預期 1 截圖描述
  - 估時：2 hr

- [ ] **T021** 建 `scenes/combat/combat_scene.tscn`（空殼）
  - 元素：PlayerArea / EnemyArea / HandArea / EnergyDisplay / EndTurnButton
  - 還不用邏輯，純佈局
  - 驗收：commit + 我讀
  - 估時：1.5 hr

- [ ] **T022** 主選單 → 戰鬥場景跳轉 work
  - Start 按鈕 → 切到 combat_scene
  - 驗收：commit + 描述你跑了一次能切到
  - 估時：30 min

- [ ] **T023** Weekly devlog #2
  - 同 T017 格式
  - 估時：30 min

- [ ] **T024** M1 自我 milestone review
  - 對照 02-learning-path.md M1 目標：「能獨立寫出簡單 2D 遊戲」
  - 跟我說：你覺得達成了嗎 / 哪些還弱 / 要不要補
  - 驗收：對話即可
  - 估時：30 min

---

# M2（W5-W8）：DFS Phase 0-2

## W5：戰鬥邏輯起步

- [ ] **T025** 看完 [Phase 2 - Combat Prototype](https://github.com/asd23353934/dice-fate-survivor/blob/master/docs/08-implementation/steps/phase-2-combat-prototype.md) doc（原 DFS repo）
  - 驗收：我問 2 問題
  - 估時：1 hr

- [ ] **T026** 寫 `scripts/core/enemy.gd`
  - 變數：hp / max_hp / damage / status_effects
  - 函數：take_damage / heal / get_intent
  - 驗收：commit + 我讀
  - 估時：2 hr

- [ ] **T027** 寫 `data/enemies/enemy_slime.tres`
  - HP 12 / 攻 3 / intent pattern: [attack 3, attack 3, defend 4]
  - 驗收：commit + 我讀
  - 估時：30 min

- [ ] **T028** 戰鬥場景 spawn 1 個 slime
  - 用 instance scene + add_child
  - 驗收：commit + 描述
  - 估時：1 hr

- [ ] **T029** 玩家可以拖卡到敵人
  - Card widget 加 drag 邏輯
  - Drop 偵測 Area2D
  - 驗收：commit + 描述跑起來能拖
  - 估時：3 hr

- [ ] **T030** 拖 STRIKE 到 slime → slime HP - 6
  - 整合 CardData.damage → enemy.take_damage
  - 驗收：commit + 描述
  - 估時：1.5 hr

- [ ] **T031** Weekly devlog #3 + 紅燈 check
  - 進度檢查：是否落後 02-learning-path.md
  - 驗收：URL + 跟我說你的判斷
  - 估時：30 min

---

## W6：戰鬥 loop

- [ ] **T032** 玩家回合結束 → 敵人回合
  - End Turn 按鈕 → 觸發 enemy turn
  - Enemy 依 intent 攻擊玩家
  - 驗收：commit + 描述
  - 估時：3 hr

- [ ] **T033** 玩家 HP 0 → 戰鬥失敗
  - Game over screen
  - 驗收：commit
  - 估時：1.5 hr

- [ ] **T034** 敵人 HP 0 → 進下一波
  - 5 波結構基礎（hardcode 每波 1 敵）
  - 驗收：commit + 描述跑完 5 波
  - 估時：2 hr

- [ ] **T035** 第 5 波殺光 → 戰鬥勝利
  - Victory screen
  - 驗收：commit
  - 估時：1 hr

- [ ] **T036** 寫 5-10 個 combat 單元測試（GdUnit4）
  - 測 `damage 計算` / `weak/vulnerable multiplier` / `block 扣傷害` 等
  - 驗收：跑 test 結果 + commit
  - 估時：2.5 hr

- [ ] **T037** Weekly devlog #4
  - 估時：30 min

---

## W7：Combo Lock + Status

- [ ] **T038** 實作 6 個基本 status
  - poison / burn / weak / vulnerable / strength / dexterity
  - 寫 `scripts/core/status_effect.gd` + 6 個 .tres
  - 驗收：commit + 我讀
  - 估時：3 hr

- [ ] **T039** 寫 status apply / tick / 衰減邏輯
  - 玩家回合開始 tick poison
  - status 倒數歸 0 移除
  - 驗收：commit + 寫對應 unit test
  - 估時：3 hr

- [ ] **T040** 實作 Combo Lock universal 機制（招牌）
  - 看 `01-design/combat-formulas.md` §1.5
  - 連續同類卡 ×1.0 → ×1.2 → ×1.5 cap
  - 出不同類 / 回合結束 reset
  - 驗收：commit + 5 個 unit test + 跑完整 demo 看 multiplier 對
  - 估時：3 hr

- [ ] **T041** Combo Lock UI（HUD 顯示）
  - 螢幕中央上方「⚡ Combo ×N」
  - 顏色依 tier 變化（白 → 藍 → 紫）
  - 驗收：commit + 描述
  - 估時：2 hr

- [ ] **T042** Weekly devlog #5
  - 估時：30 min

---

## W8：Power 卡 + 補完 phase 2/3

- [ ] **T043** 寫 Power 卡邏輯
  - Power 卡出後留場（不進棄牌）
  - `CARD_INNER_POWER`（每回合 +1 strength）
  - `CARD_BATTLE_TRANCE`（每回合 +1 draw）
  - 驗收：commit + 2 unit test
  - 估時：2.5 hr

- [ ] **T044** Item 卡（exhaust）邏輯
  - 出後進 exhaust pile，不洗回
  - `CARD_BOMB`（全敵 12 傷 exhaust）
  - 驗收：commit + unit test
  - 估時：1.5 hr

- [ ] **T045** 從 cards-pool.md 補滿 10 張卡 .tres
  - STRIKE / DEFEND / BASH / FOCUS / QUICK_STAB / GUARD / IRON_WAVE / SLASH / HEAVY_SLASH / INSPIRE
  - 驗收：commit + 我 ls 看 10 個 .tres
  - 估時：2 hr

- [ ] **T046** M2 自我 milestone review
  - 對照 02-learning-path.md M2 目標：「Phase 0-2 完成，可玩戰鬥」
  - 跑 1 場完整戰鬥（5 波，10 張卡）→ 截圖 / 描述
  - 驗收：對話
  - 估時：1 hr

- [ ] **T047** Weekly devlog #6 + monthly summary
  - 估時：1 hr

---

# M3（W9-W13）：DFS Phase 3-4（戰鬥成熟 + 棋盤）

> 詳細 task 等你完成 M2 後我細化（依當時進度調整）。

### W9-W11 高階目標

- [ ] **T048** 補完 status effects 高階互動（順序 / 邊際）
- [ ] **T049** AnimatedSprite2D Aria 互動角色（依 03-pipeline.md）
- [ ] **T050** AI 生 Aria 立繪 5 張 + 整合
- [ ] **T051** 卡牌 hover preview（hover 顯示放大版 + 預測 dmg）
- [ ] **T052** 戰鬥動畫（hit pause / screen shake / 飄字）

### W12-W13 棋盤起步

- [ ] **T053** 棋盤場景 + 20 格 closed loop
- [ ] **T054** 玩家擲骰移動
- [ ] **T055** 5 種格子（gold / item / event / combat / shop）
- [ ] **T056** 棋盤敵人 + path encounter
- [ ] **T057** 棋盤 ↔ 戰鬥場景切換
- [ ] **T058** Weekly devlogs #7-#11 + M3 milestone review

---

# M4（W14-W17）：Phase 5-6 整合 + Hub

> 細化待 M3 完成。

- [ ] **T059** 三選一獎勵 UI
- [ ] **T060** Boss 戰（DICE_LORD 2 phase）
- [ ] **T061** Hub 場景 + 2 NPCs（訓練師 + 卡牌商）
- [ ] **T062** VN 風對話系統（dialog tree + 分支）
- [ ] **T063** 永久升級系統
- [ ] **T064** Save / Load（XOR 加密 binary）
- [ ] **T065** Settings menu
- [ ] **T066-T068** Weekly devlogs + M4 milestone

---

# M5（W18-W22）：Phase 7-8 內容 + 美術

> 細化待 M4 完成。

- [ ] **T069** 補滿 10-15 張卡 .tres + illustration
- [ ] **T070** 補滿 4-5 敵人 + DICE_LORD
- [ ] **T071** 補滿 5-6 道具
- [ ] **T072** 棋盤事件 3-5 種（VN 系統 + 分支選擇）
- [ ] **T073** QTE 卡 1-2 張
- [ ] **T074** AI 生 10-15 卡 illustration + Krita 後製
- [ ] **T075** UI Figma 設計 + 整合
- [ ] **T076** Tutorial 5 popup
- [ ] **T077** i18n 繁中 + 英文補完
- [ ] **T078** BGM 3-5 首 + SFX 20-30 個
- [ ] **T079-T081** Weekly devlogs + M5 milestone

---

# M6（W23-W26）：收尾 + 應徵

> 細化待 M5 完成。

- [ ] **T082** Bug fix 大掃除
- [ ] **T083** 平衡 auto-play 100 場
- [ ] **T084** Web export + 上 itch.io
- [ ] **T085** 60-90 秒 trailer（OBS 錄 + CapCut 剪）
- [ ] **T086** 截圖 5-8 張
- [ ] **T087** GitHub README 完整化
- [ ] **T088** Live2D 整合測試（另開 repo）
- [ ] **T089** 履歷 PDF + 1111 / 104 / yourator 登錄
- [ ] **T090** 投履歷 ≥ 10 家公司
- [ ] **T091** WorkNite 客製化求職信
- [ ] **T092** Weekly devlogs #23-#26
- [ ] **T093** M6 final milestone review

---

## 紅燈處理

如出現以下任一，**立刻告訴我**：

- 連 7 天沒 commit
- 連 2 週的週六沒開工
- 進度落後 milestone > 2 週
- 連 3 次 weekly devlog 跳過
- 想做不在 scope 的東西

我會幫你 talk it out + 調整 scope。

---

## 我會主動做的事

- 每次你回報 task done → 我驗 → 標 `[x]` + commit + 告訴你下個 3 個優先
- 每月底 → 我去看 GitHub graph + 告訴你「進度評估」
- 你超過 7 天沒消息 → 我不會主動提醒（不是 push notification 服務），但你回來找我我會看你進度給意見

---

## 相關文件

- 6 月 portfolio scope：[`01-mvp-scope.md`](01-mvp-scope.md)
- 學習路徑：[`02-learning-path.md`](02-learning-path.md)
- Pipeline：[`03-pipeline.md`](03-pipeline.md)
- 程式規範：[`04-coding-standards.md`](04-coding-standards.md)
