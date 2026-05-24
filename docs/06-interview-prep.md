# Godot Junior 面試準備（口語版）

> **目的**：把 W1-W3 學到的東西，練成面試**真的講得出口**的版本。
> **目標**：投 WorkNite / Rayark / 中型 indie / 外包公司 Godot junior 缺。
> **寫作日期**：2026-05-23（W2 同日，audit 完即寫）

---

## 怎麼用這份文件

1. **不要背字**：背我的字會聽起來像 robot。把每題答案用**你自己的話**重講一次。
2. **錄音念出來**：對著手機錄音念 1-2 次，自己聽是否自然。
3. **練到能即時應答**：理想狀態是面試官問題念完，3 秒內你能開口（停太久顯示不熟）。
4. **承認不會 > 瞎掰**：junior 不需要全會。「這個我沒做過，但我會這樣 approach...」是健康答案。
5. **每個答案帶你的實例**：「我 dodge-the-creeps 就是...」「我 W3 練 EventBus 那邊...」會比講通則加分。

---

## 0. 自我介紹（30 秒，必背）

```
您好，我叫 [名字]，本來是前端工程師，做 Vue 跟 Angular 大概一兩年。

從六個月前開始全力轉 Godot，每週投 10 到 20 小時。

學習路徑都有公開在 GitHub，三個 repo：
  一個是學習文件 godot-learning-path
  一個是 dodge-the-creeps 完整的 2D 遊戲
  一個是卡牌資料系統的架構 demo card-resource-demo

主要產出是一款叫 DiceFateSurvivor 的卡牌肉鴿 demo，
整合戰鬥、棋盤、VN 對話，已經上 itch.io 可以直接玩。

對你們的職位，特別是 VN 跟卡牌戰鬥的部分，
我覺得跟我的學習路徑很 match，所以想過來看看。
```

**節奏**：講到 itch.io 那邊停一下，看對方反應。**不要一口氣全噴**。

---

## 1. 必答的 15 題深度

### Q1：velocity 用 4 個 if，不用 `Input.get_vector()`？

**Source**：`dodge-the-creeps/player.gd`

```gdscript
var velocity := Vector2.ZERO
if Input.is_action_pressed("move_right"): velocity.x += 1
if Input.is_action_pressed("move_left"):  velocity.x -= 1
if Input.is_action_pressed("move_down"):  velocity.y += 1
if Input.is_action_pressed("move_up"):    velocity.y -= 1
```

**核心觀念**：兩種都能跑。Tutorial 用 4-if 是為了教學清楚，實務 prod code 用 `Input.get_vector` 更簡潔。

**口語答**：
> 「Tutorial 用 4 個 if 是為了教學，每步看得清楚。
>
> 實務上有更簡潔的 `Input.get_vector("move_left", "move_right", "move_up", "move_down")`，一行回傳 Vector2，**自動處理斜走的 normalize 跟手把搖桿的 deadzone**。
>
> 我之後寫 DFS 會用 get_vector，dodge-the-creeps 沒改是因為跟著教學走。」

**加分**：提到 get_vector 內建 deadzone 處理（手把搖桿小幅度漂移會被忽略）。

---

### Q2：`velocity.normalized() * speed` 拿掉 normalize 會怎樣？

**Source**：`dodge-the-creeps/player.gd`

```gdscript
if velocity.length() > 0:
    velocity = velocity.normalized() * speed
```

**核心觀念**：斜走 velocity = (1, 1) 長度 √2 ≈ 1.41，直走 (1, 0) 長度 1。不 normalize 會讓斜走快 41%。

**口語答**：
> 「斜走時 velocity = (1, 1)，長度是 √2，大概 1.41。直走是 (1, 0)，長度 1。
>
> 不 normalize 直接 * speed，**斜走會比直走快 41%**。玩家會用斜跑刷地圖，遊戲不平衡。
>
> normalize 把所有方向統一成單位向量（長度恰好 1），再 * speed 不管方向都一樣快。」

**加分**：提到「向量數學基本功，2D / 3D 都要」。

---

### Q3【核心，必背】：`position += velocity * delta` 為什麼乘 delta？

