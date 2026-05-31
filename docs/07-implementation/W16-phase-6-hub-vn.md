# W16 / Phase 6 (1/2)：Hub 場景 + 2 NPCs + 自寫 VN dialog 系統

> 對應 repo：[dice-fate-survivor](https://github.com/asd23353934/com.dice-fate-survivor)
> 對應 PROGRESS milestone：M4 W16
> 寫於 Day 8（2026-05-30 ~ 05-31）

---

## 1. 目標 / 不目標

### 目標
- **Hub 場景成型**：取代 W6 placeholder，變成 meta progression 中央樞紐
- **2 個可互動 NPC**：訓練師 Akira（永久 HP / 能量上限）+ 卡牌商 Mira（永久 starter deck 加卡）
- **可重用 DialogPanel**：speaker / text / choices vbox，未來棋盤事件、戰鬥對話都能用
- **Gold 持久化**：跨 run 累積（不再每 run reset 為 0），Hub 才是花錢的地方
- **路由更新**：stage clear → Hub（不直接回主選單），玩家可以花賺到的錢

### 不目標（留後續）
- Dialogic plugin（W18 內容期再評估，現階段自寫足夠）
- NPC 立繪 / Live2D（W20 美術期）
- 對話分支樹（純線性 dialog，W18 加結構化 dialog data）
- Save / load（W17）
- 多 kit 解鎖（unlocked_kits 預留，之後做）

---

## 2. 設計決策

### A. 為什麼不裝 Dialogic plugin

| 方案 | 優點 | 缺點 |
|---|---|---|
| Dialogic 2.x | 生產級工具、分支樹、translation 友善 | 學習曲線陡、plugin 兼容風險（GdUnit4 災難回憶錄）、W16 scope 2 NPCs 用不到一半功能 |
| **自寫 DialogPanel**（採用） | 學 Godot 基礎更深、零外部依賴、量身打造 | 之後加複雜分支要自己擴 |

→ **自寫勝**：W16 只需 2 NPCs × 3 選項，殺雞用牛刀沒意義。**W18-19 內容期** 若 dialog tree 真的爆炸再評估 plugin。

### B. 為什麼 gold 持久化？

W14-W15 設定：`reset_for_new_run` 內 `gold = 0`。問題：

- Hub 是花錢的地方
- 但 reset 後 gold 歸零 → 從 main_menu 進 Hub 永遠 0 金錢
- 玩家「在棋盤辛苦賺的錢進 Hub 全消失」UX 災難

修法：**gold 跨 run 累積**（從 GameState 移出 reset 範圍）。

額外加 `gold_earned_this_run`：
- add_gold 同時 ++ 兩個
- reset 只歸零 `gold_earned_this_run`
- Stage clear panel 顯示「本場獲得 / 總金錢」兩個欄位 → UX 清楚

### C. 為什麼 Hub 才是 run 起點，不是 main menu

W15 之前：`main_menu._on_start_pressed` 內 `reset_for_new_run()` + 切 Hub。
問題：玩家進 Hub 後想升級 → 改變了 max_hp / max_energy，但這些變動只在「下次 reset」生效。如果 main_menu 已經 reset 過了，現在的 reset 就要二次跑，run_count 亂跳。

修法：
- main_menu start 只設 `current_kit`，不 reset
- Hub「開始冒險」按鈕才 `reset_for_new_run`
- 這樣玩家在 Hub 升級後立刻按出發，新值才會生效

> Vue 類比：main_menu 是 router push（純導航），Hub 才有 store mutation（state 變更）。

### D. DialogPanel API 設計：caller 自管 close vs auto-close

兩種選擇：
1. 點 choice → 執行 callback → **自動關 dialog**
2. 點 choice → 執行 callback → **保持開啟**（caller 自己 close 或 chain）

**選 2**（採用）：caller 自管 close。
- 優點：可以 chain dialog（「買成功 → 顯示確認訊息 → 玩家按繼續才關」）
- 程式碼一致：`callback` 就是「點下去要做什麼」，不混 close 邏輯
- 沒選的成本：每個分支要明寫 `_close_dialog` callback（小代價）

```gdscript
# 範例：買 HP
func _trainer_buy_hp():
    if not GameState.spend_gold(50):
        %DialogPanel.show_dialog("Akira", "錢不夠喔", [])   # 空 choices → 顯示「繼續」
        return
    GameState.purchase_max_hp_upgnt(5)
    %DialogPanel.show_dialog("Akira", "max HP +5 完成", [])  # chain
```

### E. NPC 為什麼用 Button 而不 Sprite + Area2D + click detection

- Button 內建 hover / focus / pressed 狀態（無障礙）
- Button 可裝子 Control（ColorRect + 名字 label）做成 NPC「卡片」
- 之後改 Sprite2D + AnimatedSprite2D 也只要換 Button 內視覺（行為層不動）

**Composition**：Button 是「可點容器」，視覺是 children。

### F. 卡商商品（offerings）每進 Hub roll 一次

- 進 Hub 時 `_shop_offerings = CardDatabase.get_reward_options(3)`
- 出 Hub 重進 → 重 roll（鼓勵玩家「攢點錢再來看」）
- 同次 Hub 買掉的從 offerings.erase → 不能買重複
- 不持久化 offerings（簡化；之後 W18 可加「商店 refresh 機制」）

---

## 3. 檔案清單

### 新增

| 檔案 | 角色 |
|---|---|
| `scripts/ui/dialog_panel.gd` | DialogPanel class — show_dialog / close_dialog API |
| `scenes/ui/dialog_panel.tscn` | DialogPanel 視覺（dim + inner panel + speaker / text / choices vbox） |

### 修改（重構）

| 檔案 | 變動 |
|---|---|
| `scenes/hub/hub.tscn` | placeholder 改成完整 Hub（stats panel / 2 NPCs / 開始冒險 / 返回主選單 / DialogPanel instance） |
| `scenes/hub/hub.gd` | placeholder 改成 hub logic（NPC click → dialog → action）|
| `scripts/autoload/game_state.gd` | gold 不再 reset / 加 gold_earned_this_run / 加 permanent_starter_additions / 加 purchase_card_for_starter / reset 內 append additions 到 deck |
| `scenes/main_menu.gd` | start 不再 reset_for_new_run（移到 Hub） |
| `scenes/board/board.gd` | StageClearMenuButton → Hub（不是主選單）/ stage clear panel 顯示「本場獲得 / 總金錢」 |
| `scenes/board/board.tscn` | StageClearMenuButton 文字改「🏠 前往旅店」 |
| `scenes/combat/combat.gd` | RunEndPanel 顯示「本場獲得 / 總金錢」 |

---

## 4. 實作流程

1. **GameState gold 持久化 + permanent_starter_additions**（資料層先動）
2. **DialogPanel scene + script**（reusable component）
3. **Hub scene 重做 UI**（stats panel + 2 NPCs + 按鈕）
4. **Hub.gd dialog logic**（訓練師 / 卡商）
5. **路由切換**（main_menu 不 reset、stage clear → Hub）
6. **Stats panel 顯示「本場 vs 總」金錢**

順序原則：**資料 → 元件 → 場景 → 邏輯 → 路由**。

---

## 5. 關鍵 code 解析

### 5.1 Gold 持久化 + 累積追蹤

```gdscript
# game_state.gd
var gold: int = 0                  # 持久（Hub 消費才扣）
var gold_earned_this_run: int = 0  # 本 run 賺多少

func add_gold(amount: int) -> void:
    gold += amount
    gold_earned_this_run += amount   # W16：本 run 累計
    total_gold_collected += amount
    EventBus.gold_changed.emit(gold)

func reset_for_new_run(kit):
    ...
    # 不再 gold = 0
    gold_earned_this_run = 0   # 只重設「本 run 計數器」
    ...
```

### 5.2 Permanent starter additions（卡商買的卡跨 run 帶著）

```gdscript
var permanent_starter_additions: Array[String] = []   # card ids

func purchase_card_for_starter(card_id: String) -> void:
    permanent_starter_additions.append(card_id)
    EventBus.upgrade_purchased.emit("starter_card_" + card_id)

func reset_for_new_run(kit):
    ...
    current_run_deck = CardDatabase.build_starter_deck(kit)
    for card_id in permanent_starter_additions:
        var added = CardDatabase.get_card(card_id)
        if added != null:
            current_run_deck.append(added)
    ...
```

- starter deck = kit 預設（10 張）+ 玩家累積買的卡
- 卡商買 3 張 → 下次 run deck 變 13 張

### 5.3 DialogPanel show / close API

```gdscript
class_name DialogPanel
extends Control

signal dialog_closed

func show_dialog(speaker_name: String, text: String, choices: Array = []) -> void:
    %SpeakerLabel.text = speaker_name
    %DialogText.text = text
    _clear_choices()
    if choices.is_empty():
        _add_choice_button("繼續", close_dialog)
    else:
        for choice in choices:
            var label: String = choice.get("label", "?")
            var callback: Callable = choice.get("callback", Callable())
            _add_choice_button(label, callback)
    show()

func close_dialog() -> void:
    hide()
    dialog_closed.emit()

func _add_choice_button(label: String, callback: Callable) -> void:
    var btn = Button.new()
    btn.text = label
    btn.custom_minimum_size = Vector2(0, 40)
    if callback.is_valid():
        btn.pressed.connect(callback)
    %ChoicesVBox.add_child(btn)
```

- 動態 spawn buttons（每次 show 清空 + 重 add）
- `Callable.bind(card)` partial application 模式：`_shop_buy_card.bind(card)`
- 空 choices → 自動 fallback「繼續」按鈕 → close_dialog

### 5.4 Hub Trainer dialog（branched）

```gdscript
func _on_trainer_pressed() -> void:
    %DialogPanel.show_dialog(
        "訓練師 Akira",
        "「想變強嗎？我可以幫你提升上限。」",
        [
            {"label": "+5 max HP（50 金）", "callback": _trainer_buy_hp},
            {"label": "+1 max 能量（100 金）", "callback": _trainer_buy_energy},
            {"label": "離開", "callback": _close_dialog},
        ]
    )

func _trainer_buy_hp() -> void:
    if not GameState.spend_gold(TRAINER_HP_COST):
        %DialogPanel.show_dialog("Akira", "「錢不夠喔」", [])   # 失敗 → 「繼續」
        return
    GameState.purchase_max_hp_upgrade(TRAINER_HP_BONUS)
    %DialogPanel.show_dialog("Akira", "「+5 HP 完成」", [])   # 成功 chain dialog
```

Pattern：**action → conditional dialog chain**。
- 失敗 → 顯示原因 dialog
- 成功 → 顯示結果 dialog
- 都是空 choices → 玩家按「繼續」關閉

### 5.5 Shop offerings 動態列表 + bind 卡資料

```gdscript
var _shop_offerings: Array[CardData] = []

func _ready():
    _shop_offerings.assign(CardDatabase.get_reward_options(3))

func _on_shop_pressed():
    var choices: Array = []
    for card in _shop_offerings:
        choices.append({
            "label": "%s（30 金）" % card.card_name,
            "callback": _shop_buy_card.bind(card),   # ← partial application
        })
    choices.append({"label": "離開", "callback": _close_dialog})
    %DialogPanel.show_dialog("Mira", "「今天進了幾張卡」", choices)

func _shop_buy_card(card: CardData):
    if not GameState.spend_gold(30):
        ...
        return
    GameState.purchase_card_for_starter(card.id)
    _shop_offerings.erase(card)   # 買掉的賣完從清單移除
    %DialogPanel.show_dialog("Mira", "「謝啦！%s 加進 starter」" % card.card_name, [])
```

要點：
- `.bind(card)` 預綁卡資料 → callback 不用透過全局狀態傳資料
- offerings.erase 後同次 Hub 不能再買（但出 Hub 重進會 re-roll）

### 5.6 路由：Hub 變 run 起點

```gdscript
# main_menu.gd（W16 變動）
func _on_start_pressed() -> void:
    GameState.current_kit = "KIT_SWORD"   # 只設 kit
    SceneRouter.change_scene("res://scenes/hub/hub.tscn")   # 不 reset

# hub.gd（W16 新）
func _on_to_board() -> void:
    GameState.reset_for_new_run(GameState.current_kit)   # Hub 才是 reset 點
    SceneRouter.change_scene("res://scenes/board/board.tscn")
```

效果：
- 玩家在 Hub 買 +5 HP → 按「開始冒險」→ reset 把 max_hp 設成 80 + 累積 bonus（生效）
- 不會二次 reset、不會跳過升級

### 5.7 Stage clear → Hub（不是主選單）

```gdscript
# board.gd（W16 變動）
%StageClearMenuButton.pressed.connect(_on_to_hub_pressed)

func _on_to_hub_pressed():
    SceneRouter.change_scene("res://scenes/hub/hub.tscn")

func _on_to_menu_pressed():
    # 棋盤左上「返回主選單」 — 中途放棄
    SceneRouter.change_scene("res://scenes/main_menu.tscn")
```

UX 效果：
- 勝利後到 Hub 看到「金錢 +N」可以花
- 主選單只在「放棄 run」或「玩家死亡」時回去（重啟流程）

---

## 6. 觀念對照

| Godot 概念 | 前端 / Vue / Angular |
|---|---|
| `DialogPanel` 可重用 component | `<DialogModal>` Vue component |
| `show_dialog(speaker, text, choices)` API | `props: {speaker, text, choices}` + `v-if="isOpen"` |
| `Callable.bind(card)` partial application | `() => buyCard(card)` arrow function |
| `permanent_starter_additions: Array[String]` | Pinia store array state |
| Hub 是 run 起點，不是 main menu | router 設計：把 mutation 放對 view |
| z_index = 50 overlay pattern | `position: fixed; z-index: 50` modal |
| `mouse_filter = MOUSE_FILTER_IGNORE` 子元素不擋 click | `pointer-events: none` CSS |

---

## 7. 擴充方式

### 加第 3 個 NPC（e.g., 鐵匠 — 升級卡牌）

```gdscript
# hub.tscn 加 SmithNPC Button（複製 TrainerNPC 結構，換顏色 + 名字）
# hub.gd:
func _on_smith_pressed():
    var upgradables = _get_upgradable_cards_in_deck()
    var choices: Array = []
    for card in upgradables:
        choices.append({
            "label": "升級 %s → %s（80 金）" % [card.card_name, card.upgraded_version.card_name],
            "callback": _smith_upgrade.bind(card),
        })
    choices.append({"label": "離開", "callback": _close_dialog})
    %DialogPanel.show_dialog("鐵匠", "...", choices)
```

### 加 Dialogic plugin（之後 W18-19）

當 dialog tree 真的爆炸（10+ NPC、分支樹、 portrait、 timeline）→ 改 Dialogic：
1. Asset Library 裝 Dialogic 2.x
2. 把每個 NPC 對話寫成 timeline `.dtl`
3. Hub.gd 改成 `Dialogic.start("trainer")` 取代 `show_dialog`
4. 保留 DialogPanel 給「商店式互動選單」（list of items）— Dialogic 不擅長那種

### 加 NPC 立繪（W20 美術期）

`TrainerNPC/VBox/ColorBlock` 換成 `Sprite2D` + `texture = preload("res://assets/portraits/akira.png")`。
行為層（Button.pressed）零變動。

### 加「對話歷史」功能

DialogPanel 加 `var _history: Array[Dictionary]`，每次 show 推進去；加按鈕開歷史視窗。

### 加 Hub 多場景 / 多區域

把 Hub 拆成「主廳 / 訓練場 / 商店街」三個子場景，主廳有 3 個門 Button 通往子場景。
每個子場景一個 NPC。

---

## 8. 常見錯誤（踩雷）

### 8.1 點 NPC button 時 child element 擋 click

**症狀**：點 NPC 卡片中間（ColorBlock 或 Label 範圍）沒反應，只有邊緣 padding 才觸發。

**原因**：子 Control（VBox / ColorRect / Label）預設 `mouse_filter = MOUSE_FILTER_STOP`，把 click 吃掉。

**修法**：所有子 Control 設 `mouse_filter = MOUSE_FILTER_IGNORE`（int = 2 in tscn）：
```
[node name="VBox" type="VBoxContainer" parent="TrainerNPC"]
mouse_filter = 2
```

CSS 類比：`pointer-events: none` 在子元素，讓 click 穿透到父 Button。

### 8.2 Gold reset 後 Hub 顯示 0

**原因**：W15 之前 `reset_for_new_run` 內 `gold = 0`。
W16 移掉。確認 game_state.gd `reset_for_new_run` 內**只有** `gold_earned_this_run = 0`，沒 `gold = 0`。

### 8.3 run_count 跳兩格

**原因**：main_menu reset + Hub 開始冒險 reset = 兩次 ++。

**修法**：main_menu 不 reset，Hub 才 reset。

### 8.4 Dialog show 兩次 button 加倍

**原因**：DialogPanel 沒 `_clear_choices()` 就 `_add_choice_button`，舊 button 還在。

**修法**：show_dialog 一開始就 clear：
```gdscript
func show_dialog(...):
    ...
    _clear_choices()
    if choices.is_empty(): ...
    else: ...
```

### 8.5 Hub 點「開始冒險」回到上次 run 一半的 board

**原因**：reset_for_new_run 沒呼叫，current_tile_index / board_enemies 還是上次的。

**修法**：`_on_to_board` 一定要先 reset 再切 scene：
```gdscript
func _on_to_board():
    GameState.reset_for_new_run(GameState.current_kit)
    SceneRouter.change_scene("res://scenes/board/board.tscn")
```

---

## 9. 下游依賴

| 後續週 | 依賴 W16 的什麼 |
|---|---|
| **W17 Save** | gold / permanent_max_hp_bonus / permanent_starter_additions 都要序列化進 save |
| **W18 內容** | DialogPanel 拿去寫棋盤事件（EVENT tile → 對話樹）/ 商店 refresh 機制 |
| **W19 棋盤事件** | DialogPanel + 結構化 EventData resource（label / branches / actions） |
| **W20 美術** | NPC ColorBlock 換 Sprite2D / Live2D（行為層不動） |
| **W22 整合** | Hub + Dialogic plugin 評估點（若 dialog 樹爆炸）|

---

## 10. 面試 talking point

> 「W16 我把 DFS 從『run-only』升級到『run + meta hub』。設計重點有三：
>
> **1. 路由設計**：原本 main_menu 直接 reset state 切 board，我發現玩家在 Hub 升級的 max HP 不會生效（因為 main_menu 已經 reset 過了，現在的升級要等下次 reset 才生效，玩家視角就是『升完馬上沒效果』）。改成 main_menu 只導航、Hub 才 reset，玩家升完按出發立刻生效。
>
> **2. 自寫 DialogPanel 而不裝 Dialogic plugin**：W16 scope 只 2 NPCs × 3 選項，Dialogic 學習曲線跟 plugin 兼容風險不划算。自寫 100 行 reusable component 反而學更深 — Callable.bind / signal connect / 動態 spawn button / chain dialog。Plugin 留到 W18 dialog tree 爆炸時再評估。
>
> **3. Gold 持久化 + 雙計數器**：原本 gold per run reset，Hub 永遠 0 金錢 UX 災難。改成持久化（Hub 消費才扣），但加 gold_earned_this_run 顯示『本場獲得』給 stage clear 用。**單一資料、雙視角** — 玩家看『本場』，系統看『總額』。」

---

**M4 (W14-W17) 進度：75% (3/4)**
**下一週 W17**：永久升級結算 + XOR binary Save / Load + Settings menu + M4 月底 retro
