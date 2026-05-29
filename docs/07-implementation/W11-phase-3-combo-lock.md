# W11 / DFS Phase 3 (3/3)：Combo Lock 招牌機制 + 戰鬥公式測試

> 對應 PROGRESS.md W11 / DFS docs/01-design/combat-formulas.md §1.5 + Step 2.8
> 完成日：2026-05-28
> Repo：[dice-fate-survivor](https://github.com/asd23353934/games/dice-fate-survivor)

---

## 1. 目標 / 不目標

### 目標

- **Combo Lock universal 機制**（DFS 招牌）：連續同類卡牌觸發 multiplier
  - 1 張：1.0×
  - 2 張同類：1.2×
  - 3+ 張同類：1.5× cap
  - 不同類 / End Turn 重置
- **CardData 加 CardType enum**（ATTACK / SKILL / POWER / ITEM）
- 11 張卡分類：7 Attack / 3 Skill / 1 Power
- **Combo multiplier 套用範圍**：damage / shield / status stacks
- **不套用範圍**：strength 永久 buff（避免 Power exploit）
- **戰鬥公式測試**：11 個 self-written tests（沿用 W3.5 / W4 pattern）
- **UI**：ComboLabel 顯示「連擊：攻擊 ×3 → 1.5x」

### 不目標（之後階段）

- **CardType 對應的 click 觸發重構**（SKILL / POWER 改點擊）— 留 W12+，UX 完美 vs 功能 work 的 trade-off
- **5 波結構**（Phase 2 Step 2.8）— 留 W12 棋盤前一起做
- **Combo + Item 卡互動**（Item exhaust 但仍累 combo）— 等 W19 加 Item 卡時實作
- **戰鬥動畫 / 飄字 / shake** — W22 polish
- **W11 mini-gate playtest**（DFS spec 寫的「自評戰鬥手感」步驟）— 用戶自行決定

---

## 2. 設計決策

### 為什麼 Combo Lock 是「招牌」？

DFS spec 2026-05-15 決議：
> Universal 出牌系統升級：加入 **Combo Lock 機制**（連續同類卡：×1.0 → ×1.2 → ×1.5 cap），全流派通用，每回合 reset

**設計理由**：
- **簡單直觀**：玩家秒懂「同類連打有 buff」
- **策略深度**：rush combo 3 vs 分散打牌 trade-off
- **UI 飄字爽快感**：1.5x 倍率有「爆擊」感
- **全流派通用**：劍流 / 法術流 / 召喚流都用，學一次到處用
- **DFS 跟 Slay the Spire 區隔**：StS 沒這個機制

### 為什麼 multiplier 1.0 / 1.2 / 1.5 不用 1.0 / 1.5 / 2.0？

| 倍率組 | 優 | 缺 |
|---|---|---|
| 1.0 / 1.5 / 2.0 | 爽快感強 | combo 3 太強，玩家強迫 rush |
| **1.0 / 1.2 / 1.5**（採用） | 平衡 | 1.2 視覺感不強 |
| 1.0 / 1.1 / 1.2 | 平衡保守 | 玩家無感 |

DFS 選平衡值。**1.2 不會讓「打 2 張」變必選**，1.5 是甜蜜點獎勵但不是必要。

### 為什麼 CardType **4 類**而不是 6 類？

選項：
- 6 類：Attack / Skill / Power / Item / Curse / Status
- **4 類**（採用）：Attack / Skill / Power / Item

理由：
- Curse / Status 不是「主動出牌」（沒法 combo）
- Slay the Spire 也基本只有 Attack / Skill / Power 三類，Item 是 DFS 特有

### Combo 為何不套 strength permanent buff？

```
FOCUS card_type=Power, grants_strength=3
拖 FOCUS×3 → 玩家 +9 strength？  ← 太強
拖 FOCUS×3 → 玩家 +3 strength（不乘 combo）   ← 採用
```

**因為 Power 是永久效果**，combo 倍率作用於「一次性」效果（攻擊瞬間、護盾瞬間、status 瞬間施加），不該作用於「跨回合永久」效果。

否則玩家會堆 Power combo 3 → 力量永久 +9 → 滾雪球失控。

### Combo 套用「status stacks」採 `floor` + min 1

```gdscript
var stacks = max(1, int(card_data.applies_poison_to_enemy * combo_mult))
```

例：poison 3 × 1.2 = 3.6 → int = 3（取 floor，沒進 4）
例：poison 1 × 1.0 = 1 → 1
例：poison 1 × 1.5 = 1.5 → int = 1（取 floor）

**max(1, ...)**：防止小數 status 變 0（如 poison 1 × 0.5 = 0.5 → 0 是 bug）。

### `card_type` 為什麼 enum 而不是字串？

| | Enum | String |
|---|---|---|
| Type safety | ✓ | ✗（typo 不檢查） |
| Inspector 選單 | dropdown | text field |
| 序列化 | int（小） | string |
| 比較 | == int | == string |
| 加新類 | 改 enum 然後 update .tres | 直接寫 |

**選 enum**：Inspector dropdown 防 typo + 性能好。但 GameState.on_card_played 接 `String`（從 enum 轉），方便 print 顯示。

### 為什麼自寫 test harness 而不用 GdUnit4？

延續 W3.5 決定：
- W3.5 踩過 GdUnit4 v6.0 跟 4.6.3 API drift 的雷
- self-written harness 跨版本免疫
- 11 個測試規模還小，框架 overhead 不值得

**W18-W19 內容填充時測試規模到 50+**，可能值得重新考慮 GdUnit4 v6.1+。

---

## 3. 檔案清單

```
新增：
├── tests/test_runner.gd                    ← 11 個 combat-formulas 測試
└── tests/test_runner.tscn

修改：
├── scripts/data/card_data.gd               ← 加 CardType enum + card_type 欄位 + get_type_string()
├── data/cards/defend.tres                  ← card_type = 1 (SKILL)
├── data/cards/parry.tres                   ← card_type = 1 (SKILL)
├── data/cards/brace.tres                   ← card_type = 1 (SKILL)
├── data/cards/focus.tres                   ← card_type = 2 (POWER)
├── scripts/autoload/game_state.gd          ← combo state + on_card_played / get_current_combo_multiplier / reset_combo
├── scripts/autoload/event_bus.gd           ← combo_changed signal
├── scripts/combat/enemy.gd                 ← _drop_data 套 combo 到 damage/shield/status
├── scenes/combat/combat.tscn               ← ComboLabel
└── scenes/combat/combat.gd                 ← 訂閱 combo_changed + UI + reset on turn end
```

---

## 4. 實作流程 / 順序

```
1. 讀 DFS spec：docs/01-design/combat-formulas.md §1.5 Combo Lock 規範
2. CardData 加 enum CardType + card_type 欄位 + get_type_string()
3. 改 4 張 .tres（defend/parry/brace card_type=1 SKILL，focus card_type=2 POWER）
   · Attack 類保持 default 0，省 .tres 改動
4. GameState 加 combo state:
   · current_combo_count / current_combo_type
   · on_card_played(type_str) — 連續同類 ++，換類 = 1
   · get_current_combo_multiplier() — 回 1.0 / 1.2 / 1.5
   · reset_combo() — 新戰鬥 / End Turn 用
5. EventBus 加 combo_changed signal
6. start_combat_with_deck 加 reset_combo() 初始化
7. enemy._drop_data 改：
   · 先 GameState.on_card_played(card.get_type_string())
   · 拿 combo_mult
   · damage * combo_mult / shield * combo_mult / status stacks * combo_mult（max 1 floor）
   · strength permanent buff 不乘
8. combat.gd End Turn 末加 GameState.reset_combo()
9. combat.gd 訂閱 combo_changed + 寫 _on_combo_changed / _combo_type_label
10. combat.tscn 加 ComboLabel
11. 寫 tests/test_runner.gd 11 個測試
12. F5 + F6 雙驗證
13. 寫 W11 implementation guide
14. 雙 commit + push
```

---

## 5. 關鍵 code 解析

### `GameState.on_card_played` — Combo 觸發

```gdscript
func on_card_played(card_type_string: String) -> void:
    if card_type_string == current_combo_type:
        current_combo_count += 1
    else:
        current_combo_type = card_type_string
        current_combo_count = 1
    EventBus.combo_changed.emit(
        current_combo_count,
        current_combo_type,
        get_current_combo_multiplier()
    )
```

**設計**：
- 同類 → count++
- 換類 → 重設 type + count = 1（**不是 0！** 第一張立刻算 1）
- emit 一個 signal 把 3 個值打包（不分開 emit 避免 partial state）

### `get_current_combo_multiplier` — 倍率表

```gdscript
func get_current_combo_multiplier() -> float:
    if current_combo_count <= 1: return 1.0
    if current_combo_count == 2: return 1.2
    return 1.5    # 3+ cap
```

**選簡單分支不用 array map**：3 個值寫死，可讀性 > 抽象。
DFS 平衡如果之後改 1.0/1.3/1.7 → 改 3 個字面值就好。

### `enemy._drop_data` 整合 Combo

```gdscript
func _drop_data(_at_position, drop_data):
    var card_data: CardData = drop_data.data
    if not GameState.spend_energy(card_data.cost): return

    # W11：先 update combo 再算
    GameState.on_card_played(card_data.get_type_string())
    var combo_mult = GameState.get_current_combo_multiplier()

    # 套 combo 到 damage
    if card_data.damage > 0:
        var base = GameState.compute_outgoing_damage(card_data.damage)
        receive_attack(int(base * combo_mult))

    # 套 combo 到 shield
    if card_data.shield > 0:
        GameState.gain_shield(int(card_data.shield * combo_mult))

    # 套 combo 到 status stacks（floor + min 1）
    if card_data.applies_poison_to_enemy > 0:
        apply_status("poison", max(1, int(card_data.applies_poison_to_enemy * combo_mult)))

    # ❌ strength permanent 不套（W11 spec）
    if card_data.grants_strength_to_self > 0:
        GameState.apply_player_status("strength", card_data.grants_strength_to_self)
```

**順序很重要**：
1. **先** update combo（拿 multiplier）
2. **再** 套用效果

倒過來會用「上一張卡的 multiplier」算這張，邏輯錯。

### Test harness — Float 比對

```gdscript
func _expect_float(actual: float, expected: float, description: String) -> String:
    if abs(actual - expected) < 0.001:
        return "PASS: " + description
    return "FAIL: %s | expected %.3f, got %.3f" % [description, expected, actual]
```

**重點**：Float 比對**不用 ==**，用 epsilon（0.001）。
浮點數運算累積誤差，1.0 + 0.1 + 0.1 + 0.1 ≠ 1.3 嚴格相等。

### Test runner — Setup 隔離

```gdscript
func _setup() -> void:
    GameState.player_hp = 80
    GameState.max_hp = 80
    GameState.energy = 3
    GameState.shield = 0
    GameState.player_statuses.clear()
    GameState.reset_combo()
```

**重點**：每個 test 跑前手動重置 GameState。
**省略**：discard / hand / deck（這些測試不關心）。
**好處**：test 之間無 side effect，順序無關。

---

## 6. 觀念對照（前端視角）

| Godot | 前端 / 一般軟體 |
|---|---|
| `current_combo_count / current_combo_type` | Redux state slice |
| `on_card_played` middleware | Redux action handler |
| `get_current_combo_multiplier` selector | Reselect selector / Pinia getter |
| `EventBus.combo_changed.emit(3 args)` | Single `dispatch` with payload obj |
| Float epsilon 比對 | `expect.toBeCloseTo(0.3, 5)` (Jest) |
| `_setup` test 重置 | `beforeEach` setup |
| 自寫 test runner | Custom harness (when Jest overkill) |

**整體**：W11 把戰鬥邏輯從「散落各 .gd」收斂到「GameState 集中算 multiplier」，類似 Redux 的「邏輯放 reducer / dispatcher」設計。

---

## 7. 擴充方式

### 加新 CardType（例：STATUS）

```gdscript
# CardData
enum CardType { ATTACK, SKILL, POWER, ITEM, STATUS }

func get_type_string():
    match card_type:
        # ...
        CardType.STATUS: return "Status"
```

`_combo_type_label` / spec 都不用動 — 新類自動進 combo 系統。

### 改 combo 倍率公式

DFS 平衡覺得「1.2 太弱」想改 1.3：

```gdscript
func get_current_combo_multiplier():
    if current_combo_count <= 1: return 1.0
    if current_combo_count == 2: return 1.3    # ← 改這
    return 1.5
```

**測試也要改**：`test_combo_2_same_type_gives_1_2x` 改成 1.3，**否則測試掛**——這是測試的正面價值。

### Combo + Item 卡（W19）

DFS spec 寫「Item 卡（exhaust）效果累 combo」。實作：

```gdscript
# Item 卡不討論 cost，直接 trigger
func use_item(item_data):
    GameState.on_card_played(item_data.get_type_string())    # 累 combo
    var mult = GameState.get_current_combo_multiplier()
    # apply effect with mult
```

不用改 combo 系統。

### Combo + Boss 階段（boss buff "combo immune"）

```gdscript
# 假設 Boss 有 status "combo_immune"
# enemy._drop_data 改：
var combo_mult = 1.0 if "combo_immune" in statuses else GameState.get_current_combo_multiplier()
```

機制留好接口，boss 設計時補。

### 加更多 combat-formulas 測試

接 W10 / W11 既有：
```
test_poison_tick_damage_and_decay
test_enemy_take_damage_with_vulnerable
test_enemy_weak_reduces_attack
test_combo_3_strikes_total_damage
test_focus_then_strike_strength_applies
test_focus_then_focus_grants_strength_not_multiplied  ← regression 防 Power exploit
```

W18-W19 cards 補完後測 20+。

### StatusData.tres 化（W12+）

`Dictionary{status_id: stacks}` 增至 8+ 種時抽象成 Resource：

```gdscript
class_name StatusData
extends Resource

enum DurationType { STACKING_DECAY, DURATION, PERMANENT }
@export var id: String
@export var duration_type: DurationType
@export var tick_damage_per_stack: int
@export_multiline var description: String
```

CardData 改：
```gdscript
@export var status_applications: Array[StatusApplication] = []
# 替換掉 applies_poison_to_enemy 等個別欄位
```

W11 還不到那個門檻 — **YAGNI**。

---

## 8. 常見錯誤 / debug 指南

### Combo multiplier 跟 enemy 看到的數字對不上

**症狀**：UI 顯示 ×1.5 但敵人扣血計算用 ×1.2
**根因**：`on_card_played` 跟 `get_current_combo_multiplier` 順序倒了
**修法**：在 `enemy._drop_data` 內，**先 on_card_played 後 get_current_combo_multiplier**

### Float 測試莫名 fail

**症狀**：`expected 1.2, got 1.2` 但 fail
**根因**：`==` 嚴格比對遇到 1.2000001 vs 1.1999999
**修法**：用 epsilon 比對 `abs(actual - expected) < 0.001`

### Combo 在新戰鬥沒重置

**症狀**：上一戰最後 combo 3，新戰一進去 combo 仍 3
**根因**：start_combat_with_deck 沒 reset
**修法**：W11 已加 `reset_combo()` 進 start_combat_with_deck（檢查有沒漏）

### CardType enum 在 .tres 存錯數字

**症狀**：FOCUS 應該 Power 但 combo 顯示「攻擊」
**根因**：.tres 內 `card_type = 0` (Attack default)，沒設 2
**修法**：手動編輯 .tres 改 `card_type = 2`，或 Inspector 重選 dropdown

### Strength 被乘 combo（W10 bug regression）

**症狀**：拖 FOCUS×3 → 力量 +9 而不是 +3
**根因**：`grants_strength_to_self` 路徑誤套 combo_mult
**修法**：W11 spec 規定 strength **不**乘 combo，code 跳過 strength 那段

---

## 9. 下游依賴

| 之後階段 | 怎麼用 W11 成果 |
|---|---|
| W12 棋盤 | board 上 tile 可能觸發 combat 直接套 combo（無 hand 互動） |
| W14-W15 整合 | boss 階段可能 immune combo（用 status 表達） |
| W15 Boss 多 phase | phase 切換時 reset combo（或 buff combo） |
| W16-W17 Hub | 永久升級「+0.1 combo multiplier」 |
| W18-W19 內容填充 | 新卡 / 新 status 直接接 combo 系統 |
| W20-W21 美術 | combo 變化 emit signal → 動畫 / 音效 hook |
| W22 polish | combo 3 達成 → 飄字「COMBO!」+ shake |
| 戰鬥公式測試 | 每加新 status / card effect → 補測試（regression 防護） |
| Telemetry | combo_changed signal → log 玩家 combo 偏好（rush vs spread） |

---

## 10. 面試 talking point

> 「W11 是 DFS 的招牌週 — Combo Lock universal 機制。連續同類卡（Attack/Skill/Power）→ 倍率 1.0 / 1.2 / 1.5 cap。每回合重置。
>
> 設計重點：**multiplier 套用範圍精選**。damage / shield / status stacks 套（一次性瞬間效果），strength permanent buff 不套（避免 Power exploit）。**對戰鬥節奏正確 — 玩家不會用 FOCUS combo 滾雪球**。
>
> CardType enum 用 4 類（Attack/Skill/Power/Item）。Inspector dropdown 防 typo。GameState 用 String 接 type（從 enum get_type_string 轉），方便 log + signal payload。
>
> 順序很重要：`enemy._drop_data` 內**先 on_card_played 後 get_multiplier 後 套效果**。倒序會用上一張卡 multiplier 算本張。
>
> 寫 **11 個 self-written combat-formulas 測試**，沿用 W3.5 / W4 pattern：
> - 6 個 combo（multiplier / type 切換 / cap / reset）
> - 3 個 damage 修飾鏈（strength + weak 順序）
> - 2 個 shield + take_damage 吸收溢出
>
> Float 比對用 epsilon 0.001（不是 ==）。Setup 每 test 重置 GameState 隔離。
>
> 這套架構撐得住擴充：之後加新 CardType / 改 multiplier 公式 / boss combo immune / Combo Item 卡都不用大改 — **只動局部**。」
