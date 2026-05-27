# W7 / DFS Phase 2 上半：戰鬥 prototype（拖卡攻擊）

> 對應 PROGRESS.md W7 / DFS docs/08-implementation/steps/phase-2-combat-prototype.md（Steps 2.1-2.3）
> 完成日：2026-05-28
> Repo：[dice-fate-survivor](https://github.com/asd23353934/dice-fate-survivor)

---

## 1. 目標 / 不目標

### 目標

- combat.tscn 從 placeholder 變**真戰鬥**：能量顯示 + 1 敵人 + 5 張手牌 + 拖卡攻擊
- 移植 W4 deckbuilder-prototype 的 Card / Hand / Enemy 進 DFS 結構
- CardData class + 5 張 .tres（STRIKE / DEFEND / FOCUS / HEAVY / QUICK）
- Enemy 死亡 → emit `combat_ended(true)` → 1.5 秒切回 Board
- 完整 flow：主選單 → Hub → Board → Combat → 殺敵 → 回 Board

### 不目標（W8+ 補）

- 抽 / 棄 / 洗牌堆（W8）
- 多回合（end turn 機制 W8）
- 敵人回合 / 敵人攻擊（W8）
- 玩家防禦（shield 機制 W8）
- 戰鬥失敗處理（W8）
- 多場 5 波結構（W8）
- Status effect（W11）
- Combo Lock 招牌機制（W11）

---

## 2. 設計決策

### 為什麼**直接移植 W4 code**而不重寫？

W4 deckbuilder-prototype 已經驗證過：
- Card scene + drag-and-drop ✓
- Hand container + 動態 instantiate ✓
- Enemy + take_damage + signal ✓
- 13/13 單元測試通過 ✓

重寫沒有新學習，**移植 + 改路徑 + 加 EventBus 整合** 才是 W7 的價值。

### 為什麼**只改路徑不改邏輯**？

W4 的 code 本來就乾淨：
- card.gd 用 GameState.energy（autoload，DFS 一樣）
- enemy.gd 用 GameState.spend_energy（同樣）
- Hand 用 PackedScene + @export，跟引擎無關

唯一變動：

| W4 路徑 | DFS 路徑 |
|---|---|
| `res://card.gd` | `res://scripts/combat/card.gd` |
| `res://card.tscn` | `res://scenes/combat/card.tscn` |
| `res://cards/card_data.gd` | `res://scripts/data/card_data.gd` |
| `res://cards/strike.tres` | `res://data/cards/strike.tres` |

**結構更清楚**：scripts/（邏輯）/ scenes/（視覺）/ data/（資料） 三層分離，依 DFS architecture.md。

### Enemy 為什麼**emit 兩種 signal**（local + global）？

```gdscript
func take_damage(amount: int) -> void:
    # ...
    hp_changed.emit(hp, max_hp)              # ← Local signal
    EventBus.damage_dealt.emit(amount, enemy_name)  # ← Global signal
```

**Local signal（hp_changed）**：直接訂閱這個 Enemy instance 的人用
- 例：combat.gd 訂閱 `%Enemy.died` 處理戰鬥結束
- 例：之後 Enemy 上方飄字 / HP bar 動畫

**Global signal（EventBus.damage_dealt）**：跨 scene / 跨模組訂閱者用
- AchievementSystem：累計總傷害
- AudioManager：受傷音效
- Telemetry：戰鬥數據統計
- 撒花特效系統：之後加

**為什麼兩種都要**：local 是 1:1 直接通訊，效率好且耦合明確；global 是 1:N 廣播，方便鬆耦合擴充。**並存才是 W2/W4 學到的「signal up + method down」進階版**。

### enemy HP 為什麼從 30 降到 15？

W7 沒做 W8 的「end_turn + 抽牌」，所以**一場戰鬥只能打 5 張卡 = 一手 cap 19 傷害**：

```
STRIKE 6 + HEAVY 10 + QUICK 3 = 19（總能量恰好 1+2+0=3，符合上限）
DEFEND / FOCUS 在 W7 沒機制（dmg=0）
```

敵人 HP 30 → **永遠殺不死** → 戰鬥流程跑不完 → demo 卡住。

兩種選擇：
- **降 enemy HP 到 15**（採用）：玩家用 STRIKE+HEAVY 16 dmg 就能殺，看到 VICTORY 流程
- 提前做 W8 抽牌：複雜度大、超出 W7 scope

W7 採短期妥協（HP 15），W8 補完真實流程後改回 30+。

### Enemy 5 張卡為什麼分**3 攻擊 + 2 暫無機制**？

- **STRIKE / HEAVY / QUICK** 三張覆蓋 cost 0/1/2 攻擊光譜（demo 玩家選牌空間）
- **DEFEND**：cost 1 / dmg 0，**展示 shield 機制位置**（W8 補）
- **FOCUS**：cost 2 / dmg 0，**展示 power buff 位置**（W11 補）

DEFEND / FOCUS 在 W7 是 placeholder card → 玩家拖會扣能量但不傷敵。**這是有意的**，讓後續 phase 直接擴充行為而不用新增卡。

### 為什麼 combat.tscn 用 `instance=ExtResource(...)` 而不是 `add_child` runtime 生？

兩種方式：

**A. .tscn 用 instance 包**（採用）：
```
[node name="Enemy" parent="." instance=ExtResource("2_enemy")]
```
- 編輯器看得到 + 可調 position / size
- 啟動立即顯示，沒 frame 延遲

**B. combat.gd runtime instantiate**：
```gdscript
var enemy = preload("res://scenes/combat/enemy.tscn").instantiate()
add_child(enemy)
```
- 更動態（可重新 spawn / 多隻）
- 編輯器看不到，要跑起來才知道擺哪

**W7 選 A**：只 1 隻敵人、不變、demo 用。
**W8+ 改 B**：多波 + 每波生新 enemy → 必須 runtime spawn。

---

## 3. 檔案清單

```
dice-fate-survivor/
├── scripts/
│   ├── combat/                       ← 新增資料夾
│   │   ├── card.gd                   ← W4 移植 + 路徑調整
│   │   ├── hand.gd                   ← W4 移植
│   │   └── enemy.gd                  ← W4 移植 + EventBus emit
│   └── data/
│       └── card_data.gd              ← CardData class（從 W3/W4 移植）
├── data/cards/                       ← 新增資料夾
│   ├── strike.tres                   ← cost 1 / dmg 6
│   ├── defend.tres                   ← cost 1 / dmg 0（W8 補 shield）
│   ├── focus.tres                    ← cost 2 / dmg 0（W11 補 buff）
│   ├── heavy.tres                    ← cost 2 / dmg 10
│   └── quick.tres                    ← cost 0 / dmg 3
└── scenes/combat/
    ├── card.tscn                     ← 一張卡視覺
    ├── hand.tscn                     ← HBoxContainer 容器
    ├── enemy.tscn                    ← 史萊姆 HP=15
    ├── combat.tscn                   ← 升級：背景 + 能量 label + Enemy + Hand
    └── combat.gd                     ← 升級：reset 能量 + 訂閱 died + win
```

---

## 4. 實作流程 / 順序

```
1. 設計：把 W4 deckbuilder 移植進 DFS 結構（scripts/data/、scripts/combat/、data/cards/、scenes/combat/）
2. 建 CardData class（scripts/data/card_data.gd）
3. 建 5 張 .tres（data/cards/*.tres，引用 CardData script_class）
4. 移植 card.gd / card.tscn（path 改 res://scripts/combat/、res://scenes/combat/）
   · drag preview 用 res://scenes/combat/card.tscn 新路徑
5. 移植 hand.gd / hand.tscn（無修改邏輯）
6. 移植 enemy.gd / enemy.tscn（+ EventBus.damage_dealt.emit）
7. 升級 combat.gd：
   · _ready 重置能量
   · 訂閱 enemy.died 處理勝利
   · 1.5 秒切回 Board
8. 升級 combat.tscn：背景 + EnergyLabel + MessageLabel + Enemy + Hand（用 instance）
9. Inspector 設 Hand 的 cards array 拖入 5 張 .tres
10. F5 跑全程驗證
11. 發現 enemy HP=30 殺不死 → 改 15
12. 再跑確認 VICTORY 流程順
13. 寫 implementation guide + 雙 commit + push
```

**順序邏輯**：底層先（CardData class → .tres）→ 組件（Card / Hand / Enemy 各自）→ 組合（combat.tscn 用 instance 把它們串起來）→ 邏輯（combat.gd 控制流程）→ 整合測試。

---

## 5. 關鍵 code 解析

### `card_data.gd` — Resource schema

```gdscript
class_name CardData
extends Resource

@export var card_name: String = ""
@export var cost: int = 1
@export var damage: int = 0
@export_multiline var description: String = ""

func describe() -> String:
    return "%s (cost: %d, dmg: %d) — %s" % [card_name, cost, damage, description]
```

**重點**：
- `class_name CardData` → 全域註冊，任何地方 `var c: CardData` 可用
- `extends Resource` → 可序列化、可 .tres、可 Inspector
- `@export_multiline` → 字串欄位變多行框，描述方便
- W11 之後加 enum / Array fields 都用同樣 @export pattern

### `data/cards/strike.tres` — 序列化資料

```
[gd_resource type="Resource" script_class="CardData" format=3]

[ext_resource type="Script" path="res://scripts/data/card_data.gd" id="1"]

[resource]
script = ExtResource("1")
card_name = "STRIKE"
cost = 1
damage = 6
description = "對單一敵人造成 6 點傷害。"
```

**重點**：
- `script_class="CardData"` → Godot 知道這 .tres 是 CardData 類型
- `ExtResource("1") path` → 找到 CardData 的 .gd 檔
- `card_name="STRIKE"` 等 → @export 欄位值
- 改數值**不用碰 code**，直接編 .tres

### `card.gd` 的 drag 觸發 + preview

```gdscript
class_name Card
extends PanelContainer

@export var data: CardData:
    set(value):
        data = value
        if is_node_ready():
            _refresh()

# ...

func _get_drag_data(_at_position: Vector2) -> Variant:
    if data == null:
        return null
    if GameState.energy < data.cost:
        print("[Card] 能量不足，無法打 %s（需 %d，剩 %d）" % [data.card_name, data.cost, GameState.energy])
        return null

    # Drag preview：fresh card instance（不用 duplicate）
    var preview_scene = preload("res://scenes/combat/card.tscn")
    var preview = preview_scene.instantiate()
    preview.data = data
    set_drag_preview(preview)

    return self
```

**重點**：
- `class_name Card` → Enemy.gd 用 `data is Card` type check 更乾淨
- @export var data 的 setter pattern → Inspector 即時 refresh + runtime 設 data 也觸發
- preview 用新 .tscn instance 不用 `duplicate()` → 避免 owner / unique_name 衝突
- early return `null` → 能量不足拖不起來，符合 Godot drag API

### `enemy.gd` 的 take_damage 雙廣播

```gdscript
class_name Enemy
extends PanelContainer

signal hp_changed(current: int, max_hp: int)
signal died

func take_damage(amount: int) -> void:
    if hp <= 0:
        return                                  # 死了不再扣
    hp = max(0, hp - amount)                    # clamp
    print(...)
    _refresh()
    hp_changed.emit(hp, max_hp)                 # ← Local：給直接訂閱者
    EventBus.damage_dealt.emit(amount, enemy_name)  # ← Global：給跨模組訂閱者
    if hp == 0:
        died.emit()                              # ← Local 訊號（戰鬥場景 combat.gd 接）
```

**重點**：
- `if hp <= 0: return` 防止死後重複 emit died（W4 regression test 抓過這個）
- `hp = max(0, hp - amount)` clamp，不會負值
- **雙廣播**：local 給戰鬥內邏輯、global 給全局訂閱者

### `combat.gd` orchestration

```gdscript
func _ready() -> void:
    # 1. 重置能量
    GameState.energy = GameState.max_energy
    EventBus.energy_changed.emit(GameState.energy, GameState.max_energy)

    # 2. 訂閱
    EventBus.energy_changed.connect(_on_energy_changed)
    %Enemy.died.connect(_on_enemy_died)

    # 3. 廣播戰鬥開始
    EventBus.combat_started.emit("debug_encounter")


func _on_enemy_died() -> void:
    %MessageLabel.text = "VICTORY!"
    %MessageLabel.show()
    EventBus.combat_ended.emit(true)
    await get_tree().create_timer(1.5).timeout
    SceneRouter.change_scene("res://scenes/board/board.tscn")
```

**重點**：
- 訂閱 `EventBus.energy_changed`（不是 Enemy 自己）→ 任何地方扣能量都會更新 label
- 訂閱 `%Enemy.died`（local signal）→ 處理特定 enemy 的死亡
- `await get_tree().create_timer(1.5).timeout` → W2 的 await pattern，延遲後切場景
- `SceneRouter.change_scene` → W6 寫的 fade 過場

---

## 6. 觀念對照（前端視角）

| Godot 概念 | Vue / Angular |
|---|---|
| CardData (Resource) + .tres | TypeScript interface + JSON file（schema + data 分離） |
| @export Inspector 編輯 | 設計師工具：直接編 JSON / CMS |
| script_class | TypeScript class name binding |
| PackedScene + instantiate | React `<Component />` 在 tree 內 / Vue dynamic component |
| local signal vs EventBus | child emit vs Pinia store action |
| combat.gd 訂閱兩種 signal | Vue parent 接 @child-event + 也接 store action |
| `await create_timer.timeout` | `await new Promise(setTimeout)` |
| scene composition（hub→combat→board） | React Router pages |

---

## 7. 擴充方式

### 加新卡

**範例**：W11 加 POISON_DART（中毒攻擊）

1. 建 `data/cards/poison_dart.tres`：
   ```
   [gd_resource type="Resource" script_class="CardData" format=3]
   [ext_resource type="Script" path="res://scripts/data/card_data.gd" id="1"]
   [resource]
   script = ExtResource("1")
   card_name = "POISON DART"
   cost = 1
   damage = 2
   description = "造成 2 點傷害並附 3 層中毒。"
   ```

2. **加 status_effects 欄位**到 CardData：
   ```gdscript
   @export var status_effects: Array[StatusData] = []   ← 新加
   ```

3. **修改 enemy._drop_data 處理 status**：
   ```gdscript
   func _drop_data(_at_position, drop_data):
       # ... 原本邏輯
       take_damage(card_data.damage)
       for status in card_data.status_effects:    ← 新加
           apply_status(status)
   ```

4. 加進 hand.tscn 的 cards array（Inspector 拖）

### 加多隻敵人

```gdscript
# combat.gd 改成 array
var enemies: Array[Enemy] = []

func _ready() -> void:
    # 動態 spawn 3 隻敵人
    for i in range(3):
        var enemy = preload("res://scenes/combat/enemy.tscn").instantiate()
        enemy.position = Vector2(400 + i * 250, 100)
        enemy.died.connect(_on_enemy_died.bind(enemy))
        add_child(enemy)
        enemies.append(enemy)

func _on_enemy_died(dead_enemy: Enemy) -> void:
    enemies.erase(dead_enemy)
    if enemies.is_empty():
        # 全部死光 → VICTORY
```

⚠️ drag-drop target 變多 → enemy._can_drop_data 各自處理（Godot 內建支援多 drop target）。

### 加敵人攻擊（W8 的事）

```gdscript
# enemy.gd
func enemy_turn_attack() -> void:
    var dmg = 5  # 之後從 EnemyData.tres 讀
    GameState.take_damage(dmg)

# combat.gd end_turn handler
func _on_end_turn_pressed() -> void:
    # 1. 敵人輪流攻擊
    for enemy in enemies:
        enemy.enemy_turn_attack()
    # 2. 我方回合開始
    GameState.energy = GameState.max_energy
    EventBus.energy_changed.emit(GameState.energy, GameState.max_energy)
    # 3. 抽牌
    _draw_cards(5)
```

### 加 status effect 系統（W11 招牌：Combo Lock）

設計建議：
- `scripts/combat/status.gd` 新 class
- 玩家 / 敵人持有 `Array[Status]`
- 每回合 tick + 觸發 effect
- 多種 status 衝突解決

### 加抽牌系統（W8）

設計：
```
GameState.deck      ← 牌庫
GameState.hand      ← 手牌
GameState.discard   ← 棄牌堆

func draw_card(n: int):
    for i in range(n):
        if deck.is_empty():
            shuffle_discard_into_deck()
        hand.append(deck.pop_back())
```

Hand UI 訂閱 GameState.hand 變化 → 動態 refresh 顯示。

---

## 8. 常見錯誤 / debug 指南

### Enemy 不接收 drop / 拖過去沒反應

**症狀**：拖卡到 enemy 上，沒高亮 / 放開沒事。
**根因可能**：
1. enemy.gd 沒實作 `_can_drop_data` 回 true
2. data 不是 Card type
3. Enemy 是 hp <= 0（已死）
**修法**：print debug `_can_drop_data` 內部，看為什麼回 false。

### 拖一張卡能量扣 2 次

**症狀**：拖卡後能量扣兩次。
**根因**：可能 enemy 訂閱了重複 signal。
**修法**：檢查 `_drop_data` 是否被 call 兩次（罕見，除非 enemy 被加進多個 drop target 兩次）。

### 卡牌打死敵人但沒切回 Board

**症狀**：VICTORY 出現但場景沒切。
**根因可能**：
1. SceneRouter 出錯（看 push_error 訊息）
2. await timer 沒跑（罕見）
**修法**：移除 await 直接呼叫 change_scene 測試；如果 work → timer issue。

### 拖到背景區（非 enemy）也消耗能量

**症狀**：拖卡到空白處放開，能量也扣。
**根因**：不該。Godot drag-drop 沒落在有效 target = 不會呼叫 _drop_data。
**檢查**：背景 ColorRect 是否誤設成可接 drop（mouse_filter / drag-drop method）。

### .tres 改數值沒生效

**症狀**：在編輯器改 STRIKE damage 從 6 → 8，跑遊戲還是 6。
**根因**：
1. .tres 沒存（Ctrl+S）
2. Godot 沒 reimport（重啟 / Project → Tools → Reimport）
**預防**：改完 .tres 立即看 FileSystem 是否有未存標記。

---

## 9. 下游依賴

| 之後階段 | 怎麼用 W7 成果 |
|---|---|
| W8 Phase 2 下半 | 在 combat.gd 加 end_turn / 敵人回合 / 抽棄牌堆 / 戰鬥失敗 |
| W9 Phase 3 (1/3) | 把 hand 從 hardcode 改成 GameState.hand reactive；deck pool 化 |
| W10 Phase 3 (2/3) | CardData 加 status_effects 欄位；enemy 加 status array |
| W11 Phase 3 (3/3) | Combo Lock 機制（招牌）；戰鬥 formulas 測試 |
| W12-W13 Phase 4 棋盤 | board.gd 觸發戰鬥 → change_scene to combat（用本週 combat scene）|
| W14-W15 Phase 5 整合 | 真實 encounter ID 進 combat_started.emit；reward 處理 |
| W18-W19 Phase 7 內容 | 補完 10-15 張卡 .tres / 4-5 種敵人 .tres |
| W22 Phase 9 polish | combat 加 hit pause / shake / 飄字（hook 既有 signal） |

---

## 10. 面試 talking point

> 「W7 我把 W4 deckbuilder-prototype 的核心戰鬥 code 移植進 DFS。重點不是重寫，是**結構分離**：scripts/combat/（邏輯）、scenes/combat/（視覺）、data/cards/（資料）三層分開，依架構規範。
>
> 移植過程主要兩件事：
> 一、調整 path（card.gd 從 res://card.gd 移到 scripts/combat/card.gd）
> 二、Enemy 加 EventBus.damage_dealt.emit() 廣播 — 之前 W4 prototype 只有 local signal，DFS 規模要兼顧 global subscriber（AchievementSystem、AudioManager 等）。
>
> 這個「**local + global signal 並存**」的 pattern 是 EventBus 架構的進階用法。local 是 1:1 直接通訊，效率好；global 是 1:N 廣播，鬆耦合擴充。兩者並存。
>
> 我也踩了一個設計缺口：W7 沒做 W8 的 end_turn / 抽牌，所以單場戰鬥能打卡上限 = 一手 5 張。敵人 HP 30 設得太高 → 永遠殺不死 → demo 卡住。短期降到 HP 15 demo 可跑通，W8 補完抽牌後再改回去。**這是 prototype 階段常見的權衡 — 不要為了未來的完美而卡在當下**。」