**Source**：`dodge-the-creeps/player.gd`

```gdscript
position += velocity * delta
```

**核心觀念**：delta = 距離上一 frame 過了幾秒。乘 delta = 「每**秒**移動 speed pixel」，不乘 = 「每**frame** 移動 speed pixel」，前者跨 fps 一致。

**口語答**：
> 「`delta` 是距離上一 frame 過了多少秒。`_process` 每 frame 都跑，但不同電腦 fps 不一樣，60 fps 一秒跑 60 次，30 fps 一秒跑 30 次。
>
> 如果寫 `position += velocity`，等於『**每 frame** 移動 speed pixel』，60 fps 電腦一秒移動 60×speed，30 fps 移動 30×speed，**跨機速度不一樣**。
>
> 乘 delta 之後變成『**每秒**移動 speed pixel』，跨電腦一致。
>
> 這是 game programming 101，所有 movement code 都要乘 delta。」

**注意**：`_physics_process(delta)` 的 delta 是固定值（預設 1/60），但 `_process(delta)` 隨真實 fps 變動，**兩個 delta 意義不同**。Movement 通常用 `_physics_process` 更穩。

---

### Q4【核心】：撞到敵人時 `set_deferred("disabled", true)`，為什麼不直接寫 `disabled = true`？

**Source**：`dodge-the-creeps/player.gd`

```gdscript
func _on_body_entered(_body: Node2D) -> void:
    hide()
    hit.emit()
    $CollisionShape2D.set_deferred("disabled", true)
```

**核心觀念**：`_on_body_entered` 是物理引擎在 physics step 中呼叫你的。直接改 CollisionShape 的屬性 = 在物理算到一半 mutate 它正在處理的東西，可能 crash 或行為怪。

**口語答**：
> 「`_on_body_entered` 不是我呼叫的，是 Godot 的**物理引擎在 physics step 中間**呼叫我的。
>
> 這時候 CollisionShape2D 還在被引擎用，**直接寫 `disabled = true` 等於『邊算邊改』**，可能 crash 或行為不可預期。
>
> `set_deferred` 把改寫**排到下一 idle frame**，等物理 step 跑完才動 shape，這樣才安全。
>
> Godot 官方文件有特別寫『Modifying object properties directly during physics callbacks can cause issues with the simulation state』，跟物理互動的 mutation 都要 deferred 或 `call_deferred`。」

**加分**：「`queue_free()` 內部就是 deferred 的，這就是為什麼一個 node 不會在它自己 callback 裡馬上死掉。」

---

### Q5：Player `_ready()` 裡 `hide()` 的設計意圖？

**Source**：`dodge-the-creeps/player.gd`

```gdscript
func _ready() -> void:
    screen_size = get_viewport_rect().size
    hide()
```

**核心觀念**：Player.tscn 是可重用組件，**自己不知道何時該出現**，由 parent（Main）的 `start()` 控制。

**口語答**：
> 「Player.tscn 是可重用組件，**它自己不知道何時該出現**，那是遊戲流程的事。
>
> `_ready` 裡 hide 是設『default: invisible』，由外面（Main）呼叫 `player.start(pos)` 才 show。
>
> 這樣同一個 Player.tscn 可以在主選單（看得到但不操作）、戰鬥（顯示 + 可控）、死亡畫面（隱藏）共用，**狀態由 parent 決定**。
>
> 違反這個設計，Player 自己決定 show/hide，邏輯散布，難維護。」

**對應前端**：Vue 子組件不該管自己的 v-if，那是 parent 的責任。

---

### Q6：為什麼用 VisibleOnScreenNotifier2D，不在 `_process` 裡判斷 position？

**Source**：`dodge-the-creeps/mob.gd`

```gdscript
func _on_visible_on_screen_notifier_2d_screen_exited() -> void:
    queue_free()
```

**核心觀念**：Notifier 是 Godot 內建偵測器，跟 viewport 系統整合，正確 + 效率好。

