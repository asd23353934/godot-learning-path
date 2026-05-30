# W12 / DFS Phase 4 (1/2)：棋盤 prototype（20 格 closed loop + 擲骰 + 戰鬥銜接）

> 對應 PROGRESS.md W12 / DFS docs/08-implementation/steps/phase-4-board-prototype.md（Steps 4.1-4.5）
> 完成日：2026-05-28
> Repo：[dice-fate-survivor](https://github.com/asd23353934/dice-fate-survivor)

---

## 1. 目標 / 不目標

### 目標

- **20 格 closed loop 棋盤**（7+3+7+3 對稱 perimeter，順時針）
- **5 種 TileType**：EMPTY / GOLD / COMBAT / ITEM / SHOP（W12 完整實作 EMPTY、GOLD、COMBAT；ITEM / SHOP 顯示但無機制）
- **擲骰移動**：randi 1-6 → 沿 loop 走 N 格（每格 0.18 秒 tween）
- **金錢格觸發**：踩到 → GameState.add_gold + 飄字
- **戰鬥格觸發**：踩到 → SceneRouter to combat → 戰勝回 board 同位置
- **跨場景狀態保留**：current_tile_index 存在 GameState，combat 結束自動恢復
- **UI**：Gold / HP / Tile 位置 / 擲骰結果 / Tile info 即時更新

### 不目標（W13+ 補）

- **5 波結構**（W13 / Step 4.10）：棋盤每隔幾輪 spawn 敵人
- **敵人棋盤 AI**（W13 / Step 4.8）：怪物也擲骰移動
- **雙向 path encounter**（W13 / Step 4.9）：路徑經過敵人觸發
- **道具系統 + 商店**（W13）：ITEM / SHOP 格子真實機制
- **事件格**（W13）：EVENT 隨機 popup（VN 風）
- **TileData / StageData .tres 化**（W14）：W12 hardcode in code 夠用
- **棋盤分岔路**（W13）：W12 純單向 loop，無 fork
- **通關 / 失敗結算**（W13 / Step 4.11）

---

## 2. 設計決策

### 為什麼**先做 closed loop** 不做有分岔的棋盤？

| 選項 | 優 | 缺 |
|---|---|---|
| **Closed loop**（採用） | 邏輯簡單（i+1 % 20）；視覺穩定 | 缺少策略選擇 |
| 分岔樹（Slay the Spire） | 玩家路線選擇 | 移動邏輯複雜（graph traversal） |
| 自由 grid | 最高自由度 | 平衡 + UI 都難 |

**W12 選 loop**：
- 路徑遍歷邏輯 `(i + 1) % 20` 簡單
- DFS spec 也是這個方向（玩家 + 敵人都繞圈）
- W13 加 fork 是擴充，不會推翻 W12

### 為什麼 perimeter 用 **7+3+7+3**，不用 6×6 或 5×5？

數學考量：
- 6×6 perimeter = 2*6 + 2*6 - 4 = 20 ✓
- 5×5 perimeter = 2*5 + 2*5 - 4 = 16 ❌
- 7×5 perimeter = 2*7 + 2*5 - 4 = 20 ✓（**採用**）
- 7+3+7+3 = 7 寬 5 高（含 corners）

**選 7×5（7 寬 5 高）的關鍵**：
- 1280×720 viewport 寬比高，**橫長比直高的 layout 更合身**
- 比 6×6 略寬，視覺平衡感更好
- 7 tile top 加 95px step = 570 寬，合理

### 踩過的 bug — **corner overlap**

**初版 7+3+6+4 = 20**（也是 20 但不對稱）：
```
top 7   (idx 0-6)
right 3 (idx 7-9)
bottom 6 (idx 10-15)
left 4  (idx 16-19)
```

問題：
- 左側 4 個 tile 從 y=4 往上 y=4,3,2,1 → tile 19 在 y=1
- 但 tile 0 也在 y=0（頂排左端）
- 視覺上 tile 19 跑到 tile 0 位置附近 → 看起來 **少 1 格**（右下少塊）

**修法**：改 7+3+7+3 對稱（左右側都 3 格，避免 corner 越界）。

### 為什麼 tile config **hardcode in code** 不做 .tres？

**選項對照**：

| 方案 | 何時值得 |
|---|---|
| **hardcode Array in code**（W12 採用） | prototype 階段、單一 stage |
| StageData .tres + Inspector 編輯 | 多 stage（W14+）、設計師調平衡 |
| Procedural generation | roguelike 每 run 隨機 |

**W12 hardcode 理由**：
- 只 1 個 stage（W12 規模）
- 加新 stage 是 W14 的事（Phase 5 整合）
- StageData class 寫了不用浪費

**W14 重構計畫**：抽 StageData class → data/stages/test_stage.tres → BoardDatabase 載入。

### 為什麼 dice 用 1d6 不 2d6？

```
1d6 → 1-6 步 → 跨 20 格的 1/3-1/3 多
2d6 → 2-12 步 → 跨 20 格的一半
```

**1d6 適合 20 格棋盤**：
- 一次跑 60% 棋盤太快
- 1d6 給玩家慢慢探索的節奏
- DFS spec 也用 1d6

**之後可調**：W18-W19 平衡時根據 playtest 決定。

### 為什麼 player avatar 用 **ColorRect** 不用 Sprite2D？

W12 階段：
- 美術 sprite 還沒做（W20）
- ColorRect 視覺夠用（白色方塊在彩色 tiles 中很明顯）
- code 簡單

**W20 替換時**：
```gdscript
# 現在
_player_avatar = ColorRect.new()
_player_avatar.color = Color.WHITE

# W20 改
_player_avatar = TextureRect.new()
_player_avatar.texture = preload("res://assets/characters/player_avatar.png")
```

**邏輯 0 變動，只換 1 行 + 1 個 asset**。

### 為什麼 GameState.current_tile_index 而不是 Board.current_index？

**跨場景持久性**：

```
board → combat → board
       ↑          ↑
       玩家在 tile 3
       戰勝回 board
       需要恢復到 tile 3
```

如果 current_index 在 board.gd 內：
- board scene 切走時 → free → 變數消失
- 回 board → fresh _ready → 不知道之前在哪

放 **GameState autoload**：
- autoload 跨場景持續存在 ✓
- board._ready 第一行 `_place_player_at_tile(GameState.current_tile_index)` 恢復
- combat 不用碰這個變數（自然保留）

**這就是 autoload pattern 為什麼存在** — 跨場景的 state 必須放這。

### 為什麼 connection lines **加 move_child(line, 0)** 移到最底層？

```gdscript
add_child(line)
move_child(line, 0)   # 連線放最下層（被 tile 蓋）
```

Godot 場景樹 children 渲染**順序：先加 = 先畫 = 後面被蓋**。

預設 add_child 加在最後 → 連線蓋在 tile 上面（醜）。

`move_child(line, 0)` 移到 index 0 = 最先畫 = 被 tile 蓋住。

**結果**：tile rect 邊緣不被線打斷，視覺乾淨。

### 為什麼 combat 結束**自動回 board**？

W7 combat.gd：
```gdscript
func _on_enemy_died():
    # ... show VICTORY
    await get_tree().create_timer(1.5).timeout
    SceneRouter.change_scene("res://scenes/board/board.tscn")
```

當時是 placeholder（board 也是 placeholder）。W12 board 升級為真實後，這套自動 work：
- combat 結束 → 切 res://scenes/board/board.tscn
- board._ready 從 GameState.current_tile_index 恢復位置 ✓

**W7 寫的 code 在 W12 自動 reusable** — 鬆耦合架構的價值。

---

## 3. 檔案清單

```
新增 / 大改：
└── scenes/board/board.gd         ← 從 placeholder 升級成 200+ 行 board controller

修改：
├── scenes/board/board.tscn       ← UI 改：placeholder buttons → Gold/HP/TilePos/Dice/TileInfo labels + RollDiceButton
├── scripts/autoload/game_state.gd ← 加 current_tile_index 欄位 + reset_for_new_run 歸 0
└── scripts/autoload/event_bus.gd  ← player_moved_to_tile / tile_triggered 改 tile_type: int (enum)
```

---

## 4. 實作流程 / 順序

```
1. 讀 DFS spec：docs/08-implementation/steps/phase-4-board-prototype.md（4.1-4.5）
   + docs/01-design/board-system.md（格子類型 / 路徑設計）
2. 計畫 W12 scope：closed loop / 3 種 active tile types / 戰鬥銜接
3. GameState 加 current_tile_index 欄位
4. reset_for_new_run 歸 0
5. EventBus tile signals 改用 int (enum)
6. board.gd 設計：
   · enum TileType { EMPTY, GOLD, COMBAT, ITEM, SHOP }
   · 顏色 dictionary + 名稱 dictionary
   · 20 tile config array hardcode
   · _build_tile_positions (perimeter math)
   · _build_tile_visuals (ColorRect + Label per tile)
   · _build_connection_lines (Line2D + move_child to back)
   · _build_player_avatar (ColorRect)
   · _on_roll_dice_pressed + _move_player tween
   · _resolve_current_tile (match by type)
   · _refresh_ui
7. board.tscn UI 加：Gold/HP/TilePos/Dice/TileInfo labels + RollDiceButton + ToMenuButton
8. F5 驗證 → 發現 corner overlap bug（少 1 格）
9. Fix bug：7+3+6+4 → 7+3+7+3 對稱 layout
10. F5 再驗證 → 20 格全顯示 + 戰鬥銜接 + 金錢累積
11. 寫 W12 implementation guide
12. 雙 commit + push（中文）
```

**順序邏輯**：state（GameState 加 field）→ data（tile config）→ 視覺（positions / visuals / lines / avatar）→ 互動（dice / move）→ trigger（resolve_tile）→ UI → 驗證 → debug bug → fix → 再驗證。

---

## 5. 關鍵 code 解析

### `_build_tile_positions` — perimeter 數學

```gdscript
for i in range(20):
    var pos = Vector2.ZERO
    if i <= 6:
        # 頂排 0-6 → y=0
        pos = origin + Vector2(i * step.x, 0)
    elif i <= 9:
        # 右側 7-9 → x=6, y=1..3
        pos = origin + Vector2(6 * step.x, (i - 6) * step.y)
    elif i <= 16:
        # 底排 10-16 → y=4, x=6..0（從右往左）
        pos = origin + Vector2((6 - (i - 10)) * step.x, 4 * step.y)
    else:
        # 左側 17-19 → x=0, y=3..1（從下往上）
        pos = origin + Vector2(0, (3 - (i - 17)) * step.y)
```

**4 段對應 4 邊**，每段 if 範圍要小心：
- 頂排 0-6 是 7 個（idx 0-6 含兩端）
- 右側 7-9 是 3 個（不含 corners，corners 屬於頂排和底排）
- 底排 10-16 是 7 個（含兩端 corners）
- 左側 17-19 是 3 個（不含 corners）
- 總和 7+3+7+3 = 20 ✓

**Loop 閉合**：i=19 在 (0, 1*95)，i=0 在 (0, 0) → 相鄰 95px 距離 ✓

### `_build_connection_lines` — Z-order 控制

```gdscript
func _build_connection_lines() -> void:
    for i in range(20):
        var next_i = (i + 1) % 20    # ← closed loop 關鍵
        var line = Line2D.new()
        line.add_point(_tile_positions[i])
        line.add_point(_tile_positions[next_i])
        line.width = 3
        line.default_color = Color(0.55, 0.55, 0.6, 0.6)
        add_child(line)
        move_child(line, 0)    # ← 放最底層
```

**`(i + 1) % 20`**：modulo 是 closed loop 的關鍵 — i=19 → next_i=0，自動閉合。

**`move_child(line, 0)`**：放到 children 第 0 位 → Godot 渲染順序 first child first → **先畫的被後畫的蓋**。連線蓋住會醜，所以放最底。

### `_move_player` async tween 鏈

```gdscript
func _move_player(steps: int) -> void:
    _is_moving = true
    %RollDiceButton.disabled = true

    for i in range(steps):
        var next_index = (GameState.current_tile_index + 1) % 20
        var target_pos = _tile_positions[next_index] - _player_avatar.size / 2

        var tween = create_tween()
        tween.tween_property(_player_avatar, "position", target_pos, MOVE_DURATION_PER_TILE)
        await tween.finished    # ← 等這格走完再走下一格

        GameState.current_tile_index = next_index
        EventBus.player_moved_to_tile.emit(next_index, _tile_configs[next_index][0])

    _is_moving = false
    %RollDiceButton.disabled = false
```

**重點**：
- `_is_moving` flag + button disabled → 玩家無法連點搞亂狀態
- `await tween.finished` → 每格走完才算數
- GameState.current_tile_index 在每格 update（不是最後一次性）→ 中斷也保留進度

**為什麼不平行 tween 一次走 N 格**：
- 視覺：玩家逐格跳的「擲骰移動感」
- 邏輯：每格 trigger signal，未來可加「中間格子觸發效果」（W13 path encounter）

### `_resolve_current_tile` — Match 分流

```gdscript
func _resolve_current_tile() -> void:
    var i = GameState.current_tile_index
    var config = _tile_configs[i]
    var tile_type = config[0]
    var gold_amount = config[1]
    EventBus.tile_triggered.emit(i, tile_type)

    match tile_type:
        TileType.GOLD:
            GameState.add_gold(gold_amount)
            %TileInfoLabel.text = "+%d 金錢！" % gold_amount
        TileType.COMBAT:
            %TileInfoLabel.text = "遭遇敵人！"
            await get_tree().create_timer(0.6).timeout
            SceneRouter.change_scene("res://scenes/combat/combat.tscn")
        TileType.ITEM:
            %TileInfoLabel.text = "獲得道具（W13 補機制）"
        TileType.SHOP:
            %TileInfoLabel.text = "商店（W13 補機制）"
        TileType.EMPTY:
            %TileInfoLabel.text = "空地，什麼都沒發生。"
```

**重點**：
- `match` 是 GDScript 的 switch，無需 break（每 case 自動退出）
- W13 加 ITEM / SHOP 機制只動對應 case，其他不變
- COMBAT 切場景前 0.6 秒延遲 → 玩家看清「遭遇敵人」訊息再 fade out

---

## 6. 觀念對照（前端視角）

| Godot | 前端 / 一般軟體 |
|---|---|
| TileType enum + 0..19 array | TypeScript enum + array of objects |
| `(i + 1) % 20` closed loop | 循環陣列 / ring buffer |
| ColorRect + Label per tile | `<div style="background: red">tile</div>` |
| Line2D 連線 | SVG `<line>` |
| move_child(line, 0) z-order | CSS `z-index: -1` / DOM order |
| Tween + await | CSS transition / JS animate API + Promise |
| GameState.current_tile_index | sessionStorage / Pinia state |
| board → combat → board 流程 | router navigate + state preservation |

**整體**：W12 是「**遊戲關卡資料 + 視覺渲染 + 玩家輸入 + 跨場景狀態**」的綜合考題。每塊都有前端對應。

---

## 7. 擴充方式

### 加分岔路（W13）

```gdscript
# 改 connection 從「i+1」改為 graph dictionary
var _connections: Dictionary = {
    0: [1],          # 0 → 1
    5: [6, 19],      # 5 → 6 或 19（分岔！）
    # ...
}

func _move_player(steps: int):
    for i in range(steps):
        var next_options = _connections.get(GameState.current_tile_index, [])
        var next_index = -1
        if next_options.size() == 1:
            next_index = next_options[0]
        else:
            next_index = await _show_fork_dialog(next_options)   # 玩家選
        # ... 跟原本一樣
```

### 加敵人在棋盤上（W13）

```gdscript
var _enemies_on_board: Array = []   # [{tile_index: int, enemy_type: String}]

func _spawn_enemy_at_tile(tile_index: int, enemy_type: String):
    var sprite = ColorRect.new()
    sprite.color = Color.RED
    sprite.size = Vector2(20, 20)
    sprite.position = _tile_positions[tile_index]
    add_child(sprite)
    _enemies_on_board.append({...})

# _move_player 改：每格檢查有沒有敵人 → 撞到停下 + 戰鬥
func _move_player(steps: int):
    for i in range(steps):
        var next_index = ...
        # 移動到 next_index
        if _has_enemy_at(next_index):
            _trigger_combat_with(next_index)
            break    # 剩餘步數作廢
```

### TileData .tres 化（W14）

```gdscript
# scripts/data/tile_data.gd
class_name TileData
extends Resource

enum TileType { EMPTY, GOLD, COMBAT, ITEM, SHOP }

@export var tile_type: TileType
@export var gold_amount: int = 0
@export_multiline var description: String = ""
@export var custom_icon: Texture2D
```

```gdscript
# scripts/data/stage_data.gd
class_name StageData
extends Resource

@export var stage_name: String
@export var tiles: Array[TileData] = []   # 順序 = tile index
@export var connections: Dictionary = {}
```

```
# data/stages/stage_01.tres
[gd_resource ...]
[resource]
stage_name = "森林徑"
tiles = [...20 tiles...]
```

```gdscript
# board.gd 改用 StageData：
var _stage: StageData = preload("res://data/stages/stage_01.tres")
# _tile_configs[i] → _stage.tiles[i].tile_type / .gold_amount
```

設計師可以建多個 .tres 切換 stage（森林 / 沙漠 / 城市），不動 code。

### 美術 sprite 替換（W20）

```gdscript
# 加 const PATH 映射 type → texture
const TILE_TEXTURES = {
    TileType.GOLD: preload("res://assets/board/tile_gold.png"),
    TileType.COMBAT: preload("res://assets/board/tile_combat.png"),
    # ...
}

# _build_tile_visuals 改：
var rect = TextureRect.new()   # 從 ColorRect 改 TextureRect
rect.texture = TILE_TEXTURES[tile_type]
```

連 player avatar 換 AnimatedSprite2D 都 trivial。

### 玩家行走動畫升級

```gdscript
# 加 squash & stretch
func _move_player(steps):
    for i in range(steps):
        var tween = create_tween()
        # 起跳 squash
        tween.tween_property(_player_avatar, "scale", Vector2(1.2, 0.8), 0.05)
        # 移動 + 拉伸
        tween.parallel().tween_property(_player_avatar, "position", target_pos, MOVE_DURATION_PER_TILE)
        # 落地 squash
        tween.tween_property(_player_avatar, "scale", Vector2(1.0, 1.0), 0.05)
        await tween.finished
```

---

## 8. 常見錯誤 / debug 指南

### Layout 少 1 格（W12 踩過）

**症狀**：視覺上少 1 個 tile
**根因**：perimeter 設計不對稱，某個 corner overlap
**修法**：對稱 7+3+7+3 / 6×6（2*6+2*6-4=20）/ 數學算 perimeter
**預防**：每 tile_position 印出來檢查重複

### Player 卡在某格不動

**症狀**：擲骰 → avatar 不移動
**根因**：
1. `_is_moving = true` 沒 reset → 連點被 ignore
2. tween 沒 await → for loop 跳過
3. button.disabled 沒解
**修法**：F5 看 Output [Board] 擲骰 print 有沒有；逐行加 print debug

### Combat 結束沒回 board

**症狀**：戰勝後黑屏永久
**根因**：SceneRouter.change_scene fade 中失敗（W6 加過 push_error 檢查）
**檢查**：Output 看有沒有 push_error 路徑錯訊息
**驗證**：combat.gd 的 `_on_enemy_died` 是 path correct（"res://scenes/board/board.tscn"）

### Combat 回 board，玩家在 tile 0 不在原位

**症狀**：戰前在 tile 3，戰勝後玩家在 tile 0
**根因**：GameState.current_tile_index 沒寫入 / 被 reset
**檢查**：F5 看 print [Board] ready 玩家在 tile X
**修法**：確認 _move_player 內 `GameState.current_tile_index = next_index` 有執行

### 連續快速擲骰崩潰

**症狀**：連點擲骰按鈕 → avatar 飛到亂位置 / 跳過 tile resolve
**根因**：W12 加了 `_is_moving` flag 應該擋掉，如果還能崩 → 看 flag 有沒漏 set
**修法**：button.disabled 是雙保險

### Tile color 一片黑

**症狀**：tiles 全黑色
**根因**：TILE_COLORS dict 某個 key 沒對應，或 enum 值跑掉
**檢查**：print(tile_type) 看 enum int 對不對

### Line 蓋在 tile 上面

**症狀**：連線壓在 tile 中間，視覺亂
**根因**：move_child(line, 0) 漏掉
**修法**：每個 line add_child 後立刻 move_child to 0

---

## 9. 下游依賴

| 之後階段 | 怎麼用 W12 成果 |
|---|---|
| **W13 棋盤下半** | 5 波結構 + 棋盤敵人 AI + path encounter + ITEM/SHOP/EVENT 機制 + 通關失敗 |
| W14 整合 | 棋盤 → 戰鬥 → reward 三選一 → 回棋盤 完整 loop |
| W14 Boss | 第 10 輪固定 boss spawn，棋盤 fork 走向 boss |
| W16 Hub | Hub 提供 kit 選擇 + 永久升級，棋盤 stage 用對應 kit |
| W17 SaveManager | save current_tile_index + visited_tiles + spawned_enemies |
| W18-W19 內容 | 加 5+ stages（不同主題 / 連線 / tile 配置） |
| W20 美術 | tile sprites + 背景圖 + player walking animation |
| W21 互動 | hover tooltip / move preview |
| W22 polish | dice 動畫 + move trail + tile enter effects |

---

## 10. 面試 talking point

> 「W12 我做 DFS 的第二個招牌系統 — 棋盤 prototype。20 格 closed loop（7+3+7+3 對稱 perimeter），玩家擲骰 1d6 沿圈移動，踩到對應格子觸發機制（金錢 / 戰鬥 / 道具 / 商店）。
>
> 設計重點：
>
> **一、跨場景狀態持久化**。玩家位置存在 GameState.current_tile_index（autoload），不存在 board scene 內部。所以 board → combat → board 流程中，combat 不用碰這個變數，board 重新 _ready 時自動從 GameState 恢復位置。**這就是 autoload pattern 的價值**。
>
> **二、closed loop modulo 數學**。連線用 `(i + 1) % 20` 自動閉合，移動每格 `next = (current + 1) % 20`。簡潔。
>
> **三、踩過 corner overlap bug**。初版 7+3+6+4 layout，左下 corner 重疊頂左 corner → 視覺少 1 格。修成 7+3+7+3 對稱。**算 perimeter 時 corner 是否計入要算清楚 — 6×6 perimeter = 2*6+2*6-4 = 20（減 corner 重複計算）**。
>
> **四、Z-order 控制**。連線 Line2D 用 `move_child(line, 0)` 移到最底層，避免蓋在 tile 上方。
>
> **五、漸進擴充設計**。W12 hardcode tile config，W14 抽 TileData / StageData .tres，W20 把 ColorRect 換成 TextureRect、ColorRect avatar 換成 AnimatedSprite2D — **邏輯 code 不變**。
>
> 這套架構撐到商業 release 規模 — 加新 stage / 加 sprite / 加分岔路 / 加 wave 結構都不用大改。」
