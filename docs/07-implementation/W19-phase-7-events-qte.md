# W19 / Phase 7 (2/2)：棋盤事件 VN 系統 + ITEM/SHOP tile + QTE 卡（致敬 WorkNite）

> 對應 repo：[dice-fate-survivor](https://github.com/asd23353934/dice-fate-survivor)
> 對應 PROGRESS milestone：M5 W19
> 寫於 Day 8（2026-06-01）

---

## 1. 目標 / 不目標

### 目標
- **棋盤事件 VN 系統**：EventData / EventChoice resource + 3 個事件 .tres + DialogPanel 分支選擇
- **統一 board effect 系統**：GameState.apply_board_effect 一個入口處理所有 board 後果
- **ITEM tile 實作**：踩到 → 即時隨機正面 buff（不做 inventory）
- **SHOP tile 實作**：board 內即時小商店（花當前 gold，跟 Hub meta shop 區隔）
- **QTE 卡（致敬 WorkNite）**：限時連點 mini-game，點越多傷害越高
- **pending combat buff**：board 撿的「下場戰鬥」buff，combat 開場套用
- **修 reward 池 bug**：排除 _plus 升級版卡（之前會被直接 offer）

### 不目標
- 多段對話樹 / 條件分支（線性事件足夠，之後可擴）
- 事件立繪（W20 美術期）
- ITEM inventory 系統（即時 buff 取代）
- QTE 多種模式（只做連點，之後可加按方向 / 節奏）
- Dialogic plugin（自寫 DialogPanel 重用即可）

---

## 2. 設計決策

### A. 為什麼棋盤事件對 DFS / WorkNite 履歷重要？

WorkNite 是**台灣 VN/H game 工作室**。VN（視覺小說）核心就是「對話 + 分支選擇 + 後果」。
- W16 已建 DialogPanel（NPC 對話）
- W19 把它升級成「資料化的分支事件系統」
- **這是 deckbuilder 套 VN 元素的混血** — 正好對應 WorkNite「敘事 + 玩法」雙軌
- 面試展示：「我做的不只是戰鬥，還有資料驅動的 VN 事件系統」

### B. EventData + EventChoice：為什麼用 sub-resource 而非平行 array？

兩種資料化方案：

| 方案 | event_data.gd | .tres 寫法 |
|---|---|---|
| 平行 array | `choice_labels: Array[String]` + `choice_effects: Array[String]` + ... | 4 個平行陣列，index 對齊易錯 |
| **sub-resource**（採用） | `choices: Array`（裝 EventChoice） | 每個 choice 一個 SubResource，獨立完整 |

→ **sub-resource 勝**：
- 每個選項是完整物件（label + result + effect 綁在一起）
- Inspector 可 add/remove EventChoice element
- 不會發生「label 有 3 個但 effect 只填 2 個」的對齊 bug

### C. 為什麼 choices 用 untyped `Array` 不用 `Array[EventChoice]`？

- 手寫 .tres 的 **typed array 序列化語法脆弱**（`Array[ExtResource(...)]([...])` 容易寫錯導致 load 失敗）
- untyped `Array` 的 .tres 寫法單純：`choices = [SubResource("a"), SubResource("b")]`
- runtime element 仍是 EventChoice（存取 `.label` / `.effect_type` 正常）
- **trade-off**：失去 compile-time 型別檢查，換 .tres 健壯性。內容期手寫多，選健壯。

> 這是「型別安全 vs 手寫健壯」的實務取捨。如果用 Inspector 編輯（不手寫）就可以用 typed。

### D. 統一 effect 系統：apply_board_effect 一個入口

所有 board 後果（事件選擇 / ITEM / SHOP 購買）都走同一個方法：

```gdscript
func apply_board_effect(effect_type: String, amount: int) -> String:
    match effect_type:
        "heal": ...
        "gold": ...
        "next_combat_shield": ...
        "next_combat_strength": ...
        "max_hp": ...
        "add_random_card": ...
        "damage": ...
        "nothing": ...
```

好處：
- **DRY**：事件 / ITEM / SHOP 不各寫一套效果邏輯
- **回傳描述字串**：UI 直接顯示「回復 15 HP（現 70/80）」
- **新效果一處加**：之後加 "remove_card" / "max_energy" 只改這個 match

### E. 為什麼 ITEM 即時 buff 不做 inventory？

| 方案 | 複雜度 | 適合 |
|---|---|---|
| inventory（撿了存著，戰鬥中用） | 高（UI + 槽位 + 使用時機） | 完整 RPG |
| **即時 buff**（採用） | 低（撿到立刻生效） | roguelike deckbuilder |

DFS 是 roguelike，「即時生效」符合 Slay the Spire 的 potion-on-pickup 精神。inventory 之後 W22 整合期再評估。

### F. board SHOP vs Hub shop 區隔

| | Hub shop（Mira） | board SHOP tile |
|---|---|---|
| 性質 | meta progression（跨 run） | run 內即時 |
| 賣 | 永久 starter 卡 | 即時補給（HP / buff） |
| 貨幣 | 持久 gold | 持久 gold（同個池） |
| 時機 | run 之間 | run 進行中 |

兩個都花 gold，但「買什麼」「持久性」不同 — 區隔玩家決策層次。

### G. 為什麼 board 撿的 buff 要「下場戰鬥」生效，不立刻？

- 護盾 / 力量是**戰鬥內**概念（戰鬥外沒有「護盾」可言）
- 玩家在 board 撿到「鐵盾碎片」→ 存 `pending_combat_shield` → 進下場戰鬥開場套用
- heal 例外（HP 是跨場景持久的）→ 立刻生效

技術：combat `_ready` 在 `start_combat_with_deck`（會 reset shield/status）**之後**呼叫 `consume_pending_combat_buffs`，避免被 reset 清掉。

### H. QTE 卡架構：為什麼用 EventBus 解耦？

QTE 卡的傷害需要「等玩家點完」才知道。但 enemy._drop_data 是 drag-drop callback，不適合在裡面跑 3 秒 mini-game。

解法：**EventBus 解耦**
```
enemy._drop_data 偵測 is_qte
  → 花能量 + combo + discard（正常流程）
  → emit EventBus.qte_requested(card, enemy)
  → return（跳過正常傷害）

combat.gd 監聽 qte_requested
  → await QtePanel.run_qte()  ← 跑 mini-game 拿點擊數
  → 算傷害（base + hits×bonus，套 strength + combo）
  → enemy.receive_attack(dmg)
```

enemy 不需知道 QtePanel 存在，combat 統籌 mini-game。**單一職責 + 事件解耦**。

---

## 3. 檔案清單

### 新增

| 檔案 | 角色 |
|---|---|
| `scripts/data/event_choice.gd` | EventChoice resource（label / result / effect / gold_cost） |
| `scripts/data/event_data.gd` | EventData resource（title / description / choices） |
| `data/events/mysterious_merchant.tres` | 神秘商人（買護身符 / 買卷軸 / 離開） |
| `data/events/wounded_traveler.tres` | 受傷旅人（幫助得 gold / 無視） |
| `data/events/ancient_altar.tres` | 古老祭壇（獻血得 max HP / 供金得力量 / 離開） |
| `scripts/ui/qte_panel.gd` | QtePanel — run_qte() 限時連點 |
| `scenes/ui/qte_panel.tscn` | QTE overlay UI |
| `data/cards/frenzy.tres` + `frenzy_plus.tres` | 狂亂連擊 QTE 卡 |

### 修改

| 檔案 | 變動 |
|---|---|
| `scripts/autoload/game_state.gd` | +`apply_board_effect` / +`consume_pending_combat_buffs` / +`pending_combat_shield/strength` |
| `scripts/autoload/event_bus.gd` | +`qte_requested(card, enemy)` |
| `scripts/autoload/card_database.gd` | +`get_obtainable_cards`（排除 _plus）/ get_reward_options 改用它 |
| `scripts/data/card_data.gd` | +`is_qte` / `qte_base_damage` / `qte_bonus_per_hit` |
| `scripts/combat/enemy.gd` | `_drop_data` 加 QTE 分支 |
| `scenes/combat/combat.gd` | 連 `qte_requested` + `_on_qte_requested` handler + `consume_pending_combat_buffs` |
| `scenes/combat/combat.tscn` | +QtePanel instance |
| `scenes/board/board.gd` | +EVENT TileType / tile configs 加事件道具格 / `_resolve_event/item/shop_tile` |
| `scenes/board/board.tscn` | +DialogPanel instance |

---

## 4. 關鍵 code 解析

### 4.1 EventChoice / EventData schema

```gdscript
class_name EventChoice extends Resource
@export var label: String
@export_multiline var result_text: String
@export var effect_type: String = "nothing"
@export var effect_amount: int = 0
@export var gold_cost: int = 0      # 需花錢的選項

class_name EventData extends Resource
@export var id: String
@export var title: String
@export_multiline var description: String
@export var choices: Array          # 裝 EventChoice
```

### 4.2 事件 .tres 用 SubResource 內嵌

```ini
[gd_resource type="Resource" script_class="EventData" load_steps=6 format=3]
[ext_resource path="res://scripts/data/event_data.gd" id="1_event"]
[ext_resource path="res://scripts/data/event_choice.gd" id="2_choice"]

[sub_resource type="Resource" id="choice_buy_shield"]
script = ExtResource("2_choice")
label = "買下護身符（30 金）"
effect_type = "next_combat_shield"
effect_amount = 10
gold_cost = 30

[resource]
script = ExtResource("1_event")
title = "神秘商人"
choices = [SubResource("choice_buy_shield"), ...]
```

一個檔案內：event 主體 + 多個 choice sub-resource，自包含。

### 4.3 board 事件 tile 處理（async dialog）

```gdscript
func _resolve_event_tile() -> void:
    var event: EventData = load(BOARD_EVENTS[randi() % BOARD_EVENTS.size()])
    var choices: Array = []
    for choice in event.choices:
        var affordable = GameState.gold >= choice.gold_cost
        var label = choice.label
        if choice.gold_cost > 0 and not affordable:
            label += "（金錢不足）"
        choices.append({
            "label": label,
            "callback": _on_event_choice.bind(event, choice, affordable),
        })
    %DialogPanel.show_dialog(event.title, event.description, choices)
    await %DialogPanel.dialog_closed   # ← 等玩家走完才 return
```

關鍵：`await dialog_closed` 讓 `_process_movement` 在事件期間不 re-enable 骰子按鈕（沿用 W14 的 caller-managed lock pattern）。

### 4.4 事件選擇 → 套 effect → chain 結果 dialog

```gdscript
func _on_event_choice(_event, choice, affordable):
    if choice.gold_cost > 0 and not affordable:
        %DialogPanel.show_dialog("……", "金錢不足。", [
            {"label": "回去", "callback": _reopen_current_event}])
        return
    if choice.gold_cost > 0:
        GameState.spend_gold(choice.gold_cost)
    var result_detail = GameState.apply_board_effect(choice.effect_type, choice.effect_amount)
    %DialogPanel.show_dialog("", "%s\n\n（%s）" % [choice.result_text, result_detail], [])
    _refresh_ui()
```

chain：選擇 → 套 effect → 顯示結果 dialog（空 choices = 「繼續」）→ 玩家按繼續 → close → dialog_closed emit → 解除 await。

### 4.5 統一 effect + pending combat buff

```gdscript
func apply_board_effect(effect_type, amount) -> String:
    match effect_type:
        "heal": heal(amount); return "回復 %d HP" % amount
        "next_combat_shield": pending_combat_shield += amount; return "下場 +%d 護盾" % amount
        "next_combat_strength": pending_combat_strength += amount; return "下場 +%d 力量" % amount
        ...

func consume_pending_combat_buffs():
    if pending_combat_shield > 0:
        gain_shield(pending_combat_shield)
        pending_combat_shield = 0
    if pending_combat_strength > 0:
        apply_player_status("strength", pending_combat_strength)
        pending_combat_strength = 0
```

combat `_ready`：
```gdscript
GameState.start_combat_with_deck(...)   # reset shield/status
GameState.consume_pending_combat_buffs()  # ← 之後才套 board buff
```

### 4.6 QTE 卡 enemy 分支

```gdscript
func _drop_data(_pos, drop_data):
    ...
    GameState.spend_energy(cost)
    GameState.on_card_played(type_string)
    var combo_mult = GameState.get_current_combo_multiplier()

    # W19：QTE 卡傷害交給 combat（mini-game 結果決定）
    if card_data.is_qte:
        EventBus.card_played.emit(card_data, self)
        GameState.discard_card(card_data)
        EventBus.qte_requested.emit(card_data, self)
        return
    ...
```

### 4.7 QtePanel coroutine

```gdscript
func run_qte(duration := 3.0) -> int:
    _hits = 0
    _active = true
    %HitButton.disabled = false
    show()
    var remaining = duration
    while remaining > 0.0 and _active:
        await get_tree().create_timer(0.1).timeout
        remaining -= 0.1
        %TimeBar.value = remaining / duration * 100.0
    _active = false
    %HitButton.disabled = true
    hide()
    return _hits

func _on_hit_pressed():
    if _active:
        _hits += 1
        %HitLabel.text = "連擊數：%d" % _hits
```

關鍵：`while + await timer(0.1)` 的迴圈讓引擎在每次 await 間隙處理 button input → `_hits++`。`run_qte` 是回傳 int 的 coroutine（`await %QtePanel.run_qte()` 拿到點擊數）。

### 4.8 combat QTE handler

```gdscript
func _on_qte_requested(card_data, enemy_node):
    if enemy_node.hp <= 0: return
    var combo_mult = GameState.get_current_combo_multiplier()
    var hits = await %QtePanel.run_qte(3.0)
    var base = card_data.qte_base_damage + hits * card_data.qte_bonus_per_hit
    var final_dmg = int(GameState.compute_outgoing_damage(base) * combo_mult)
    if enemy_node.hp > 0:
        enemy_node.receive_attack(final_dmg)
```

combo / strength 照常套用（QTE 不破壞既有傷害修飾鏈）。

### 4.9 修 reward 池排除 _plus

```gdscript
func get_obtainable_cards() -> Array[CardData]:
    var result: Array[CardData] = []
    for card in _cards.values():
        if card.id.ends_with("_plus"):
            continue   # 升級版只能透過升級取得，不直接 offer
        result.append(card)
    return result
```

之前 reward 可能跳出「STRIKE+」直接送，破壞升級系統意義。W19 修。

---

## 5. 觀念對照

| Godot 概念 | 前端 / Vue / Angular |
|---|---|
| EventData / EventChoice sub-resource | nested data model / JSON schema with sub-objects |
| `apply_board_effect` match dispatch | reducer pattern（action type → state change） |
| `await %DialogPanel.dialog_closed` | `await modal.waitForClose()` |
| EventBus `qte_requested` 解耦 | event bus / pub-sub 跨 component 通訊 |
| QtePanel `run_qte()` coroutine 回傳值 | `async function runQTE(): Promise<number>` |
| pending_combat_buff | deferred state / staged mutation |
| `id.ends_with("_plus")` 過濾 | array filter by naming convention |

---

## 6. 擴充方式

### 加更多事件（純 .tres）

複製 event 範本，改 title / description / choices。丟進 `data/events/`，加 path 進 `BOARD_EVENTS`。

### 加多段對話（VN 樹）

EventData 加 `next_event_id: String`，choice 觸發後 load 下一個 event → chain 對話。或 EventChoice 加 `follow_up_text: Array[String]` 多段敘述。

### 加條件分支（依玩家狀態）

EventChoice 加 `require_min_hp: int` / `require_card: String`，board 顯示時檢查條件決定 enable/disable。

### 加 QTE 多模式

QtePanel 加 `mode: String`（"mash" / "timing" / "sequence"），CardData 加 `qte_mode`。timing 模式 = 在指針掃過綠區時按；sequence = 按出正確方向序列。

### 加事件立繪（W20）

DialogPanel 加 `portrait: TextureRect`，EventData 加 `portrait_path: String`。

---

## 7. 常見錯誤

### 7.1 typed array .tres 序列化失敗

**症狀**：event .tres load 噴 parse error。

**原因**：`Array[EventChoice]` 手寫 .tres 的 `Array[ExtResource(...)]([...])` 語法寫錯。

**修法**：W19 改用 untyped `Array`，.tres 寫 `choices = [SubResource(...), ...]`。

### 7.2 pending combat buff 被 start_combat reset 清掉

**症狀**：board 撿護盾，進戰鬥沒有。

**原因**：`consume_pending_combat_buffs` 在 `start_combat_with_deck`（reset shield）之前呼叫。

**修法**：順序 — 先 start_combat_with_deck，再 consume buff。

### 7.3 QTE 點擊沒被計數

**症狀**：run_qte 一直回 0。

**原因**：`_active` 沒設 true，或 button.pressed 沒連 / disabled。

**修法**：run_qte 開頭 `_active = true` + `%HitButton.disabled = false`，_ready 連 button.pressed。

### 7.4 reward 跳出 _plus 卡

**症狀**：reward 直接 offer「STRIKE+」。

**修法**：get_reward_options 改用 get_obtainable_cards（過濾 _plus）。

### 7.5 事件 dialog 期間骰子按鈕還能按

**症狀**：事件對話開著，玩家又擲骰。

**原因**：`_resolve_event_tile` 沒 await dialog_closed 就 return → caller re-enable 按鈕。

**修法**：`await %DialogPanel.dialog_closed` 確保 dialog 關閉才 return（+ DialogPanel 的 dim ColorRect mouse_filter STOP 擋點擊）。

---

## 8. 下游依賴

| 後續週 | 依賴 W19 的什麼 |
|---|---|
| **W20 美術** | 事件加立繪 / QTE 加動畫 / DialogPanel portrait |
| **W21 音樂** | 事件 / QTE 加音效（AudioManager） |
| **W22 整合** | 事件 chain / inventory 系統 / QTE 多模式 |
| **W23 平衡** | 事件 effect 數值 / QTE 傷害 tune |
| **履歷** | 「資料驅動 VN 事件系統 + QTE」是 WorkNite 對口亮點 |

---

## 9. 面試 talking point

> 「W19 我為 DFS 加了三個內容系統：棋盤事件 VN、ITEM/SHOP tile、QTE 卡。
>
> 重點是 **VN 事件系統** — 因為 WorkNite 是 VN 工作室。我做了 EventData + EventChoice 兩層 resource，每個事件一個 .tres，裡面用 sub-resource 內嵌選項。踩到事件格 → DialogPanel 顯示分支 → 選擇套用後果。這就是簡化版的視覺小說對話樹，data-driven，designer 可以純編 .tres 加事件。
>
> 所有 board 後果（事件 / 道具 / 商店）統一走一個 `apply_board_effect(type, amount)` 入口 — reducer pattern，DRY，新效果一處加。
>
> **QTE 卡**致敬 WorkNite（VN/H game 常有 QTE）。技術上用 EventBus 解耦：卡片在 enemy._drop_data 偵測到是 QTE → emit signal，combat scene 監聽 → 跑限時連點 mini-game（一個回傳點擊數的 coroutine）→ 依點擊數算傷害。enemy 不需知道 QTE UI 存在，combat 統籌。單一職責。
>
> 順手修了一個 reward 系統 bug — 之前升級版卡（STRIKE+）會被直接 offer，破壞升級系統。加了 `get_obtainable_cards` 用命名慣例（id 結尾 _plus）過濾。」

---

**M5 (W18-W22) 進度：40% (2/5)**
**內容池**：17 base + 10 upgrade = 27 張卡 / 5 隻敵人 / 3 事件 / 1 QTE 卡
**下一週 W20**：Phase 8 美術（1/2）— Aria 立繪 + 卡牌 illustration
