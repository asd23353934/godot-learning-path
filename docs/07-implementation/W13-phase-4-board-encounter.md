# W13 / DFS Phase 4 (2/2)：棋盤敵人 + Path Encounter + 勝利條件（M3 收尾）

> 對應 PROGRESS.md W13 / DFS docs/08-implementation/steps/phase-4-board-prototype.md（Steps 4.8 + 4.9 + 4.11 部分）
> 完成日：2026-05-28
> Repo：[dice-fate-survivor](https://github.com/asd23353934/dice-fate-survivor)

---

## 1. 目標 / 不目標

### 目標

- **3 隻棋盤敵人** spawn 在 tile 4 / 11 / 17（hardcode 起始位置）
- **視覺**：每個 enemy tile 顯示暗紅色標記
- **Path encounter**：玩家移動 per-step 檢測，**走到** enemy tile 就停下 + 觸發戰鬥
- **戰鬥銜接**：勝利後該 enemy 從 board 永久消失（GameState 同步）
- **勝利條件**：3 隻敵人全清 → STAGE CLEARED → 2 秒回主選單
- **UI**：剩餘敵人計數 + 回合計數

### 不目標（W14+ 補）

- **敵人在棋盤上移動**（Step 4.8 完整版）：W13 enemy 固定不動，W14 加 random AI
- **完整 5 波結構**（Step 4.10）：W13 hardcode 3 隻 vs W14+ 每輪 spawn
- **Boss 第 10 輪**：W14 整合階段做
- **棋盤 ITEM / SHOP / EVENT 完整機制**（Steps 4.6, 4.7）：W14-W15 補
- **戰鬥失敗處理**：仍是直接回主選單（簡化版），W14 加 reward 流程

---

## 2. 設計決策

### 為什麼 board enemy ≠ COMBAT tile？

DFS 有**兩種戰鬥觸發源**：

| 來源 | 行為 | W13 範例 |
|---|---|---|
| **COMBAT tile**（W12） | 一次性事件格，踩到觸發 | tile 3 / 8 / 12 / 17 |
| **Board enemy**（W13） | 持久 entity 在板上，戰勝後永久消失 | tile 4 / 11 / 17 |

**為什麼分開**：
- COMBAT tile 像「**陷阱 / 事件**」（一格固定設計）
- Board enemy 像「**怪物 spawn**」（**會減少 / 累積 / 移動**，是 roguelike 核心壓力源）
- DFS 棋盤敵人是「**倖存者壓力**」設計核心（spec 2026-05-15）

### 為什麼用 **per-step collision check**？

```gdscript
for i in range(steps):
    # 走一格
    GameState.current_tile_index = next_index
    if next_index in GameState.board_enemies:
        encountered = true
        break    # ← 剩餘步數作廢
```

**選項對照**：

| 方案 | 行為 | 缺點 |
|---|---|---|
| **per-step 檢測**（採用） | 路徑經過敵人即停 | code 稍多 |
| 只檢測終點 | 跳過敵人也算遭遇？需另外判 | 不直覺（玩家覺得「我跨過去了」就該安全） |
| 路徑全跑完再算 | 純算到終點 | 反 Slay the Spire 規則 |

**per-step 是業界標準**（Mario Party、桌遊「大富翁」風格）。玩家走「**經過敵人**」應該停下，符合直覺。

### 為什麼用 `pending_remove_enemy_tile` 而不是直接刪？

**問題**：玩家撞 enemy 時 board 還 active。如果立刻 `GameState.board_enemies.erase(...)` → enemy 消失 → 切到 combat scene → 玩家輸了還是贏了？

**解法**：暫存 "pending remove"：
- 撞 enemy → 設 `pending_remove_enemy_tile = current_tile`
- 切 combat
- combat 結束（VICTORY only）→ 切回 board
- board._ready 看到 pending_remove 不是 -1 → 移除 enemy → reset pending

如果 combat 失敗（player_died）：pending 不被清，但反正切到主選單也不會走到 board._ready 的 remove 邏輯。下次玩家新 run 會 reset_for_new_run 重設一切。

**這是「跨場景兩階段提交」pattern** — 操作 commit 在「實際發生」的場景，不在「決定」的場景。

對應前端：API 「optimistic update」 vs 「server confirm」分離。

### 為什麼勝利條件 = 全清而不是「第 10 回合 boss」？

DFS spec 是「**第 10 輪 boss 結算**」，但 W13 簡化：
- W13 無 boss 機制
- W13 無 enemy 移動，所以「敵人累積」邏輯沒實現
- **「全清 3 隻 hardcode」是 MVP demo 路徑**

**W14 升級時**：
- 改成「擲骰用完 N 次進下一輪」+「第 10 輪 spawn boss」
- 全清條件改為「第 10 輪殺 boss」

W13 hardcode 3 隻 trial = 玩家能玩到完整 win / lose 流程。

### 為什麼 enemy 視覺是**小暗紅點** 不是大 sprite？

- W13 美術 placeholder 階段
- 暗紅色（Color(0.2, 0.05, 0.05)）跟 tile 顏色（GOLD 黃 / EMPTY 灰）對比夠
- 放 tile 右上角不擋 tile name label
- 24×24 px 視覺夠明顯

**W20 美術階段**：換成 enemy AnimatedSprite2D + 動畫。

### Enemy 起始位置 hardcode 4 / 11 / 17 — 為什麼這 3 個？

挑「**均勻分布在 perimeter**」的 3 格：
- tile 4 = 頂排中段（玩家從 0 走 4 步遇到）
- tile 11 = 底排右段
- tile 17 = 左側中段（玩家繞完一圈會遇到）

**設計意圖**：玩家擲 1d6，平均 3.5 步 → 第一次擲骰大概率撞到 tile 4 → 立刻體驗 path encounter（不用走半圈才碰）。

**W14 升級**：改成 random spawn position，每 run 不同位置。

### Turn counter 為什麼從 0 開始不從 1？

GameState.turn_number 預設 0，每次擲骰 +1，**所以第一次擲骰後 = 回合 1**。

如果預設 1，第一次擲後 = 回合 2，玩家會困惑。

預設 0 + 擲骰 +1 = **第 N 次擲骰 = 回合 N**，最直覺。

---

## 3. 檔案清單

```
修改：
├── scripts/autoload/game_state.gd      ← 加 board_enemies / pending_remove_enemy_tile / turn_number
├── scripts/autoload/event_bus.gd       ← 加 enemy_removed_from_board / board_stage_cleared signals
├── scenes/board/board.gd               ← +75 行：render enemies / per-step check / encounter trigger / 全清勝利
└── scenes/board/board.tscn             ← 加 EnemiesLeftLabel / TurnLabel
```

---

## 4. 實作流程 / 順序

```
1. GameState 加 3 個欄位：board_enemies / pending_remove_enemy_tile / turn_number
2. reset_for_new_run 設 board_enemies = [4, 11, 17]（hardcode 起始位置）
3. EventBus 加 enemy_removed_from_board / board_stage_cleared signals
4. board.gd：
   · _ready 開頭：處理 pending_remove（戰勝返場時觸發）
   · _render_board_enemies：為每 tile_index 加暗紅 ColorRect 標記
   · _ready 末：檢查 board_enemies.is_empty() → 全清勝利
5. board.gd _move_player 改：
   · 回 bool（true = 撞到敵人提前結束）
   · per-step check `if next_index in GameState.board_enemies`
6. board.gd 加 _trigger_board_encounter：設 pending_remove + 切 combat
7. _on_roll_dice_pressed 改：根據 encountered_enemy 分流（戰鬥 vs tile resolve）
8. _on_roll_dice_pressed 開頭 turn_number += 1
9. _refresh_ui 加 EnemiesLeftLabel / TurnLabel
10. board.tscn 加 EnemiesLeftLabel / TurnLabel UI
11. F5 驗證 → 全清 STAGE CLEARED → 回主選單
12. 寫 W13 implementation guide
13. PROGRESS.md 加 M3 retro
14. 雙 commit + push
```

---

## 5. 關鍵 code 解析

### `_move_player` 改為回 bool + per-step check

```gdscript
func _move_player(steps: int) -> bool:
    # W13 改：回 bool（true = 遭遇 board enemy 提前停下）
    _is_moving = true
    %RollDiceButton.disabled = true

    var encountered_enemy = false
    for i in range(steps):
        var next_index = (GameState.current_tile_index + 1) % 20
        var target_pos = _tile_positions[next_index] - _player_avatar.size / 2

        var tween = create_tween()
        tween.tween_property(_player_avatar, "position", target_pos, MOVE_DURATION_PER_TILE)
        await tween.finished

        GameState.current_tile_index = next_index
        EventBus.player_moved_to_tile.emit(next_index, _tile_configs[next_index][0])

        # W13：per-step 檢測 board enemy
        if next_index in GameState.board_enemies:
            encountered_enemy = true
            break   # 撞到，剩餘步數作廢

    _is_moving = false
    %RollDiceButton.disabled = false
    _refresh_ui()
    return encountered_enemy
```

**重點**：
- 回 bool 給 caller，**用 return 值溝通 比 emit signal 簡單**（caller 同一個 function 立刻要決定）
- `if next_index in GameState.board_enemies` — `in` 運算 GDScript 對 Array 是 O(n) but n 才 3 個，無感
- `break` 立刻停 + 剩餘步數作廢（Mario Party 風格）

### `_trigger_board_encounter` — 兩階段提交

```gdscript
func _trigger_board_encounter() -> void:
    GameState.pending_remove_enemy_tile = GameState.current_tile_index
    %TileInfoLabel.text = "遭遇棋盤敵人！戰鬥開始..."
    print("[Board] 路徑遭遇 tile %d 的敵人 → combat" % GameState.current_tile_index)
    await get_tree().create_timer(0.6).timeout
    SceneRouter.change_scene("res://scenes/combat/combat.tscn")
```

**設計**：
- pending_remove **不立刻刪 enemy** → combat 還沒打贏前 enemy 還在
- 0.6 秒延遲 → 玩家看清「遭遇」訊息
- SceneRouter fade out → combat scene

### `_ready` 開頭處理 pending_remove

```gdscript
func _ready() -> void:
    # ... 建 layout 跟 visuals
    _place_player_at_tile(GameState.current_tile_index)

    # W13：處理戰鬥結束的 pending remove
    if GameState.pending_remove_enemy_tile != -1:
        var tile_to_clear = GameState.pending_remove_enemy_tile
        GameState.board_enemies.erase(tile_to_clear)
        EventBus.enemy_removed_from_board.emit(tile_to_clear)
        GameState.pending_remove_enemy_tile = -1
        print("[Board] 戰鬥勝利，清除 tile %d 的敵人，剩 %d 隻" % [
            tile_to_clear, GameState.board_enemies.size()
        ])

    # W13：渲染剩餘 board enemies
    _render_board_enemies()

    # W13：勝利條件檢查
    if GameState.board_enemies.is_empty():
        _on_all_enemies_cleared()
        return
    # ... 其他 setup
```

**順序很重要**：
1. 先 place player（無論如何要先放）
2. 處理 pending remove（如果有）
3. render enemies（基於 updated board_enemies）
4. 檢查全清（基於 updated 數量）

如果順序錯 → 可能 render 還沒移除的 enemy / 全清檢查跑在 remove 之前。

### `_render_board_enemies` clean + rebuild

```gdscript
func _render_board_enemies() -> void:
    # 清掉舊 visuals
    for tile_idx in _enemy_visuals.keys():
        _enemy_visuals[tile_idx].queue_free()
    _enemy_visuals.clear()

    # 為每個 board enemy 加 visual
    for tile_idx in GameState.board_enemies:
        var marker = ColorRect.new()
        marker.color = Color(0.2, 0.05, 0.05, 1)
        marker.size = Vector2(24, 24)
        marker.position = _tile_positions[tile_idx] + Vector2(15, -30)
        add_child(marker)
        _enemy_visuals[tile_idx] = marker
```

**Reactive pattern**：跟 W8 Hand 一樣，**整把 clear + rebuild**。3 隻 enemies 範圍可接受。

**marker 位置 `Vector2(15, -30)`**：tile 右上角 offset，不擋 tile name label。

### 勝利條件處理

```gdscript
func _on_all_enemies_cleared() -> void:
    %TileInfoLabel.text = "🎉 STAGE CLEARED！"
    %RollDiceButton.disabled = true
    EventBus.board_stage_cleared.emit()
    print("[Board] 全清！回主選單")
    await get_tree().create_timer(2.0).timeout
    SceneRouter.change_scene("res://scenes/main_menu.tscn")
```

**重點**：
- disable button → 防止玩家在過場時繼續擲骰
- 2 秒延遲讓玩家看到 STAGE CLEARED 訊息
- 切回主選單（W14 改成 reward 場景）

---

## 6. 觀念對照（前端視角）

| Godot | 前端 / 一般軟體 |
|---|---|
| `pending_remove_enemy_tile` 跨場景傳遞 | sessionStorage / URL params / Redux persisted state |
| `_render_board_enemies` clear + rebuild | React component re-render |
| per-step collision check | path raytracing / segment intersection |
| `enemy_removed_from_board` signal | DELETE API → store update → UI re-render |
| Win condition 檢查在 _ready | route guards / mounted hook 判 state |
| ColorRect placeholder for enemy | Material UI Avatar with default icon |

---

## 7. 擴充方式

### Enemy 在棋盤移動（W14 Step 4.8 完整版）

```gdscript
# board.gd 加：玩家動完 enemy 動
func _on_roll_dice_pressed():
    # ... player moves
    await _enemies_move_turn()

func _enemies_move_turn():
    var new_positions: Array[int] = []
    for enemy_tile in GameState.board_enemies:
        var enemy_roll = randi_range(1, 3)   # enemy 1d3 較慢
        var new_pos = (enemy_tile + enemy_roll) % 20
        # 檢查移動路徑是否經過玩家
        for step in range(1, enemy_roll + 1):
            var check_pos = (enemy_tile + step) % 20
            if check_pos == GameState.current_tile_index:
                # 撞到玩家
                _trigger_board_encounter_from_enemy(enemy_tile)
                return
        new_positions.append(new_pos)
    GameState.board_enemies = new_positions
    _render_board_enemies()
```

### Wave 系統（W14 Step 4.10）

```gdscript
# GameState 加
var current_round: int = 1
var dice_rolls_per_round: int = 20
var rolls_used_this_round: int = 0

# board.gd 加
func _on_roll_dice_pressed():
    rolls_used += 1
    if rolls_used >= 20:
        _advance_to_next_round()

func _advance_to_next_round():
    current_round += 1
    rolls_used = 0
    # spawn 新 enemy
    _spawn_enemies_for_round(current_round)
    if current_round == 5:
        _spawn_elite()
    if current_round == 10:
        _spawn_boss()
```

### Boss 機制（W14）

```gdscript
# GameState
var board_boss_active: bool = false

func _spawn_boss():
    board_boss_active = true
    GameState.board_enemies.append(10)   # boss 在中央底
    # boss 視覺：更大、紫色等
```

Boss 用同樣 combat scene（但載入更難的 enemy data）。

### 戰鬥失敗 reward 流程（W14 Step 4.11）

```gdscript
# combat.gd _on_player_died 改：
# W14 加：先去 reward / shop 場景，不是直接主選單
SceneRouter.change_scene("res://scenes/reward/death_screen.tscn")
```

`death_screen.tscn` 顯示「失敗 + 取得 X gold meta」+ 主選單按鈕。

### ITEM 機制（W14 Step 4.6）

```gdscript
# scripts/data/item_data.gd
class_name ItemData
extends Resource

@export var id: String
@export var item_name: String
@export var heal_amount: int = 0
@export var bonus_max_hp: int = 0

# data/items/small_potion.tres
# GameState
var inventory: Array[ItemData] = []  # 已有空 list

func add_item(item: ItemData):
    if inventory.size() >= 6: return false
    inventory.append(item)
    EventBus.item_added.emit(item)
    return true

# board.gd _resolve_current_tile
TileType.ITEM:
    var item = ItemDatabase.get_random()
    if GameState.add_item(item):
        %TileInfoLabel.text = "+%s" % item.item_name
    else:
        %TileInfoLabel.text = "背包滿，無法獲得 %s" % item.item_name
```

跟 CardDatabase 同 pattern，新增 ItemDatabase autoload。

### SHOP / EVENT 機制（W14）

```gdscript
TileType.SHOP:
    SceneRouter.change_scene("res://scenes/shop/shop.tscn")   # 新 scene

TileType.EVENT:
    var event_data = EventDatabase.get_random()
    _show_event_popup(event_data)   # AcceptDialog 顯示文字 + 選項
```

EVENT 之後 W16 用 Dialogic 重做（VN 風）。

---

## 8. 常見錯誤 / debug 指南

### Combat 勝利後 enemy 沒消失

**症狀**：戰勝回 board，enemy 還在原 tile
**根因**：
1. pending_remove_enemy_tile 沒設（_trigger_board_encounter 漏寫）
2. board._ready 沒處理 pending_remove
**修法**：print debug `[Board] pending_remove = %d` 在 _ready 看值對不對

### 玩家撞 enemy 後繼續走完剩餘步數

**症狀**：擲 5、走到第 2 格遇 enemy，但繼續走完 5 格才停
**根因**：_move_player for loop 沒 `break`
**修法**：if encountered 後 `break` 確認有寫

### Combat 勝利後 enemy 消失，但**所有 board enemies 都被清空**

**症狀**：踩 enemy 1 個，戰勝後 3 個全消失
**根因**：用 `GameState.board_enemies.clear()` 而非 `.erase(specific_tile)`
**修法**：用 erase 不要 clear

### 玩家死亡後仍切回 board

**症狀**：戰鬥失敗，board scene 開啟，玩家在 0 HP 漂浮
**根因**：combat 的 _on_player_died 切到 main_menu 應該對。檢查 SceneRouter 對的 path
**修法**：W7-W8 寫的 _on_player_died 應該正確，看 Output

### 進場景 STAGE CLEARED 立刻觸發（沒玩過）

**症狀**：剛進 board 就 STAGE CLEARED
**根因**：reset_for_new_run 沒 set board_enemies，或 set 成空 array
**修法**：reset_for_new_run 內確認 `board_enemies = [4, 11, 17]`

### Enemy marker 跟 tile 文字重疊

**症狀**：暗紅 marker 蓋住「金錢 / 戰鬥」字
**根因**：marker position 設錯
**修法**：marker 位置 = tile_position + Vector2(15, -30) 在右上角

### Turn counter 永遠 0

**症狀**：擲了好多次「回合 0」不變
**根因**：_on_roll_dice_pressed 沒 `GameState.turn_number += 1`
**修法**：roll button handler 開頭加 increment

---

## 9. 下游依賴

| 之後階段 | 怎麼用 W13 成果 |
|---|---|
| **W14 Phase 5 (1/2)** | 補完 Step 4.8（enemy 移動）+ Step 4.10（wave 結構）+ Step 4.11（reward 流程） |
| W14 reward 三選一 | board enemy 戰勝改成切 reward 場景，不是直接回 board |
| W15 Boss | 第 10 輪 boss 用 board_enemies append [center_tile]，combat 載 boss data |
| W16 Hub VN | event_tile 改用 Dialogic 顯示分支對話 |
| W17 Save | 序列化 board_enemies / current_tile_index / current_round |
| W18 內容 | 加 EnemyBoardData.tres pool（多種 mob / elite / boss） |
| W19 內容 | 加 EventData.tres pool（VN 風事件） |
| W20 美術 | enemy markers 改 AnimatedSprite2D（敵人動畫） |
| W22 polish | enemy spawn 進場動畫 / encounter shake effect / 勝利特效 |

---

## 10. 面試 talking point

> 「W13 我做棋盤敵人 + path encounter 機制。3 隻敵人 spawn 在固定 tile，玩家擲骰移動時 **per-step 檢測**，**走到** 任一敵人格立刻停 + 觸發戰鬥。剩餘步數作廢。
>
> 設計重點：
>
> **一、兩階段提交跨場景**。玩家撞 enemy → 設 GameState.pending_remove_enemy_tile → 切 combat scene → 戰勝回 board → board._ready 偵測 pending_remove 然後執行 erase。**操作 commit 在「實際發生」的場景**（board），不在「決定」的場景（combat）。對應前端 optimistic update + server confirm pattern。
>
> **二、board enemy ≠ COMBAT tile**。前者是 persistent entity（可移動 / 累積 / 被消滅），後者是事件格（觸發完事件 tile 仍在）。**DFS 棋盤敵人是「倖存者壓力」設計核心** — 殺不完會累積。
>
> **三、per-step collision 是業界標準**（Mario Party / 大富翁風格）。玩家走「**經過敵人**」應該停下，符合直覺。
>
> **四、視覺 placeholder vs 邏輯完整**。enemy marker 是 24×24 ColorRect 暗紅色，**美術 W20 換 AnimatedSprite2D 不動 code**。邏輯（spawn / move / collide / remove）這週做完。
>
> 勝利條件：3 隻全清 → STAGE CLEARED → 回主選單。W14 升級 wave 結構：第 5 輪精英、第 10 輪 boss、20 dice rolls / round。
>
> 這個 pattern 撐到 release — 加更多敵人、加 boss、加 enemy 移動 AI 都不用大改架構。」