**口語答**：
> 「兩個都做得到，但 Notifier 是 Godot 內建的，**跟 viewport 系統綁好**，自動處理 camera 移動、邊界判斷。
>
> 我自己在 `_process` 比較座標可以做，但：
> 一、每 frame 跑，浪費 CPU
> 二、沒考慮 camera 移動，camera 滾動時邊界錯
> 三、邊界判斷自己寫容易漏角落
>
> 效能上 100 個 mob 各自 `_process` 比 100 個 Notifier 慢，因為 Notifier 是引擎內部 batched 處理。
>
> **內建的省事又正確**，不重新造輪子。」

---

### Q7：Mob spawn 那段 `+ PI / 2` 是什麼意思？

**Source**：`dodge-the-creeps/main.gd`

```gdscript
var direction = mob_spawn_location.rotation + PI / 2
direction += randf_range(-PI / 4, PI / 4)
```

**核心觀念**：PathFollow2D 的 `rotation` 是該點的**切線方向**（沿路徑走的方向）。切線 + 90° = 法線方向 = 朝路徑內側（如果路徑順時針畫）。

**口語答**：
> 「PathFollow2D 沿著 Path2D 走，它的 rotation 是『該點的**切線方向**』。比如路徑是矩形的上邊，切線是水平向右。
>
> 但我要 mob 朝**畫面內**飛，不是沿著邊跑。所以**切線轉 90 度 = 法線方向**，朝矩形內側。
>
> `PI/2` 是 90 度的弧度（GDScript 三角函數用弧度不是角度）。
>
> 老實說這段我畫圖確認過方向才理解。如果以後做不規則路徑 spawn，我會再 review 一次方向計算。」

**注意**：路徑畫順時針，+PI/2 朝內；逆時針要 -PI/2。**承認「我畫圖試出來的」誠實加分**。

---

### Q8：為什麼 `@export var mob_scene: PackedScene`，不直接 `preload("res://mob.tscn")`？

**Source**：`dodge-the-creeps/main.gd`

```gdscript
@export var mob_scene: PackedScene
```

**核心觀念**：@export 是 dependency injection，Inspector 拖檔，可在編輯器替換。preload 寫死路徑，要改要動 code。

**口語答**：
> 「preload 是把路徑寫死在 code 裡，要換 mob 種類要改 code。
>
> @export 是把 .tscn 變成『**可注入的設定**』，Inspector 拖檔指定。
>
> 好處：
> 一、設計師（我自己）不用碰 code 換敵人
> 二、同個 Main 場景可以複製多份配不同 mob，像關卡 1 用一般敵人、關卡 2 用 boss
> 三、單元測試時可以注入 mock mob
>
> **這是 dependency injection 概念，跟 Vue props 注入子組件一樣**。」

---

### Q9：為什麼用 3 個獨立 Timer，不合成 1 個？

**Source**：`dodge-the-creeps/main.tscn`

```
MobTimer   wait_time = 0.5
ScoreTimer wait_time = 1.0
StartTimer wait_time = 2.0, one_shot = true
```

**核心觀念**：Timer node 是 Godot 封裝好的計時器，wait_time + timeout signal，各管各的 callback，code 乾淨。

**口語答**：
> 「分三個是因為 Godot 的 Timer node 是封裝好的：設 wait_time + 接 timeout signal，**不用自己寫倒數邏輯**。
>
> 三個各管各的 signal handler，code 乾淨易讀。
>
> 合成一個的話，要在 timeout handler 裡用 if 或 state machine 分流，自己管多少時間到該做什麼，反而複雜。
>
> 而且 Timer 內建處理**暫停遊戲時自動 pause**，自己寫倒數還要處理這個。」

---

### Q10：`await $MessageTimer.timeout` 拿掉會怎樣？

**Source**：`dodge-the-creeps/hud.gd`

```gdscript
func show_game_over() -> void:
    show_message("Game Over")
    await $MessageTimer.timeout
    $MessageLabel.text = "Dodge the Creeps!"
    $MessageLabel.show()
    await get_tree().create_timer(1.0).timeout
    $StartButton.show()
```

**核心觀念**：await 暫停 function 等 signal。沒它的話三段操作會在同 frame 全跑完，玩家來不及讀。

