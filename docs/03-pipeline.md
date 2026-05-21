# Portfolio Pipeline：美術 + 互動角色 + VN + Live2D

> 6 月 portfolio MVP 的具體製作流程
> 配合：[`01-mvp-scope.md`](01-mvp-scope.md) scope / [`02-learning-path.md`](02-learning-path.md) 時程

---

## 1. 美術 pipeline（Slay the Spire 厚塗風）

### 風格選擇理由

| 風格 | 為什麼選 / 不選 |
|------|--------------|
| **Slay the Spire 厚塗 + 強描邊** ✅ | AI 友善 + 不用手繪 + 廣譜接受 + 與 deckbuilder genre 一致 |
| Pixel art | AI 友善但 WorkNite 不對齊 |
| Anime CG 立繪 | 需手繪 |
| Vector / flat | 不適合 deckbuilder 氛圍 |
| 水彩 / 油畫 | AI 處理難統一 |

### 統一色板

```
主色 1：深紫     #2D1B4E    panel / 背景
主色 2：金黃     #D4AF37    accent / 高光 / 標題
主色 3：暗紅     #8B1A1A    傷害 / 危險
輔色 1：白       #F5F5DC    文字
輔色 2：暗灰     #3A3A3A    陰影 / 細節
```

所有美術都用這 5 色為主，求**統一感 > 單張精美**。

### AI 生圖 prompt template

```
# 卡牌 illustration prompt template
masterpiece, best quality, highly detailed,
[CARD CONTENT 如 "warrior swinging sword" / "shield blocking arrow"],
thick painting style, strong ink outlines, limited color palette,
dark purple background, gold accents, dramatic lighting,
fantasy art, oil painting texture,
no text, no UI, no logo,
single subject, centered composition

# Negative prompt
photorealistic, blurry, text, logo, watermark,
multiple subjects, busy background, modern, anime
```

工具推薦順序：
1. **NovelAI** (`$10-25/月`) — 一致性好，速度快
2. **Stable Diffusion local + ControlNet** — 免費，可控
3. **Midjourney** (`$10/月`) — 美但風格固定

### Krita 後製流程（每張 30-60 分鐘）

```
1. AI 原圖 import 進 Krita
2. 套統一 action（必裝）：
   - 飽和 -8%（去 AI default 過飽和）
   - Curves 加深 shadows（增強對比）
   - 加 grain noise 1-2%（破 AI 平滑）
3. 修錯誤：
   - 手指 / 文字 / 邊緣亂線（5-10 分鐘）
4. 加手繪線稿描邊：
   - 用 ink brush 描重要輪廓 5-10 分鐘
   - 不需精細，看得出「手描感」就好
5. 統一色板套用（Filter → Color Adjustment → 限定 palette）
6. Export PNG 512x512
```

### 卡牌邊框（Figma 自做，不用 AI）

設計規範依 `docs/04-art/card-frame.md`：
- 256 × 384（2:3）
- 頂部：能量成本（左上）+ 類別 icon（右上）
- 中央：illustration 256 × 256
- 名稱列：白字 + 黑色描邊 24px bold
- 效果描述：14px regular
- 邊框依類別變色：
  - Attack: 暗紅 + 細金邊
  - Skill: 深藍 + 細金邊
  - Power: 深紫 + 細金邊
  - Item: 深綠 + 細金邊

Figma 一個 frame 設計完，其他卡 copy + 改 illustration 即可。

### 字型

- 繁中：**思源宋體 ExtraBold**（標題）+ **思源黑體 Regular**（內文）
- 英文：**Cinzel**（標題 fantasy）+ **Inter**（內文）
- 數字（傷害飄字）：**Bebas Neue**（窄體醒目）

全免費商用。

---

## 2. AnimatedSprite2D 互動角色（**重點 portfolio piece**）

### 需要的美術

AI 生 Aria 立繪 5-8 張（同一角色 + 同一風格，用 LoRA 確保一致）：

```
aria_idle_neutral.png      預設站姿，平靜
aria_idle_smile.png        微笑
aria_attack_pose.png       攻擊姿（握劍前傾）
aria_hurt.png              受傷喘氣
aria_victory.png           勝利舉拳
aria_thinking.png          思考（手摸下巴）
aria_blush.png             害羞（被摸頭時）
aria_battle_ready.png      戰鬥準備
```

每張 512 × 768（4:6 比例，立繪標準）。

### Godot 場景結構

