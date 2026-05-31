# W17 / Phase 6 (2/2)：SaveManager + SettingsManager + 升級分階 + M4 收尾

> 對應 repo：[dice-fate-survivor](https://github.com/asd23353934/dice-fate-survivor)
> 對應 PROGRESS milestone：M4 W17（**M4 收尾 100%**）
> 寫於 Day 8（2026-05-31）

---

## 1. 目標 / 不目標

### 目標
- **SaveManager autoload**：XOR 加密 binary save / load，存永久進度
- **SettingsManager 實作**：ConfigFile 存偏好（音量 / 解析度 / 全螢幕 / locale），autoload _ready 套用到引擎
- **Settings menu scene**：完整 UI（OptionButton / CheckBox / Slider × 3 / 重置存檔）
- **升級分階 cap**：HP max 2 階 / Energy max 1 階，防無限堆破壞挑戰
- **Auto-save 時機**：升級 / 買卡後立刻 save，autoload _ready 自動 load

### 不目標
- HMAC + AES 真加密（XOR 是 casual mod prevention，W23+ 考慮升級）
- 多 slot 存檔（1 slot 夠，W18+ 加 slot 選擇）
- 雲端 sync / Steam Cloud（M6 / 履歷期再評估）
- 完整 i18n（locale 預留欄位，W18 內容期才實作 TranslationServer）
- 鍵位重綁定（之後）
- 跨 scene 主動「儲存提示」icon

---

## 2. 設計決策

### A. 為什麼 Save 跟 Settings 分兩個檔案？

| 內容 | 檔案 | 格式 | 加密 |
|---|---|---|---|
| **遊戲進度**（gold / 升級 / 卡片解鎖） | `user://save.dat` | JSON → XOR binary | 是（防手改） |
| **偏好設定**（音量 / 解析度 / 語言） | `user://settings.cfg` | ConfigFile（INI） | 否（玩家本來就該能改） |

理由：
- **重置存檔不該掉設定**：玩家刪檔重玩，不會想連音量都重設
- **手改的政治正確性差別**：手改 gold 是作弊，手改音量是合理偏好
- **同檔混合**反而會踩到「我想 mod 我的設定但碰到加密」UX 問題

### B. XOR 加密 vs HMAC + AES

| 方案 | 強度 | 開發成本 | 適用 |
|---|---|---|---|
| 明文 JSON | 0 | 0 | 純單機 prototype |
| **XOR with key**（採用） | 弱 — casual prevention | 5 分鐘 | indie / casual / 學習 project |
| HMAC-SHA256 + AES | 強 | 1-2 hr + crypto lib | 商業作品 / 競技對戰 |

選 XOR 的理由：
- **檔案打開不是明文** → 一般玩家不會用 hex editor 改
- **不需外部 crypto 函式庫** → 純 PackedByteArray + `^` 一行
- **key hardcode 在 source**，知道的人擋不住 — 但 DFS 不是 multi-player 對戰，作弊只是讓玩家自己沒成就感

> 對應面試 talking point：能講「為什麼這場景選 XOR 不選 AES」比講「我用了 AES」更有架構思維。

### C. 升級分階為什麼 HP 2 / Energy 1

- **HP +5 × 2 = +10**：最終 max_hp = 90（起 80 → +12.5%）
  - 撐 boss phase 2 攻擊 12 + weak 仍有壓力（不是無腦贏）
- **Energy +1 × 1 = +1**：最終 max_energy = 4
  - 多 1 energy 意義大（可多打一張 Skill 過卡或多一張 Attack 串 combo）
  - 但 +2 就會 trivialize 戰鬥（cost 1 卡可以打 4 張）

設計準則：**升級帶來成長感，但不破壞戰鬥挑戰**。實際數字可以在 W23 playtest 後再 tune。

### D. ConfigFile vs JSON for settings

| 方案 | 優點 | 缺點 |
|---|---|---|
| **ConfigFile**（採用） | Godot 內建、INI-like 玩家熟悉、API 簡潔 | 不支援巢狀結構 |
| JSON | 跨語言通用、結構靈活 | 玩家編輯時 syntax 容易壞 |

settings 是 flat key-value（master_volume / resolution_index / locale），用 ConfigFile 最自然。

### E. 自動 save 時機

- **升級 / 買卡後**：meta state 變動的明確時機 → 立刻 save
- **不 every gold_changed**：戰鬥內 gold +5 +10 每場很多次，I/O 太頻繁
- **autoload _ready load**：遊戲啟動就讀檔（GameState 在 SaveManager 前 ready，可被 populate）

如果 app crash 時最壞情況：lose 一次 hub interaction 之前的進度。可接受。

### F. setter pattern → apply + save 一條龍

```gdscript
# settings_manager.gd
func set_master_volume(vol: float) -> void:
    master_volume = clamp(vol, 0.0, 1.0)
    apply_settings()    # 立刻套到 AudioServer
    save_settings()     # 立刻持久
```

UI slider 改數值 → 立刻聽到音量變 + 設定持久。不用「套用」按鈕，UX 直覺。

---

## 3. 檔案清單

### 新增

| 檔案 | 角色 |
|---|---|
| `scripts/autoload/save_manager.gd` | SaveManager — XOR binary save/load API |
| `scenes/settings/settings.tscn` | Settings menu UI |
| `scenes/settings/settings.gd` | Settings menu controller |

### 修改

| 檔案 | 變動 |
|---|---|
| `scripts/autoload/settings_manager.gd` | stub → 完整實作（ConfigFile + apply_settings 套到 DisplayServer/AudioServer） |
| `scripts/autoload/game_state.gd` | +`MAX_*_CAP` / +`max_*_upgrade_purchases` / +`can_upgrade_*()` / +`reset_permanent_state()` |
| `scenes/hub/hub.gd` | trainer dialog 顯示「N/MAX 階」/ 達 cap 不能買 / 升級買卡後 `SaveManager.save_game()` |
| `scenes/main_menu.gd` | Settings button → settings scene |
| `project.godot` | autoload 加 SaveManager |

---

## 4. 實作流程

1. **GameState 加 cap 機制**（資料層基礎）
2. **SaveManager autoload**（核心服務）
3. **project.godot autoload 註冊**（讓 SaveManager 跑起來）
4. **SettingsManager 從 stub 變實作**（升級既有 autoload）
5. **Settings menu scene + UI**
6. **Hub 整合 cap + auto-save**
7. **main_menu 連 Settings 入口**

順序原則：**autoload 服務 → scene 整合**。autoload 提供 API，scene 消費。

---

## 5. 關鍵 code 解析

### 5.1 XOR 加密 + JSON 序列化

```gdscript
const SAVE_PATH = "user://save.dat"
const XOR_KEY = 0x42

func save_game() -> bool:
    var data = {"version": 1, "gold": GameState.gold, ...}
    var json_str = JSON.stringify(data)
    var bytes = json_str.to_utf8_buffer()
    _xor_inplace(bytes)
    var f = FileAccess.open(SAVE_PATH, FileAccess.WRITE)
    f.store_buffer(bytes)
    f.close()
    return true

func _xor_inplace(bytes: PackedByteArray) -> void:
    for i in range(bytes.size()):
        bytes[i] = bytes[i] ^ XOR_KEY
```

要點：
- XOR 是 **symmetric**（同 op 加密 / 解密），同一個 `_xor_inplace` 兩用
- `to_utf8_buffer` → PackedByteArray，可以 in-place 操作
- `JSON.stringify` 處理 nested Array / Dictionary 自動

### 5.2 Save load 含版本檢查

```gdscript
const SAVE_VERSION = 1

func load_game() -> bool:
    ...
    var data = json.data
    var version = data.get("version", 0)
    if version != SAVE_VERSION:
        push_warning("[SaveManager] 存檔版本不符（檔=%d / 期望=%d）" % [version, SAVE_VERSION])
        return false
    ...
```

之後升 SAVE_VERSION = 2 時加 migration：
```gdscript
if version == 1:
    data = _migrate_v1_to_v2(data)
```

### 5.3 ConfigFile settings persistence

```gdscript
func save_settings() -> void:
    var cfg = ConfigFile.new()
    cfg.set_value("audio", "master_volume", master_volume)
    cfg.set_value("display", "resolution_index", resolution_index)
    cfg.set_value("display", "fullscreen", fullscreen)
    cfg.save(SETTINGS_PATH)

func load_settings() -> void:
    var cfg = ConfigFile.new()
    if cfg.load(SETTINGS_PATH) != OK:
        return   # 首次遊玩，用預設值
    master_volume = cfg.get_value("audio", "master_volume", 1.0)
    ...
```

INI 結構：
```
[audio]
master_volume=0.75

[display]
resolution_index=1
fullscreen=false
```

玩家手動編輯也行，符合「偏好設定該透明」的理念。

### 5.4 apply_settings 套用到 engine

```gdscript
func apply_settings() -> void:
    # 音量：linear → dB（AudioServer 用 dB scale）
    var db = linear_to_db(max(master_volume, 0.0001))   # clamp 防無聲不可逆
    AudioServer.set_bus_volume_db(0, db)

    # 視窗模式
    if fullscreen:
        DisplayServer.window_set_mode(DisplayServer.WINDOW_MODE_FULLSCREEN)
    else:
        DisplayServer.window_set_mode(DisplayServer.WINDOW_MODE_WINDOWED)
        DisplayServer.window_set_size(RESOLUTIONS[resolution_index])
```

要點：
- **linear_to_db(0) = -inf**，會讓 audio bus 完全 mute 且無法恢復 → `max(v, 0.0001)` 守底
- DisplayServer / AudioServer 是 Godot 引擎層 singleton

### 5.5 升級分階 cap

```gdscript
# game_state.gd
const MAX_HP_UPGRADE_CAP = 2
var max_hp_upgrade_purchases: int = 0

func can_upgrade_max_hp() -> bool:
    return max_hp_upgrade_purchases < MAX_HP_UPGRADE_CAP

func purchase_max_hp_upgrade(amount: int = 10) -> void:
    permanent_max_hp_bonus += amount
    max_hp_upgrade_purchases += 1
    EventBus.upgrade_purchased.emit("max_hp")
```

```gdscript
# hub.gd
func _on_trainer_pressed():
    var hp_label = "+%d max HP（%d 金，%d/%d 階）" % [...]
    var hp_callback = _trainer_buy_hp
    if not GameState.can_upgrade_max_hp():
        hp_label = "max HP 已達上限（%d/%d）"
        hp_callback = _trainer_capped_msg
    %DialogPanel.show_dialog(..., [{"label": hp_label, "callback": hp_callback}, ...])
```

達 cap 不隱藏選項，改顯示 capped 狀態 → 玩家知道「這選項存在，但我已用完」。

### 5.6 Settings menu setter 立刻套用

```gdscript
# settings.gd
func _on_master_volume_changed(value: float) -> void:
    SettingsManager.set_master_volume(value)
    %MasterVolumeLabel.text = "%d%%" % int(value * 100)

# settings_manager.gd
func set_master_volume(vol: float) -> void:
    master_volume = clamp(vol, 0.0, 1.0)
    apply_settings()    # 立刻套
    save_settings()     # 立刻存
```

UX：拖 slider → 立刻聽到音量變化 + 設定永久保留。沒有「套用」按鈕。

### 5.7 重置存檔 destructive 二次確認

```gdscript
func _on_delete_save_pressed() -> void:
    %DialogPanel.show_dialog(
        "⚠ 警告",
        "確定要刪除存檔嗎？\n所有 gold / 升級 / 解鎖卡牌都會永久消失。",
        [
            {"label": "確定刪除", "callback": _confirm_delete},
            {"label": "取消", "callback": _close_dialog},
        ]
    )

func _confirm_delete() -> void:
    var ok = SaveManager.delete_save()
    _refresh_save_info()
    %DialogPanel.show_dialog("⚠ 警告",
        "✓ 存檔已刪除" if ok else "刪除失敗", [])
```

destructive UI 一定要二次確認 — 一鍵刪是 anti-pattern。

---

## 6. 觀念對照

| Godot 概念 | 前端 / Vue / Angular |
|---|---|
| Autoload `SaveManager` | Pinia / Vuex store with persistence plugin |
| ConfigFile (INI) | localStorage / sessionStorage（key-value） |
| XOR + binary save | obfuscated JSON 在 localStorage |
| `AudioServer.set_bus_volume_db` | Web Audio API GainNode |
| `DisplayServer.window_set_mode` | document.requestFullscreen / window.resizeTo |
| setter → apply + save | Vue computed setter with watch + persist |
| `linear_to_db` | dB = 20 × log10(linear)（音訊工程公式） |
| destructive 二次確認 dialog | `confirm()` 取代 / Modal with explicit confirm button |

---

## 7. 擴充方式

### 加多 slot（W18+）

```gdscript
const SAVE_PATHS = ["user://save_1.dat", "user://save_2.dat", "user://save_3.dat"]
var current_slot: int = 0

func save_game(slot: int = -1):
    var idx = slot if slot >= 0 else current_slot
    var path = SAVE_PATHS[idx]
    ...
```

UI 加 slot selector 在 main_menu。

### 升 SAVE_VERSION 加 migration

```gdscript
const SAVE_VERSION = 2

func load_game():
    var version = data.get("version", 0)
    if version == 1:
        data = _migrate_v1_to_v2(data)
        version = 2
    if version != SAVE_VERSION:
        return false
    ...

func _migrate_v1_to_v2(d: Dictionary) -> Dictionary:
    # v2 加了 unlocked_achievements，從空 array 起步
    d["unlocked_achievements"] = []
    return d
```

### 加 HMAC tamper detection（升級加密）

```gdscript
const HMAC_KEY = "your-secret"

func save_game():
    var json_str = JSON.stringify(data)
    var hmac = Crypto.new().hmac_digest(HashingContext.HASH_SHA256, HMAC_KEY.to_utf8_buffer(), json_str.to_utf8_buffer())
    var wrapper = {"data": json_str, "sig": Marshalls.raw_to_base64(hmac)}
    var bytes = JSON.stringify(wrapper).to_utf8_buffer()
    _xor_inplace(bytes)
    ...

func load_game():
    ...
    var wrapper = JSON.parse_string(json_str)
    var expected_sig = Crypto.new().hmac_digest(HashingContext.HASH_SHA256, HMAC_KEY.to_utf8_buffer(), wrapper.data.to_utf8_buffer())
    var got_sig = Marshalls.base64_to_raw(wrapper.sig)
    if got_sig != expected_sig:
        push_error("save tampered!")
        return false
```

擋手改 + 擋 XOR key 暴力反推。

### 加 i18n（W18 內容期）

```gdscript
# settings_manager.gd
func set_locale(loc: String):
    locale = loc
    TranslationServer.set_locale(loc)
    save_settings()
```

`tr("HUB_TITLE")` 在所有 label，搭配 .csv 翻譯表。

### 加 controller rebinding（按鍵重綁）

加 `key_bindings: Dictionary` 到 settings.cfg，UI 用 `InputEvent.is_action_pressed("custom_action_id")` 動態映射。

---

## 8. 常見錯誤

### 8.1 SAVE_PATH 用 `res://` 不能寫入

**症狀**：`open_error=13`（FileAccess.ERR_FILE_NOT_FOUND / write 失敗）。

**原因**：`res://` 是唯讀 game assets，runtime 不能寫。

**修法**：用 `user://` — Godot 自動 map 到 OS 適當位置：
- Windows: `%APPDATA%\Godot\app_userdata\<project>\`
- Linux: `~/.local/share/godot/app_userdata/<project>/`
- Mac: `~/Library/Application Support/Godot/app_userdata/<project>/`

### 8.2 XOR encryption 後 to_utf8 失敗

**症狀**：load 時 `json.parse` 噴錯，bytes 含非 UTF-8 字元。

**原因**：忘了在 load 時 XOR（XOR 是 symmetric，但要手動呼叫）。

**修法**：load 一定要 `_xor_inplace(bytes)` 才 `get_string_from_utf8()`。

### 8.3 ConfigFile load 失敗但 settings 仍清空

**症狀**：cfg.load 失敗（首次遊玩）→ settings 全變空字串。

**錯誤**：
```gdscript
cfg.load(PATH)   # 沒檢查 return value
master_volume = cfg.get_value("audio", "master_volume", 1.0)   # ← cfg 是空，get 不到值 → default
```

實際上 ConfigFile.load 失敗時 cfg 是空 ConfigFile，`get_value(..., default)` 會回 default，所以 OK。但建議顯式檢查：

```gdscript
var err = cfg.load(PATH)
if err != OK:
    print("首次遊玩，用預設值")
    return
```

### 8.4 linear_to_db(0) 無聲後再拉滑桿沒聲音

**症狀**：master_volume = 0 → bus = -inf dB → 之後 master = 0.5 還是無聲。

**原因**：`linear_to_db(0)` = -inf，AudioServer bus 進入「永久 mute」狀態（某些版本）。

**修法**：`max(v, 0.0001)` clamp 守底：
```gdscript
var db = linear_to_db(max(master_volume, 0.0001))
AudioServer.set_bus_volume_db(0, db)
```

### 8.5 Autoload 順序錯：SaveManager 在 GameState 前

**症狀**：SaveManager._ready 跑時 `GameState.gold` 還沒初始化（雖然 .gd default value 是 0，但 _ready order 影響其他邏輯）。

**修法**：autoload 順序明確：
```
SettingsManager → EventBus → GameState → CardDatabase → AudioManager → SaveManager → SceneRouter
```

SaveManager 在 **GameState 之後**（要 populate GameState fields），在 SceneRouter 之前（無 SceneRouter 依賴）。

### 8.6 Settings menu 改解析度視窗跑掉

**症狀**：1280×720 改 1920×1080，視窗變大但跑出螢幕邊界。

**修法**：apply_settings 內 resize 後置中：
```gdscript
DisplayServer.window_set_size(RESOLUTIONS[idx])
var screen_size = DisplayServer.screen_get_size()
var win_size = RESOLUTIONS[idx]
DisplayServer.window_set_position((screen_size - win_size) / 2)
```

---

## 9. 下游依賴

| 後續週 | 依賴 W17 的什麼 |
|---|---|
| **W18 內容** | save 結構穩定，加新 unlocked_cards / achievements 用 Dictionary.get + default |
| **W18 i18n** | SettingsManager.locale + TranslationServer.set_locale |
| **W19 棋盤事件** | unlocked_events 加進 save |
| **W20 美術** | settings 加 resolution 選項，UI 隨之 scale |
| **W21 音樂** | AudioManager 訂閱 SettingsManager.bgm_volume 變動 |
| **W22 整合** | Save migration（v1 → v2）|
| **W23-26 履歷** | 「我寫了完整 save + settings 系統」是面試亮點 |

---

## 10. 面試 talking point

> 「W17 我幫 DFS 加了 save + settings 雙系統，故意分兩個檔案不混。
>
> **save.dat** 用 XOR + JSON 加密 binary，存遊戲進度（gold / 升級 / 卡片解鎖）。
> XOR 不是真加密 — 只是讓檔案打開不是明文 JSON，擋一般玩家用記事本改數值。
> 知道我用 XOR + key 0x42 的人擋不住，但 DFS 不是 multi-player 對戰，
> 作弊只是讓玩家自己沒成就感，所以 XOR 的 trade-off 合理。
>
> **settings.cfg** 用 ConfigFile（INI）存偏好（音量 / 解析度 / 全螢幕）。
> 故意不加密 — 玩家本來就該能手改音量。混在 save 內反而會破壞 mod-friendliness。
>
> 重點設計：**升級分階 cap** — HP 2 階 / Energy 1 階。
> 沒這個就無限買 HP，玩家撐到不會死，戰鬥失去意義。
> cap 設定是『讓玩家感受到成長，但 boss phase 2 攻擊 12 + weak 仍能造成壓力』平衡點。
>
> 還有 **setter pattern** — set_master_volume(v) 自動 apply (套到 AudioServer) + save_settings()。
> 玩家拖 slider 立刻聽到 + 設定持久，不需要『套用』按鈕。」

---

**M4 (W14-W17) 完成 100% — DFS 從 prototype 升級成完整 meta loop game**
**整體進度：17/26 週 = 65.4%**
**下一站 M5（W18-W22）**：內容填充（補卡 / 補敵人 / 棋盤事件）+ Phase 8 美術（Aria 立繪 / Live2D）