**口語答**：
> 「await 是『**暫停 function 等 signal emit**』，類似 JS 的 await Promise。
>
> `show_game_over` 流程：先 show 'Game Over' → 等 2 秒 → 改成 'Dodge the Creeps!' → 等 1 秒 → 顯示 Start 按鈕。
>
> 如果拿掉 await，這 3 件事**同一 frame 全做完**，玩家根本看不到 'Game Over' 跟標題，瞬間就 Start 按鈕回來。
>
> **await 製造時序**，讓玩家有時間讀訊息。」

**加分**：「`get_tree().create_timer(1.0).timeout` 是一次性 delay signal，短暫等待不用建 Timer node，超方便。」

---

### Q11【核心架構】：HUD 為什麼用 signal start_game 通知 Main，不是 Main 直接抓 HUD？

**Source**：`dodge-the-creeps/hud.gd`

```gdscript
signal start_game

func _on_start_button_pressed() -> void:
    $StartButton.hide()
    start_game.emit()
```

**核心觀念**：HUD 用 signal 廣播，**不認識誰處理 = 變 reusable component**。直接 call 會綁死，挪不走。

**口語答**：
> 「HUD 用 emit signal 而不認識誰處理，是為了讓 HUD 變 reusable component。
>
> 如果 hud.gd 直接寫 `Main.new_game()` 或 `get_parent().new_game()`，HUD 就**綁死在 Main 上**。未來戰鬥場景或主選單想用同個 HUD 就要改 HUD 的 code。
>
> 改成 emit signal，**誰想接 start_game 就 connect，HUD 完全不認識訂閱者**。換 scene 也能重用。
>
> 這跟 Vue 的 `emit` / Angular 的 `@Output()` 一樣，**子組件廣播，parent 決定怎麼處理**。」

**設計原則**：「**Signal up, method down**」—— child 用 signal 向上通知，parent 直接 call child method 向下。

---

### Q12：CardData 為什麼 `extends Resource`，不是 `RefCounted`？

**Source**：`card-resource-demo/cards/card_data.gd`

```gdscript
class_name CardData
extends Resource
```

**核心觀念**：Resource 是 RefCounted 的子類，多三個能力：序列化、Inspector 編輯、可拖檔。

**口語答**：
> 「Resource 是 RefCounted 的子類，多三個能力：
>
> 一、可以**序列化成 .tres 檔**存硬碟，可以版控 git。
>
> 二、Inspector **GUI 編輯**，欄位看得到改得到。
>
> 三、可以拖到 `@export var x: CardData` 欄位當資料注入。
>
> RefCounted 只能 runtime `new` 出來用，沒辦法存檔。
>
> 80 張卡如果寫成 RefCounted，全部寫在 code 裡硬編碼。改成 Resource + .tres，**設計師改數值不碰 code**，這是商業遊戲標配。Slay the Spire / Hades / DFS 都這樣做。」

---

### Q13【核心】：Autoload 為什麼必須 `extends Node`？

**Source**：`card-resource-demo/autoload/game_state.gd`

```gdscript
extends Node
```

**核心觀念**：Autoload 啟動時自動 instantiate + add_child 到場景樹頂層。**只有 Node 系列能掛場景樹**。

**口語答**：
> 「Autoload 的機制是 Godot 啟動時**自動 `instantiate` 然後 `add_child` 到場景樹的 root**。
>
> 場景樹只能放 Node 跟它的子類（Node2D / Control / Area2D 等）。
>
> 寫 `extends Resource` 會錯，因為 Resource 是資料不是節點，**沒辦法 add_child**。
>
> Object 雖然是所有東西的祖先，但沒有 `_ready` `_process` 等生命週期，autoload 通常需要 `_ready` 做初始化。
>
> 我的 GameState 跟 EventBus 都 extends Node，因為 EventBus 需要 emit signal，**signal 機制是 Object/Node 的功能**。
>
> 官方文件原話：『You can create an Autoload to load a scene or a script that **inherits from Node**』。」

---

### Q14【核心 pattern】：GameState 修改狀態同時 emit EventBus 的設計

**Source**：`card-resource-demo/autoload/game_state.gd`

