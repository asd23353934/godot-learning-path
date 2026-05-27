# 學習進度（週級）

> 起始日：2026-05-22
> 配合 [`docs/02-learning-path.md`](docs/02-learning-path.md) 月度切片
> 細項任務見 [`docs/05-tasks.md`](docs/05-tasks.md)

---

## Status Legend

```
[ ]   Todo
[~]   In Progress
[x]   Done
[!]   Blocked
[s]   Skipped
```

---

## 進度總覽

```
M1 (W1-W4)   ██████████  100% Godot 基礎 + 2 tutorial
M2 (W5-W8)   ██████████  100% DFS Phase 0-2（戰鬥 prototype）
M3 (W9-W13)  ░░░░░░░░░░  0%   DFS Phase 3-4（戰鬥成熟 + 棋盤）
M4 (W14-W17) ░░░░░░░░░░  0%   DFS Phase 5-6（整合 + Hub Meta）
M5 (W18-W22) ░░░░░░░░░░  0%   DFS Phase 7-8（內容 + 美術）
M6 (W23-W26) ░░░░░░░░░░  0%   收尾 + Live2D + 履歷
```

完成週數：**0 / 26**

---

## M1（W1-W4）：Godot 基礎 + 2 tutorial

### W1（2026-05-22 ~ 05-28）：安裝 + GDScript 起步

- 目標時數：10-15 hr
- [x] 裝 Godot 4.6.2 Standard（實際裝 4.6.3.stable，同系列）
- [x] VSCode Godot Tools 擴充（geequlim 出的；雙擊 .gd 可跳 VSCode 開啟）
- [ ] 看完 Godot Docs - Getting Started（philosophy / nodes / signals）
- [x] Hello world：按鈕 → console print
- [x] GDScript 練 100 行（fizzbuzz / 排序 / class）
- [ ] 週末 retro：3 行心得寫在下方

週記：

```
Day 2（2026-05-23）即時筆記：
- 完成 Godot 4.6.3 安裝 + 第一個 hello world（Control + Button + button.gd）
- 觀念：場景樹 ≈ Vue 組件樹；signal (pressed.connect) ≈ event emit；.tscn 場景與 .gd 腳本分離
- GDScript 對應：func ≈ function、_ready ≈ mounted、@onready 之後練、type hint 用 : 寫
- 完成 VSCode Godot Tools 擴充 + practice.gd 三題全過（FizzBuzz / Bubble Sort / Card class）
- GDScript 觀念：
  · 沒有 import：class_name 自動全域註冊（class_name Card + extends RefCounted）
  · Array 是 reference type，函式內改外面看得到，不用 return
  · sort_custom 的 callable 回傳 bool（a 該在前嗎），不是 JS 的 a-b 數字差
  · 沒有 switch，有 match（比對值不是條件；條件判斷還是用 if/elif）
  · 字串模板用 "%s %d" % [a, b]，不是 backtick
  · F5 跑主場景、F6 跑當前場景（hello-godot 一個 project 多場景就要分清楚）
- 踩雷：
  · range(20) 是 0~19 不是 1~20（FizzBuzz 首尾要 range(1, n+1)）
  · 累加字串法 FizzBuzz 不用寫 if % 15（多寫會重複串）
  · 建場景後忘記附加 script 到 Node 上，跑了但 _ready 沒呼叫，Output 空白
- 下一步：讀 Godot Docs - Getting Started（philosophy / nodes / signals）+ 週末 retro

（每週做完寫 1-3 行：學到什麼、卡哪裡、下週調整）
```

---

### W2：Dodge the Creeps tutorial

