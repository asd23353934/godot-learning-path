# W6 / DFS Phase 1：Walking Skeleton（場景切換 flow）

> 對應 PROGRESS.md W6 / DFS docs/08-implementation/steps/phase-1-skeleton.md
> 完成日：2026-05-28
> Repo：[dice-fate-survivor](https://github.com/asd23353934/dice-fate-survivor) commit `555b77f`

---

## 1. 目標 / 不目標

### 目標

- **整個遊戲 flow 跑得通**（雖然每個畫面都是 placeholder）
- 主選單 → Hub → Board → Combat → 假勝利/失敗 → 回主選單
- 場景切換用 SceneRouter 帶 fade in/out 過場
- 驗證 autoload **跨場景持續存在**（換場景後 GameState 還是同一個）

### 不目標（後續階段）

- 任何真實遊戲邏輯（戰鬥、棋盤、Hub 都是空殼）
- 美術 / 音效
- 設定 menu（W17）
- 圖鑑（之後）
- i18n（之後）
- 存檔（W17）

---

## 2. 設計決策

### 為什麼**先做 walking skeleton** 而不是直接做戰鬥？

「Walking skeleton」是軟體工程 pattern：先有最小可運行的端到端骨架，**之後每個部分都在骨架上長肉**。

**好處**：
- 早期發現架構問題（場景切換 / autoload 持續性 / signal 跨 scene）
- 每個後續 phase 可以**獨立測試**（W7 寫戰鬥只專注戰鬥，不用煩切場）
- 給設計師 / 自己玩 placeholder 確認 game flow 順
- 比「先寫完戰鬥 → 再煩怎麼切場」省心

**反例**：先卡在 combat 寫到一半才發現 SceneRouter 還沒做，要中斷去寫 router。

### 為什麼 SceneRouter 用 **CanvasLayer + ColorRect** 做 fade？

選項對照：

| 方案 | 優 | 缺 |
|---|---|---|
| **CanvasLayer + ColorRect**（採用） | 跨場景持續存在、autoload 持有、層級可控 | 要寫 setup code |
| 每個 scene 自己有 fade overlay | scene 內封閉 | 重複 code 80 行 × 場景數，noise 多 |
| Camera2D 的 modulate | 簡單 | 影響整個 viewport 渲染，副作用多 |
| AnimationPlayer | 視覺強 | overkill；fade 一行 Tween 就好 |

**選 CanvasLayer 的關鍵**：
- autoload 持有，**整個遊戲生命週期共用一個 fade overlay**
- `CanvasLayer.layer = 100` 蓋過所有 2D scene 渲染
- 不會被 scene 切換破壞（autoload 不會 free）

### 為什麼 fade duration **預設 0.3 秒**？

UX research：

```
< 0.15s   太快 → 玩家看不到過場，感覺像 crash
0.2-0.4s  剛好 → 流暢但不卡
> 0.6s    太慢 → 連續換場時煩
```

0.3s 是業界 sweet spot。Slay the Spire / Hades 都接近這個值。

### 為什麼**刪掉 W5 的 main.tscn**，不留著當 debug scene？

兩種選擇：
- **刪掉**（採用）：保持 root 乾淨，只有 game-relevant scene
- 留著當 debug 入口

**選刪掉**：
- main.tscn 是 W5 任務（驗證 autoload）的副產物，任務完成就無用
- 留在 root 會混淆「main 是什麼」（Godot 沒禁止但 main_menu 才是 entry）
- F5 走 main_menu.tscn 後 main.tscn 永遠不會被執行 → 死碼
- 之後想 debug autoload 可以暫時加 print 到任何 scene 的 _ready

---

## 3. 檔案清單

```
dice-fate-survivor/
├── project.godot                       ← run/main_scene 從 main.tscn 改 scenes/main_menu.tscn
├── scripts/
│   └── autoload/
│       └── scene_router.gd             ← 從 stub 升級成 fade transition 實作
└── scenes/
    ├── main_menu.tscn / main_menu.gd   ← 主選單 4 按鈕
    ├── hub/
    │   ├── hub.tscn
    │   └── hub.gd                      ← Hub placeholder
    ├── board/
    │   ├── board.tscn
    │   └── board.gd                    ← Board placeholder
    └── combat/
        ├── combat.tscn
        └── combat.gd                   ← Combat placeholder（含假勝負）

刪除：
- main.tscn （W5 bootstrap，被 main_menu.tscn 取代）
- scripts/main.gd
```

---

## 4. 實作流程 / 順序

```
1. 設計 SceneRouter fade 機制：用 CanvasLayer + ColorRect overlay
2. 升級 scripts/autoload/scene_router.gd：
   · _ready 內 _setup_fade_overlay()
   · change_scene 改成 async function 用 await
3. 寫 scenes/main_menu.tscn + .gd：4 按鈕、開始按鈕呼叫 reset_for_new_run + change_scene
4. 寫 scenes/hub/hub.tscn + .gd：2 個 nav button
5. 寫 scenes/board/board.tscn + .gd：2 個 nav button
6. 寫 scenes/combat/combat.tscn + .gd：3 個按鈕（假勝、假敗、返回）
   · 假勝 emit combat_ended(true)
   · 假敗 emit combat_ended(false) + run_ended(false)
7. project.godot 改 run/main_scene = "res://scenes/main_menu.tscn"
8. 刪舊 main.tscn / scripts/main.gd
9. F5 跑全程：主選單 → Hub → Board → Combat → 假勝回 Board → ... → 回主選單
10. Commit + push
```

**順序邏輯**：先把基礎建設（SceneRouter）做完，再做依賴它的 scene。每個 scene 寫完都能獨立測試（雖然 F6 nav button 會錯，因為其他 scene 還沒建）。

---

## 5. 關鍵 code 解析

### `scene_router.gd` 完整解析

```gdscript
extends Node

var _fade_layer: CanvasLayer
var _fade_rect: ColorRect


func _ready() -> void:
    print("[SceneRouter] autoload ready")
    _setup_fade_overlay()


func _setup_fade_overlay() -> void:
    # 1. 建一個 CanvasLayer 作為 fade 容器
    _fade_layer = CanvasLayer.new()
    _fade_layer.layer = 100        # ← 重點：layer 數字大 = 渲染順序在上
    add_child(_fade_layer)         # 掛在 SceneRouter 自身 = autoload，永存

    # 2. 建一個 ColorRect 鋪滿整個螢幕
    _fade_rect = ColorRect.new()
    _fade_rect.color = Color.BLACK
    _fade_rect.modulate.a = 0.0   # 初始全透明
    _fade_rect.set_anchors_preset(Control.PRESET_FULL_RECT)   # 撐滿
    _fade_rect.mouse_filter = Control.MOUSE_FILTER_IGNORE     # 不擋滑鼠（透明時玩家還是能點下層）
    _fade_layer.add_child(_fade_rect)


func change_scene(scene_path: String, fade_duration: float = 0.3) -> void:
    # Phase 1: 黑屏出現（α 0 → 1）
    var tween_out = create_tween()
    tween_out.tween_property(_fade_rect, "modulate:a", 1.0, fade_duration)
    await tween_out.finished

    # Phase 2: 切場景（玩家看不到，因為黑屏蓋著）
    var err = get_tree().change_scene_to_file(scene_path)
    if err != OK:
        push_error("[SceneRouter] 切場景失敗：%s (err=%d)" % [scene_path, err])
        _fade_rect.modulate.a = 0.0
        return

    EventBus.scene_changed.emit(scene_path)

    # Phase 3: 等新場景 _ready 完
    await get_tree().process_frame

    # Phase 4: 黑屏淡掉（α 1 → 0）
    var tween_in = create_tween()
    tween_in.tween_property(_fade_rect, "modulate:a", 0.0, fade_duration)
```

**深入觀念**：

1. **CanvasLayer.layer**
   - Godot 渲染順序由 layer 決定，**數字大蓋過數字小**
   - 預設 scene 的 root 是 layer 0
   - 我們的 fade overlay 是 layer 100 → 永遠最上
   - HUD / Pause menu 之後可能用 layer 50 → 蓋場景但被 fade 蓋

2. **autoload 持有的 add_child 永久存在**
   - `add_child(_fade_layer)` 把 CanvasLayer 掛在 SceneRouter 底下
   - SceneRouter 是 autoload，整個遊戲不會 free
   - 所以這個 CanvasLayer **跨場景持續存在** → fade overlay 隨時可用

3. **`await` async pattern**
   - `await tween.finished` 暫停這個 function，等 tween 跑完
   - GDScript 4.x 的 await 跟 JS / Python async 概念一致
   - 必須在 function 簽名外加沒寫 async 沒關係，**有 await 自動是 async**

4. **`get_tree().process_frame`**
   - 等下一 idle frame
   - 用途：確保切場景後新 scene 已經 _ready，再執行下一段
   - 不等的話 fade in 可能在新 scene 還沒準備好的時候跑 → 視覺上有閃

5. **`push_error` vs `print`**
   - `push_error()` 把訊息推到 Godot 的 error panel + Output（紅色）
   - `print()` 只到 Output（白色）
   - 真錯誤用 `push_error`，debug 訊息用 `print`

### `main_menu.gd`

```gdscript
extends Control


func _ready() -> void:
    %StartButton.pressed.connect(_on_start_pressed)
    %SettingsButton.pressed.connect(_on_settings_pressed)
    %CodexButton.pressed.connect(_on_codex_pressed)
    %QuitButton.pressed.connect(_on_quit_pressed)
    print("[MainMenu] ready，GameState.run_count=", GameState.run_count)


func _on_start_pressed() -> void:
    GameState.reset_for_new_run("KIT_SWORD")         # 1. 重置 current_run
    SceneRouter.change_scene("res://scenes/hub/hub.tscn")  # 2. 切到 Hub


func _on_quit_pressed() -> void:
    get_tree().quit()
```

**重點**：
- `%StartButton` 是 unique_name_in_owner 寫法（W4 學的）
- `_on_start_pressed` 順序：reset run state **再**切場景。先切場景 reset 不到資料時可能造成 Hub _ready 讀到舊資料
- `get_tree().quit()` 正確關閉 Godot

### `combat.gd` 的假勝負 + signal

```gdscript
func _on_fake_win() -> void:
    print("[Combat] 假勝利 — 回 Board")
    EventBus.combat_ended.emit(true)         # ← 廣播
    SceneRouter.change_scene("res://scenes/board/board.tscn")


func _on_fake_lose() -> void:
    print("[Combat] 假失敗 — 回主選單")
    EventBus.combat_ended.emit(false)
    EventBus.run_ended.emit(false)            # ← 失敗時連 run 也結束
    SceneRouter.change_scene("res://scenes/main_menu.tscn")
```

**重點**：雖然現在沒人接這些 signal，但**先 emit 對的 signal** 讓之後加成就系統 / 統計 / 音效時直接訂閱即可。

---

## 6. 觀念對照（前端視角）

| Godot 概念 | Vue / Angular / React |
|---|---|
| SceneRouter.change_scene | Vue Router.push / React Router useNavigate |
| Fade transition via CanvasLayer | Vue `<Transition>` / React Framer Motion AnimatePresence |
| autoload 跨場景持續 | Pinia store / Angular root service / Context Provider 包整個 app |
| `await tween.finished` | `await new Promise(...)` |
| `get_tree().process_frame` | `await nextTick()` / `requestAnimationFrame` |
| %NodeName unique_name | DOM `id` selector / Vue ref |
| 場景切換 emit signal | router.afterEach hook |

---

## 7. 擴充方式

### 加新場景

**範例**：W11 加 Combo Lock 教學場景。

1. 建 `scenes/tutorial/combo_lock_tutorial.tscn` + `.gd`
2. tutorial.gd 寫：
   ```gdscript
   extends Control
   func _ready() -> void:
       %BackButton.pressed.connect(_on_back)

   func _on_back() -> void:
       SceneRouter.change_scene("res://scenes/main_menu.tscn")
   ```
3. 從其他 scene call：`SceneRouter.change_scene("res://scenes/tutorial/combo_lock_tutorial.tscn")`

**就這樣**。SceneRouter 自動 fade。

### 改變 fade 顏色 / 時長

```gdscript
# scene_router.gd

# 改顏色（如白色 flash）：
_fade_rect.color = Color.WHITE

# 改時長：
SceneRouter.change_scene("res://xxx.tscn", 0.5)   # 0.5 秒

# 不要 fade（瞬切）：
SceneRouter.change_scene("res://xxx.tscn", 0.0)
```

### 加不同 transition 樣式（slide / wipe）

之後 W22 polish 可以加：

```gdscript
# scene_router.gd
enum TransitionStyle { FADE, SLIDE_LEFT, WIPE_TOP }

func change_scene(path: String, style: TransitionStyle = TransitionStyle.FADE, duration: float = 0.3) -> void:
    match style:
        TransitionStyle.FADE:
            await _fade_transition(path, duration)
        TransitionStyle.SLIDE_LEFT:
            await _slide_transition(path, duration, Vector2(-1, 0))
        # ...
```

各 transition 用不同 Tween 配方。

### 加 Loading 畫面（大場景）

如果某 scene 載入時間 > 0.5 秒：

```gdscript
func change_scene(path: String, duration: float = 0.3) -> void:
    # 1. Fade out
    ...
    # 2. 顯示 Loading text
    _loading_label.show()
    # 3. async load
    ResourceLoader.load_threaded_request(path)
    while ResourceLoader.load_threaded_get_status(path) == ResourceLoader.THREAD_LOAD_IN_PROGRESS:
        await get_tree().process_frame
    # 4. Loading 隱藏 + change_scene_to_packed + fade in
    _loading_label.hide()
    ...
```

DFS MVP 規模還用不到，但 spec 要求時可加。

### 場景間傳遞資料

**問題**：Hub 點「進關卡」要傳「選了哪個 kit」給 Board scene。

**做法 A — 用 GameState（推薦）**：
```gdscript
# Hub
GameState.current_kit = "KIT_SPELL"
SceneRouter.change_scene("res://scenes/board/board.tscn")

# Board._ready
print("玩家用：", GameState.current_kit)
```

**做法 B — 用 EventBus（事件式）**：
```gdscript
# Hub
EventBus.kit_selected.emit("KIT_SPELL")
SceneRouter.change_scene(...)
# Board 之前 connect kit_selected signal
```

**做法 C — SceneRouter 加參數**：
```gdscript
SceneRouter.change_scene(path, 0.3, {"kit": "KIT_SPELL"})
# 新 scene _ready 內讀 SceneRouter.last_args
```

**選 A** 對 DFS 最自然 —— GameState 本來就是全域狀態源。

---

## 8. 常見錯誤 / debug 指南

### Fade 黑屏不消失

**症狀**：切場景後永遠黑屏。
**根因**：
1. `change_scene_to_file` 回傳 err 但你沒檢查
2. 新 scene `_ready` crash 導致 fade in tween 永遠跑不到
**修法**：看 push_error 訊息；F5 看 Output 有沒有錯
**預防**：scene_router.gd 已加 err 檢查 + reset alpha。

### 按鈕沒反應

**症狀**：點按鈕沒事發生。
**根因**：
1. signal 沒 connect
2. fade overlay mouse_filter 不是 IGNORE → 擋掉 input
**修法**：
1. 查 `pressed.connect()` 寫法（W4 學過）
2. 查 scene_router.gd 的 `_fade_rect.mouse_filter = Control.MOUSE_FILTER_IGNORE`

### autoload 在新場景拿不到資料

**症狀**：Hub._ready 印 GameState.player_hp 是 0。
**根因**：忘記呼叫 `reset_for_new_run()` 初始化。
**修法**：main_menu 點開始時要呼叫 reset。

### 連續快速點切場景按鈕崩潰

**症狀**：玩家連點兩次按鈕 → fade 跑一半被中斷 → 視覺亂。
**根因**：change_scene 沒擋並發。
**修法（之後可加）**：
```gdscript
var _is_transitioning: bool = false

func change_scene(path: String, duration: float = 0.3) -> void:
    if _is_transitioning:
        return        # 忽略
    _is_transitioning = true
    # ... 原本邏輯 ...
    _is_transitioning = false
```

### 黑屏切場景時還能 click 下層

**症狀**：fade 1.0 黑屏時，玩家還能點到底下的 button。
**根因**：mouse_filter = IGNORE 讓 ColorRect 不接 input → 穿透到底下。
**修法**：fade 滿時改 mouse_filter = STOP，淡掉時改回 IGNORE：
```gdscript
# 進場 fade 滿時
_fade_rect.mouse_filter = Control.MOUSE_FILTER_STOP

# fade 完時
_fade_rect.mouse_filter = Control.MOUSE_FILTER_IGNORE
```

W6 沒做這個但**可以之後加**避免 race condition。

---

## 9. 下游依賴

| 之後階段 | 怎麼用 W6 的成果 |
|---|---|
| W7 Phase 2 戰鬥 prototype | combat.tscn 不再是 placeholder，加入 W4 移植的 Card/Hand/Enemy |
| W12-W13 Phase 4 棋盤 | board.tscn 改成真實 20 格 closed loop |
| W14-W17 Phase 5-6 Hub | hub.tscn 加 4 個 NPC（訓練師 / 卡牌商 / ...） |
| W16 VN 系統 | 新增 `scenes/vn/dialog.tscn`，從 hub / 棋盤事件 call |
| W22 polish | SceneRouter 加 slide / wipe transition；UI 加 fade 進場動畫 |
| W23+ 主選單美化 | main_menu.tscn 加背景圖 / Live2D 立繪 / 動態文字 |
| W25 Live2D demo | 新增 `scenes/demo/live2d_demo.tscn`，從主選單 codex 進入 |

---

## 10. 面試 talking point

> 「我用 walking skeleton pattern 起手 DFS 開發：先把整套 game flow 串起來（主選單 → Hub → Board → Combat → 結算），雖然每個 scene 都只是 placeholder，但場景切換 + autoload 跨場景持續性都驗證過了。
>
> 場景切換用我寫的 SceneRouter autoload 處理，帶 fade in/out 過場。技術上是用 CanvasLayer.layer=100 蓋過所有 scene，配 ColorRect 全螢幕 + Tween modulate.a。autoload 持有 = 跨 scene 不會 free。
>
> change_scene 是 async function，用 await 串：fade out → 切 scene → 等 process_frame → fade in。整套 30 行 code。
>
> 我刻意把假勝/假敗按鈕 emit 對應 EventBus.combat_ended / run_ended signal，雖然現在沒人接，但之後加成就系統、音效、統計時直接訂閱就好，不用回頭改 combat.gd。**先設計好事件介面，比之後 retrofit 省心**。」