```gdscript
func take_damage(amount: int) -> void:
    player_hp = max(0, player_hp - amount)
    EventBus.damage_dealt.emit(amount, "Player")
    EventBus.player_hp_changed.emit(player_hp, max_hp)
```

**核心觀念**：State owner 修改自己後立刻廣播 = encapsulation + single source of truth + 保證通知不漏。

**口語答**：
> 「`take_damage` 同時做兩件事：**改 HP + emit signal**。
>
> 這樣 GameState 是 single source of truth，HP 怎麼算它一個地方說了算。
>
> 如果讓外面自己改 HP + 自己 emit signal，戰鬥邏輯、技能效果、debug 工具都會修改 GameState，**一定有人會漏寫 emit**，造成 UI 不更新的 bug。
>
> 集中在 GameState 內 emit，保證『**改了一定發通知**』。
>
> 這跟 Vuex / Redux 的 mutation pattern 一樣，state 的改動只能透過 mutation function，不能外面亂改。」

---

### Q15【bug 抓蟲】：`reset_for_new_run()` 有什麼問題？

**Source**：`card-resource-demo/autoload/game_state.gd`（修復前）

```gdscript
func reset_for_new_run() -> void:
    player_hp = max_hp
    energy = max_energy
    gold = 0
    current_floor = 1
    run_count += 1
    print(...)

    EventBus.run_started.emit(run_count)
    EventBus.player_hp_changed.emit(player_hp, max_hp)
    EventBus.energy_changed.emit(energy, max_energy)
    # ↓ 以下是重複貼上的 bug ↓
    player_hp = max_hp
    energy = max_energy
    gold = 0
    current_floor = 1
    run_count += 1
    print(...)
```

**口語答**：
> 「我自己 audit 時抓到的 bug：function body **重複貼了兩次**。
>
> 影響：
> 一、`run_count` 會 += 2 而不是 +1
> 二、EventBus 的 3 個 signal 都 emit 兩次，訂閱者跑兩次 → UI 重置兩次、音效播兩次、可能閃爍
> 三、`player_hp = max_hp` 兩次沒事，但其他邏輯重跑可能引發問題
>
> 已經修了，commit 在 GitHub 上可以看。
>
> **這個案例的教訓是寫完要自己讀過一次**，AI 工具寫 code 我也會發生這種狀況，要養成 review 自己 commit 的習慣。」

**這題拿來吹「我會 self code-review」是面試金句。**

---

## 2. AI 時代額外考點

### Q：「你平常會用什麼 AI 工具寫 code？怎麼用？」

> 「主要用 Claude 跟 GitHub Copilot。
>
> 我的習慣是**不會直接讓 AI 寫完整 feature**。先自己想清楚架構，比如要哪幾個 node、signal 怎麼接，**然後叫 AI 幫我寫骨架**。
>
> 然後逐行讀過，AI 常常會加我沒要求的東西，或用過時的 API，這些挑掉。
>
> 學東西的時候我會反過來：先讓 AI 解釋觀念，再去看官方文件對照，最後自己手寫一遍練熟。」

### Q：「AI 給你的 code 出錯怎麼辦？」

> 「最常遇到的是 AI 給的 Godot 4 code 其實是 Godot 3 寫法。比如：
>
> - `connect("pressed", self, "_on_pressed")` → 4 是 `pressed.connect(_on_pressed)`
> - `tool` → `@tool`
> - `KinematicBody2D` → `CharacterBody2D`
> - `yield(get_tree(), "idle_frame")` → `await get_tree().process_frame`
>
> 我的 debug 流程：
> 一、看 Godot 官方文件確認 API 版本
> 二、跑起來把 error 訊息丟回 AI 問
> 三、不行就讀文件 + 編輯器補全慢慢試」

### Q：「給你一段 AI 寫的 code，你會看什麼？」

> 「四件事：
>
> 一、API 版本對不對（Godot 4 vs 3）
> 二、有沒有 null 沒處理（`$Node` 在 _ready 前可能 null）
> 三、signal 連線有沒有清掉（scene free 後 dangling connection）
> 四、效能（`_process` 每 frame 跑重操作？）
>
> 然後問自己：『如果我沒用 AI，我會這樣寫嗎？』不會就改。」