- 目標時數：10-15 hr
- [x] 跟著 [Your First 2D Game](https://docs.godotengine.org/en/stable/getting_started/first_2d_game/index.html) 做完（Part 1-7 全部完成）
- [ ] 改成自己版本（如 mob 改成卡牌掉下來閃避）← 選配
- [x] 學到 scene 組合 / signal 連接 / Timer / 隨機 spawn / HUD
- [ ] 週末 retro

週記：

```
Day 2（2026-05-23）即時筆記（W1 提前完成 → 同日跳 W2）：

完成範圍：
- 重構專案結構：godot-projects 改為父資料夾（hello-godot / dodge-the-creeps 兩個子專案並存）
- W2 Part 1：viewport 480x720 + canvas_items + keep（手機直式）
- W2 Part 2-3：Player 場景（Area2D + AnimatedSprite2D + CollisionShape2D），方向鍵移動 + 動畫切換 + 邊界 clamp
- W2 Part 4：Mob 場景（RigidBody2D + VisibleOnScreenNotifier2D），隨機 3 種敵人動畫 + 出畫面自動 free
- W2 Part 5：Main 主場景組合（3 個 Timer + Path2D + PathFollow2D + Marker2D），signal 連線網跑通
- W2 Part 6：HUD（CanvasLayer + Label + Button），分數顯示 / Start 按鈕 / Game Over 訊息

新 Node 觀念：
- Area2D：「我會偵測別人撞我」（玩家用）
- RigidBody2D：「給速度就被引擎推著走」（敵人用，gravity_scale = 0 不要掉）
- AnimatedSprite2D + SpriteFrames（Resource）：node 是顯示器 / SpriteFrames 是動畫資料
- CollisionShape2D：給 Area2D / RigidBody2D 配 hitbox 形狀（CapsuleShape2D 等）
- VisibleOnScreenNotifier2D：出畫面 emit screen_exited → queue_free 自清（避免記憶體爆）
- Path2D + PathFollow2D：定義曲線 + 沿線跑的指標，progress_ratio 0~1 自動算座標
- Marker2D：純座標標記，runtime 看不見（玩家起始點 / spawn point）
- Timer：one_shot 一次性 vs 重複；wait_time + autostart + timeout signal
- CanvasLayer：UI 圖層，不受 camera 影響（≈ CSS position: fixed）
- Label / Button：基本 UI 元件，anchor 預設快速排版（≈ CSS Flexbox + 響應式）
- Theme override：局部 styling（≈ inline style；整套主題用 Theme resource）

GDScript 新語法：
- @export var x: PackedScene → Inspector 可拖檔欄位（≈ Vue props，dependency injection）
- instantiate() → 從 PackedScene 開模子（≈ new Class()）
- add_child(node) → 加進場景樹才會被處理（沒加就放在記憶體沒人理）
- queue_free() → 自殺（≈ React v-if false）
- await signal → 暫停 function 等 signal（≈ JS await Promise，但只能等 signal）
- get_tree().create_timer(1.0).timeout → 一次性 delay signal（短 delay 不用建 Timer node）
- randf() / randf_range() → 隨機數
- Vector2(x, 0).rotated(angle) → 向量旋轉（mob spawn 方向計算）
- $Path/To/Node → 取得子節點（≈ get_node()）
- position.clamp(min, max) → 夾範圍
- velocity.normalized() * speed → 斜走不超速

設計 pattern：
- 場景 = 可重用組件（.tscn ≈ Vue .vue）
- Node 各司其職：Area2D 偵測 / Sprite 顯示 / Collision 形狀分開（組合優於繼承）
- Signal up + method down（≈ React/Vue events up + props down）
  · Parent → Child：直接 $Child.method() call（parent 握有 child 引用）
  · Child → Parent：emit signal 廣播（child 不該知道 parent）
- @export 注入而非 preload 寫死（設計師可在 Inspector 換）
- 自訂 signal 完整週期：signal 宣告 → emit() → connect

踩雷：
- mod vs mob typo（檔名打錯，code 找不到 node）
- MapSpawnLocation vs MobSpawnLocation typo（運行時 null error）
- Player root 不該有變形（scene 實例化會被 parent 覆蓋），scale 要設在 AnimatedSprite2D
- := 型別推斷對 $Node.prop 失效（要明寫 type 或用 = 改 normal 指派）
- 沒設 input action 方向鍵無效（要在專案設定 → 輸入對應）
- F5 跑主場景 vs F6 跑當前場景（多場景專案要切清楚）
- hide() 沒取消註解 standalone 測試看不到 player（建 Main 後恢復）
- Path2D 沒封閉（要點 5 個點回起點才會圍成矩形）
- @export var mob_scene 必須拖 mob.tscn 到 Inspector，不然 instantiate null error

Part 7 補完（同日下午）：
- 加 ColorRect 當背景（暗色，撐滿錨點 → 整個矩形）
- 加 AudioStreamPlayer Music（House In a Forest Loop.ogg，匯入時勾 Loop 重新匯入）
- 加 AudioStreamPlayer DeathSound（gameover.wav）
- main.gd 在 new_game 加 $Music.play()，game_over 加 $Music.stop() + $DeathSound.play()
- 鍵盤啟動：hud.gd 加 _unhandled_input → 按 ui_accept (Enter/Space) emit start_game
- 學到：Godot 內建 ui_* action（ui_accept/cancel/select/left/right/up/down/text_submit）
  不要重綁，自訂操作另開名字（如 move_left）

W2 完整里程碑：能跑、能玩、有音樂、有 UI、可重複開始
GitHub repo：asd23353934/dodge-the-creeps（已 push）

下一步：W3 Custom Resource (.tres) + Autoload
```

---

### W3：Custom Resource (.tres) + Autoload

- 目標時數：10-15 hr
- [~] 看 Resources 文件（邊做邊查，沒系統性讀完）
- [x] 寫 `CardData.gd`（extends Resource）+ 建 3 張測試卡 .tres
- [x] 寫 `GameState.gd` autoload（player_hp / energy + run state + take_damage / heal / spend_energy methods）
- [x] 寫 `EventBus.gd` autoload（11 個 signal：戰鬥 / 玩家狀態 / 遊戲流程）
- [ ] 週末 retro

週記：

```
Day 2（同 2026-05-23，下午接 W3，新專案 card-resource-demo）：

完成範圍：
- 建專案 card-resource-demo（純資料 + 邏輯地基，無遊戲畫面）
- cards/ 資料夾：card_data.gd（extends Resource）+ strike/defend/focus.tres
- autoload/ 資料夾：game_state.gd（狀態 + 操作 API）+ event_bus.gd（11 個 signal）
- main.gd 驗證：讀 .tres + 訂閱 EventBus + 操作 GameState 觸發連鎖反應

Custom Resource 觀念：
- class_name X extends Resource → 可序列化的資料 class
- @export 欄位 → Inspector 直接編輯
- @export_multiline → 多行文字框（描述用）
- .tres 是文字格式可版控、可在 Inspector 編
- 對應前端：JSON 設定檔 + schema + 內建 GUI 編輯器
- 拖到 @export var x: Array[CardData] 欄位 → main scene 持有資料

Autoload（singleton）觀念：
- 註冊在 專案 → 設定 → Autoload → 給 node 名稱（如 GameState）
- 啟動時自動 instance 到場景樹最頂 / 全 scene 都能用 / 永遠存在
- 任何地方寫 GameState.xxx 直接訪問，不用 import / get_node
- 對應前端：Vue Pinia / Angular root service / React Zustand
- autoload _ready 比所有 scene 早跑

EventBus pattern：
- 一個專門裝 signal 的 autoload，emit / connect 都在這
- A 不用認識 B / B 不用認識 A，靠 EventBus 中轉
- N 個訂閱者一起接（emit 一次全跑）
- 解耦遠距 scene 間通訊（如戰鬥 → UI / 音效 / 成就系統）
- 對應前端：Redux dispatch / RxJS Subject / EventEmitter

設計原則（autoload 不要濫用）：
- 適合：全域狀態（hp/gold/save）、跨場景資源（音樂）、事件中心
- 不適合：單一 scene 內部狀態、相鄰 node 通訊（用 signal）
- 過度用 autoload → 全 scene 都依賴它 → 難測試 / 難重構

GameState 也 emit signal（重要 pattern）：
- take_damage 改完 player_hp 後 → 自動 emit EventBus.damage_dealt + player_hp_changed
- GameState 只管「改狀態 + 廣播」，完全不管誰會反應
- UI 訂閱 player_hp_changed 自動更新血條，戰鬥邏輯訂閱 damage_dealt 播飄字
- 這是 DFS 戰鬥系統的核心模式

踩雷：
- Resource 必須 extends Resource（不是 RefCounted 也不是 Node）
- autoload 必須是 Node 或子類（不是 Resource）
- @export var x = "default" 預設值在 .tres 第一次建立才用
- Array[CardData] 拖檔到 Inspector，每個元素獨立拖入

W3.5 補強（2026-05-26）：
- 自寫 test_runner：card-resource-demo 加 test/ 資料夾，9 個單元測試
- 起因：原本想用 GdUnit4 framework，發現 AssetLib 最新 v6.0.0 跟 Godot 4.6.3 API drift
  · FileAccess.get_as_text() 4.6 改為 0 參數（舊版可傳 bool）
  · plugin parse error → 連鎖 10 個 compile fail → 整 project 跑不起來
- Pivot：純 GDScript test harness（無 framework dependency）
  · 9 個測試覆蓋 take_damage / heal / spend_energy / reset_for_new_run
  · 含 signal emit 驗證（damage_dealt / player_died）
  · regression test 防 848f6cf 的 dup-body bug 復發
  · `_setup()` reset state + clear log，每個 test 隔離
- 副產出：event_bus.gd 加 @warning_ignore_start("unused_signal")（EventBus pattern 標準誤判）
- 學習：plugin 不能信「Godot 4」標籤，要看 minor 版本；AssetLib 常滯後 GitHub
- 面試 prep：interview-prep.md 加 Q16 talking point

下一步：W4 Deckbuilder tutorial（把所有 W1-W3 知識整合做卡牌戰鬥 prototype）
```

---

### W4：第 2 個 tutorial + 卡牌 prototype

- 目標時數：10-15 hr
- [s] 跟一個 YouTube「Godot 4 Deckbuilder Tutorial」← 跳過，從 W1-W3 pattern 直接整合
- [x] 試做：手牌顯示 + 拖卡到敵人 + 扣 HP（**完成**）
- [ ] 月底 retro：M1 哪裡超時 / 哪裡輕鬆

週記：

```
Day 5（2026-05-26）W4 開工 — deckbuilder-prototype：

決策：不照 YouTube tutorial，直接從 W1-W3 累積 pattern 整合
- 理由 1：W1-W3 已建立 CardData (Resource) + GameState (autoload) + EventBus
- 理由 2：跟 tutorial 容易抄成「會用 framework 但不懂為什麼」
- 理由 3：W4 主要新東西是 UI 排版 + drag-and-drop，這個比較好查 Godot 官方文件

完成範圍（Day 5）：
- 建專案 deckbuilder-prototype，PowerShell 複製 W3 的 autoload/cards/test 全套
- 加 2 張新卡（HEAVY cost 2/dmg 10 + QUICK cost 0/dmg 3），共 5 張覆蓋設計光譜
- 註冊 autoload，踩到 autoload 順序 bug，已寫進 hsin-dev-notes/godot/errors.md
- README 含 WIP 進度表，提交 GitHub
- 寫 Card.tscn 視覺場景（PanelContainer + MarginContainer + VBox + Labels）

踩雷紀錄：
- autoload 順序錯：GameState 先、EventBus 後 → 30+ Parse Error
  · Godot 4 parser 嚴格，autoload 之間 forward reference 不行
  · 規則：被依賴的（EventBus）擺前面、依賴的（GameState）擺後面
  · 寫進 errors.md 給未來查

下一步（下個 session）：
- card.gd 寫 setup_with_data(data: CardData) method
- Hand.tscn 容器 + 動態 instantiate Card × 5
- Enemy.tscn + ProgressBar HP bar
- Drag-and-drop 三大 method：_get_drag_data / _can_drop_data / _drop_data
- Main scene 組合
- 補 Enemy.take_damage 測試（複用 W3.5 test_runner pattern）

Day 6（2026-05-27）W4 全部完成：

完成範圍：
- Card.tscn / hand.tscn / enemy.tscn / main.tscn 全部寫完跑通
- Drag-and-drop 完整 work：拖卡到敵人 → 扣能量 + 扣 HP + 卡 queue_free
- 能量不足無法拖（_get_drag_data 早期 return null）
- 史萊姆 HP 歸零 → main.gd 顯示 VICTORY
- Enemy 加 4 個測試（take_damage 邊界 + signal + dead ignores）
- 共 13/13 tests 全綠（GameState 9 + Enemy 4）

W4 新觀念（補進 interview-prep 對應 Q）：
- PanelContainer / MarginContainer / VBoxContainer / HBoxContainer 排版 hierarchy
- ProgressBar 視覺化數值
- unique_name_in_owner + %NodeName 全域引用語法
- @export var x: Type 的 setter 模式（Inspector 改值即時反映）
- PackedScene 動態 instantiate × N 個（Hand 內部）
- Godot 內建 drag-and-drop 三大 method
  · _get_drag_data：開始拖時 call，return null 取消 / 設 drag preview
  · _can_drop_data：拖到上面 call，bool 決定能不能放
  · _drop_data：放開 call，真正處理
- drag preview 用 PackedScene.instantiate 而非 duplicate（owner / unique_name 衝突）
- class_name X 讓 type check 簡潔（data is Card）

設計討論（要寫進 interview-prep）：
- 拖拉 vs 點選+目標：兩派都業界主流，DFS 用拖拉因為 Slay the Spire 是
- 業界測試現況：遊戲業 coverage 一般低於後端，junior 寫測試是「異常加分」
- 自寫 test runner vs framework：pattern 一樣，產品階段才換 framework

踩雷紀錄（無，這次很順）：
- 主要修一個 Edit 失敗（event_bus.gd 沒先 Read），無影響

M1 完成（W1-W4 全部 ✅）：
- W1：Godot 安裝 + GDScript 100 行
- W2：dodge-the-creeps 完整可玩
- W3：Custom Resource + Autoload + 9 個測試
- W3.5：test harness pivot（GdUnit4 v6.0 incompat）
- W4：Deckbuilder prototype + 13 個測試

GitHub repos（5 個全部 push）：
- godot-learning-path
- dodge-the-creeps
- card-resource-demo
- deckbuilder-prototype（新加 5 個檔 + README + screenshot）
- hsin-dev-notes

下一站：W5 進 M2 DFS Phase 0 setup（建主專案 dice-fate-survivor 結構）
```

**M1 完成驗收**：能獨立寫出簡單 2D 遊戲。

---

## M2（W5-W8）：DFS Phase 0-2

### W5：Phase 0 setup

- 目標時數：10-15 hr
- [x] 建 DFS 專案結構（5 個 autoload + scripts/scenes/data/assets/tests 資料夾骨架）
- [x] 設 Godot project settings（1280x720 / canvas_items / keep / debug warnings）
- [x] Git repo + commit pattern 開始（DFS repo 已 push 過 docs，現在加 Godot project）

週記：

```
Day 7（2026-05-28）M2 開工 — DFS Phase 0：

完成範圍：
- 把 dice-fate-survivor 從 Desktop 搬到 Desktop/godot-projects/（5 個 project 並列管理）
- 在 DFS repo 內建 Godot project（注意 docs/ + SPEC.md 已存在，不勾「版本控制中繼資料」避免覆蓋）
- 建 5 個 autoload（按 DFS architecture.md 順序註冊）：
  · SettingsManager（stub，W17 補完）
  · EventBus（41 signal：玩家狀態 / 戰鬥 / 棋盤 / Hub / 流程 五層）
  · GameState（permanent + current_run 拆分，take_damage / heal / spend_energy / spend_gold
    / reset_for_new_run / reset_energy_for_new_wave / purchase upgrade / unlock_card）
  · AudioManager（stub，W21 補完）
  · SceneRouter（stub，W17 補完）
- 設 project.godot：1280x720 / canvas_items / keep / exclude_addons warning / Forward+
- 建資料夾骨架：scripts/ data/ scenes/ assets/ tests/（用 .gitkeep 讓 git 追蹤空資料夾）
- 寫 scripts/main.gd + main.tscn bootstrap 場景：F5 印 5 autoload ready 訊息
- F5 跑通：41 signals / HP 80/80 / 全 stub OK

W5 比 W3 W4 多學到：
- 大 project 的 autoload 依賴順序（11 個 spec / 先做 5 個）
- 在「已有 docs 的 repo」內建 Godot project 的技巧（不勾版本控制中繼資料）
- .gitkeep 維持空資料夾結構於 git
- Permanent vs Current Run state 拆分（DFS 跨 run 升級系統的關鍵設計）
- EventBus 多層 signal（41 個）統一管理 vs 散落各 scene

新觀念對應 interview-prep（W5 talking point）：
- Q18：DFS 11 autoload 怎麼設計？順序為何？依賴關係？
- Q19：Permanent vs Current Run state 分開好處？

下一步（W6 起）：
- W6 Phase 1：walking skeleton（主選單 → 棋盤 → 戰鬥 → 結算 → 主選單 全空殼）
- 用 SceneRouter 串場景切換
- 用 EventBus 串跨場景狀態傳遞
```

---

### W6：Phase 1 walking skeleton

- 目標時數：10-15 hr
- [x] 主選單 → Hub → Board → Combat → 假勝利/失敗 → 回主選單（全 placeholder）
- [x] 場景切換 + 全局 state 串起來（SceneRouter fade + autoload 跨場景持續）

週記：

```
Day 7（2026-05-28）M2 加速 — W6 Phase 1 walking skeleton：

完成範圍：
- 升級 SceneRouter：從 stub 變真實作
  · 在 autoload 自己 _ready 時建 CanvasLayer + ColorRect 全螢幕黑屏
  · CanvasLayer.layer = 100 → 永遠在最上層蓋場景
  · change_scene(path, duration=0.3)：fade out 0→1 → 切場景 → fade in 1→0
  · mouse_filter=IGNORE 不擋滑鼠
- 建主選單 main_menu.tscn：DiceFateSurvivor 標題 + 4 按鈕（開始/設定/圖鑑/離開）
- 建 3 個 placeholder：scenes/hub/hub.tscn、scenes/board/board.tscn、scenes/combat/combat.tscn
- 每個 placeholder 都有 navigation button 串起 flow
- combat 多了「假勝利」「假失敗」按鈕（測試 EventBus.combat_ended / run_ended）
- project.godot main_scene 從 main.tscn 改成 scenes/main_menu.tscn
- 刪掉 W5 的 bootstrap main.tscn + scripts/main.gd（被取代）

W6 新概念：
- CanvasLayer + layer 數字決定渲染層級（autoload 持有的 CanvasLayer 跨場景永存）
- Tween 用法：create_tween() → tween_property() → await tween.finished
- await get_tree().process_frame：等下一 frame（讓 _ready 完成）
- get_tree().change_scene_to_file(path) 切場景 API
- set_anchors_preset(Control.PRESET_FULL_RECT) 程式設定 anchor
- 整套 game flow 串接：玩家可以走完整一輪雖無真內容

驗收：F5 跑通完整 flow（主選單 → Hub → Board → Combat → 假勝利 → Board → ... → 回主選單），fade 流暢，autoload 跨場景持續 ready。

下一步（W7 起）：Phase 2 戰鬥 prototype
- 把 W4 deckbuilder-prototype 的 Card / Hand / Enemy 移植進 DFS combat scene
- 玩家 + 1 敵人戰鬥
- 5 張 hardcode 卡 → 拖卡攻擊（重用 W4 drag-drop）
- 之後 W9-W11 加 .tres 資料化 / status / Combo Lock
```

---

### W7：Phase 2 戰鬥 prototype (上)

- 目標時數：10-15 hr
- [x] 戰鬥場景：玩家 + 1 敵人（史萊姆 HP 15 demo）
- [x] 手牌 UI（hardcode 5 張卡 — STRIKE/DEFEND/FOCUS/HEAVY/QUICK）
- [x] 拖卡釋放 → 扣敵人 HP（移植 W4 drag-drop）

週記：

```
Day 7（2026-05-28）M2 加速 — W7 Phase 2 上半：

完成範圍（同一個 session 內 W5+W6+W7）：
- 把 W4 deckbuilder-prototype 完整移植進 DFS
- CardData class 移到 scripts/data/card_data.gd
- 5 張 .tres 移到 data/cards/（用 script_class="CardData" 鏈結）
- Card / Hand / Enemy script 移到 scripts/combat/
- Card / Hand / Enemy scene 移到 scenes/combat/
- combat.tscn 從 placeholder 升級成真戰鬥（背景 + 能量 label + 1 隻 enemy + 5 張手牌）
- combat.gd orchestrate：進場 reset 能量 + 訂閱 enemy.died + 勝利 1.5 秒切回 board
- enemy.gd 加 EventBus.damage_dealt.emit（W4 沒有，DFS 需要 global 訂閱）

W7 新觀念：
- code 移植 / 重組的工程實踐（不重寫，調 path + 加 EventBus）
- local + global signal 並存 pattern（hp_changed local + damage_dealt global）
- .tres 用 script_class 鏈結 .gd（跨 repo 也能 work）
- combat scene 用 instance=ExtResource 嵌入 sub-scene（編輯器可見 + immediate 顯示）
- Inspector Array[CardData] 拖檔配置（不寫 code 就能設 5 張卡）

設計權衡：
- enemy HP 從 30 降到 15（demo 用），W8 加抽牌後改回
- DEFEND / FOCUS 在 W7 dmg=0 placeholder，W8/W11 補 shield/buff 機制

下一步（W8 起）：Phase 2 下半
- 加 end_turn 按鈕 + 敵人回合
- 加抽牌系統（GameState.deck / hand / discard）
- DEFEND 真的給 shield、FOCUS 真的加 buff（基礎）
- 戰鬥失敗處理（玩家 HP=0 → main_menu）
- 多場戰鬥（5 波結構雛形）

implementation guide 補完：docs/07-implementation/W7-phase-2-combat-prototype-upper.md
（10 段：目標 / 設計決策 / 檔案 / 流程 / code 解析 / Vue 對照 /
擴充方式 / 常見錯誤 / 下游依賴 / 面試 talking point）
```

---

### W8：Phase 2 戰鬥 prototype (下)

- 目標時數：10-15 hr
- [x] 敵人回合（每回合 attack 4，take_damage 吸 shield）
- [~] 5 波結構基礎（保留 W9 / Phase 3 做，W8 只做單場戰鬥）
- [ ] 月底 retro

週記：

```
Day 7（2026-05-28）M2 完成 — W8 Phase 2 下半：

完成範圍（同 Day 7 內 W5+W6+W7+W8 全做完）：
- CardData 加 shield 欄位；DEFEND.tres shield=5
- GameState 大改：deck/hand/discard cycle、shield、抽棄牌堆、take_damage 先扣 shield
- EventBus 加 3 signal：shield_changed / hand_changed / deck_state_changed
- Hand 重構為 reactive：訂閱 hand_changed + 從 GameState.hand 讀（single source of truth）
- Enemy 加 attack_damage + enemy_turn_attack；drop 處理 damage / shield 兩種卡
- enemy HP 改回 30、attack_damage = 4
- combat.tscn 加 EndTurnButton / ShieldLabel / PlayerHPLabel / DeckCountLabel / DiscardCountLabel
- combat.gd turn loop orchestration：
  · _ready 訂閱 + start_combat_with_deck + draw_card(5)
  · End Turn：棄手牌 → 敵人攻擊 → 抽 5 + 能量回滿
  · 玩家死 / 敵人死 → 對應結局 → 1.5 秒切場景

W8 新觀念：
- Hand reactive pattern（Vue 響應式 / React state 對應）
- single source of truth：GameState.hand 是唯一真實狀態
- shield + HP 雙層 take_damage：封裝於 GameState（鬆耦合）
- deck cycle：抽空自動洗 discard 回去
- Slay the Spire 規則：每回合棄整手 + 抽 5 新
- pop_back() vs shift()：stack 模式 O(1) 效率

設計討論（寫進 interview-prep）：
- Q：DEFEND 拖到敵人 UX 怪不怪？
  · 怪，W8 範圍可接受（功能 work > UX 完美）
  · W11 加 card_type enum + click 自觸發，正式 refactor
- Q：為什麼 Hand 整把 rebuild 不漸進 diff？
  · 5 張小 → rebuild < 1ms，比 diff 邏輯複雜性簡單
  · 規模上去再優化

M2 整體完成（W5-W8 全部）：
- W5：5 autoload + 結構（Phase 0）
- W6：SceneRouter fade + 4 場景（Phase 1 walking skeleton）
- W7：戰鬥 prototype 上半（移植 W4 拖卡攻擊）
- W8：戰鬥 prototype 下半（抽棄牌 + shield + turn loop + 勝負）

下一步 W9 進 M3：Phase 3 戰鬥成熟
- 加 5 波結構（Step 2.8）
- Status effect 6 種（poison / weak / vulnerable / strength / dex / shield）
- Combo Lock 招牌機制
- 卡牌資料化 .tres 池（hardcode → loader）
- GdUnit4 / 自寫 harness 補戰鬥公式測試
```

---

## M3（W9-W13）：DFS Phase 3-4

### W9：Phase 3 戰鬥成熟 (1/3) — 卡牌資料化

- 目標時數：10-15 hr
- [ ] hardcode 卡改成從 .tres 讀
- [ ] 卡牌池基本建立

週記：

```

```

---

### W10：Phase 3 戰鬥成熟 (2/3) — Status + Power

- 目標時數：10-15 hr
- [ ] 6 種 status（poison / weak / vulnerable / strength / dex / shield）
- [ ] Power 卡實作

週記：

```

```

---

### W11：Phase 3 戰鬥成熟 (3/3) — Combo Lock + 測試

- 目標時數：10-15 hr
- [ ] Combo Lock universal 機制（招牌）
- [ ] GdUnit4 寫 5-10 個 combat-formulas 測試

週記：

```

```

---

### W12：Phase 4 棋盤 prototype (1/2)

- 目標時數：10-15 hr
- [ ] 簡化版棋盤（20 格 closed loop）
- [ ] 5 種格子（gold / item / event / combat / shop）
- [ ] 玩家擲骰移動

週記：

```

```

---

### W13：Phase 4 棋盤 prototype (2/2)

- 目標時數：10-15 hr
- [ ] 棋盤敵人移動（簡單 random AI）
- [ ] Path encounter（移動經過觸發戰鬥）
- [ ] 月底 retro

週記：

```

```

---

## M4（W14-W17）：DFS Phase 5-6 整合 + Hub

### W14：Phase 5 整合 (1/2)

- 目標時數：10-15 hr
- [ ] 完整 1 個 run loop：選 kit → 進棋盤 → 戰鬥 → 結算 → 再來
- [ ] 三選一獎勵 UI

週記：

```

```

---

### W15：Phase 5 整合 (2/2) — Boss

- 目標時數：10-15 hr
- [ ] Boss 戰（DICE_LORD 簡化 2 phase）

週記：

```

```

---

### W16：Phase 6 Hub + VN

- 目標時數：10-15 hr
- [ ] Hub 場景（簡單 2D）+ 2 NPCs（訓練師 + 卡牌商）
- [ ] VN 風對話系統（dialog tree + 分支選項）

週記：

```

```

---

### W17：Phase 6 Save / Settings

- 目標時數：10-15 hr
- [ ] 永久升級（HP 2 階 / 能量 1 階）
- [ ] Save / load（XOR 加密 binary，1 slot）
- [ ] Settings menu（解析度 / 音量 / 語言）
- [ ] 月底 retro

週記：

```

```

---

## M5（W18-W22）：Phase 7-8 內容 + 美術

### W18：Phase 7 內容填充 (1/2)

- 目標時數：10-15 hr
- [ ] 補滿 10-15 張卡 .tres
- [ ] 補滿 4-5 敵人 + DICE_LORD

週記：

```

```

---

### W19：Phase 7 內容填充 (2/2)

- 目標時數：10-15 hr
- [ ] 補滿 5-6 道具
- [ ] 棋盤事件 3-5 種（VN 系統 + 分支）
- [ ] QTE 卡 1-2 張（致敬 WorkNite）

週記：

```

```

---

### W20：Phase 8 美術 (1/2) — Aria 立繪

- 目標時數：10-15 hr
- [ ] AI 生 5-8 張 Aria 立繪
- [ ] 10-15 張卡 illustration（AI + Krita 修）

週記：

```

```

---

### W21：Phase 8 美術 (2/2) — UI + 互動

- 目標時數：10-15 hr
- [ ] 卡牌邊框 + UI（Figma）
- [ ] AnimatedSprite2D + Tween + Area2D 互動角色
- [ ] BGM 3-5 首 + SFX 20-30 個

週記：

```

```

---

### W22：Phase 9 polish 開始

- 目標時數：10-15 hr
- [ ] 戰鬥 hit pause / screen shake / 飄字
- [ ] Tutorial 5 popup
- [ ] i18n 補完（繁中 + 英文）
- [ ] 月底 retro

週記：

```

```

---

## M6（W23-W26）：收尾 + Live2D + 履歷

### W23：Polish 收尾

- 目標時數：10-15 hr
- [ ] Bug fix（連跑 5 場 run 找 bug）
- [ ] 平衡（auto-play 100 場驗）
- [ ] 音樂 / SFX 最終整合

週記：

```

```

---

### W24：Web export + itch.io + trailer

- 目標時數：10-15 hr
- [ ] HTML5 export 測試 + 上 itch.io
- [ ] 60-90 秒 trailer（OBS 錄 + DaVinci / CapCut 剪）
- [ ] 截圖 5-8 張

週記：

```

```

---

### W25：Live2D 整合測試

- 目標時數：10-15 hr
- [ ] 下載 Live2D 官方免費模型
- [ ] 裝 GDCubism plugin
- [ ] 30 秒 demo（hover / 點擊 / lip sync）
- [ ] 開新 GitHub repo（小 + 獨立 + 1 README）

週記：

```

```

---

### W26：履歷 + 應徵

- 目標時數：10-15 hr
- [ ] 1111 / 104 / yourator 履歷 + 連結
- [ ] 履歷 PDF
- [ ] 內推（Twitter / Discord / FB 社團找）
- [ ] 投 WorkNite + 廣譜 10-20 家

週記：

```

```

**M6 完成驗收**：履歷投出 + 至少 1 個面試機會。

---

## 整體 retro 記錄

完成每個 M 後在這邊記 3-5 行整體心得：

### M1 retro
```

```

### M2 retro
```

```

### M3 retro
```

```

### M4 retro
```

```

### M5 retro
```

```

### M6 retro
```

```
