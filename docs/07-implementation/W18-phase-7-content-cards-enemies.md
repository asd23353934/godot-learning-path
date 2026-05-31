# W18 / Phase 7 (1/2)：內容填充 — 補 5 張新卡 + 5 張升級版 + 3 隻敵人 + draw_cards 機制

> 對應 repo：[dice-fate-survivor](https://github.com/asd23353934/dice-fate-survivor)
> 對應 PROGRESS milestone：M5 W18（**M5 開工**）
> 寫於 Day 8（2026-05-31）

---

## 1. 目標 / 不目標

### 目標
- **5 張新卡**：INFLAME / IRON_WAVE / SURVIVOR / BATTLE_TRANCE + 修 POMMEL_STRIKE 的「抽 1」mechanic
- **5 張既有卡升級版**：CLEAVE+ / QUICK+ / PARRY+ / BRACE+ / FOCUS+（補齊升級樹）
- **3 隻新敵人**：SLIME / SPIKE_HOUND / STONE_GUARD（一隻 elite）
- **draw_cards 卡片欄位**：CardData 加新欄，enemy.gd `_drop_data` 整合
- **per-encounter 隨機敵人**：非 boss tile 從 4 種敵人池隨機選

### 不目標（W19 / 之後）
- 多 ATK target（cleave 註解寫過 multi，但 DFS 只 1 enemy combat — 之後做）
- ITEM 卡（道具系統，W19）
- 棋盤事件 tile（EVENT，W19）
- AOE 機制
- exhaust / single-use 卡
- TranslationServer i18n（W19）

---

## 2. 設計決策

### A. 為什麼新卡只加 5 張，不一次補 10+ 張？

| 方案 | 優點 | 缺點 |
|---|---|---|
| 一次加 15 張卡 | 內容豐富 | 平衡 nightmare，每張都要思考 reward/shop pool 影響 |
| **W18 加 5 + W19 加 5**（採用） | 漸進加，每加完 playtest | 內容生成節奏稍慢 |

→ **漸進加好**：每張新卡進入 reward 池都會影響 build path，5 張一批容易回溯找問題卡。

### B. 為什麼起始 deck（kit_starter_decks）不加新卡

- 起始 deck 10 張是 W10-W15 平衡測過的甜蜜點
- 新卡（如 BATTLE_TRANCE 0 費抽 3）改起始 deck 會 trivialize 早期戰鬥
- **新卡只透過 reward / 卡商**：玩家「贏一場戰鬥」才解鎖 build path 的能力
- 之後若想開 KIT 2 / KIT 3，再針對性編製 starter

### C. draw_cards 為什麼放卡片資料層，不放卡片 callback

| 方案 | 優點 | 缺點 |
|---|---|---|
| **欄位 driven**（採用） | 純 .tres 編輯，無 code 變動，新卡即插即用 | 複雜效果（條件抽、scry）這個 schema 不夠 |
| Script callback per card | 每張卡可任意自定義邏輯 | code/data 耦合，每卡都要寫 script |

→ **資料 driven**：DFS scope 內絕大多數卡可用「damage / shield / status / draw」幾個欄位組合表達。複雜卡之後 W19 再評估 callback 系統。

### D. draw_cards 觸發時機放 discard 之後

```gdscript
EventBus.card_played.emit(card_data, self)
GameState.discard_card(card_data)

# W18：draw 在 discard 後
if card_data.draw_cards > 0:
    GameState.draw_card(card_data.draw_cards)
```

- 若先 draw 再 discard：新抽的卡可能被誤包進 `hand` 然後 discard_hand 掃到（不會，但語意危險）
- discard 後 draw：當下的這張卡確定離開 hand，再抽新的乾淨

### E. 升級規則：cost / damage / shield 三選一強化

| 卡 | 升級類型 | 原 → 升級 |
|---|---|---|
| CLEAVE | damage 加成 | 7 → 10 |
| QUICK | damage 加成 | 3 → 5 |
| PARRY | shield 加成 | 3 → 5 |
| BRACE | shield 加成 | 12 → 16 |
| FOCUS | cost 降低 | 2 cost → 1 cost（damage 不變） |
| POMMEL_STRIKE | damage 加成 | 9 → 12（draw 不變） |
| INFLAME | strength 加成 | +2 → +3 |
| IRON_WAVE | damage + shield 同步 | 5/5 → 7/7 |
| SURVIVOR | shield 加成 | 8 → 11 |
| BATTLE_TRANCE | draw 加成 | 3 → 4 |

設計原則：**升級不改 cost** 為主（除 FOCUS — Power 卡 cost-1 是經典升級），保持 deck combo curve 穩定。

### F. STONE_GUARD 設為非 boss 但有 phase 2

- 想證明 EnemyData 的 `has_phase_2` 不只 boss 可用（**composition over inheritance** 證明）
- 玩家在 stage 內就可能遇到「會切階段」的精英敵 — 增加 surprise / 緊張感
- 之後可加更多「精英」敵人（HP > 35, has_phase_2 = true, is_boss = false）

### G. 新卡平衡核心：「每張卡都該有 build path 定位」

| 卡 | build 定位 |
|---|---|
| INFLAME | 低 cost Power 起手，給 strength build |
| IRON_WAVE | 攻防一體，cost 效率高，雜役起手 |
| SURVIVOR | 純高護盾，hold 致命一擊 |
| BATTLE_TRANCE | 0 cost 抽 3 = 無腦過卡 + 連 combo 套利器 |
| POMMEL_STRIKE | 高傷害 + 抽 1，攻擊 build 的 tempo 牌 |

每張卡都對應一個玩家想做的事，**不該有「為加而加」的填空卡**。

---

## 3. 檔案清單

### 新增 — 5 張卡 .tres

| 檔案 | 卡片 |
|---|---|
| `data/cards/inflame.tres` | INFLAME（POWER, cost 1, +2 strength） |
| `data/cards/iron_wave.tres` | IRON WAVE（ATTACK, cost 1, dmg 5 + shield 5） |
| `data/cards/survivor.tres` | SURVIVOR（SKILL, cost 1, shield 8） |
| `data/cards/battle_trance.tres` | BATTLE TRANCE（SKILL, cost 0, draw 3） |
| (修改) `data/cards/pommel_strike.tres` | 加 draw_cards = 1 |

### 新增 — 升級版 .tres × 10（補齊 + 新卡的）

| 檔案 | 對應升級 |
|---|---|
| `inflame_plus.tres` | +2 → +3 strength |
| `iron_wave_plus.tres` | 5/5 → 7/7 |
| `survivor_plus.tres` | shield 8 → 11 |
| `battle_trance_plus.tres` | draw 3 → 4 |
| `pommel_strike_plus.tres` | dmg 9 → 12 |
| `cleave_plus.tres` | dmg 7 → 10 |
| `quick_plus.tres` | dmg 3 → 5 |
| `parry_plus.tres` | shield 3 → 5 |
| `brace_plus.tres` | shield 12 → 16 |
| `focus_plus.tres` | cost 2 → 1 |

### 新增 — 3 隻敵人 .tres

| 檔案 | 敵人 |
|---|---|
| `data/enemies/slime.tres` | 腐肉史萊姆（HP 25, ATK 6, applies weak 1） |
| `data/enemies/spike_hound.tres` | 鐵刺獵犬（HP 22, ATK 9, applies vulnerable 1） |
| `data/enemies/stone_guard.tres` | 石牢守衛（HP 40, ATK 7→11 phase 2，**精英非 boss**） |

### 修改

| 檔案 | 變動 |
|---|---|
| `scripts/data/card_data.gd` | +`draw_cards: int` / `get_upgrade_diff` 加抽牌欄 |
| `scripts/combat/enemy.gd` | `_drop_data` discard 後 trigger `draw_cards` |
| `scenes/combat/combat.gd` | +`NON_BOSS_POOL` / `_decide_enemy_data` 隨機選非 boss |
| 既有 `cleave/quick/parry/brace/focus.tres` | 加 `upgraded_version` 連結升級版 |

---

## 4. 實作流程

1. **CardData schema 擴**：加 `draw_cards` 欄位 + diff 顯示
2. **enemy.gd 整合**：discard 後 draw
3. **新卡 .tres × 5 + upgrade × 10**：純 data，無 code
4. **新敵人 .tres × 3**：純 data
5. **combat.gd 隨機池**：preload + 改 `_decide_enemy_data`
6. **既有卡 link upgrade**：補既有 5 張 `upgraded_version`

順序原則：**schema → engine → data → integration**。schema 是 contract，先穩定。

---

## 5. 關鍵 code 解析

### 5.1 CardData `draw_cards` 欄位

```gdscript
# === W18：抽牌效果 ===
@export var draw_cards: int = 0
```

預設 0 = 不抽牌。`@export` 讓 Inspector 可編輯，新卡 .tres 設值即可。

### 5.2 enemy.gd 整合 draw

```gdscript
func _drop_data(_at_position, drop_data):
    ...
    EventBus.card_played.emit(card_data, self)
    GameState.discard_card(card_data)

    # W18：draw 在 discard 後（確保當前卡離開 hand）
    if card_data.draw_cards > 0:
        print("[Enemy] %s 觸發抽牌效果 ×%d" % [card_data.card_name, card_data.draw_cards])
        GameState.draw_card(card_data.draw_cards)
```

要點：
- discard 後 draw 避免「打 BATTLE_TRANCE 抽 3 → 結果這張本身被洗進新 hand」異常
- `GameState.draw_card(n)` 已有實作（W8 加，會自動 reshuffle discard 進 deck）

### 5.3 BATTLE_TRANCE 0 cost + 3 draw 的設計

`cost = 0`、`damage = 0`、`shield = 0`、`draw_cards = 3`。

效果：免費抽 3 張。極端 tempo 卡。
- 連 combo 用：抽到的卡可能繼續同類連擊
- combo lock 不會被打斷（因為 SKILL 類，連續 SKILL 一樣維持 multiplier）
- **不過量原因**：要在 reward 池抽到才能拿，shop 才能定期買，不是無限可得

### 5.4 升級版 .tres pattern（範例 FOCUS+）

```ini
# focus.tres
id = "focus"
cost = 2
grants_strength_to_self = 3
upgraded_version = ExtResource("2_upgrade")   # → focus_plus.tres

# focus_plus.tres
id = "focus_plus"
cost = 1                # ← 唯一變動
grants_strength_to_self = 3
```

cost-1 是 Power 卡標準升級。讓 Power build 玩家「升級 FOCUS」變成關鍵抉擇。

### 5.5 STONE_GUARD 精英敵設定

```ini
id = "stone_guard"
max_hp = 40              # 大塊頭
attack_damage = 7
has_phase_2 = true       # ← 非 boss 也能 2 phase
phase_2_threshold = 0.4  # HP < 40% 觸發
phase_2_attack_damage = 11
phase_2_announce_text = "石牢守衛切換攻擊姿態 — 攻擊力大幅提升！"
is_boss = false          # ← 但不算 boss
```

設計目的：證明 EnemyData 的 `has_phase_2` 跟 `is_boss` 解耦。實際玩法：玩家以為「啊只是隻普通敵」結果打到一半被切階段嚇到 — surprise mechanic。

### 5.6 隨機敵人池

```gdscript
const NON_BOSS_POOL: Array[EnemyData] = [
    RAT_DATA, SLIME_DATA, SPIKE_HOUND_DATA, STONE_GUARD_DATA
]

func _decide_enemy_data() -> EnemyData:
    if GameState.current_tile_index == GameState.boss_tile_index:
        return DICE_LORD_DATA
    var picked = NON_BOSS_POOL[randi() % NON_BOSS_POOL.size()]
    print("[Combat] 普通 tile → 隨機 %s" % picked.display_name)
    return picked
```

要點：
- `randi() % size` 是 GDScript 慣用 random index
- 之後可加 weight（STONE_GUARD 機率較低，畢竟比較硬）
- per-encounter 隨機 = 同 run 玩 3 場戰鬥可能撞到 3 種敵人 → 戰鬥多樣性

---

## 6. 觀念對照

| Godot 概念 | 前端 / Vue / Angular |
|---|---|
| `@export var draw_cards: int = 0` | TypeScript interface 加新欄位（向下相容） |
| 5 + 5 + 3 個 `.tres` 純資料 | Strapi CMS 新建 content type 實例 |
| `randi() % size` | `Math.floor(Math.random() * arr.length)` |
| `const NON_BOSS_POOL` | static lookup table / enum-based dictionary |
| 升級版指向 upgrade_version | linked list / hierarchy chain |
| Discard 後 draw 順序 | mutate state in correct sequence to avoid race |

---

## 7. 擴充方式

### 加更多卡（W19 +5）

純 .tres 編輯，例：

```ini
# data/cards/whirlwind.tres
id = "whirlwind"
card_name = "WHIRLWIND"
cost = 2
damage = 5
# whirlwind 應該攻擊 N 次（之後加 attack_count 欄位）
```

新欄位（如 `attack_count`）→ CardData 加 @export → enemy.gd `_drop_data` 處理 → 完工。

### 加敵人池 weighted random

```gdscript
const NON_BOSS_POOL_WEIGHTED = [
    {data: RAT_DATA, weight: 40},
    {data: SLIME_DATA, weight: 30},
    {data: SPIKE_HOUND_DATA, weight: 20},
    {data: STONE_GUARD_DATA, weight: 10},  # 精英較稀
]

func _decide_enemy_data():
    var total_weight = 0
    for entry in NON_BOSS_POOL_WEIGHTED:
        total_weight += entry.weight
    var roll = randi() % total_weight
    var cumulative = 0
    for entry in NON_BOSS_POOL_WEIGHTED:
        cumulative += entry.weight
        if roll < cumulative:
            return entry.data
```

### 加 condition draw（POMMEL+ 升級為「條件抽 2」）

CardData 加 `draw_cards_on_kill: int`，enemy.gd 在 `died.emit` 後檢查 `_last_played_card.draw_cards_on_kill`。

### 加 single-use exhaust 卡

CardData 加 `@export var exhaust: bool = false`，enemy.gd `_drop_data` 內：
```gdscript
if card_data.exhaust:
    GameState.exhaust_card(card_data)   # 不進 discard，永久退出戰鬥
else:
    GameState.discard_card(card_data)
```

### 加更多敵人 + 不同行為

新建 `cultist.tres`（HP 30 / 第一回合不攻擊只 +3 strength / 第二回合起重擊）。
需要 EnemyData 加 `first_turn_skip_attack: bool` + `first_turn_buff: int` 欄位。

---

## 8. 常見錯誤

### 8.1 .tres 升級版 path 寫錯導致 load 失敗

**症狀**：開遊戲 CardDatabase 警告 `[CardDatabase] 跳過 X.tres（無 id 欄位）` 或 `找不到 res://data/cards/y.tres`。

**原因**：`upgraded_version = ExtResource("2_upgrade")` 的 `path` 寫錯（typo / 路徑前綴）。

**修法**：所有升級 path 用絕對 `res://data/cards/X_plus.tres`，避免相對路徑。

### 8.2 draw 在 discard 前

**症狀**：BATTLE_TRANCE 抽 3 張，結果手牌變 3 張不是原來 + 3。

**原因**：discard 前抽 → 抽到的卡進 hand → 然後本卡（BATTLE_TRANCE 自己）discard → hand 計數變 (old + 3 - 1) = old + 2 ✗

**修法**：discard 卡再 draw，順序對：
```gdscript
GameState.discard_card(card_data)
if card_data.draw_cards > 0:
    GameState.draw_card(card_data.draw_cards)
```

### 8.3 隨機池 size = 0 crash

**症狀**：`NON_BOSS_POOL` 不小心改成空 → `randi() % 0` divide by zero。

**修法**：加 guard：
```gdscript
if NON_BOSS_POOL.is_empty():
    push_error("NON_BOSS_POOL 是空，fallback RAT")
    return RAT_DATA
```

### 8.4 STONE_GUARD phase_2_threshold 寫成 4（int 不是 0.4）

**症狀**：phase 2 從不觸發。

**原因**：`@export var phase_2_threshold: float`，.tres 寫 `phase_2_threshold = 4` 是有效 float（4.0）→ HP 永遠 < 40 × 4 不成立。

**修法**：.tres 寫 `phase_2_threshold = 0.4`（小數點明確）。

### 8.5 新卡進入 reward pool 後玩家不認識

**現象**：reward 跳 BATTLE_TRANCE，玩家看「0 cost / draw 3 / SKILL」一臉問號。

**修法**：W18 已在 description 寫「爆發 tempo 必備。」之類 hint。內容期之後加 tooltip / 詳細展示（W20 美術期）。

---

## 9. 下游依賴

| 後續週 | 依賴 W18 的什麼 |
|---|---|
| **W19 內容** | draw_cards 機制可重用（其他抽牌卡） / 敵人 .tres pattern 多開 5-10 隻 |
| **W19 棋盤事件** | EnemyData / CardData 加 callback 機制（複雜效果不夠） |
| **W19 ITEM** | 加 `item_type` enum + 戰鬥外可用 |
| **W20 美術** | 每張卡加 illustration 路徑欄位 / 每隻敵人加 sprite |
| **W23 平衡** | 玩家反饋 → tune cost / damage / draw 數字 |

---

## 10. 面試 talking point

> 「W18 我為 DFS 補了 5 張新卡 + 5 張升級版 + 3 隻敵人 + 加新卡牌欄位 `draw_cards`。
>
> 設計重點：**用 .tres 資料 driven 取代 script callback**。新卡只要丟 .tres 進 `data/cards/`，CardDatabase 自動 scan 收進池子，reward / shop 自動分發。
>
> 整個 W18 內容期，code 動的不多（只 enemy.gd `_drop_data` 加 `if draw_cards > 0` 三行），主要是寫 .tres。
> 對比傳統「每張卡 = 一個 script」的設計，這套架構讓內容生成是 designer-level（不需 coder），擴展性是 N+1 而不是 N×M。
>
> 還有 STONE_GUARD 設計：HP 40、`has_phase_2 = true`、`is_boss = false`。
> 這證明 W15 的 EnemyData composition 設計成立 — 'phase 2' 是 capability，跟 'boss' 是 categorization 解耦。
> 玩家原本以為遇到一隻普通敵，結果打到一半被切階段嚇到 — surprise mechanic 自然從 data 衍生出來。
>
> draw_cards 在 discard 後執行是另一個小細節 — 順序錯會 break BATTLE_TRANCE 的計數。」

---

**M5 (W18-W22) 進度：20% (1/5)**
**內容池總覽**：16 base + 9 upgrade = 25 張卡 / 5 隻敵人（4 普通 + 1 boss）
**下一週 W19**：補滿 5 張卡 + 5 隻敵人 + 棋盤事件 3-5 種 + QTE 卡致敬 WorkNite