---

## 3. 反問面試官的問題（必準備 2-3 題）

面試最後對方一定會問「你有什麼想問」，**沒問是 NG**。

### 推薦 3 題

**1. 技術 / 學習方向**

> 「想問貴公司目前用 Godot 開發比較常踩的雷有哪些？或者有什麼是我準備面試前可以多看的？」

→ 看對方答的內容知道公司在哪個段位 + 你之後可以加強什麼。

**2. Junior 起點 / 職涯**

> 「Junior 進來之後通常會先做什麼樣的任務？需要多久才會獨立負責一個模組？」

→ 評估學習曲線。

**3. 架構偏好**

> 「貴公司在卡牌系統 / VN 對話 / 戰鬥的架構上，比較傾向用 autoload 還是 component 多一點？」

→ 展示你有思考架構，不只看表面。

---

## 4. 紅線清單（避免說錯）

| 不要說 | 改說 |
|---|---|
| 「Unity 不好所以選 Godot」 | 「Godot 適合我的需求 / 開源生態」 |
| 「AI 幫我寫」 | 「我用 AI 加速架構，每段 code 我都能解釋為什麼」 |
| 「我什麼都會」 | 「這個我沒做過，但我會這樣 approach…」 |
| 「我會寫所有的東西」 | 「以我的程度，這部分我會先 prototype 看看，遇到瓶頸再學」 |
| 「我學了 6 個月所以很厲害」 | 「6 個月帶我從零到能 ship 一個 deckbuilder demo」 |

---

## 5. 心法 5 條

1. **每個答案都帶實例**：「我 dodge-the-creeps 就是…」「我 W3 練的是…」
2. **承認不會**：junior 不需要全會，「我會這樣 approach」很 OK
3. **節奏**：講 30-45 秒就停，看對方反應再深入
4. **問問題**：面試是雙向，多問代表你有興趣
5. **自我介紹練到反射**：開場 30 秒要流暢

---

## 6. 練習計畫

### 第 1 週：背 ❌ 那 4 題（Q3 / Q4 / Q11 / Q13）

每天念 3 遍，**錄音聽自己有沒有卡頓**。

### 第 2 週：模擬 1 小時面試

找朋友或對著手機，從自我介紹開始，**問 5-8 題隨機抽**。

### 第 3 週：再 audit 自己後續做的 code

W4 deckbuilder 做完後，重複這份文件的格式，audit 一次新 code。

### 持續：每次寫新 code

問自己「**這段我能用 30 秒解釋為什麼這樣寫嗎**」。能 = 過關。不能 = 還沒內化。

---

## 7. 進度對照

寫這份文件時的 audit 結果：

| 等級 | 數量 | 題號 |
|---|---|---|
| ✓ 能講清 | 2 | 5、15 |
| ❓ 大概懂 | 9 | 1, 2, 6, 8, 9, 10, 12, 14 |
| ❌ 完全沒 | 4 | 3, 4, 7, 11, 13 |

**下次 audit 目標**（W4 後）：

| 等級 | 目標 |
|---|---|
| ✓ | ≥ 10 |
| ❓ | ≤ 4 |
| ❌ | ≤ 1 |

---

## 附錄：常用 Godot 4 vs Godot 3 差異對照

AI 訓練資料 Godot 3 比 4 多，常踩雷：

| Godot 3 寫法 | Godot 4 正確寫法 |
|---|---|
| `connect("pressed", self, "_on_pressed")` | `pressed.connect(_on_pressed)` |
| `tool` (header) | `@tool` |
| `onready var x` | `@onready var x` |
| `export var x` | `@export var x` |
| `KinematicBody2D` | `CharacterBody2D` |
| `move_and_slide(velocity, Vector2.UP)` | `velocity = ...; move_and_slide()` |
| `yield(get_tree(), "idle_frame")` | `await get_tree().process_frame` |
| `Vector2(0, 0).rotated(PI)` | 同（這個沒變） |
| `print_debug(x)` | 同 |

面試官超愛丟 Godot 3 寫法測你，看你會不會挑出來。
