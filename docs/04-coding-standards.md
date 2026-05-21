# Coding Standards — Godot 4.6.2

> Portfolio MVP 程式規範。每條規則標來源。
> 適用版本：Godot 4.6.2 Standard / GDScript / GdUnit4 v6.0.0

---

## 來源標記

| 標記 | 來源 |
|------|------|
| `[Official]` | Godot 官方 docs |
| `[Community]` | GDQuest / 社群共識 |
| `[General]` | 通用 software engineering |
| `[My opinion]` | 個人建議，可改 |

---

## 1. 版本鎖定 `[Official]`

| 項目 | 版本 | Link |
|------|------|------|
| Godot Engine | **4.6.2 Standard**（不要 .NET / 不要 4.7 dev）| [release](https://godotengine.org/blog/release/) |
| GdUnit4 | **v6.0.0**（基於 Godot 4.5+）| [releases](https://github.com/MikeSchulze/gdUnit4/releases) |
| VSCode + Godot Tools | 最新 | [marketplace](https://marketplace.visualstudio.com/items?itemName=geequlim.godot-tools) |
| gdformat | 最新 | [godot-gdscript-toolkit](https://github.com/Scony/godot-gdscript-toolkit) |
| GDCubism plugin（如做 Live2D 測試）| 鎖 commit hash | - |

寫進 `README.md` Requirements section。

---

## 2. 資料夾結構 `[Official + My opinion]`

來源：[Project organization](https://docs.godotengine.org/en/stable/tutorials/best_practices/project_organization.html)

```
res://
├── addons/                    # 第三方 plugin
├── scenes/                    # .tscn
│   ├── main/
│   ├── combat/
│   ├── board/
│   ├── hub/
│   └── ui/
├── scripts/                   # .gd（不放 scene 內的）
│   ├── autoload/
│   ├── core/
│   ├── ui/
│   └── util/
├── data/                      # .tres
│   ├── cards/
│   ├── enemies/
│   ├── items/
│   ├── stages/
│   └── dialogs/
├── assets/                    # 美術 / 音樂 / 字型
│   ├── illustrations/
│   ├── ui/
│   ├── characters/
│   ├── audio/
│   └── fonts/
├── tests/                     # GdUnit4
├── translations/              # i18n
├── project.godot
└── README.md
```

規則：
- 檔名 / 資料夾名 `snake_case`（防 Windows 大小寫匯出問題）
- 第三方資源放 `addons/`

---

## 3. 命名慣例 `[Official]`

來源：[GDScript style guide - Naming](https://docs.godotengine.org/en/stable/tutorials/scripting/gdscript/gdscript_styleguide.html#naming-conventions)

| 類型 | 慣例 | 範例 |
|------|------|------|
| 檔名 | `snake_case` | `card_data.gd` |
| `class_name` | `PascalCase` | `CardData` |
| 函數 | `snake_case` | `play_card()` |
| 變數 | `snake_case` | `player_hp` |
| 訊號 | `snake_case` 動詞過去式 | `card_played` |
| 常數 | `CONSTANT_CASE` | `MAX_HAND_SIZE` |
| 列舉名稱 | `PascalCase` | `enum CardType` |
| 列舉成員 | `CONSTANT_CASE` | `{ATTACK, SKILL, POWER}` |
| 私有（前綴 `_`）| `_snake_case` | `_internal_state` |
| Node 名（場景內） | `PascalCase` | `HandArea` |
| 資料 ID `[Community]` | `PREFIX_NAME` | `CARD_STRIKE` |
| i18n key `[Community]` | `section.subsection.detail` | `card.strike.name` |

---

## 4. Code 順序 `[Official]`

來源：[GDScript style guide - Code order](https://docs.godotengine.org/en/stable/tutorials/scripting/gdscript/gdscript_styleguide.html#code-order)

```gdscript
# 1. @tool / @icon / @static_unload
@tool

# 2. class_name
class_name CombatManager

# 3. extends
extends Node

# 4. 文件註解 ##
## 戰鬥場景主邏輯

# 5. signal
signal card_played(card: CardData)

# 6. enum
enum State { IDLE, PLAYER_TURN, ENEMY_TURN }

# 7. const
const MAX_ENERGY: int = 3

# 8. static var
static var instance: CombatManager

# 9. @export var
@export var debug_mode: bool = false

# 10. var
var current_state: State = State.IDLE

# 11. @onready var
@onready var hand_area: Node2D = $HandArea

# 12. _init()
func _init() -> void: pass

# 13. _ready() + 其他虛擬方法
func _ready() -> void: pass

# 14. 公開方法 → 私有方法
func play_card(card: CardData) -> bool: pass
func _try_play(card: CardData) -> bool: pass
```

---

## 5. Formatting `[Official]`

來源：[GDScript style guide - Formatting](https://docs.godotengine.org/en/stable/tutorials/scripting/gdscript/gdscript_styleguide.html#formatting)

| 項目 | 規則 |
|------|------|
| 縮排 | **Tab**（不要空格）|
| 行長 | max **100 chars**，盡量 < 80 |
| 函數 / class 前後 | **2 空行** |
| 函數內邏輯分段 | 1 空行 |
| 字串引號 | 預設**雙引號** `""` |
| 多行陣列 / 字典 / 列舉 | 尾部加逗號 |
| 運算子前後 | 1 空格 |

```gdscript
# ❌
func play_card(card,energy):
  return card.damage*2

# ✅
func play_card(card: CardData, energy: int) -> int:
	return card.damage * 2
```

---

## 6. 布林運算子 `[Official]`

```gdscript
# ❌
if hp > 0 && energy > 0: pass
if !is_dead: pass

# ✅
if hp > 0 and energy > 0: pass
if not is_dead: pass
```

---

## 7. Typed GDScript `[Official]`

來源：[Static typing](https://docs.godotengine.org/en/stable/tutorials/scripting/gdscript/static_typing.html)

```gdscript
# 型別明確 → :=
var direction := Vector2(1, 0)
var count := 0

# 型別曖昧 / 明示 → : Type =
var hp: int = 0
var card: CardData = null

# get_node 一律明示型別
@onready var bar: ProgressBar = $UI/HPBar

# Array / Dictionary typed
var cards: Array[CardData] = []
var stats: Dictionary[String, int] = {}

# 函數 signature typed
func play_card(card: CardData, target: Enemy) -> bool: pass
```

---

## 8. Docstring `##` `[Official]`

來源：[GDScript style guide - Comments](https://docs.godotengine.org/en/stable/tutorials/scripting/gdscript/gdscript_styleguide.html#comments)

```gdscript
## 戰鬥場景主邏輯
## 處理出牌、敵人 AI、波次切換
class_name CombatManager
extends Node


## 玩家當前 HP
## 修改時必觸發 [signal player_hp_changed]
var player_hp: int = 75


## 嘗試出 [param card] 對 [param target]。
## 回傳 [code]true[/code] 表示出牌成功。
func play_card(card: CardData, target: Enemy) -> bool:
    # 一般註解用一個 #
    return true
```

`##` 出現在 F1 docs，`#` 只出現在 source。

---

## 9. Scene Composition `[Official]`

來源：[Scene organization](https://docs.godotengine.org/en/stable/tutorials/best_practices/scene_organization.html)

父子通訊 5 種方法（推薦度由高到低）：

1. **Signal 連接**
2. 方法呼叫
3. Callable 屬性
4. 節點 reference 注入
5. NodePath 注入

```gdscript
# ❌ 硬路徑
@onready var player = get_node("/root/Main/Game/Player")

# ❌ 編輯器內 connect signal（scene 複用會壞）

# ✅ EventBus 解耦
func _ready():
    EventBus.player_hp_changed.connect(_on_hp_changed)

# ✅ 依賴注入
@export var target_player: Player
```

---

## 10. Autoload `[Official + My opinion]`

該用 autoload：
- 全局事件 bus
- 全局狀態
- 單例服務
- 全局工具

不該用：
- 暫時的 UI
- 單一場景內邏輯
- 數量變動的物件（敵 / 卡 / 道具用 instance）

DFS 6 個 autoload：

| Autoload | 用途 |
|----------|------|
| `EventBus` | 全局 signal hub |
| `GameState` | 當前 run + 玩家 HP/energy/deck |
| `SaveManager` | 存檔讀寫 |
| `SettingsManager` | 音量 / 解析度 / 語言 |
| `Localization` | tr() wrapper |
| `AudioManager` | BGM / SFX |

---

## 11. Signal Pattern `[Community]`

來源：[GDQuest GDScript Guidelines](https://gdquest.gitbook.io/gdquests-guidelines/godot-gdscript-guidelines)

規則：

1. **命名動詞過去式**：`card_played` ✅ / `play_card` ❌
2. **typed param**：`signal card_played(card: CardData)` ✅
3. **Signal 解耦取代直接呼叫**
4. **不在 signal handler 裡 emit 同 signal**（無限 loop）

```gdscript
# ❌
ui.hp_bar.update(player.hp)
audio.play("hurt")

# ✅
EventBus.player_hp_changed.emit(player.hp, player.max_hp)
```

---

## 12. Resource Pattern `[Community + General]`

規則：

1. 所有 game data 用 `.tres`（卡 / 敵 / 道具 / kit）
2. 不要 hard-code 內容到 script
3. Resource class 定義 schema
4. Resource 不能循環引用（用 ID string 替代直接 reference）

```gdscript
# ❌
func get_strike_damage() -> int:
    return 6

# ✅
func get_card_damage(card: CardData) -> int:
    return card.damage

# ❌ 循環引用
class_name Card extends Resource
@export var gems: Array[Gem]  # 如 Gem 也引用 Card 會炸

# ✅ ID 引用
class_name Card extends Resource
@export var gem_ids: Array[String]
```

---

## 13. Godot 4 Gotchas

### `queue_free()` 不要 `free()` `[Official]`

```gdscript
# ❌ 立即 free，可能 crash
some_node.free()

# ✅ 下一幀 free
some_node.queue_free()
```

### `@onready` 不要 `_ready get_node()` `[Community]`

```gdscript
# ❌
var sprite: Sprite2D
func _ready() -> void:
    sprite = get_node("Sprite")

# ✅
@onready var sprite: Sprite2D = $Sprite
```

### 不要在 `_process` instantiate `[General]`

```gdscript
# ❌ 每幀生 node，FPS 掉
func _process(delta: float) -> void:
    var bullet = Bullet.new()
    add_child(bullet)

# ✅ 用 timer / input 觸發
func _on_fire_pressed() -> void:
    var bullet = Bullet.new()
    add_child(bullet)
```

### Tween 用 `create_tween()` `[Official]`

```gdscript
# ❌ Godot 3 風（已 deprecated）
$Tween.interpolate_property(...)

# ✅ Godot 4
var tw = create_tween()
tw.tween_property(self, "modulate:a", 0.0, 1.0)
```

### Magic number 用 const `[General]`

```gdscript
# ❌
if score > 100:
    player.hp += 5

# ✅
const SCORE_HEAL_THRESHOLD: int = 100
const HEAL_AMOUNT: int = 5
if score > SCORE_HEAL_THRESHOLD:
    player.hp += HEAL_AMOUNT
```

### Resource 用 `.tres` 不要 `.res` `[Community]`

- `.tres` 純文字，git diff 可讀
- `.res` binary，git 看不出改了什麼

### UI 用 Control，遊戲物件用 Sprite2D `[Official]`

- 按鈕 / panel / label → `Button` / `Panel` / `Label`
- 角色 / 子彈 → `Sprite2D` / `AnimatedSprite2D`

### `await` 用法 `[General]`

```gdscript
# ❌
func _process(delta: float) -> void:
    await get_tree().create_timer(1.0).timeout

# ✅
func play_card_animation() -> void:
    sprite.play("card_slide")
    await sprite.animation_finished
    EventBus.card_played.emit(card)
```

---

## 14. Git Workflow `[General]`

來源：[Conventional Commits](https://www.conventionalcommits.org/zh-hant/v1.0.0/)

### Commit format

```
type(scope): summary
```

| Type | 用途 |
|------|------|
| `feat` | 加新功能 |
| `fix` | 修 bug |
| `refactor` | 重構不改行為 |
| `docs` | 改文件 |
| `test` | 加測試 |
| `chore` | 雜事（package / config）|
| `style` | 格式（gdformat 結果）|

範例：

```
feat(combat): add Combo Lock universal mechanic
fix(card): STRIKE damage not applying vulnerability multiplier
refactor(autoload): rename EventManager → EventBus
docs(readme): update Godot version requirement to 4.6.2
test(combat-formulas): add 10 unit tests for damage calculation
```

規則：
- 1 commit 1 件事
- commit message 寫「為什麼」
- 每天至少 1 commit

---

## 15. 自我 Review Checklist

```
[ ] 所有變數 / 函數 typed
[ ] 沒有 magic number（全 const）
[ ] 沒有 hardcode 卡 ID 的 if/else
[ ] Signal 取代直接呼叫
[ ] 沒有循環引用 .tres
[ ] 沒有 hard 路徑 get_node("../../...")
[ ] 改 combat 邏輯加了 1-2 unit test
[ ] Run 過 1 場完整遊戲沒 crash
[ ] gdformat check 通過
[ ] commit message 符合 conventional format
[ ] 沒有 commit binary / .tmp / .DS_Store
```

---

## 16. 工具設定

### `.vscode/settings.json`

```json
{
    "[gdscript]": {
        "editor.insertSpaces": false,
        "editor.tabSize": 4
    }
}
```

### gdformat 安裝

```bash
pip install "gdtoolkit==4.*"
gdformat --version
gdformat path/to/file.gd
gdformat .
```

### `.gitignore`

```gitignore
.godot/
.import/
export.cfg
export_presets.cfg
*.import
.DS_Store
Thumbs.db
.idea/
build/
exports/
```

### `.editorconfig`

```ini
root = true

[*]
charset = utf-8
end_of_line = lf
insert_final_newline = true
trim_trailing_whitespace = true

[*.gd]
indent_style = tab
indent_size = 4

[*.md]
indent_style = space
indent_size = 2
```

### Pre-commit hook（選用）

`.git/hooks/pre-commit`:

```bash
#!/bin/bash
gdformat --check $(git diff --cached --name-only --diff-filter=ACM | grep '\.gd$')
```

---

## 17. 驗證來源

| 規則 | 驗的地方 |
|------|--------|
| GDScript style | [docs.godotengine.org/en/stable/tutorials/scripting/gdscript/gdscript_styleguide.html](https://docs.godotengine.org/en/stable/tutorials/scripting/gdscript/gdscript_styleguide.html) |
| Best practices | [docs.godotengine.org/en/stable/tutorials/best_practices/](https://docs.godotengine.org/en/stable/tutorials/best_practices/) |
| Class reference | [docs.godotengine.org/en/stable/classes/](https://docs.godotengine.org/en/stable/classes/) |
| GdUnit4 | [github.com/MikeSchulze/gdUnit4](https://github.com/MikeSchulze/gdUnit4) |
| Conventional Commits | [conventionalcommits.org/zh-hant/v1.0.0/](https://www.conventionalcommits.org/zh-hant/v1.0.0/) |
| GDQuest 社群指南 | [gdquest.gitbook.io/gdquests-guidelines/godot-gdscript-guidelines](https://gdquest.gitbook.io/gdquests-guidelines/godot-gdscript-guidelines) |

---

## 相關文件

- 6 月 portfolio scope：[`01-mvp-scope.md`](01-mvp-scope.md)
- 學習路徑：[`02-learning-path.md`](02-learning-path.md)
- Pipeline：[`03-pipeline.md`](03-pipeline.md)
