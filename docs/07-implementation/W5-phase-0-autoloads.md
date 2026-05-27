# W5 / DFS Phase 0：5 個 Autoload + 資料夾骨架

> 對應 PROGRESS.md W5 / DFS docs/08-implementation/steps/phase-0-setup.md
> 完成日：2026-05-28
> Repo：[dice-fate-survivor](https://github.com/asd23353934/dice-fate-survivor) commit `311fb1f`

---

## 1. 目標 / 不目標

### 目標

- 在 DFS repo 內建 Godot project（與既有 docs/ + SPEC.md 共存）
- 建立**核心 5 個 autoload**：SettingsManager、EventBus、GameState、AudioManager、SceneRouter
- 設好 Godot project settings（1280x720、Forward+、debug warnings）
- 建好資料夾骨架（scripts/、scenes/、data/、assets/、tests/）
- F5 跑通 bootstrap 場景：5 個 autoload 印出 ready 訊息

### 不目標（之後階段補）

- 11 個 autoload 完整版（剩 6 個延後到對應週次）
- 任何真實遊戲邏輯
- UI / 美術 / 音效
- 資料 (.tres 卡片 / 敵人) — W7+ 才填
- 存檔系統 — W17

---

## 2. 設計決策

### 為什麼**先做 5 個**而不是 11 個？

DFS architecture.md 列了 11 個 autoload：

```
1. SettingsManager   ← W5 stub
2. Localization      ← W17 補
3. EventBus          ← W5 完整
4. SaveManager       ← W17 補
5. GameState         ← W5 完整
6. AudioManager      ← W5 stub
7. SceneRouter       ← W5 stub（W6 升級）
8. CameraShakeManager ← W22 polish
9. TutorialManager   ← W23+
10. Telemetry        ← M6 launch
11. DebugConsole     ← 開發中加
```

**先 5 個**的理由：
- **YAGNI**（You Aren't Gonna Need It）：W5 用不到的別建，避免維護負擔
- **編譯成本**：每個 autoload 都會在啟動時被 parse + _ready，多餘的拖慢開發循環
- **學習負擔**：一次寫 11 個 stub 太多噪音
- **彈性**：之後加 autoload 就是在 project.godot 多一行，無痛

### 為什麼**註冊順序很重要**？

Godot 啟動時 autoload **依宣告順序逐一 parse + _ready**。如果 A 用到 B 但 A 比 B 早 parse，會 `Parse Error: Identifier "B" not declared`。

我們的依賴關係：

```
SettingsManager ← 無依賴（最底層）
EventBus        ← 無依賴（signal hub）
GameState       ← 依賴 EventBus（emit signals）
AudioManager    ← 依賴 SettingsManager（讀 bgm/sfx volume）+ EventBus（聽事件播音）
SceneRouter     ← 依賴 EventBus（emit scene_changed）
```

所以順序：`SettingsManager → EventBus → GameState → AudioManager → SceneRouter`

> **W5 踩過這個雷**：第一次寫成 GameState 先、EventBus 後 → 30+ Parse Error。詳見 [hsin-dev-notes errors.md](https://github.com/asd23353934/hsin-dev-notes/blob/master/godot/errors.md)。

### 為什麼 GameState 拆 Permanent vs Current Run？

DFS 是 roguelite — 死掉 run 重置但有 meta 升級。狀態分兩類：

```
Permanent（跨 run 保留 / 之後存檔）：
  permanent_max_hp_bonus / permanent_energy_bonus
  unlocked_cards / unlocked_kits / total_gold_collected / run_count

Current Run（每 run reset_for_new_run 重置）：
  current_kit / floor_number / player_hp / energy / gold
  deck / hand / discard_pile / inventory
```

**好處**：
- `reset_for_new_run()` 只清 current_run 區塊，不影響升級紀錄
- 之後 SaveManager 只存 Permanent 區塊就好
- 程式碼自然分群，讀程式碼一眼看出哪些是進度、哪些是場上狀態

**反例**：所有變數混一起，每次新 run 要記得「哪些要 reset 哪些不能」→ 容易漏 → bug。

### 為什麼 EventBus 有 41 個 signal 而不是按需要才加？

DFS 三大層（戰鬥 / 棋盤 / Hub）需要的 signal **已經在 spec 階段想清楚**，一次寫齊好處：

- 寫業務邏輯時不會被「要不要加 signal」中斷
- 所有 signal 集中一處 = 之後 audit 容易
- 沒被 emit 的 signal 等於沒成本（只是 declare）

`@warning_ignore_start("unused_signal")` 整檔靜音，避免「沒人 emit」警告噪音。

### 為什麼路徑寫 `*res://...` 不寫 `*uid://...`？

兩種都可：

```
GameState="*res://scripts/autoload/game_state.gd"   ← path-based
GameState="*uid://dikw4dusy5oa0"                    ← UID-based
```

**path-based 好處**：human-readable、easier to debug、grep 找得到。
**UID-based 好處**：檔案改路徑不會壞、Godot 自動處理。

W3 用 UID-based 後來覺得不夠透明，W5 起改用 path-based。

---

## 3. 檔案清單

```
dice-fate-survivor/
├── project.godot                    ← 註冊 5 autoload + viewport + warnings
├── main.tscn                        ← bootstrap 驗證場景（W6 被 main_menu.tscn 取代）
├── scripts/
│   ├── main.gd                      ← bootstrap script，print 5 autoload 狀態
│   ├── autoload/
│   │   ├── settings_manager.gd      ← stub（locale / 音量預設值）
│   │   ├── event_bus.gd             ← 41 個 signal，分 5 層
│   │   ├── game_state.gd            ← Permanent + Current Run state + API
│   │   ├── audio_manager.gd         ← stub（play_bgm / play_sfx 空殼）
│   │   └── scene_router.gd          ← stub（change_scene 簡單版，W6 升級 fade）
│   └── data/                        ← .gitkeep（之後放 CardData / EnemyData class）
├── scenes/                          ← .gitkeep（之後放各場景）
├── data/                            ← .gitkeep（之後放 .tres）
│   ├── cards/ enemies/ items/ starting_kits/
├── assets/                          ← .gitkeep（之後放美術音樂）
└── tests/                           ← .gitkeep（之後放 test_runner）
```

---

## 4. 實作流程 / 順序

```
1. 評估 DFS 計畫：要 11 個 autoload，但 W5 只做核心 5 個
2. 在 DFS repo（已有 docs/ + SPEC.md）建 Godot project
   ⚠️ 不勾「版本控制中繼資料」（避免覆蓋既有 .gitignore）
3. 建資料夾骨架 + .gitkeep（讓 git 追蹤空資料夾）
4. 寫 EventBus（最底層、無依賴）
5. 寫 GameState（依賴 EventBus 的 emit）
6. 寫 SettingsManager / AudioManager / SceneRouter 三個 stub
7. project.godot 加 [autoload] 區塊（順序：Settings → Event → Game → Audio → Router）
8. project.godot 加 [debug] gdscript/warnings/exclude_addons=true（之後裝 plugin 不誤判）
9. project.godot 加 [display] 1280x720 + canvas_items + keep
10. 寫 main.gd + main.tscn 驗證場景
11. F5 跑：看 5 個 ready 訊息 + bootstrap 印出
12. Commit + push
```

**順序邏輯**：底層先（EventBus）→ 依賴它的（GameState）→ stub（其他 3 個）→ 註冊 → 驗證。

---

## 5. 關鍵 code 解析

### `event_bus.gd` — 41 signal 分 5 層

```gdscript
extends Node

@warning_ignore_start("unused_signal")
# ↑ 整檔靜音 unused_signal 警告（EventBus pattern 標準誤判）

# 玩家狀態（GameState 內部 emit）
signal player_hp_changed(current: int, max_hp: int)
signal player_died
# ...

# 戰鬥層
signal combat_started(encounter_id: String)
signal card_played(card: Resource, target)
# ...

# 棋盤層
signal dice_rolled(value: int)
# ...

# Hub / Meta
signal upgrade_purchased(upgrade_id: String)
# ...

# 遊戲流程
signal run_started(run_number: int)
signal scene_changed(scene_path: String)


func _ready() -> void:
    print("[EventBus] autoload ready，", _count_signals(), " 個 signal 待命")

func _count_signals() -> int:
    return get_signal_list().size()
```

**重點**：
- `@warning_ignore_start` 是 Godot 4.3+ 新功能，整檔靜音指定警告
- `signal X(arg: Type)` 寫型別 = signal 簽名清楚 + 訂閱端 IDE 補全
- `get_signal_list()` 是 Node 內建 method，回傳所有 signal（包括繼承）
- 印出 41 是因為 Node base class 也有 30 個（child_entered_tree 等內建）+ 我們 11 個自訂

### `game_state.gd` — State + API

關鍵 method：

```gdscript
func take_damage(amount: int) -> void:
    if player_hp <= 0:
        return                                    # 死了不再扣
    player_hp = max(0, player_hp - amount)        # clamp 0
    print("[GameState] 玩家受 %d 點傷害，剩 %d HP" % [amount, player_hp])
    EventBus.damage_dealt.emit(amount, "Player")
    EventBus.player_hp_changed.emit(player_hp, max_hp)
    if player_hp == 0:
        print("[GameState] 玩家死亡")
        EventBus.player_died.emit()
```

**Pattern**：**改 state + 立刻 emit signal**（W3 學的 pattern 延續）。

```gdscript
func reset_for_new_run(kit: String = "KIT_SWORD") -> void:
    current_kit = kit
    floor_number = 1
    max_hp = 80 + permanent_max_hp_bonus           # ← Permanent 加成生效
    player_hp = max_hp
    max_energy = 3 + permanent_energy_bonus
    energy = max_energy
    gold = 0
    deck.clear()                                    # 清 run 狀態
    discard_pile.clear()
    hand.clear()
    inventory.clear()
    run_count += 1
    # emit 3 個 signal 廣播
```

**重點**：
- `max_hp = 80 + permanent_max_hp_bonus` 顯示 Permanent 加成怎麼生效
- 不清 `unlocked_cards / unlocked_kits`（這些跨 run 保留）
- `run_count` 是統計，每次新 run +1

### `scene_router.gd` (W5 stub 版)

```gdscript
func change_scene(scene_path: String) -> void:
    get_tree().change_scene_to_file(scene_path)
    EventBus.scene_changed.emit(scene_path)
```

**最簡實作**：呼叫 Godot 內建 API + emit signal。W6 會升級加 fade。

### `project.godot` autoload 區塊

```
[autoload]

SettingsManager="*res://scripts/autoload/settings_manager.gd"
EventBus="*res://scripts/autoload/event_bus.gd"
GameState="*res://scripts/autoload/game_state.gd"
AudioManager="*res://scripts/autoload/audio_manager.gd"
SceneRouter="*res://scripts/autoload/scene_router.gd"
```

**`*` 前綴**意義：**啟用** 該 autoload（沒 `*` 就是 disabled）。
**順序**：依依賴關係，被依賴的在前面。

---

## 6. 觀念對照（前端視角）

| Godot autoload | Vue / Angular / React |
|---|---|
| SettingsManager（locale / volume） | Pinia store / Angular service / React Context |
| EventBus（signal hub） | Vue mitt / Angular Subject / RxJS Subject |
| GameState（global state） | Pinia / Vuex / Redux store |
| AudioManager（cross-scene 持續） | Angular root-provided service / React 全域 Audio singleton |
| SceneRouter（場景切換） | Vue Router / React Router |

**最大差別**：
- Godot autoload 是「**真實存在的 Node**」掛在場景樹根 → 有 _ready / _process 生命週期
- 前端 store 是「**純物件**」由框架管理 → 沒生命週期

**好處**：autoload 可以自己跑 _process 做定時任務、自己 connect signal、自己 add_child（如 SceneRouter 的 fade overlay）。

---

## 7. 擴充方式

### 加新 autoload（最常見擴充）

**範例**：W17 加 SaveManager。

1. 寫 `scripts/autoload/save_manager.gd`
2. `project.godot` 的 `[autoload]` 加一行：
   ```
   SaveManager="*res://scripts/autoload/save_manager.gd"
   ```
3. **位置**：依依賴順序插入。如果 SaveManager 用 SettingsManager 但不用 EventBus，可以插在 SettingsManager 後、EventBus 前；如果用 EventBus，插在 EventBus 後。
4. 重啟 Godot（Project Settings 自動讀取，但有時要重啟）
5. F5 應該看到新 autoload 印 ready 訊息

### 在 EventBus 加新 signal

```gdscript
# event_bus.gd 對應分類加一行：

# === 棋盤層 ===
signal player_picked_up_item(item_id: String, tile_index: int)  ← 新增
```

訂閱：

```gdscript
# 任何 .gd
func _ready() -> void:
    EventBus.player_picked_up_item.connect(_on_picked_up)

func _on_picked_up(item_id: String, tile_index: int) -> void:
    # 反應邏輯
```

Emit：

```gdscript
EventBus.player_picked_up_item.emit("HEAL_POTION", 12)
```

### 在 GameState 加新 state

範例：加「目前難度」。

```gdscript
# game_state.gd
var difficulty: int = 1   # 1=Normal, 2=Hard, 3=Insane

# reset_for_new_run 內保留：difficulty 不重置（玩家設定，跨 run 保留）
# 或重置：reset_for_new_run 內 difficulty = 1（每 run 重選）
```

**判斷依據**：跨 run 保留 → 放 Permanent 區塊頂部 + 不放進 reset_for_new_run；單 run → 放 Current Run 區塊 + 加進 reset。

### Permanent / Current Run 邊界判斷

**判斷規則**：

| 屬性 | Permanent | Current Run |
|---|---|---|
| 玩家經驗 / 進度 / 解鎖 | ✓ | |
| 每 run 重新算的數字 | | ✓ |
| 設定（語言 / 音量） | 不在 GameState（在 SettingsManager） | |
| 暫時 buff / debuff | | ✓ |
| 統計（run_count / total_gold） | ✓ | |
| 當前敵人 HP / 卡組組合 | | ✓ |

---

## 8. 常見錯誤 / debug 指南

### autoload 順序錯 → Parse Error 連鎖

**症狀**：30+ 個 `Identifier "X" not declared in the current scope.`
**根因**：autoload A 用了 autoload B 的 identifier，但 A 在 B 之前註冊。
**修法**：調整 `[autoload]` 順序，被依賴的擺前面。
**預防**：寫 autoload 時心裡先想「我用到誰」→ 確保那個在我前面。

### `@warning_ignore_start` 找不到識別字

**症狀**：`Parse Error: Identifier "warning_ignore_start" is not declared.`
**根因**：Godot < 4.3。
**修法**：升級 Godot 或改用 per-line `@warning_ignore`。

### 新 autoload 加了但沒生效

**症狀**：Output 沒看到該 autoload 的 ready 訊息。
**根因**：Project Settings → Autoload 那一筆**沒勾啟用** 或 `*` 前綴漏掉。
**修法**：勾啟用 / 加 `*`，重啟 Godot。

### Inspector 拖檔覆蓋不到 autoload 屬性

**症狀**：在 scene 內無法 `@export` 一個 autoload 進來。
**根因**：autoload 是全域 singleton，不需要 inject（直接寫 `GameState.xxx` 就拿到）。
**對策**：別嘗試把 autoload 當 PackedScene 拖檔；直接 reference。

### 在 autoload 內呼叫另一個 autoload 失敗（啟動時）

**症狀**：autoload A 的 `_init()` 或 `_ready()` 內呼叫 B → null 或 crash。
**根因**：A 比 B 早建構時，B 還不存在。
**修法**：
1. 確認註冊順序（B 在 A 之前）
2. 如果順序對還是壞 → 把跨 autoload 呼叫**延後到 _ready** 或更晚（_process 第一次 frame）

---

## 9. 下游依賴

| 之後階段 | 依賴 W5 的什麼 |
|---|---|
| W6 Phase 1 walking skeleton | SceneRouter（升級 fade）、EventBus.scene_changed |
| W7 Phase 2 戰鬥 prototype | GameState.spend_energy / take_damage、EventBus.combat_started/card_played |
| W11 Phase 3 戰鬥成熟 | EventBus.status_applied/removed、GameState.reset_energy_for_new_wave |
| W12-W13 Phase 4 棋盤 | EventBus.dice_rolled / tile_triggered / enemy_spawned |
| W14-W17 Phase 5-6 整合 + Hub | GameState.purchase_max_hp_upgrade、unlock_card、所有 Hub-related signal |
| W17 SaveManager | GameState 的 Permanent 區塊（讀寫存檔） |
| W21 AudioManager 補完 | SettingsManager 的 bgm/sfx volume |
| W22 polish | EventBus 全部既有 signal（為 hit pause / shake hook） |

---

## 10. 面試 talking point

> 「DFS 我規劃了 11 個 autoload，但 Phase 0 只先做核心 5 個（SettingsManager / EventBus / GameState / AudioManager / SceneRouter）。
>
> 順序很重要：被依賴的擺前面。GameState 在內部呼叫 EventBus.emit()，所以 EventBus 必須先註冊。
>
> GameState 我刻意拆成 Permanent vs Current Run 兩塊。Permanent 是跨 run 保留的（玩家升級紀錄、解鎖卡牌），Current Run 是每 run 重置的（HP、能量、手牌）。reset_for_new_run() 只清 Current Run 區塊，Permanent 永遠不動。這樣 SaveManager 之後存檔也很乾淨：只存 Permanent 區塊。
>
> 我有踩過 autoload 順序錯的雷 —— Parse Error 連鎖 30 多個。寫進 hsin-dev-notes 個人筆記，下次新專案直接查。」
