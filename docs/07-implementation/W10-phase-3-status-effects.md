# W10 / DFS Phase 3 (2/3)：Status Effect 系統（poison / weak / vulnerable / strength）

> 對應 PROGRESS.md W10 / DFS docs/08-implementation/steps/phase-3-combat-mature.md（status 段）
> 完成日：2026-05-28
> Repo：[dice-fate-survivor](https://github.com/asd23353934/dice-fate-survivor)

---

## 1. 目標 / 不目標

### 目標

- **4 種 status**：poison（中毒）/ weak（虛弱）/ vulnerable（脆弱）/ strength（力量）
- **CardData 加 4 status 欄位**：applies_X_to_enemy / grants_X_to_self
- **GameState player_statuses**：Dictionary + apply / tick / compute_outgoing_damage
- **Enemy.statuses**：Dictionary + apply / tick / receive_attack（含 vulnerable）
- **回合結束 tick**：poison 扣血、weak/vulnerable 衰減、strength 永久
- **UI status labels**：玩家 / 敵人 status 即時顯示
- **填補 FOCUS placeholder**：W8 dmg=0 → W10 grants_strength=3
- **新增 POISON_DART**：dmg=2 + applies_poison=3，示範 status 系統

### 不目標（W11+ 補）

- **dexterity** 狀態（同 strength 模式，省去學習重複）
- **shield** 列為 status（W8 是獨立變數，保留）
- **status icons / 動畫** — W22 polish
- **StatusData.tres**（每個 status 一個 Resource）— 4 種還沒到值得抽象
- **Combo Lock 招牌機制** — W11 寫
- **status 互動 stack 邏輯**（如多 stack vs duration 怎麼合）— W11 補

---

## 2. 設計決策

### 為什麼用 `Dictionary{status_id: stacks}` 不用 enum + array？

選項對照：

| 方案 | 優 | 缺 |
|---|---|---|
| **Dictionary**（採用） | 加新 status 不改 enum；序列化簡單；遠端查找 O(1) | 沒 type safety（字串 typo 沒人檢查） |
| Enum + Array{StatusData} | 編譯期 type check；IDE autocomplete | 加新 status 要改 enum；遠端查找慢 |
| StatusData (Resource) | 可 .tres、可帶 icon / description | W10 4 種太少不值得抽象 |

**選 Dictionary 的關鍵**：W10 是 prototype 階段，**靈活性 > type safety**。W11 / W18 確定 status 列表穩定（10+ 種）才考慮 .tres 化。

### Status 4 種的時間語意

| Status | tick 時行為 | 來源 |
|---|---|---|
| **poison** | 扣血 = stacks，stacks -= 1 | 卡牌：applies_poison_to_enemy |
| **weak** | 回合 -1，到 0 移除 | 卡牌：applies_weak_to_enemy |
| **vulnerable** | 回合 -1，到 0 移除 | 卡牌：applies_vulnerable_to_enemy |
| **strength** | **永久不衰減**（戰鬥內） | 卡牌：grants_strength_to_self |

**設計差別**：
- **stacking + decay**（poison）：壓力遞減型
- **duration**（weak / vulnerable）：時效型
- **permanent**（strength）：累積型（永遠加）

跟 Slay the Spire 行為一致。

### 為什麼 status 修飾在「**輸出端 vs 受傷端**」分開？

```
玩家攻擊：base_damage = card.damage
  ↓ GameState.compute_outgoing_damage(base)
  · + strength
  · × 0.75 if weak
  ↓ 算出 modified_damage
  ↓ Enemy.receive_attack(modified_damage)
  · × 1.5 if vulnerable
  ↓ Enemy.take_damage(final)
```

| 修飾位置 | 責任 |
|---|---|
| `GameState.compute_outgoing_damage` | 玩家攻擊輸出端：strength + / weak − |
| `Enemy.receive_attack` | 敵人受傷端：vulnerable + |

**為什麼分開**：
- strength / weak 屬於攻擊者「自己的狀態」
- vulnerable 屬於受擊者「自己的狀態」
- 各自封裝，不交叉依賴

對比反例 — 全部塞 enemy._drop_data：
```gdscript
# bad: enemy 知道太多攻擊者細節
var dmg = card.damage
dmg += GameState.player_statuses.get("strength", 0)
if "weak" in GameState.player_statuses: dmg *= 0.75
if "vulnerable" in statuses: dmg *= 1.5
take_damage(dmg)
```

**鬆耦合 = 各自 own 自己的狀態邏輯**。

### 為什麼 tick 順序這樣排？

```
End Turn 流程：
  1. 玩家棄手牌
  2. 敵人 tick statuses    ← 中毒先扣血
  3. 敵人攻擊              ← 攻擊有 weak 修飾
  4. 玩家 tick statuses    ← 玩家也吃 poison（如果有）
  5. 玩家抽 5 + 重置能量
```

**順序意圖**：
- 敵人先 tick → 敵人中毒先扣（玩家上回合打的 poison 立刻見效）
- **再** 敵人攻擊（這時 weak 還在效，攻擊衰減）
- 玩家 tick **在敵人攻擊之後** → 玩家可能被攻擊死，避免 tick 死人邏輯

替代方案：tick 全在 turn 開始，但 Slay the Spire 就是 turn end tick，沿用減少新概念負擔。

### 為什麼 status UI 用文字（不用 icons）？

- W10 是邏輯驗證階段
- icons 需要美術資源（要 W18-W21）
- 文字「中毒 3」清楚 + 不用 asset

W22 polish 時加 icons + tooltip。

### 為什麼新增 POISON_DART 而不只更新既有卡？

兩種拓展：

| 方案 | 行為 |
|---|---|
| **加新卡 POISON_DART** | 新 .tres，pool 從 10 → 11 張 |
| 改 STRIKE 加 poison | 既有卡多功能化 |

**選加新卡**：
- POISON_DART **存在感明確**（玩家拖出來就知道用 poison）
- STRIKE 維持「基本攻擊」純粹角色
- 未來有更多 status 卡時邊界清楚

---

## 3. 檔案清單

```
新增：
└── data/cards/poison_dart.tres        ← 1 dmg 2 + poison 3

修改：
├── scripts/data/card_data.gd           ← 加 4 個 status @export 欄位
├── data/cards/focus.tres               ← grants_strength_to_self = 3
├── data/cards/bash.tres                ← applies_weak_to_enemy = 2
├── scripts/autoload/game_state.gd      ← player_statuses + apply/tick/compute methods
├── scripts/autoload/event_bus.gd       ← 加 player_status_changed / enemy_status_changed
├── scripts/autoload/card_database.gd   ← KIT_SWORD 加 focus + poison_dart
├── scripts/combat/enemy.gd             ← statuses dict + apply/tick + receive_attack
├── scenes/combat/combat.tscn           ← PlayerStatusLabel + EnemyStatusLabel
└── scenes/combat/combat.gd             ← tick at end turn + status UI handlers
```

---

## 4. 實作流程 / 順序

```
1. 設計：4 status 類別 + 修飾在攻擊輸出 vs 受傷端 分離
2. CardData 加 4 status @export 欄位（默認 0）
3. 修 FOCUS / BASH .tres 填新欄位（W8 placeholder 補完）
4. 加 POISON_DART .tres（新 status 卡示範）
5. GameState 加 player_statuses + apply_player_status / tick_player_statuses
6. GameState 加 compute_outgoing_damage（攻擊端修飾）
7. EventBus 加 player_status_changed / enemy_status_changed
8. Enemy 加 statuses + apply_status / tick_statuses
9. Enemy 加 receive_attack（vulnerable 修飾）
10. Enemy.enemy_turn_attack 加 weak 修飾
11. Enemy._drop_data 補：套 status_to_enemy + status_to_self
12. combat.gd end turn 流程加：tick enemy → enemy attack → tick player
13. combat.gd 加 status UI handler + _format_statuses helper
14. combat.tscn 加 PlayerStatusLabel + EnemyStatusLabel
15. CardDatabase kit_starter_decks 加 focus / poison_dart
16. F5 驗證
17. 寫 W10 guide
18. 雙 commit + push
```

---

## 5. 關鍵 code 解析

### `GameState.compute_outgoing_damage`

```gdscript
func compute_outgoing_damage(base: int) -> int:
    var dmg = base
    dmg += player_statuses.get("strength", 0)
    if "weak" in player_statuses:
        dmg = int(dmg * 0.75)
    return max(0, dmg)
```

**順序很重要**：先 + strength 再 × weak（Slay the Spire 標準）。

例：base 6、strength 3、weak 1 → (6+3) × 0.75 = 6.75 → int = 6
反例：base 6、strength 3、weak 0 → 6+3 = 9
反例：base 6、strength 0、weak 1 → 6 × 0.75 = 4.5 → int = 4

`max(0, ...)` 防衛性：避免 status 過度時負數。

### `Enemy.receive_attack` vs `Enemy.take_damage`

```gdscript
func receive_attack(base_damage: int) -> void:
    if hp <= 0 or base_damage <= 0:
        return
    var final = base_damage
    if "vulnerable" in statuses:
        final = int(final * 1.5)
    take_damage(final)
```

**兩個 method 分開**：
- `take_damage(N)` — 直接扣 HP（無修飾）。中毒 tick / 真實傷害用這個。
- `receive_attack(N)` — 攻擊邏輯（含 vulnerable）。卡牌出招用這個。

**為什麼**：
- 中毒不該被 vulnerable 雙重修飾（中毒在 tick 階段已是「最終值」）
- 真實傷害（例如環境傷害）不該被 status 影響

### `Enemy.apply_status` + tick 流程

```gdscript
func apply_status(status_id: String, stacks: int) -> void:
    if hp <= 0 or stacks <= 0:
        return
    statuses[status_id] = statuses.get(status_id, 0) + stacks
    EventBus.enemy_status_changed.emit(self, statuses.duplicate())


func tick_statuses() -> void:
    if hp <= 0:
        return
    # poison：扣血 + 衰減 1
    if "poison" in statuses:
        var stacks = statuses["poison"]
        take_damage(stacks)
        statuses["poison"] -= 1
        if statuses["poison"] <= 0:
            statuses.erase("poison")
    # weak / vulnerable：duration -1
    for sid in ["weak", "vulnerable"]:
        if sid in statuses:
            statuses[sid] -= 1
            if statuses[sid] <= 0:
                statuses.erase(sid)
    # strength：永久（不在 tick 處理）
    EventBus.enemy_status_changed.emit(self, statuses.duplicate())
```

**重點**：
- `statuses.duplicate()` 廣播時傳 copy（防止訂閱者誤改）
- `for sid in ["weak", "vulnerable"]` 短 array 直接 iterate（4 種還用不到 lookup table）
- `if "poison" in statuses` 先檢查再讀，避免 KeyError

### `combat.gd` 的 End Turn 順序

```gdscript
func _on_end_turn_pressed() -> void:
    # 1. 棄手牌
    GameState.discard_hand()

    # 2. 敵人先 tick（poison 立刻見效）
    if %Enemy.hp > 0:
        %Enemy.tick_statuses()

    # 3. 敵人攻擊（含 weak 修飾）
    if %Enemy.hp > 0:
        %Enemy.enemy_turn_attack()

    # 4. 玩家 tick（玩家中毒等）
    if GameState.player_hp > 0:
        GameState.tick_player_statuses()

    # 5. 新回合：抽 5 + 重置能量
    if GameState.player_hp > 0:
        GameState.energy = GameState.max_energy
        EventBus.energy_changed.emit(GameState.energy, GameState.max_energy)
        GameState.draw_card(5)
```

**重點**：
- 每步都 `if hp > 0` 檢查 — 防止中途死亡的 dirty state
- 順序：敵人 tick → 敵人攻擊 → 玩家 tick → 玩家新回合

### `combat.gd._format_statuses` UI helper

```gdscript
func _format_statuses(statuses: Dictionary) -> String:
    if statuses.is_empty():
        return ""
    var parts: Array[String] = []
    for sid in statuses:
        parts.append("%s %d" % [_status_label(sid), statuses[sid]])
    return " | ".join(parts)


func _status_label(status_id: String) -> String:
    match status_id:
        "poison": return "中毒"
        "weak": return "虛弱"
        "vulnerable": return "脆弱"
        "strength": return "力量"
        "dexterity": return "敏捷"
        _: return status_id
```

**重點**：
- `_status_label` 用 match 把 id 翻譯成中文
- W11 多語系時改成 `tr("status_" + status_id)` 走 i18n
- `" | ".join(parts)` 多 status 用「|」分隔（多 status 視覺好看）

---

## 6. 觀念對照（前端視角）

| Godot | 前端 / 一般軟體 |
|---|---|
| `player_statuses: Dictionary{id: stacks}` | Vuex state.statuses / React useState({}) |
| `apply_player_status` 內 emit | mutation + dispatch event |
| `compute_outgoing_damage` 修飾鏈 | Redux middleware / Pinia getter |
| `tick_statuses` 回合結束 | Cron / scheduled task / game loop tick |
| status_id 字串 key | enum (Java) / const string / Symbol (JS) |
| `_format_statuses` 渲染 | Vue computed / React render function |
| EventBus.player_status_changed | Vuex `subscribe` / RxJS Subject |

**核心思想：「狀態變動 → 廣播 → UI 響應」全套**。跟前端 reactive store 一致。

---

## 7. 擴充方式

### 加新 status：dexterity

```gdscript
# CardData 加：
@export var grants_dexterity_to_self: int = 0
```

```gdscript
# GameState：dexterity 修飾 shield 增益
func compute_shield_gain(base: int) -> int:
    return base + player_statuses.get("dexterity", 0)

# enemy._drop_data 改：
if card_data.shield > 0:
    var modified_shield = GameState.compute_shield_gain(card_data.shield)
    GameState.gain_shield(modified_shield)
```

```gdscript
# tick：dexterity 不衰減（同 strength 永久）
```

```gdscript
# _status_label 加：
"dexterity": return "敏捷"
```

加 1 種 status ≈ 1 個 CardData 欄位 + 1 個 modifier method + 1 行 label。**架構撐得住線性擴充**。

### 加給敵人施加 strength（buff 敵人）

```gdscript
# 假設新敵人 type 自己 buff 自己：
# enemy.gd
func enemy_turn_attack():
    apply_status("strength", 2)   ← 敵人自 buff
    var dmg = attack_damage + statuses.get("strength", 0)  ← 包含 self strength
    if "weak" in statuses:
        dmg = int(dmg * 0.75)
    GameState.take_damage(dmg)
```

DFS Boss 階段（W15）會用這 pattern。

### 加 stacks 上限

某些 status 應該有 cap（如「中毒上限 10 層」）：

```gdscript
const STATUS_CAPS = {
    "poison": 10,
    "weak": 5,
    "strength": 99,
}

func apply_status(status_id, stacks):
    var new_stacks = statuses.get(status_id, 0) + stacks
    if status_id in STATUS_CAPS:
        new_stacks = min(new_stacks, STATUS_CAPS[status_id])
    statuses[status_id] = new_stacks
```

之後平衡時加。

### StatusData.tres 化（W11-W12 重構）

當 status 增加到 10+ 種，Dictionary 不夠用：

```gdscript
class_name StatusData
extends Resource

enum DurationType { STACKING_DECAY, DURATION, PERMANENT }
enum TickEffect { POISON_DMG, NONE }
enum DamageModifier { OUTGOING_PLUS, OUTGOING_MULT, INCOMING_MULT }

@export var id: String
@export var status_name: String
@export var icon: Texture2D
@export var duration_type: DurationType
@export var tick_effect: TickEffect
@export var damage_modifier: DamageModifier
@export var modifier_value: float
@export_multiline var description: String
```

跑 StatusDatabase autoload load 所有 .tres → fully data-driven。

但 W10 才 4 種 → **過度抽象 = waste**。等需要時再 refactor。

### Status 互動：burning 點燃 weak（W12 例）

某些 status 之間有互動：「攻擊 weak 敵人額外點燃」

```gdscript
# enemy.receive_attack 改：
func receive_attack(base_damage):
    if hp <= 0 or base_damage <= 0: return
    var final = base_damage
    if "vulnerable" in statuses:
        final = int(final * 1.5)
    take_damage(final)
    # W12+：weak 敵人受傷時 +1 burn
    if "weak" in statuses and "burn" in GameState.player_passives:
        apply_status("burn", 1)
```

這種「組合反應」是中後期遊戲設計核心。W11+ 補。

---

## 8. 常見錯誤 / debug 指南

### Status 無限疊加沒衰減

**症狀**：weak 永遠不消失
**根因**：tick_statuses 沒被 call
**修法**：combat.gd 的 End Turn 內加 `Enemy.tick_statuses()` / `GameState.tick_player_statuses()`

### Strength 加成不生效

**症狀**：拖 FOCUS 後拖 STRIKE 還是 6 dmg（應該 9）
**根因**：enemy._drop_data 沒呼叫 `GameState.compute_outgoing_damage`
**檢查**：log 「[GameState] compute_outgoing_damage(6) → ?」確認流程

### Poison 扣血超模型

**症狀**：中毒 3 結果扣 9 HP
**根因**：tick 邏輯錯（如多次 emit / 多次 take_damage）
**修法**：tick 只 take_damage(stacks) 一次

### UI status label 沒更新

**症狀**：status 變了 UI 不變
**根因**：
1. label 沒 connect status_changed
2. status_changed emit 時資料是空（沒 duplicate）
**修法**：emit 時 `statuses.duplicate()` + handler 用 `_format_statuses`

### Status_id 字串 typo

**症狀**：apply_status("posion", 3)（typo）後 `_format_statuses` 印「posion 3」
**根因**：Dictionary 無 type check
**預防**：W11+ 用 const string：
```gdscript
const STATUS_POISON = "poison"
const STATUS_WEAK = "weak"
```

或更安全：用 enum + mapping。但 W10 trade-off 接受。

### 敵人死亡後 status tick 繼續

**症狀**：敵人 HP 0 但 status 還 tick
**修法**：tick_statuses 開頭 `if hp <= 0: return`（已有）

### Status icon UI 沒對齊（W22 時）

**症狀**：多 status 排版亂
**修法**：W22 美術階段用 HBoxContainer + icons + Tooltip 重做。W10 純文字夠用。

---

## 9. 下游依賴

| 之後階段 | 怎麼用 W10 成果 |
|---|---|
| **W11 Combo Lock** | combo 機制可能用 status 表達（如「lock counter」） |
| **W11 卡牌測試** | GdUnit4 / 自寫 harness 測 compute_outgoing_damage 邊界 |
| **W11 dexterity** | 同 strength pattern 加 |
| **W12-W13 棋盤** | 棋盤 tile 觸發給敵人 status（如毒池格） |
| **W14-W15 整合 + Boss** | Boss 含複雜 status pattern（self-strength 累積） |
| **W18-W19 內容** | 加 5+ 種敵人，各自 attack_pattern + status 偏好 |
| **W18-W19 卡** | 加 5+ 張新 status 卡（如 寒冰、燃燒、印記） |
| **W22 polish** | status 改 icons + tooltip + 動畫 |
| **i18n** | _status_label 改 tr() |
| **StatusData.tres**（W11+） | 4 → 10 種時抽象成 Resource |

---

## 10. 面試 talking point

> 「W10 我實作 Status Effect 系統 — 4 種狀態（poison / weak / vulnerable / strength），覆蓋三種時間語意：stacking + decay（中毒）、duration（虛弱 / 脆弱）、permanent（力量）。
>
> 設計重點：**修飾鏈拆分輸出端 / 受傷端**。strength 跟 weak 屬於攻擊者「自己的狀態」，所以在 `GameState.compute_outgoing_damage` 內處理。vulnerable 屬於受擊者「自己的狀態」，在 `Enemy.receive_attack` 內處理。**各自封裝、不交叉**。
>
> 用 `Dictionary{status_id: stacks}` 不用 enum + array。**W10 4 種太少不值得抽象成 StatusData.tres**。等 W11+ 增加到 10+ 種再重構。**漸進式抽象 = 不 over-engineer**。
>
> 順序很重要：End Turn → 敵人先 tick（中毒立刻見效）→ 敵人攻擊（含 weak 衰減）→ 玩家 tick → 玩家新回合。每步檢查 hp > 0 防 dirty state。
>
> 整套撐得住擴充 — 加 dexterity 就是 +1 CardData 欄位、+1 modifier method、+1 label translation。**架構正確 = 線性擴充無痛**。
>
> 對應前端：跟 reactive store + middleware + computed render 同 pattern。**遊戲狀態管理跟前端應用狀態管理本質上一致**。」
