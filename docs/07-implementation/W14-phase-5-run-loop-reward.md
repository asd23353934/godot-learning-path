# W14 / Phase 5 (1/2)：Run loop 整合 + 三選一獎勵

> 對應 repo：[dice-fate-survivor](https://github.com/asd23353934/dice-fate-survivor)
> 對應 PROGRESS milestone：M4 W14
> 寫於 Day 7（2026-05-29）

---

## 1. 目標 / 不目標

### 目標
- **完整 1 個 run loop**：主選單 → 棋盤 → combat → reward → 回棋盤 → combat → reward → 全清 → 結算 → 主選單
- **三選一獎勵 UI**：戰勝後從全卡池隨機抽 3 張，玩家選 1 加入 deck（或 Skip）
- **persistent run deck**：reward 加的卡會跨戰鬥保留，下一場可抽到
- **stage clear 結算**：全清時顯示 HP / 金錢 / 回合 / 勝場 / 牌組大小 panel
- **prepare M4 後續整合空間**：reward signal / stats signal 留給 W15 Boss / W16 Hub / W17 Save

### 不目標（留給後續）
- Boss 戰（W15）
- Run end 詳細結算（玩家死亡時，W15 補 panel）
- Reward 加 rarity / 卡牌移除 / 升級分頁（W18-W19 內容期）
- 多 stage 串接（W15 或 M5 整合）
- Card preview 大圖 / 動畫（M5 美術期）

---

## 2. 設計決策（選 X 不選 Y 的理由）

### A. 為什麼新獨立 reward scene，不在 combat scene 內 popup？

| 方案 | 優點 | 缺點 |
|---|---|---|
| **A. 獨立 scene（採用）** | 流程清晰、SceneRouter fade 自然、reward 可獨立擴充 rarity/排版 | 多一次 scene 切換成本 |
| B. Combat 內 popup CanvasLayer | 無 scene 切換 | combat.gd 變得肥（戰鬥邏輯 + UI flow 都在），單一職責崩 |

→ **A 勝**：DFS 早期就走 SceneRouter pattern（主選單/棋盤/戰鬥），多一個節點符合慣例，且 reward 之後要擴大成「卡牌移除 / 升級」共用入口。

### B. 為什麼 reward UI 不複用 Card.tscn？

- Card.tscn 內建 `_get_drag_data`（戰鬥拖拉用）
- 在 reward 場景拖卡無意義，但會誤觸 drag preview，玩家會困惑
- 解法考慮：
  - 改 Card.gd 加 `is_interactive: bool = true` flag（侵入式）
  - 改 mouse_filter = MOUSE_FILTER_IGNORE 並另起 Button overlay（hack）
  - **新做純 Button + 動態 Label 填卡資訊（採用）** — 單向資料流、無耦合
- 代價：reward 卡片視覺跟戰鬥不一樣，但對 prototype 階段無所謂；M5 美術期統一改

### C. 為什麼把 deck 分成 `current_run_deck`（persistent）與 `deck`（per-combat shuffle）？

關鍵語意分離：

| 名稱 | 生命週期 | 用途 |
|---|---|---|
| `current_run_deck` | 整個 run | 玩家擁有的所有卡，reward 加卡會 grow |
| `deck` | 戰鬥內 | 抽牌堆（每戰鬥重洗，戰鬥結束清空） |
| `discard` | 戰鬥內 | 棄牌堆 |
| `hand` | 戰鬥內 | 手牌 |

戰鬥開始時 `deck = current_run_deck.duplicate()` + shuffle — duplicate 避免戰鬥洗牌污染 persistent state。

> Vue 類比：currentRunDeck 像 Vuex store（持久），deck 像 component-local computed（每次 mount 重算）。

### D. 為什麼勝場 +1 放 combat 而不放 board？

- 勝場是「戰鬥內的事件」，combat scene 最知道
- 放 board 要靠 EventBus 訂閱，多一層 indirection
- combats_won_this_run 之後拿來 unlock / scoring — 算戰鬥成就，放 combat 語意更乾淨

### E. 為什麼 player 死亡不進 reward？

- Reward 是「勝利的獎賞」
- Run end 邏輯應該另起 panel（W15 補）
- 現在 player 死直接回 main menu，UX 略糙但 W14 scope 不處理

---

## 3. 檔案清單

### 新增

| 檔案 | 角色 |
|---|---|
| `scenes/reward/reward.tscn` | Reward 場景：標題 + 3 個 Slot Button + Skip + Status label |
| `scripts/reward/reward.gd` | Reward 邏輯：抽 3 張、填 UI、click → 加卡 → 切 board |

### 修改

| 檔案 | 變動 |
|---|---|
| `scripts/autoload/game_state.gd` | +`current_run_deck` / +`combats_won_this_run` / +`add_card_to_run_deck` / `reset_for_new_run` build deck |
| `scripts/autoload/event_bus.gd` | +4 signal: `reward_offered` / `reward_selected` / `reward_skipped` / `stage_cleared_with_stats` / `run_deck_changed` |
| `scripts/autoload/card_database.gd` | +`get_reward_options(n)` |
| `scenes/combat/combat.gd` | 用 `current_run_deck` / 勝利 → reward scene / `combats_won_this_run++` |
| `scenes/board/board.gd` | 全清 → `_show_stage_clear_panel`（取代 timer 回主選單） |
| `scenes/board/board.tscn` | +`StageClearPanel`（初始 hidden） |

---

## 4. 實作流程（建構順序）

1. **GameState 加 state**：`current_run_deck` / `combats_won_this_run`
   - 不加 reward state 在 GameState — reward 是 UI 層，只透過 method 操作 GameState
2. **EventBus 加 signal**：reward 4 個 + run_deck_changed
   - 全集中宣告，未來別處想監聽 reward（成就系統 / Codex）零成本
3. **CardDatabase 加 `get_reward_options`**：random pool 抽 n 張
   - 之後可加 rarity filter / weight / kit-specific pool
4. **Reward scene + script**：純 Button + 動態 fill
5. **combat 改 deck 來源 + 勝利導流**
6. **board stage clear panel**

順序原則：**autoload 先（資料層）→ scene 後（UI 層）**。autoload 不依賴 scene，反之依賴單向。

---

## 5. 關鍵 code 解析

### 5.1 `GameState.reset_for_new_run` build run deck

```gdscript
func reset_for_new_run(kit: String = "KIT_SWORD") -> void:
    ...
    combats_won_this_run = 0   # W14：勝場歸零

    # W14：build persistent run deck
    current_run_deck = CardDatabase.build_starter_deck(kit)
    EventBus.run_deck_changed.emit(current_run_deck.duplicate())

    run_count += 1
    ...
```

要點：
- `combats_won_this_run = 0` 在 W14 加，跟 `run_count += 1` 等其他 reset 同層級
- 用 `duplicate()` emit，避免外部監聽者拿到 mutable ref 誤改

### 5.2 `add_card_to_run_deck`

```gdscript
func add_card_to_run_deck(card: CardData) -> void:
    if card == null:
        return
    current_run_deck.append(card)
    EventBus.run_deck_changed.emit(current_run_deck.duplicate())
    print("[GameState] 加入新卡 %s，run deck 現 %d 張" % [card.card_name, current_run_deck.size()])
```

- null 防呆（reward UI 出錯時保護）
- emit duplicate 防外部誤改

### 5.3 `CardDatabase.get_reward_options`

```gdscript
func get_reward_options(count: int = 3) -> Array[CardData]:
    var pool = get_all_cards()
    pool.shuffle()
    var result: Array[CardData] = []
    for i in range(min(count, pool.size())):
        result.append(pool[i])
    return result
```

- `get_all_cards()` 回 array of CardData，shuffle 後取前 n
- `min(count, pool.size())` 防卡池小於請求數
- 之後升級點：加 `exclude: Array[String]` 參數排除手上已有的卡

### 5.4 Reward scene 主 flow

```gdscript
func _ready() -> void:
    _options = CardDatabase.get_reward_options(3)
    EventBus.reward_offered.emit(_options.duplicate())

    %SkipButton.pressed.connect(_on_skip_pressed)

    var slots = [%Slot1, %Slot2, %Slot3]
    for i in range(3):
        var slot: Button = slots[i]
        var card: CardData = _options[i]
        _populate_slot(slot, card)
        slot.pressed.connect(_on_card_selected.bind(card))
```

- `.bind(card)` 是 GDScript 的 partial application — 預先綁 card 參數
- 等同 Vue 寫 `@click="onSelect(card)"`，但用 signal 連接
- 不能直接寫 `slot.pressed.connect(_on_card_selected(card))` — 那是「立刻呼叫」

### 5.5 防 race condition：選後禁用所有按鈕

```gdscript
func _on_card_selected(card: CardData) -> void:
    GameState.add_card_to_run_deck(card)
    EventBus.reward_selected.emit(card)
    %StatusLabel.text = "已加入：%s" % card.card_name
    _disable_all_slots()
    await get_tree().create_timer(0.5).timeout
    SceneRouter.change_scene("res://scenes/board/board.tscn")
```

- 0.5 秒視覺回饋，避免「按下沒反應」感
- 在 await 前先 `_disable_all_slots()`，防玩家在過渡期重複點

### 5.6 combat 勝利改導 reward

```gdscript
func _on_enemy_died() -> void:
    %MessageLabel.text = "VICTORY!"
    %MessageLabel.show()
    %EndTurnButton.disabled = true
    GameState.combats_won_this_run += 1   # W14
    EventBus.combat_ended.emit(true)
    await get_tree().create_timer(1.2).timeout
    SceneRouter.change_scene("res://scenes/reward/reward.tscn")   # W14：取代 board
```

- timer 縮到 1.2 秒（W13 是 1.5）— reward scene 自己有 0.5 秒回饋，避免總體拖太久
- player_died 路徑不動（W15 補 run end panel）

### 5.7 board stage clear panel

```gdscript
func _on_all_enemies_cleared() -> void:
    %TileInfoLabel.text = "🎉 STAGE CLEARED！"
    %RollDiceButton.disabled = true
    EventBus.board_stage_cleared.emit()
    _show_stage_clear_panel()

func _show_stage_clear_panel() -> void:
    var stats_text = "存活 HP：%d / %d\n金錢：%d\n累計回合：%d\n戰勝場次：%d\n當前牌組：%d 張" % [
        GameState.player_hp, GameState.max_hp,
        GameState.gold, GameState.turn_number,
        GameState.combats_won_this_run,
        GameState.current_run_deck.size(),
    ]
    %StatsLabel.text = stats_text
    EventBus.stage_cleared_with_stats.emit({...})
    %StageClearPanel.show()
    %StageClearMenuButton.pressed.connect(_on_to_menu_pressed)
```

- W13 是 `timer 2 秒 → 自動回主選單`，W14 改 panel + 玩家手動點按鈕
- 多兩個動作（看數據 + 點按鈕）讓 stage end 有儀式感
- emit `stage_cleared_with_stats(Dictionary)` — Dictionary 傳值靈活，之後新增欄位不破 API

---

## 6. 觀念對照（前端 / Vue / Angular）

| Godot 概念 | Vue / Angular 類比 |
|---|---|
| `current_run_deck` autoload var | Vuex/Pinia store state（跨 component 持久） |
| `EventBus.reward_offered` signal | event bus / RxJS Subject |
| `Callable.bind(card)` | 等同 `() => onSelect(card)` partial function |
| SceneRouter.change_scene | router.push('/reward') |
| Panel `visible = false` 初始 | `v-if="showPanel"` 條件渲染 |
| `await timer.timeout` | `await new Promise(r => setTimeout(r, 500))` |
| `Array.duplicate()` | `[...array]` shallow copy |

---

## 7. 擴充方式（之後加 X 怎麼動）

### 加新 reward 來源（Boss reward）

W15 Boss 死亡後，給高品質獎勵：

```gdscript
# CardDatabase.gd
func get_boss_reward_options(count: int = 3) -> Array[CardData]:
    var pool = get_all_cards().filter(func(c): return c.rarity >= 2)
    pool.shuffle()
    return pool.slice(0, count)
```

reward.gd 加 `var is_boss_reward: bool`，setter 切換 `CardDatabase.get_boss_reward_options`。

### 加「刪卡」reward

```gdscript
# game_state.gd
func remove_card_from_run_deck(card: CardData) -> void:
    current_run_deck.erase(card)
    EventBus.run_deck_changed.emit(current_run_deck.duplicate())
```

新 scene `scenes/reward/remove_card.tscn` 顯示 run deck 列表，click 一張 → remove。

### 加 rarity 系統

```gdscript
# card_data.gd
enum Rarity { COMMON, UNCOMMON, RARE }
@export var rarity: Rarity = Rarity.COMMON

# card_database.gd
func get_reward_options(count: int = 3, rarity_weights: Dictionary = {0: 60, 1: 30, 2: 10}) -> Array[CardData]:
    # 依權重抽
```

### 加 stage 串接（W15）

```gdscript
# board.gd
func _on_next_stage_pressed() -> void:
    GameState.floor_number += 1
    GameState.board_enemies = [3, 8, 14, 19]  # 第二 stage 4 隻
    SceneRouter.change_scene("res://scenes/board/board.tscn")
```

`board.tscn` StageClearPanel 加「下一 stage」按鈕，連這個 method。

### 加 Hub 結算（W16）

Hub scene 訂閱 `EventBus.stage_cleared_with_stats`，把 gold 累積到 permanent currency。

---

## 8. 常見錯誤（踩雷 + 修法）

### 8.1 Reward 加的卡沒在下一場戰鬥出現

**原因**：combat.gd 沒從 `current_run_deck` 載，還是用 `CardDatabase.build_starter_deck`。

**檢查**：
```gdscript
# combat.gd _ready 應該長這樣
if GameState.current_run_deck.is_empty():
    var kit = ...
    GameState.current_run_deck = CardDatabase.build_starter_deck(kit)
print("[Combat] 載入 run deck %d 張" % GameState.current_run_deck.size())
GameState.start_combat_with_deck(GameState.current_run_deck)
```

### 8.2 Reward Skip 後 stage clear 直接觸發但 panel 沒顯示

**原因**：board scene reload 後 `%StageClearMenuButton.pressed` 已連過一次，第二次 `.connect()` 會報「Signal already connected」。

**修法**：board._ready 結尾的 `_show_stage_clear_panel` 改成檢查：
```gdscript
if not %StageClearMenuButton.pressed.is_connected(_on_to_menu_pressed):
    %StageClearMenuButton.pressed.connect(_on_to_menu_pressed)
```

W14 目前用 1-shot pattern（panel 顯示後玩家手動點 → 回主選單，整個 scene 銷毀），不會二次 enter，所以不會踩到。但 W15 之後加「下一 stage」就要小心。

### 8.3 `Callable.bind` 在 for loop 內全部綁到最後一個 card

**症狀**：3 個 Slot 點起來都加同一張卡。

**原因**：閉包捕獲 loop variable 的經典坑。GDScript 4.6 的 `for i in range(...)` 每次迭代有新作用域，所以 `var card = _options[i]` 是 fresh — **GDScript 沒有 JS pre-ES6 var 的問題**。

但如果寫成：
```gdscript
var card  # 外面宣告
for i in range(3):
    card = _options[i]
    slot.pressed.connect(_on_card_selected.bind(card))   # ❌ 所有 bind 都吃同一個 card ref
```

那就會踩到。建議：**在 loop 內 var**。

### 8.4 reward scene 開了但 3 個 Slot 是空的

**原因**：`get_node("VBoxContainer/NameLabel")` path 寫錯（複製貼上沒換）。

**修法**：用 unique_name_in_owner + `%` 抓也可以，但 Slot1/2/3 各自有 NameLabel 名字會撞。W14 用相對 path 比較乾淨。

### 8.5 `combats_won_this_run` 沒歸零，第二 run 還累積

**修法**：確保 `reset_for_new_run` 內加 `combats_won_this_run = 0`。

---

## 9. 下游依賴（之後哪些功能會 build on this）

| 後續週 | 依賴 W14 的什麼 |
|---|---|
| **W15 Boss** | combat 勝利 → reward 流程（Boss 用同流程，可能用 `get_boss_reward_options`） |
| **W15 Run end panel** | 玩家死亡的 panel pattern 參考 W14 stage clear panel |
| **W16 Hub** | `stage_cleared_with_stats` signal → Hub 累積 permanent gold |
| **W17 Save** | `current_run_deck` 要序列化（用 card.id 存 array） |
| **W18-W19 內容** | reward pool 直接從 CardDatabase 出，加卡只要丟 .tres 進 `data/cards/` |
| **W19 棋盤事件** | event tile 也可能用 reward scene 變體（找寶箱 → 3 選 1） |

---

## 10. 面試 talking point（30 秒可講完）

> 「W14 我把 DFS 從『單戰鬥 prototype』升級到『完整 run loop』。
>
> 設計重點：把 deck 分成 `current_run_deck`（persistent，跨戰鬥）跟戰鬥用 `deck`（每場 shuffle 一份 duplicate）。reward 加卡只 mutate persistent 那層，戰鬥內 shuffle 不會污染。
>
> 流程：戰勝 → reward scene 從全卡池隨機抽 3 張 → 玩家選 1 加進 run deck → 回棋盤 → 下一場戰鬥就抽得到。
>
> 全清時開結算 panel，emit `stage_cleared_with_stats(Dictionary)` signal，給之後 Hub meta-progression 訂閱。Dictionary 而不是固定參數 — 之後加欄位不破 API。
>
> 沒複用 Card.tscn 做 reward 視覺，因為 Card 有 `_get_drag_data`，在 reward 場景拖卡無意義會誤觸 — 改純 Button + 動態 Label，單向資料流、零耦合。」

---

**M4 (W14-W17) 進度：25% (1/4)**
**下一週 W15**：Boss 戰（DICE_LORD 簡化 2 phase）+ Run end panel 完整化