```
Aria (Node2D)
├── Sprite (AnimatedSprite2D)         # 8 張立繪
├── HeadArea (Area2D)                 # 頭部點擊區
│   └── CollisionShape2D (CircleShape2D)
├── BodyArea (Area2D)                 # 身體點擊區
│   └── CollisionShape2D (RectangleShape2D)
├── FootArea (Area2D)                 # 腳部點擊區
│   └── CollisionShape2D (RectangleShape2D)
└── EmotionPopup (Node2D)             # 浮字效果
    └── Label
```

### 完整範例 code

```gdscript
extends Node2D
class_name AriaInteractive

@onready var sprite: AnimatedSprite2D = $Sprite
@onready var head_area: Area2D = $HeadArea
@onready var body_area: Area2D = $BodyArea
@onready var foot_area: Area2D = $FootArea
@onready var emotion_popup: Node2D = $EmotionPopup

var current_state: String = "idle_neutral"
var is_hover: bool = false
var breath_tween: Tween

func _ready():
    # 設預設 sprite
    sprite.play("idle_neutral")
    
    # 呼吸動畫（永遠 loop）
    start_breath_animation()
    
    # 點擊事件
    head_area.input_event.connect(_on_area_click.bind("head"))
    body_area.input_event.connect(_on_area_click.bind("body"))
    foot_area.input_event.connect(_on_area_click.bind("foot"))
    
    # Hover 事件
    body_area.mouse_entered.connect(_on_hover_enter)
    body_area.mouse_exited.connect(_on_hover_exit)
    
    # 監聽遊戲 state
    EventBus.player_hp_changed.connect(_on_hp_changed)
    EventBus.combat_won.connect(_on_combat_won)
    EventBus.combat_lost.connect(_on_combat_lost)
    EventBus.combo_tier_reached.connect(_on_combo_tier)
    EventBus.card_played.connect(_on_card_played)

func start_breath_animation():
    breath_tween = create_tween().set_loops()
    breath_tween.tween_property(sprite, "scale", Vector2(1.01, 0.99), 1.5)\
        .set_ease(Tween.EASE_IN_OUT)
    breath_tween.tween_property(sprite, "scale", Vector2(1.0, 1.0), 1.5)\
        .set_ease(Tween.EASE_IN_OUT)

func _on_area_click(_viewport, event, _shape, part: String):
    if not (event is InputEventMouseButton and event.pressed):
        return
    match part:
        "head":
            play_state("blush")
            show_emotion("不要摸頭啦~")
            await get_tree().create_timer(2.0).timeout
            play_state("idle_neutral")
        "body":
            show_emotion("...")
        "foot":
            play_state("hurt")
            show_emotion("好癢")
            await get_tree().create_timer(1.5).timeout
            play_state("idle_neutral")

func _on_hover_enter():
    is_hover = true
    var tween = create_tween()
    tween.tween_property(sprite, "scale", Vector2(1.05, 1.05), 0.2)

func _on_hover_exit():
    is_hover = false
    var tween = create_tween()
    tween.tween_property(sprite, "scale", Vector2(1.0, 1.0), 0.2)

func _on_hp_changed(hp: int, max_hp: int):
    var ratio = float(hp) / max_hp
    if ratio < 0.3 and current_state != "hurt":
        play_state("hurt")
    elif ratio > 0.7 and current_state == "hurt":
        play_state("idle_neutral")

func _on_combat_won():
    play_state("victory")

func _on_combat_lost():
    play_state("hurt")

func _on_combo_tier(tier: int):
    if tier >= 3:
        play_state("battle_ready")
        # 加粒子效果
        $ComboParticles.emitting = true

func _on_card_played(card: CardData):
    if card.card_type == "Attack":
        play_state("attack_pose")
        await get_tree().create_timer(0.5).timeout
        play_state("idle_neutral")

func play_state(state_name: String):
    if current_state == state_name:
        return
    current_state = state_name
    sprite.play(state_name)

func show_emotion(text: String):
    var label = emotion_popup.get_node("Label") as Label
    label.text = text
    var tween = create_tween()
    emotion_popup.modulate.a = 1.0
    emotion_popup.position.y = -50
    tween.parallel().tween_property(emotion_popup, "position:y", -100, 1.0)
    tween.parallel().tween_property(emotion_popup, "modulate:a", 0.0, 1.0)
```

工時：寫 + 測 = **15-25 hr**（含美術 import 整合）。

### 互動模式設計

