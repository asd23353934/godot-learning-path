# W8 / DFS Phase 2 下半：完整回合制戰鬥（抽棄牌 + Shield + 敵人攻擊 + 勝負）

> 對應 PROGRESS.md W8 / DFS docs/08-implementation/steps/phase-2-combat-prototype.md（Steps 2.4-2.7）
> 完成日：2026-05-28
> Repo：[dice-fate-survivor](https://github.com/asd23353934/games/dice-fate-survivor)

---

## 1. 目標 / 不目標

### 目標

- **抽 / 棄 / 洗牌堆**完整 cycle（GameState 為 source of truth）
- **Shield 系統**：DEFEND 卡給玩家護盾、take_damage 先吸盾
- **End Turn 按鈕** + 敵人攻擊回合 + 玩家能量重置 + 重新抽 5 張
- **戰鬥勝負**：殺光敵人 → VICTORY → Board；玩家 HP=0 → DEFEATED → 主選單
- Hand 從 W7 的 `@export cards` 重構為 **reactive**（訂閱 GameState.hand）
- Enemy HP 改回 30、加 attack_damage = 4
- DEFEND 真正給 5 護盾（W7 是 placeholder）

### 不目標（之後階段）

- **5 波結構**（Step 2.8，留 W9）：W8 仍是單場 1 隻敵人
- **Status effect** 系統（W11+）：中毒 / 力量 / 虛弱
- **多 target 卡**：W11 拆 card_type / target_type enum
- **自定 deck / kit**：W9-W10 加
- **click 自觸發**：W11 重構（目前 DEFEND 還是 drag 到敵人）
- 動畫、音效、特效（W22 polish）

---

## 2. 設計決策

### 為什麼 Hand 改 reactive（不再 @export）？

**W7 設計**：

```gdscript
# hand.gd
@export var cards: Array[CardData] = []   # Inspector 拖 5 張

func _ready():
    setup_hand(cards)
```

**問題**：手牌變動（抽 / 棄 / 打）UI 不會自動更新。每次都要 setup_hand() 重 build。

**W8 改 reactive**：

```gdscript
# hand.gd
func _ready():
    EventBus.hand_changed.connect(_on_hand_changed)
    _rebuild(GameState.hand)

func _on_hand_changed(new_hand: Array):
    _rebuild(new_hand)
```

GameState.hand 是 **single source of truth**。任何地方改 hand → emit `hand_changed` → Hand UI 自動 rebuild。

**對應前端**：Vue 3 reactivity / React useState — UI 自動跟 state 同步。

### 為什麼 take_damage 先扣 shield？

```gdscript
func take_damage(amount):
    var damage_to_hp = amount
    if shield > 0:
        var absorbed = min(shield, amount)
        shield -= absorbed
        damage_to_hp -= absorbed
    if damage_to_hp > 0:
        player_hp = max(0, player_hp - damage_to_hp)
```

**理由**：shield 是「臨時 HP」，先消耗才合理。Slay the Spire / Hearthstone 同模式。

**Edge case 處理**：
- 傷害 ≤ shield → shield 扣，HP 不動，沒 emit player_hp_changed
- 傷害 > shield → shield 歸 0，多的扣 HP
- shield = 0 → 直接扣 HP

### 為什麼 End Turn 棄整手牌？

```gdscript
func _on_end_turn_pressed():
    GameState.discard_hand()         # 棄手牌
    %Enemy.enemy_turn_attack()       # 敵人打
    GameState.energy = max_energy    # 重置能量
    GameState.draw_card(5)           # 抽新的
```

**Slay the Spire 規則**：每回合棄掉所有手牌，下回合抽 5 新的。**保留手牌會讓 deck cycle 變慢、玩家累積牌過多打不出**。

也讓玩家**思考「這回合留 X 張下回合」是錯的**，迫使用完才換新批。

### enemy HP 為什麼從 W7 demo 15 改回 30？

W7：因為沒抽牌、單回合，要 enemy HP 低才能殺。
W8：有 end_turn 抽牌循環，**多回合戰鬥**，enemy HP 30+ 才合理。

W8 enemy 設定：
- HP = 30
- attack_damage = 4
- 玩家 deck 10 張，每回合抽 5
- 預計 1-2 回合可殺（玩家 HP 80，可撐 20 回合，**戰鬥空間夠**）

### 為什麼 DEFEND drop 到敵人也 work（非自觸發）？

UX 怪：「我打你你給我盾」😅。**但 W8 不重要**：

- W8 只 1 隻敵人，玩家不會誤 drop 別處
- 加 click handler + 自觸發需要重構 card.gd / enemy.gd（W11 一次做）
- W8 trade-off：**功能 work > UX 完美**，prototype 階段省時間

**W11 會正式 refactor**：card_type enum 分流（ATTACK 拖、SKILL 點）。

### 為什麼起始 deck hardcode 10 張在 combat.tscn 內？

```
3× STRIKE / 2× HEAVY / 2× DEFEND / 2× QUICK / 1× FOCUS
```

理由：
- W8 不做 deck builder UI，玩家不選 deck
- Inspector 拖一次 = 不寫 code
- W14-W15（Phase 5 整合）加 kit 系統時改成從 GameState.kit 動態 build
- 可調平衡（之後玩太簡單 → 改卡組組成）

---

## 3. 檔案清單

```
變動：
├── scripts/data/card_data.gd            ← 加 shield 欄位
├── scripts/autoload/game_state.gd       ← 加 deck/discard cycle + shield API（大改）
├── scripts/autoload/event_bus.gd        ← 加 3 個 signal（shield/hand/deck_state）
├── scripts/combat/hand.gd               ← 重構為 reactive（訂閱 hand_changed）
├── scripts/combat/enemy.gd              ← 加 attack_damage + enemy_turn_attack
├── scenes/combat/enemy.tscn             ← max_hp 30、attack_damage 4
├── scenes/combat/combat.tscn            ← 加 EndTurnButton / ShieldLabel / 3 個 Labels
├── scenes/combat/combat.gd              ← turn loop orchestration
└── data/cards/defend.tres               ← shield = 5
```

---

## 4. 實作流程 / 順序

```
1. 設計：W7 戰鬥單回合，W8 補成多回合 turn loop
   · 需要：deck/hand/discard cycle、shield、end turn、敵人攻擊、勝負
2. CardData 加 shield 欄位（DEFEND 才能真的給 shield）
3. defend.tres 設 shield=5
4. GameState 大改：
   · 加 shield 變數 + gain_shield method
   · take_damage 改先扣 shield
   · 加 start_combat_with_deck / draw_card / discard_card / discard_hand
5. EventBus 加 3 個 signal：shield_changed / hand_changed / deck_state_changed
6. Hand 改 reactive：訂閱 hand_changed + 從 GameState.hand 讀
7. Enemy 加 attack_damage + enemy_turn_attack；drop 處理 shield 卡
8. enemy.tscn：HP 30、attack_damage 4
9. combat.tscn：加 EndTurnButton / ShieldLabel / PlayerHPLabel / DeckCount / DiscardCount
   · starter_deck @export 10 張
10. combat.gd：訂閱所有相關 signal + turn loop orchestration + 勝負處理
11. F5 跑：拖卡 / DEFEND 加盾 / End Turn / 敵人攻擊吸盾扣血 / 抽新牌 / 勝負
12. 寫 W8 implementation guide
13. 雙 commit + push（中文 message）
```

---

## 5. 關鍵 code 解析

### `GameState.take_damage` 加 shield 邏輯

```gdscript
func take_damage(amount: int) -> void:
    if player_hp <= 0:
        return

    var damage_to_hp = amount
    if shield > 0:
        var absorbed = min(shield, amount)
        shield -= absorbed
        damage_to_hp -= absorbed
        EventBus.shield_changed.emit(shield)

    if damage_to_hp > 0:
        player_hp = max(0, player_hp - damage_to_hp)
        EventBus.damage_dealt.emit(damage_to_hp, "Player")
        EventBus.player_hp_changed.emit(player_hp, max_hp)
        if player_hp == 0:
            EventBus.player_died.emit()
```

**重點**：
- shield 全吸 → 不 emit damage_dealt 給 HP（玩家「沒受傷」）
- shield 部分吸 → emit 兩個 signal（shield + HP）
- HP 0 → emit player_died（戰鬥失敗觸發）

### `GameState.draw_card` 含洗牌邏輯

```gdscript
func draw_card(n: int = 1) -> void:
    for i in range(n):
        if deck.is_empty():
            if discard.is_empty():
                break                    # 完全沒牌可抽
            # 洗 discard 回 deck
            deck = discard.duplicate()
            deck.shuffle()
            discard.clear()
        var drawn = deck.pop_back()
        hand.append(drawn)

    EventBus.hand_changed.emit(hand)
    EventBus.deck_state_changed.emit(deck.size(), discard.size())
```

**重點**：
- 抽牌堆空 + 棄牌堆也空 → break（不再 emit，防止無限 loop）
- 抽牌堆空但棄牌堆有 → 自動洗回去（自然 cycle）
- `pop_back()` 用 stack 性質，效率 O(1)（前端 `Array.shift()` 是 O(n)）
- 每抽完 N 張才 emit 一次，避免單張一 emit 拖慢 UI

### `Hand.gd` reactive pattern

```gdscript
extends HBoxContainer

@export var card_scene: PackedScene

func _ready() -> void:
    EventBus.hand_changed.connect(_on_hand_changed)
    _rebuild(GameState.hand)

func _on_hand_changed(new_hand: Array) -> void:
    _rebuild(new_hand)

func _rebuild(card_list: Array) -> void:
    # 清舊
    for child in get_children():
        child.queue_free()
    # 建新
    for card_data in card_list:
        var card_instance = card_scene.instantiate()
        card_instance.data = card_data
        add_child(card_instance)
```

**重點**：
- 不持有 cards 變數，**完全 reactive**
- 每次 hand_changed 整把砍重建
- 效率 OK（5 張卡，rebuild 不到 1ms）
- 對應 React：function component re-render；對應 Vue：響應式 + v-for

**為什麼整把 rebuild 不漸進 diff**：
- 5 張小 → 整把砍乾淨
- 漸進 diff 邏輯複雜（哪張新加 / 哪張該移除 / 哪張只變動）
- prototype 階段：簡單 > 效能
- 之後規模上去再優化

### `combat.gd` turn loop

```gdscript
func _on_end_turn_pressed() -> void:
    # 1. 棄手牌
    GameState.discard_hand()

    # 2. 敵人攻擊
    if %Enemy.hp > 0:
        %Enemy.enemy_turn_attack()

    # 3. 玩家還活著 → 抽新牌 + 重置能量
    if GameState.player_hp > 0:
        GameState.energy = GameState.max_energy
        EventBus.energy_changed.emit(GameState.energy, GameState.max_energy)
        GameState.draw_card(5)
```

**順序很重要**：
1. **先棄手牌**（不是先敵人攻擊）— 玩家手牌都 commit 完
2. **敵人攻擊** — 玩家承受傷害（吸盾扣血）
3. **檢查玩家是否還活著** — 沒死才繼續抽
4. **重置能量 + 抽 5 張** — 下回合準備

**為什麼最後一步檢查 player_hp**：玩家死了不該還抽牌，避免新回合開始的疊代後續邏輯。

### `Enemy.enemy_turn_attack`

```gdscript
func enemy_turn_attack() -> void:
    if hp <= 0:
        return                                       # 死了不該再攻擊
    GameState.take_damage(attack_damage)
```

**簡單**：呼叫 GameState.take_damage，shield 邏輯在 GameState 內部處理。Enemy 不知道玩家有沒有 shield → 鬆耦合。

---

## 6. 觀念對照（前端視角）

| Godot 概念 | Vue / Angular / React |
|---|---|
| Hand 訂閱 `hand_changed` + rebuild | React `useEffect(() => render, [state])` |
| GameState.hand 為 source of truth | Pinia / Vuex / Redux store |
| take_damage 內部 emit signal | mutation 內部 + commit dispatch |
| EventBus signal 分類（player / combat / board） | Event bus / global state slices |
| shield 邏輯封裝在 GameState | 業務邏輯放 store action，不放 component |
| Enemy.enemy_turn_attack 呼叫 GameState | child 不直接動 sibling，透過 store/dispatch |
| `Array.pop_back()` (Godot) | `array.pop()` (JS) — stack 模式 |
| Array.duplicate() + shuffle | `[...arr].sort(() => Math.random() - 0.5)` |

---

## 7. 擴充方式

### 加新卡：抽 2 張牌的 SKILL 卡（W11 加）

```gdscript
# CardData 加 enum + on_play 邏輯參數
@export var draw_amount: int = 0   # 打出時抽 N 張
```

```
# data/cards/calculate.tres
[gd_resource type="Resource" script_class="CardData" format=3]
[ext_resource type="Script" path="..." id="1"]
[resource]
card_name = "CALCULATE"
cost = 1
damage = 0
draw_amount = 2
description = "抽 2 張牌。"
```

在 enemy._drop_data（或 W11 改成 card._play_skill）：
```gdscript
if card_data.draw_amount > 0:
    GameState.draw_card(card_data.draw_amount)
```

### 加 enemy 攻擊變化（多回合 pattern）

目前 enemy 每回合死定攻擊 4。實際遊戲：

```gdscript
# enemy.gd
var attack_pattern: Array[int] = [4, 4, 6, 0]   # 第 4 回合 buff，第 5 回合大攻
var current_turn: int = 0

func enemy_turn_attack():
    var dmg = attack_pattern[current_turn % attack_pattern.size()]
    current_turn += 1
    if dmg > 0:
        GameState.take_damage(dmg)
    else:
        # buff turn — 之後加 status effect
        gain_strength(3)
```

UI 加「下回合敵人意圖」icon（看玩家可預測） — Slay the Spire 經典設計。

### 加 5 波結構（Step 2.8，W9 補）

```gdscript
# combat.gd
@export var waves: Array[Resource] = []   # 每波 EnemyEncounter
var current_wave: int = 0

func _on_enemy_died():
    current_wave += 1
    if current_wave >= waves.size():
        _victory()
    else:
        _spawn_wave(current_wave)
```

每波 spawn 不同敵人組合。HP / deck / shield 跨波延續（不重置）。

### 加 click 自觸發（W11 重構）

詳見上一次回答的 enum + click handler 設計。core idea：

```gdscript
# card.gd 加
func _gui_input(event):
    if event.is_pressed() and data.card_type == CardData.CardType.SKILL:
        _trigger_self()

func _trigger_self():
    if GameState.energy < data.cost: return
    GameState.spend_energy(data.cost)
    # apply effects
    GameState.discard_card(data)
```

### 加自動洗牌動畫

目前 `discard.shuffle()` 是 instant。之後加：

```gdscript
func shuffle_discard_into_deck():
    EventBus.shuffle_started.emit()    # UI 播洗牌動畫
    await get_tree().create_timer(0.5).timeout
    deck = discard.duplicate()
    deck.shuffle()
    discard.clear()
    EventBus.shuffle_finished.emit()
```

---

## 8. 常見錯誤 / debug 指南

### `hand_changed` emit 後 UI 沒更新

**症狀**：打出卡 → console 印「棄牌」但 UI 還有卡。
**根因**：
1. Hand 沒 connect EventBus.hand_changed
2. Hand 的 `_rebuild` 內 `card_scene` 是 null（@export 沒拖）
**修法**：F5 看 Hand._ready 有沒有 print；Inspector 確認 card_scene 拖了 card.tscn

### Shield 沒吸傷害 / 全吸後玩家還是 HP 扣

**症狀**：明明有 5 shield，被打 4 dmg 還是扣 HP。
**根因**：take_damage 流程錯。
**修法**：print 看 shield 變數、damage_to_hp 計算

### 抽牌堆永遠不洗

**症狀**：deck 空了 + discard 有 5 張，但 draw_card 沒洗。
**根因**：draw_card 內 if deck.is_empty() 判斷漏 discard 路徑。
**檢查**：跑後 Output `[GameState] 抽牌堆空，洗 X 張棄牌堆回去` 有沒有出現。

### End Turn 後手牌沒抽到 5 張

**症狀**：End Turn → 只抽 3 張。
**根因**：deck + discard 加起來 < 5。
**正常情況**：W8 deck=10，第一回合棄 5 + 抽 5 後，deck=0 / discard=5 + hand=5。第二回合棄 5 後 deck=0 / discard=10 + hand=0 → 抽 5 會洗 → 抽 5 OK。
**Edge case**：玩家把 deck + discard 全用光（不太可能 W8 deck size 10）。

### 敵人死亡後 EndTurnButton 還能點

**症狀**：VICTORY 出來但 End Turn 按鈕仍 enabled。
**根因**：disable 該邏輯漏寫。
**修法**：W8 已加 `%EndTurnButton.disabled = true` 在勝負 handler 內。

### 玩家死亡瞬間敵人還在攻擊

**症狀**：玩家 HP 1 → End Turn → 敵人攻擊 4 → 玩家應該死 → 但 console 顯示「新回合」訊息。
**根因**：combat.gd End Turn 順序錯。
**檢查**：先 enemy 攻擊、再判斷 player_hp > 0 才抽牌。W8 已正確處理。

---

## 9. 下游依賴

| 之後階段 | 怎麼用 W8 成果 |
|---|---|
| W9 Phase 3 (1/3) | 加 5 波結構（Step 2.8）；hardcode deck 改成從 GameState.kit 讀 |
| W10 Phase 3 (2/3) | 加 6 種 status（poison/weak/vulnerable/strength/dex/shield-as-status） |
| W11 Phase 3 (3/3) | Combo Lock 機制 + card_type enum + click 自觸發；GdUnit4 戰鬥公式測試 |
| W12-W13 Phase 4 棋盤 | board.gd 觸發戰鬥，玩家 deck / HP / shield 跨棋盤戰鬥延續 |
| W14-W15 Phase 5 整合 | 完整 run loop：kit 選擇 → 棋盤 → 戰鬥（用本週這套）→ 結算 |
| W18-W19 Phase 7 內容 | 10-15 張卡 + 4-5 種敵人；每種敵人有獨特 attack_pattern |
| W22 polish | 抽牌 / 棄牌 / 攻擊飄字 / hit flash / shake；hook 既有 signal |

---

## 10. 面試 talking point

> 「W8 完成完整回合制戰鬥：抽 / 棄 / 洗牌堆、shield 系統、End Turn 按鈕、敵人攻擊回合、勝負流程。
>
> 重點重構：W7 的 Hand 用 `@export var cards` 是 static，W8 改 reactive — 訂閱 EventBus.hand_changed，從 GameState.hand 讀。**single source of truth + 自動 UI 同步**，跟 Vue 響應式 / React state 一樣概念。
>
> 戰鬥 turn loop 設計遵循 Slay the Spire 規則：每回合棄整手抽 5 新，玩家不能囤牌。能量每回合重置（max=3），玩家 HP 80 / 敵人 HP 30 / 敵人攻擊 4 — 1-2 回合可結束。
>
> Shield 邏輯放在 `GameState.take_damage()` 內封裝：先扣 shield、再扣 HP。Enemy 呼叫 `GameState.take_damage()` 完全不知道有沒有 shield — **鬆耦合**。
>
> 我目前 DEFEND 還是 drag-to-enemy（給玩家加 shield），UX 不完美但 W8 範圍能 work。**W11 會 refactor 加 card_type enum** 分 ATTACK / SKILL / POWER 三類，SKILL 改 click 觸發。**prototype 階段的 trade-off：功能先做出來，UX 完美留下個階段**。」
