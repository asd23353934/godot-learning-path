# W15 / Phase 5 (2/2)：Boss 2-phase + Run End panel + 敵人 data-driven

> 對應 repo：[dice-fate-survivor](https://github.com/asd23353934/dice-fate-survivor)
> 對應 PROGRESS milestone：M4 W15
> 寫於 Day 7（2026-05-29）

---

## 1. 目標 / 不目標

### 目標
- **EnemyData resource**：抽出敵人資料（鏡像 W7 CardData 模式），敵人變 data-driven
- **DICE_LORD boss**：HP 60、2 phase 切換（HP < 50% 觸發狂暴 + 施加玩家 weak）
- **board boss tile 視覺**：tile 17 用金色大 marker，跟普通敵人區分
- **RunEndPanel**：玩家死亡顯示結算 panel（與 stage clear panel 對稱）
- **stage clear panel 加 Boss 擊敗**：boss_defeated_this_run 顯示
- **修 W14 cosmetic**：EnemiesLeftLabel 殘留問題

### 不目標
- 多 boss（W18 內容期）
- Boss 獨立 reward pool（W18 加 rarity 後做）
- 多 stage 串接（W17 之後或 M5）
- EnemyDatabase autoload（2 隻太少，直接 preload 即可；之後 5+ 隻再包）

---

## 2. 設計決策

### A. 為什麼抽 EnemyData resource？

- W7-W14：enemy.gd 用 `@export var enemy_name/max_hp/attack_damage`，單支 hardcode
- W15+：要 2 種敵人（rat + boss），單支 .tscn 不夠
- 三種方案：
  1. **複製 enemy.tscn → boss.tscn 改 @export** — 笨方法，每多一隻就一份 .tscn，邏輯沒共用
  2. **EnemyData .tres + enemy.gd 吃 data**（採用）— 資料 / 邏輯分離，新敵人只丟 .tres
  3. **EnemyDatabase autoload mirror CardDatabase** — 之後加再說（2 隻太少）

→ **方案 2 勝**：與 W9 CardDatabase 同個 pattern，玩家加 reward 卡用 .tres、敵人也用 .tres，data-driven 一致性高。

### B. 為什麼 2 phase 不開 boss.gd 繼承 enemy.gd？

- 繼承會固化「boss 是特殊種類」這個 assumption
- 但「2-phase 是行為特性」，不一定只 boss 才有
- 之後可能有「狂戰士」普通敵也有 phase 2（hp 低時加力）
- 用 `has_phase_2: bool` 在 EnemyData 內標 → 任何敵人都可以開 phase 2，符合 **composition over inheritance**

> Vue 類比：不是「BossComponent extends EnemyComponent」，是「EnemyComponent props.has_phase_2 = true」。

### C. 為什麼 boss tile 用 `boss_tile_index: int` 不用 enum / Array

- W15 只有 1 隻 boss → int 足夠
- Array `boss_tile_indices: Array[int]` 更通用但 over-engineered for current scope
- W18 之後可能多 boss → 改 Array 容易（reset_for_new_run 一行）
- **避免 premature abstraction**：能用 int 解決就用 int

### D. RunEndPanel 為什麼放 combat.tscn 內部，不開獨立 run_end.tscn？

- 跟 stage clear panel（在 board.tscn 內）對稱
- 玩家死亡的「上下文」是「戰鬥剛打輸」，玩家視角希望看到「最後一場戰鬥的延續」
- 切到新 scene 反而打斷情境
- 之後加「重玩當前戰鬥」按鈕也比較自然（按下時 reload combat scene）

### E. enemy.gd 保留 @export var enemy_name 而不刪

- enemy.tscn 還在用 `@export var enemy_name = "史萊姆"`（W7 寫的）
- 直接刪會跳「property not found」警告
- W15 加 `@export var enemy_data: EnemyData` 為主，舊 @export 保留為「fallback / 向下相容」
- set_data 會覆蓋這些值
- **不破 W7-W14 任何測試**

---

## 3. 檔案清單

### 新增

| 檔案 | 角色 |
|---|---|
| `scripts/data/enemy_data.gd` | EnemyData Resource class（id / hp / 攻擊 / 2-phase 欄位） |
| `data/enemies/rat.tres` | RAT 普通敵（HP 18, ATK 5, no phase 2） |
| `data/enemies/dice_lord.tres` | DICE_LORD boss（HP 60, ATK 8 → 12, phase 2 + weak） |

### 修改

| 檔案 | 變動 |
|---|---|
| `scripts/autoload/game_state.gd` | +`boss_tile_index` / +`boss_defeated_this_run` / `reset_for_new_run` 設 boss tile = 17 |
| `scripts/combat/enemy.gd` | +`@export enemy_data` / +`set_data()` / +`_enter_phase_2()` / +`phase_changed` signal / enemy_turn_attack 加 apply status |
| `scenes/combat/combat.gd` | preload RAT/DICE_LORD / +`_decide_enemy_data()` / +`_on_enemy_phase_changed` / `_on_player_died` 改顯示 RunEndPanel / boss 死亡標 `boss_defeated_this_run` |
| `scenes/combat/combat.tscn` | +`RunEndPanel`（hidden 初始） |
| `scenes/board/board.gd` | `_render_board_enemies` boss tile 用金色大 marker / stage clear panel 加 boss 擊敗 / `_on_all_enemies_cleared` 補 `_refresh_ui()` 修 W14 殘留 |

---

## 4. 實作流程（建構順序）

1. **EnemyData class**（資料 schema 先定）
2. **rat.tres + dice_lord.tres**（資料實例）
3. **enemy.gd 重構**（set_data + phase 邏輯）
4. **GameState 加 boss state**
5. **combat.gd 接 data + phase listener + RunEndPanel**
6. **combat.tscn 加 RunEndPanel UI**
7. **board.gd boss tile 視覺 + 結算欄位**

順序原則：**Resource schema → 實例 → 行為 → UI / 整合**。schema 是契約，先定才能讓下游 reference。

---

## 5. 關鍵 code 解析

### 5.1 EnemyData schema 設計

```gdscript
class_name EnemyData
extends Resource

@export var id: String = ""
@export var display_name: String = "敵人"
@export var max_hp: int = 18

# Phase 1（始終生效）
@export var attack_damage: int = 5
@export var applies_weak_on_attack: int = 0
@export var applies_vulnerable_on_attack: int = 0

# Phase 2（boss / 多階段用）
@export var has_phase_2: bool = false
@export var phase_2_threshold: float = 0.5
@export var phase_2_attack_damage: int = 0         # 0 = 沿用 phase 1
@export var phase_2_applies_weak: int = 0
@export var phase_2_applies_vulnerable: int = 0
@export_multiline var phase_2_announce_text: String = ""

@export var is_boss: bool = false                  # reward / 結算區分
```

設計重點：
- `phase_2_attack_damage = 0` 代表「沿用 phase 1」— 避免 boss 設計時必須填全部欄位
- `applies_weak_on_attack` vs `phase_2_applies_weak` 分開 — phase 2 可以加 status 是 phase 1 沒有的特性
- `is_boss` 給 combat.gd 標 `boss_defeated_this_run` 用，不直接影響行為

### 5.2 enemy.gd set_data 模式

```gdscript
@export var enemy_data: EnemyData

func _ready() -> void:
    if enemy_data != null:
        set_data(enemy_data)
    else:
        hp = max_hp     # 向下相容 W7 @export 路徑
        _refresh()


func set_data(data: EnemyData) -> void:
    enemy_data = data
    enemy_name = data.display_name
    max_hp = data.max_hp
    attack_damage = data.attack_damage
    current_apply_weak = data.applies_weak_on_attack
    current_apply_vulnerable = data.applies_vulnerable_on_attack
    _has_phase_2 = data.has_phase_2
    _phase_2_threshold = data.phase_2_threshold
    current_phase = 1
    hp = max_hp
    statuses.clear()
    _refresh()
```

要點：
- **fallback 路徑**：enemy_data null → 用 legacy @export（讓 W14 之前的 tscn 還能跑）
- **覆寫，不增量**：set_data 把所有 state 重設一遍（避免 leftover 上一場 phase 狀態）
- combat.gd 在 _ready 內呼叫，timing 是「Enemy 自己的 _ready 跑完之後」（child first → parent）

### 5.3 take_damage 觸發 phase 2

```gdscript
func take_damage(amount: int) -> void:
    if hp <= 0:
        return
    hp = max(0, hp - amount)
    _refresh()
    hp_changed.emit(hp, max_hp)
    EventBus.damage_dealt.emit(amount, enemy_name)
    if hp == 0:
        died.emit()
        return

    # W15：phase 2 觸發（只能觸發一次，靠 current_phase == 1 守門）
    if _has_phase_2 and current_phase == 1 and float(hp) / float(max_hp) < _phase_2_threshold:
        _enter_phase_2()
```

要點：
- **float division** — 不寫 `hp / max_hp < 0.5` 因為 int 除法會 truncate
- `current_phase == 1` 守門 — phase 2 不會觸發兩次（即使被打多次都已經是 phase 2）
- **死亡優先**：如果這次攻擊直接 hp = 0 → 走 died，phase 2 不觸發
  > 邊界 case：HP 31/60 被打 31 → hp = 0 → died，不進 phase 2 ✓

### 5.4 phase 2 切換 + announce banner

enemy.gd：
```gdscript
func _enter_phase_2() -> void:
    current_phase = 2
    if enemy_data.phase_2_attack_damage > 0:
        attack_damage = enemy_data.phase_2_attack_damage
    if enemy_data.phase_2_applies_weak > 0:
        current_apply_weak = enemy_data.phase_2_applies_weak
    if enemy_data.phase_2_applies_vulnerable > 0:
        current_apply_vulnerable = enemy_data.phase_2_applies_vulnerable
    phase_changed.emit(2, enemy_data.phase_2_announce_text)
```

combat.gd：
```gdscript
%Enemy.phase_changed.connect(_on_enemy_phase_changed)

func _on_enemy_phase_changed(new_phase: int, announce: String) -> void:
    var text = announce if announce != "" else "Phase %d!" % new_phase
    %MessageLabel.text = text
    %MessageLabel.show()
    await get_tree().create_timer(1.6).timeout
    if %Enemy.hp > 0:           # 防玩家此時也死了/敵人也死了
        %MessageLabel.hide()
```

要點：
- `phase_changed.emit(int, String)` — phase 號碼 + announce 文字一起傳，避免 listener 還要去查 `enemy_data.phase_2_announce_text`
- announce 空字串時 fallback 用「Phase N!」— UX 不至於空白
- await 之後檢查 `%Enemy.hp > 0` — 防 race（玩家 phase 2 切換瞬間就被秒）

### 5.5 玩家死亡 → RunEndPanel

```gdscript
func _on_player_died() -> void:
    %MessageLabel.text = "DEFEATED"
    %MessageLabel.show()
    %EndTurnButton.disabled = true
    EventBus.combat_ended.emit(false)
    EventBus.run_ended.emit(false)
    _show_run_end_panel()


func _show_run_end_panel() -> void:
    var stats_text = "戰勝場次：%d\n累計回合：%d\n金錢累積：%d\n牌組大小：%d 張\n擊敗 Boss：%s" % [
        GameState.combats_won_this_run,
        GameState.turn_number,
        GameState.gold,
        GameState.current_run_deck.size(),
        "是" if GameState.boss_defeated_this_run else "否",
    ]
    %RunEndStatsLabel.text = stats_text
    %RunEndPanel.show()
```

跟 stage clear panel 設計對稱：
- 都顯示 5 個關鍵 stats
- 都用 panel + 按鈕（玩家自己點返回，不自動 timer）
- 都在原 scene 內 toggle visibility，不開新 scene

### 5.6 enemy attack 施加玩家 status

```gdscript
func enemy_turn_attack() -> void:
    if hp <= 0:
        return
    var dmg = attack_damage
    if "weak" in statuses:
        dmg = int(dmg * 0.75)
    GameState.take_damage(dmg)

    # W15：強敵 / boss 攻擊時施加玩家 status
    if current_apply_weak > 0:
        GameState.apply_player_status("weak", current_apply_weak)
    if current_apply_vulnerable > 0:
        GameState.apply_player_status("vulnerable", current_apply_vulnerable)
```

- 順序：先扣 HP 再 apply status（如果 player_hp 變 0，apply status 沒效果，但不會 crash）
- `current_apply_weak` 是 runtime var（會被 phase 2 切換覆寫），不是直接讀 enemy_data — phase 1/2 共用同個 attack path

### 5.7 board boss tile 視覺區分

```gdscript
for tile_idx in GameState.board_enemies:
    var marker = ColorRect.new()
    var is_boss_tile = (tile_idx == GameState.boss_tile_index)
    if is_boss_tile:
        marker.color = Color(0.95, 0.7, 0.15, 1)   # 金色
        marker.size = Vector2(32, 32)              # 較大
        marker.position = _tile_positions[tile_idx] + Vector2(12, -36)
    else:
        marker.color = Color(0.2, 0.05, 0.05, 1)   # 暗紅
        marker.size = Vector2(24, 24)
        marker.position = _tile_positions[tile_idx] + Vector2(15, -30)
    add_child(marker)
    _enemy_visuals[tile_idx] = marker
```

- 玩家一進 board 就看得到「右下角金色那隻是 boss」 — UX feedback
- 之後可加 BossNameLabel 顯示 "DICE LORD" 文字
- 純 code 生成 marker — 沒有用 Sprite2D，因為還沒美術 assets

---

## 6. 觀念對照（前端 / Vue / Angular）

| Godot 概念 | Vue / Angular 類比 |
|---|---|
| `EnemyData` resource | TypeScript interface / Pinia store type schema |
| `enemy_data: EnemyData` @export | `@Input() data: EnemyData` Angular component prop |
| `set_data(data)` | `ngOnChanges` / Vue `watch` 重設 internal state |
| `phase_changed` signal | `@Output() phaseChanged = new EventEmitter()` |
| `Resource .tres` | JSON 設定檔 / Strapi CMS content type 實例 |
| Composition over inheritance | Vue 用 `props.is_boss` 取代 `<BossButton>` 繼承 |
| `boss_tile_index: int` | 簡單 enum 字段（避免 over-design Array） |

---

## 7. 擴充方式

### 加新敵人（5 分鐘）

1. 開 Godot Inspector，建新 EnemyData：File → New Resource → EnemyData
2. 填欄位（display_name = "毒蛇"，max_hp = 25，applies_vulnerable_on_attack = 1）
3. 存成 `data/enemies/snake.tres`
4. combat.gd 加 `const SNAKE_DATA = preload("...")`
5. `_decide_enemy_data` 加邏輯（哪個 tile 用 snake）

無需改 enemy.gd / enemy.tscn — 純資料化擴充。

### 加多 boss（W18）

```gdscript
# game_state.gd
var boss_tile_indices: Array[int] = []     # 改 Array

# reset_for_new_run:
boss_tile_indices = [17]                    # 之後加 [10, 17] 等

# combat.gd _decide_enemy_data:
if GameState.current_tile_index in GameState.boss_tile_indices:
    return DICE_LORD_DATA    # 之後依 tile 選不同 boss
```

### 加 phase 3（少數高難 boss）

EnemyData：
```gdscript
@export var has_phase_3: bool = false
@export var phase_3_threshold: float = 0.2   # HP < 20%
@export var phase_3_attack_damage: int = 0
@export var phase_3_applies_vulnerable: int = 0
```

enemy.gd take_damage 加：
```gdscript
if _has_phase_3 and current_phase == 2 and float(hp) / float(max_hp) < _phase_3_threshold:
    _enter_phase_3()
```

### 加 enemy retaliation 機制（被打反擊）

enemy.gd take_damage 結尾：
```gdscript
if enemy_data.has_retaliation:
    GameState.take_damage(enemy_data.retaliation_damage)
```

### 加 RunEndPanel「重玩」按鈕

```gdscript
%RunEndReplayButton.pressed.connect(_on_replay_pressed)

func _on_replay_pressed() -> void:
    SceneRouter.change_scene("res://scenes/combat/combat.tscn")  # 不重置 GameState
```

---

## 8. 常見錯誤（踩雷）

### 8.1 enemy_data 設了但 _ready 跑時還是用 default

**原因**：combat.gd 在 `%Enemy.set_data(rat)` 之前，Enemy._ready 已經跑過 — 此時 enemy_data 還是 null（除非 tscn 編輯時 set 過）。

**解法**：Enemy._ready 內處理「沒 data 用 default」即可：
```gdscript
func _ready() -> void:
    if enemy_data != null:
        set_data(enemy_data)
    else:
        hp = max_hp
        _refresh()
```

之後 combat.gd set_data 一進來會覆蓋。

### 8.2 phase 2 觸發兩次

**症狀**：banner 顯示兩次。

**原因**：忘了 `current_phase == 1` 守門。

**修法**：
```gdscript
if _has_phase_2 and current_phase == 1 and float(hp) / float(max_hp) < _phase_2_threshold:
    _enter_phase_2()
```

### 8.3 int 除法導致 phase 永遠不觸發

**症狀**：boss HP 從 60 打到 29，沒切 phase 2。

**錯誤**：
```gdscript
if hp / max_hp < 0.5:    # ❌ int 除法 → 29 / 60 = 0
```

**修法**：
```gdscript
if float(hp) / float(max_hp) < 0.5:    # ✓ 0.4833
```

### 8.4 enemy.tscn 跳「property X not found」警告

**原因**：W14 enemy.gd 還有 `@export var enemy_name`，W15 改成 runtime var → tscn 載入時找不到。

**修法**：保留 @export（即使 set_data 會覆寫），維持向下相容。

### 8.5 RunEndPanel 顯示但點按鈕沒反應

**原因**：忘了在 combat.gd _ready 連接 `%RunEndMenuButton.pressed.connect(_on_run_end_menu_pressed)`。

**檢查**：所有 panel 按鈕都要在 _ready 一開始 connect，不要在 show 時才連（panel show 多次會 connect 多次）。

### 8.6 同一場戰鬥死兩次（player_died 和 enemy_died 同時觸發）

**症狀**：罕見 edge case，玩家 hp = 1 出毒卡，enemy hp = 1 → poison tick：
- enemy 中毒死 → die.emit → _on_enemy_died → reward scene
- player 也中毒（如果有 poison_to_self）→ player_died → RunEndPanel

兩個都 emit 結果 race。

**修法**：在 combat.gd 加 `var _combat_resolved: bool = false`，第一個觸發就 set true，第二個 early return：
```gdscript
func _on_enemy_died() -> void:
    if _combat_resolved: return
    _combat_resolved = true
    ...

func _on_player_died() -> void:
    if _combat_resolved: return
    _combat_resolved = true
    ...
```

W15 暫不處理（DFS 目前無 poison_to_self 卡，不會踩到）。

---

## 9. 下游依賴

| 後續週 | 依賴 W15 的什麼 |
|---|---|
| **W16 Hub** | `boss_defeated_this_run` → meta progression（首次擊敗 boss 解鎖獎勵） |
| **W17 Save** | EnemyData 不存（preload 即可）；boss_defeated_this_run 需序列化 |
| **W18 內容** | 直接丟新 .tres 進 `data/enemies/` 就增加敵人 — combat.gd 加 case 即可 |
| **W19 棋盤事件** | 用 EnemyData pattern 寫 EventData / ItemData |
| **W22 美術** | Enemy 視覺從 ColorRect 換 Sprite2D，data-driven 結構不變 |

---

## 10. 面試 talking point

> 「W15 我把 DFS 敵人系統從 hardcoded 改 data-driven。同一支 enemy.gd + 同一個 enemy.tscn，靠 EnemyData resource (.tres) 決定每個敵人的 HP / 攻擊力 / 是否多階段 / 階段切換條件。
>
> Boss (DICE_LORD) 不是 boss.gd 繼承 enemy.gd — 是 `has_phase_2: bool` 寫在 EnemyData 內。**composition over inheritance**，這樣之後若有普通敵人也想開 phase 2（狂戰士低 HP 加力），不用改架構。
>
> Phase 2 觸發點放在 take_damage 內，`current_phase == 1 && float(hp)/float(max_hp) < threshold` 守門，避免重複觸發 + int 除法 truncate 陷阱。
>
> 玩家死亡的 RunEndPanel 跟 stage clear panel 設計對稱 — 都顯示 5 個關鍵 stats、都在原 scene 內 toggle visibility、都靠玩家手動點按鈕，不自動 timer 切場景。對稱設計讓 UX 一致、之後維護成本低。」

---

**M4 (W14-W17) 進度：50% (2/4)**
**下一週 W16**：Hub 場景 + 2 NPCs + Dialogic plugin VN 對話