| Trigger | 反應 | 用在 |
|---------|------|------|
| Hover Aria | 微微放大 1.05x + idle_smile | Hub + 戰鬥 |
| 點頭 | blush + 「不要摸頭啦」 | Hub easter egg |
| 點身體 | 「...」 | Hub easter egg |
| 點腳 | hurt + 「好癢」 | Hub easter egg |
| HP < 30% | 自動 hurt | 戰鬥中 |
| HP > 70% | 自動 idle_neutral | 戰鬥中 |
| Combo×3 | battle_ready + 粒子 | 戰鬥中 |
| 出 Attack 卡 | attack_pose 0.5s | 戰鬥中 |
| 勝利 | victory | 戰勝結算 |
| 失敗 | hurt | 戰敗結算 |
| 對話「喜怒哀樂」| 對應 sprite | VN 對話中 |

---

## 3. VN 對話系統（**前端強項展示**）

### Dialog tree 結構（JSON / .tres）

```json
{
  "dialog_id": "siler_first_meet",
  "nodes": [
    {
      "id": "start",
      "speaker": "席勒",
      "text": "一張卡的背後，是一段命運。要看看嗎？",
      "sprite": "idle_neutral",
      "next": "branch_1"
    },
    {
      "id": "branch_1",
      "type": "choice",
      "options": [
        {"text": "好啊。", "next": "show_cards", "effect": null},
        {"text": "不了。", "next": "decline", "effect": null},
        {"text": "妳是誰？", "next": "intro", "effect": "+1_lore"}
      ]
    },
    {
      "id": "show_cards",
      "speaker": "席勒",
      "text": "（攤開幾張卡）這幾張不錯。",
      "sprite": "smile",
      "next": "open_shop"
    },
    {
      "id": "intro",
      "speaker": "席勒",
      "text": "我是席勒。賣卡的。也賣記憶。",
      "sprite": "thinking",
      "condition": "player_first_meet",
      "next": "branch_2"
    }
  ]
}
```

### Godot Dialog Manager 範例

```gdscript
extends Control
class_name DialogManager

@onready var speaker_label: Label = $SpeakerLabel
@onready var text_label: RichTextLabel = $TextLabel
@onready var choice_container: VBoxContainer = $ChoiceContainer
@onready var sprite_anchor: Node2D = $SpriteAnchor

var current_dialog: Dictionary
var current_node_id: String

func start_dialog(dialog_id: String):
    var path = "res://data/dialogs/%s.json" % dialog_id
    var file = FileAccess.open(path, FileAccess.READ)
    current_dialog = JSON.parse_string(file.get_as_text())
    current_node_id = "start"
    show()
    show_node(current_node_id)

func show_node(node_id: String):
    var node = _find_node(node_id)
    if node == null:
        end_dialog()
        return
    
    if node.get("type") == "choice":
        _show_choices(node.options)
    else:
        speaker_label.text = node.speaker
        text_label.text = node.text
        _set_sprite(node.get("sprite", "idle_neutral"))
        
        # 自動 advance（如有 next）
        if node.has("next"):
            await _wait_for_click()
            show_node(node.next)

func _show_choices(options: Array):
    for child in choice_container.get_children():
        child.queue_free()
    for opt in options:
        var btn = Button.new()
        btn.text = opt.text
        btn.pressed.connect(_on_choice_selected.bind(opt))
        choice_container.add_child(btn)

func _on_choice_selected(option: Dictionary):
    if option.has("effect"):
        _apply_effect(option.effect)
    show_node(option.next)

func _set_sprite(sprite_name: String):
    var aria = sprite_anchor.get_node("Aria") as AriaInteractive
    aria.play_state(sprite_name)

func _wait_for_click():
    await Input.mouse_button_just_pressed[MOUSE_BUTTON_LEFT]

func _apply_effect(effect: String):
    match effect:
        "+1_lore":
            GameState.unlock_lore("siler_intro")
        "+10_gold":
            GameState.add_gold(10)

func end_dialog():
    hide()
    EventBus.dialog_ended.emit()

func _find_node(node_id: String) -> Dictionary:
    for n in current_dialog.nodes:
        if n.id == node_id:
            return n
    return {}
```

用在哪裡：
- Hub NPC 對話
- 棋盤事件格（3-5 種文字選擇題）
- 戰鬥前 boss 對白

工時：寫 + 測 + 3 個 dialog 範例 = **20-30 hr**

---

## 4. QTE mini-game（**致敬 WorkNite**）

### 設計

某些卡是 QTE 卡，出時觸發 mini-game：

```
玩家出 CARD_QTE_FOCUS（凝神專注）
   ↓
螢幕中央顯示：[← → ↑ ↓ SPACE] 隨機 3-5 個鍵
   ↓
玩家 5 秒內按完
   ↓
依正確率：
   - 100%: 該回合所有卡 +50% dmg
   - 80%: +30% dmg
   - 60%: +10% dmg
   - <60%: 無效果
```

### Godot QTE 範例

```gdscript
extends Control
class_name QTEMiniGame

const KEY_ICONS = {
    "left": "res://ui/key_left.png",
    "right": "res://ui/key_right.png",
    "up": "res://ui/key_up.png",
    "down": "res://ui/key_down.png",
    "space": "res://ui/key_space.png"
}

@onready var key_container: HBoxContainer = $KeyContainer
@onready var timer_bar: ProgressBar = $TimerBar
@onready var result_label: Label = $ResultLabel

var key_sequence: Array[String] = []
var current_index: int = 0
var correct_count: int = 0
var time_limit: float = 5.0
var time_elapsed: float = 0.0
var is_active: bool = false

signal qte_completed(success_rate: float)

func start_qte(num_keys: int = 4):
    key_sequence = _generate_random_keys(num_keys)
    current_index = 0
    correct_count = 0
    time_elapsed = 0.0
    is_active = true
    _render_keys()
    show()

func _generate_random_keys(n: int) -> Array[String]:
    var keys = ["left", "right", "up", "down", "space"]
    var result: Array[String] = []
    for i in n:
        result.append(keys.pick_random())
    return result

func _render_keys():
    for child in key_container.get_children():
        child.queue_free()
    for key in key_sequence:
        var icon = TextureRect.new()
        icon.texture = load(KEY_ICONS[key])
        key_container.add_child(icon)

func _process(delta):
    if not is_active:
        return
    time_elapsed += delta
    timer_bar.value = (time_limit - time_elapsed) / time_limit * 100
    if time_elapsed >= time_limit:
        _end_qte()

func _input(event):
    if not is_active:
        return
    if not event is InputEventKey or not event.pressed:
        return
    var pressed_key = _get_key_name(event)
    if pressed_key == "":
        return
    
    var expected = key_sequence[current_index]
    if pressed_key == expected:
        correct_count += 1
        _highlight_correct(current_index)
    else:
        _highlight_wrong(current_index)
    
    current_index += 1
    if current_index >= key_sequence.size():
        _end_qte()

func _get_key_name(event: InputEventKey) -> String:
    match event.keycode:
        KEY_LEFT: return "left"
        KEY_RIGHT: return "right"
        KEY_UP: return "up"
        KEY_DOWN: return "down"
        KEY_SPACE: return "space"
    return ""

func _highlight_correct(idx: int):
    var icon = key_container.get_child(idx) as TextureRect
    var tween = create_tween()
    tween.tween_property(icon, "modulate", Color.GREEN, 0.1)

func _highlight_wrong(idx: int):
    var icon = key_container.get_child(idx) as TextureRect
    var tween = create_tween()
    tween.tween_property(icon, "modulate", Color.RED, 0.1)

func _end_qte():
    is_active = false
    var success_rate = float(correct_count) / key_sequence.size()
    var bonus = _get_bonus(success_rate)
    result_label.text = "正確率 %d%% → +%d%% dmg" % [success_rate * 100, bonus * 100]
    await get_tree().create_timer(1.5).timeout
    qte_completed.emit(success_rate)
    hide()

func _get_bonus(rate: float) -> float:
    if rate >= 1.0: return 0.5
    elif rate >= 0.8: return 0.3
    elif rate >= 0.6: return 0.1
    return 0.0
```

工時：寫 + 測 + 1-2 QTE 卡整合 = **10-15 hr**

履歷話術：「實作 5 鍵 QTE mini-game，致敬 WorkNite Ride Me Taxi Driver 玩法。可調難度。」

---

## 5. Live2D 整合測試（**補 WorkNite 履歷 gap**）

### 為什麼做這個

WorkNite 是 VN 工作室，Live2D 是標配。即使 demo 用 sprite swap，**面試你要能說「我會 Live2D 整合」**。

不做完整 Live2D 立繪（30-50 hr），改做**整合測試**（10-15 hr）。

### Steps

1. **下載 Live2D 官方免費模型**
   - [Cubism Sample Data](https://www.live2d.com/zh-CHT/learn/sample/)
   - 推薦：Hiyori 或 Mark（complete sample）

2. **裝 Godot Live2D plugin**
   - 選項 1：[GDCubism](https://github.com/MizunagiKB/gd_cubism) — 開源
   - 選項 2：Cubism SDK for Godot（官方，較新）

3. **import 模型進 Godot**
   - .moc3 / .model3.json / texture
   - 設 CubismModel 節點

4. **寫 GDScript 控制**
   ```gdscript
   extends Node2D
   
   @onready var model: GDCubismUserModel = $Model
   
   func _process(_delta):
       # 眼睛追游標
       var mouse = get_global_mouse_position()
       var dir = mouse - global_position
       model.set_parameter_value("ParamAngleX", dir.x * 0.01)
       model.set_parameter_value("ParamAngleY", -dir.y * 0.01)
   
   func _input(event):
       if event is InputEventMouseButton and event.pressed:
           # 點擊 → 嘴巴開合
           var tween = create_tween()
           tween.tween_method(_set_mouth_open, 0.0, 1.0, 0.3)
           tween.tween_method(_set_mouth_open, 1.0, 0.0, 0.3)
   
   func _set_mouth_open(value: float):
       model.set_parameter_value("ParamMouthOpenY", value)
   
   func play_expression(expr_name: String):
       # 切換表情組
       model.start_motion("Expression", expr_name)
   ```

5. **錄 30 秒 demo + 寫 GitHub README**

### Repo 結構

```
godot-live2d-integration-test/
├── README.md            # 為什麼做這個 + 學到什麼
├── project.godot
├── addons/
│   └── gd_cubism/       # plugin
├── models/
│   └── hiyori/          # 免費模型
├── scenes/
│   └── main.tscn
├── scripts/
│   └── live2d_controller.gd
└── demo.gif             # 30 秒 GIF（履歷用）
```

工時：**10-15 hr 全部完成**。

履歷加分話術：
> 「Live2D Cubism 整合：用 GDCubism plugin 載入官方模型，實作 mouse-tracking eye / click-to-blink / expression switching / lip sync。GitHub: [link]。本 portfolio 因 scope 用 AnimatedSprite2D，架構設計成可無痛切換 Live2D。」

---

## 6. 工具清單 + 學習順序

### 第一個月一定要裝

| 工具 | 用途 | 學多久 |
|------|------|------|
| Godot 4.6.2 Standard | 主引擎 | 1 個月 hands-on |
| VSCode + Godot Tools | 程式編輯 + LSP | 1 天 |
| Git + GitHub Desktop | 版控 | 你已會 |
| Krita | 美術修圖 + AI 後製 | 1 週 |
| Figma | UI 設計 | 你應該已會 |

### 第二個月加裝

| 工具 | 用途 |
|------|------|
| NovelAI / SD local | AI 美術 |
| GdUnit4 (Godot plugin) | 單元測試 |
| OBS Studio | 錄 trailer |

### 第五個月加裝

| 工具 | 用途 |
|------|------|
| GDCubism plugin | Live2D 整合測試 |
| DaVinci Resolve / CapCut | 剪 trailer |
| Audacity | 音效後製 |

---

## 7. itch.io / Steam page / 求職平台

### itch.io（必上）

- 上傳 HTML5 build（玩家不用下載）
- 截圖 5-8 張
- 60-90 秒 trailer 嵌入
- 標題 + tagline + 描述（繁中 + 英文）
- 標籤：deckbuilder, roguelite, dice, card, godot
- 開放 free download / pay-what-you-want

### Steam page（可選，6 月內不上架）

- 早期可建 Steam page demo（要花 $100 Steam Direct）
- 6 月內**不上架**，避免分心 + 不增加履歷分
- 留待面試後再決定

### 求職平台

| 平台 | 用法 |
|------|------|
| 1111 人力 | 沃兎奈 + 廣譜 indie + 中小遊戲公司 |
| 104 | 中大型遊戲公司（沒專做 Godot 也投，他們會看 Unity / Unreal）|
| yourator | 新創 / 偏軟體 + 遊戲公司 |
| Discord 中文遊戲開發社群 | 內推機會 |
| Twitter / X | follow 台灣 indie dev，看 hiring 訊息 |
| Bahamut 遊戲開發板 | 看 hiring + 自己 self-promote |

---

## 8. 履歷 / GitHub README template

### 履歷 1 頁

```
[你的名字]
Godot Game Engineer（Junior）/ Frontend Background

✉ email | 🐙 github.com/[user] | 🎮 [user].itch.io

== 經歷 ==
- 6 個月自學 Godot，從 Vue/Angular 前端轉遊戲開發
- 完成 DiceFateSurvivor demo（deckbuilder + VN + 棋盤 hybrid）
  - 上架 itch.io / HTML5 直接可玩
  - 81 GitHub commits / 250+ unit tests / i18n 繁英完整

== 作品 ==
1. DiceFateSurvivor [itch link] [GitHub link] [trailer link]
   - Godot 4.6.2 + GDScript + .tres data-driven
   - 系統：戰鬥 + 棋盤 + Hub + Save / Load
   - Combo Lock universal 機制（連續同類卡 ×1.5）
   - VN 對話 + 分支選項 + Aria 互動立繪
   - QTE mini-game（致敬 WorkNite）

2. Live2D Integration Test [GitHub link] [demo gif]
   - GDCubism plugin / Cubism SDK
   - mouse-tracking / click interaction / lip sync

== 技術 ==
- 引擎：Godot 4.6.2 (GDScript), 熟基本 Unity (C#)
- 前端：Vue 3, Angular, TypeScript, React 基礎
- 工具：Git, Figma, Krita, Aseprite, Live2D 整合
- 設計：data-driven, signal-based architecture, unit testing
- 語言：繁中（母語）, 英文（讀寫 OK）

== 求學 ==
[簡單列即可]

== 為什麼想加入貴公司 ==
[1-2 句，每家客製]
```

### GitHub README template

```markdown
# DiceFateSurvivor 🎲

> 大富翁式擲骰棋盤 × Slay the Spire 風卡牌 × Vampire Survivors 壓力的 hybrid roguelite

[![Play on itch.io](https://img.shields.io/badge/Play-itch.io-red)](link)
[![Trailer](https://img.shields.io/badge/Watch-Trailer-blue)](link)

## 截圖

[5-8 張]

## 玩法

[60 秒能讀完]

## 技術亮點

- **Godot 4.6.2 + GDScript** 完全 data-driven (.tres Resource)
- **Combo Lock universal**：連續同類卡 ×1.5 dmg/block bonus
- **AnimatedSprite2D + Tween 互動角色**：Aria 隨遊戲 state 變立繪 + 點擊互動
- **VN 對話系統**：dialog tree + 分支選項 + state 影響
- **QTE mini-game**：5 鍵反應賺取戰鬥 bonus
- **6 autoload + signal pattern**：scene-tree-based event-driven
- **250+ unit tests** (GdUnit4)：戰鬥公式 + Combo Lock 完整覆蓋
- **i18n**：繁中 + 英文完整翻譯
- **XOR 加密 binary save**：v1 schema + migration framework

## 架構

[一張架構圖：autoload + scene 流向]

## 學到什麼

- 從前端 reactive 思維轉到 game loop / signal-based 思維
- data-driven design 大幅降低 hard-code（10 張卡可擴 100 張）
- 兼職 6 個月內 scope 控制：原野心 spec 81 卡，砍到 15 卡 demo

## 開發

```bash
git clone ...
godot -e project.godot
```

## License

MIT
```

---

## 9. 萬一中途想改 scope

容易發生：做著做著想加東西。**對自己嚴格**：

- 加東西 = 砍另一個東西（一定要）
- 加之前先 commit 當前進度（避免後悔）
- 寫進 `01-mvp-scope.md` 的「Out of scope」list（明示「這次不做」）
- 如果真的想加，記到 `docs/_archive/v1-full-vision-2026-05/` 的 spec 裡，未來 EA 再做

---

## 相關文件

- 6 月 portfolio scope：[`01-mvp-scope.md`](01-mvp-scope.md)
- 學習路徑：[`02-learning-path.md`](02-learning-path.md)
- 既有戰鬥公式：[`docs/01-design/combat-formulas.md`](https://github.com/asd23353934/dice-fate-survivor/blob/master/docs/01-design/combat-formulas.md)（原 DFS repo）
- 既有 phase 切片：[`docs/08-implementation/steps/`](https://github.com/asd23353934/dice-fate-survivor/tree/master/docs/08-implementation/steps)（原 DFS repo）
- 既有 Godot stack：[`docs/07-tech/stack.md`](https://github.com/asd23353934/dice-fate-survivor/blob/master/docs/07-tech/stack.md)（原 DFS repo）
- 完整野心版（封存）：[`docs/_archive/v1-full-vision-2026-05/`](https://github.com/asd23353934/dice-fate-survivor/tree/master/docs/_archive/v1-full-vision-2026-05)（原 DFS repo）
