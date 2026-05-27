# W9 / DFS Phase 3 (1/3)：卡牌資料化（CardDatabase autoload）

> 對應 PROGRESS.md W9 / DFS docs/08-implementation/steps/phase-3-combat-mature.md（卡池建立段）
> 完成日：2026-05-28
> Repo：[dice-fate-survivor](https://github.com/asd23353934/dice-fate-survivor)

---

## 1. 目標 / 不目標

### 目標

- **CardDatabase autoload**：啟動時掃 `data/cards/` 載所有 .tres 進 dictionary
- **id-based lookup**：`CardDatabase.get_card("strike")` 回 CardData
- **kit 起始牌組定義**：`kit_starter_decks: Dictionary` hardcode，**之後 .tres 化**
- **combat.gd 改用 CardDatabase**：移除 starter_deck @export，依 `GameState.current_kit` 動態載
- **加 5 張新卡到池**：bash / cleave / parry / brace / pommel_strike（10 張總計）
- **CardData 加 `id` 欄位**：唯一識別符

### 不目標（之後階段）

- KitData.tres（W10-W11 重構，目前 kit 定義在 CardDatabase 內 hardcode dictionary）
- card pool 概念（reward 三選一）— W14 整合時做
- 卡牌稀有度 / illustration / 圖鑑（W18-W19）
- 解鎖卡片進池機制（GameState.unlocked_cards 已有欄位但無消費邏輯）

---

## 2. 設計決策

### 為什麼**現在**抽 CardDatabase？

W7-W8 只 5 張卡 → 用 Inspector 拖檔夠用：

```
combat.tscn:
  starter_deck = [strike, strike, defend, heavy, ...]   ← 手動排
```

問題在規模：
- DFS MVP 計畫 10-15 張卡，最終 80+
- 每張卡 Inspector 拖一次 = N 次手動操作
- 換 kit 要重設 deck → 大量重複
- 找卡 by id 不存在 → 沒中央 registry

**W9 抽 CardDatabase 解決三件事**：

| 問題 | 解法 |
|---|---|
| Inspector 拖檔擴展不了 | `CardDatabase.get_card("strike")` |
| 一份卡牌池資料散落各 scene | 中央 dictionary `_cards: {id: CardData}` |
| 換 kit 要手動重組 | `CardDatabase.build_starter_deck("KIT_SWORD")` |

### 為什麼 **id-based** 不是直接 CardData ref？

**過渡兼容性 vs 直接 ref**：

| 用 id (string) | 用 ref (CardData) |
|---|---|
| 存檔時序列化簡單（字串） | 序列化要 path resolve |
| `kit_starter_decks` 內可寫 string | 要直接 `preload(...)` 一堆 ref |
| Card pool dictionary lookup | 慢 |
| 未來 `.tres` 重命名 / 移動只改 id | path 變要全找 |

**主要為了 save 系統 + 之後 kit/run 資料化的長期相容**。

但 runtime 還是用 `CardData` ref（id 只是查表 key）：

```gdscript
var deck: Array[CardData] = CardDatabase.build_starter_deck("KIT_SWORD")
# deck 內是真實 CardData ref，不是 id strings
```

### 為什麼 CardDatabase 在 GameState 之後？

```
SettingsManager → EventBus → GameState → CardDatabase → AudioManager → SceneRouter
```

- CardDatabase 不依賴 GameState（不會呼叫它）
- CardData 不 emit EventBus signal（純資料 class）
- 但放後面是**為了 GameState 的 reset_for_new_run 有可能呼叫 CardDatabase**（未來擴充）

如果 GameState reset 時需要 `CardDatabase.build_starter_deck(kit)`，CardDatabase 必須先 ready → 順序對。

### 為什麼用 `DirAccess` 動態 scan，不 hardcode load list？

**選項 A — Hardcode**：
```gdscript
const CARD_PATHS = [
    "res://data/cards/strike.tres",
    "res://data/cards/defend.tres",
    # ... 80 行
]
```

**選項 B — Dynamic scan**（採用）：
```gdscript
var dir = DirAccess.open("res://data/cards/")
# 遞迴掃 .tres
```

**選 B**：加新卡只放 .tres 進資料夾，**code 不用動**。設計師（你自己）友善。

代價：啟動慢一點點（80 張卡 < 100ms，無感）。

### 為什麼 kit_starter_decks **用 Dictionary** 不 .tres？

W9 階段：
- 1 個 kit（KIT_SWORD）
- Dictionary hardcode 10 行夠用

未來（W11-W14）改 .tres：
- KitData class（class_name KitData extends Resource）
- 每個 kit 一個 .tres：starter_deck_ids / starting_hp_bonus / passive_ability
- CardDatabase 改成載入所有 .tres + KitData.tres

**漸進式抽象**：先 Dictionary 確認 pattern，**確認需要**才 .tres 化。

---

## 3. 檔案清單

```
新增：
├── scripts/autoload/card_database.gd       ← W9 核心
├── data/cards/bash.tres                    ← W9 新卡 1
├── data/cards/cleave.tres                  ← W9 新卡 2
├── data/cards/parry.tres                   ← W9 新卡 3
├── data/cards/brace.tres                   ← W9 新卡 4
└── data/cards/pommel_strike.tres           ← W9 新卡 5

修改：
├── scripts/data/card_data.gd               ← 加 id 欄位
├── data/cards/strike.tres                  ← 加 id="strike"
├── data/cards/defend.tres                  ← 加 id="defend"
├── data/cards/focus.tres                   ← 加 id="focus"
├── data/cards/heavy.tres                   ← 加 id="heavy"
├── data/cards/quick.tres                   ← 加 id="quick"
├── project.godot                           ← 註冊 CardDatabase autoload
├── scenes/combat/combat.tscn               ← 移除 starter_deck @export
└── scenes/combat/combat.gd                 ← 改用 CardDatabase.build_starter_deck
```

---

## 4. 實作流程 / 順序

```
1. 設計：W7-W8 卡牌 .tres 直接拖入 combat.tscn 適合小 prototype
   W9+ 規模上去 → 需要中央 CardDatabase + id-based 查找
2. CardData 加 id 欄位（@export var id: String）
3. 5 張既有卡補 id（strike/defend/focus/heavy/quick）
4. 加 5 張新卡（bash/cleave/parry/brace/pommel_strike）
   · 覆蓋 cost 光譜 0/1/2
   · 含 attack（damage > 0）/ shield（shield > 0）/ skill placeholder
5. 寫 CardDatabase autoload
   · _ready 內 DirAccess scan data/cards/
   · 載入時 push_warning 跳過無 id 卡
   · 提供 get_card / get_all_cards / build_starter_deck API
   · 內含 kit_starter_decks dictionary（W9 只 KIT_SWORD）
6. 註冊 project.godot autoload（在 GameState 後、AudioManager 前）
7. combat.gd 改：移除 @export starter_deck，改 CardDatabase.build_starter_deck(GameState.current_kit)
8. combat.tscn 移除 starter_deck = [...] 那段，load_steps 從 11 改回 5
9. F5 跑：看 "[CardDatabase] autoload ready，載入 10 張卡" + 戰鬥能玩
10. 寫 W9 implementation guide
11. 雙 commit + push（中文 message）
```

---

## 5. 關鍵 code 解析

### `CardDatabase._load_all_cards` — Dynamic scan

```gdscript
func _load_all_cards() -> void:
    var dir = DirAccess.open(CARDS_FOLDER)
    if dir == null:
        push_error("[CardDatabase] 找不到 %s" % CARDS_FOLDER)
        return

    dir.list_dir_begin()
    var file_name = dir.get_next()
    while file_name != "":
        if file_name.ends_with(".tres"):
            var card: CardData = load(CARDS_FOLDER + file_name)
            if card != null and card.id != "":
                if card.id in _cards:
                    push_warning("[CardDatabase] 重複 id: %s（覆蓋）" % card.id)
                _cards[card.id] = card
            else:
                push_warning("[CardDatabase] 跳過 %s（無 id 欄位）" % file_name)
        file_name = dir.get_next()
    dir.list_dir_end()
```

**重點**：
- `DirAccess.open(path)` 取得目錄 handle（path 是 `res://...` 也 work）
- `list_dir_begin` / `get_next` / `list_dir_end` 是 Godot 的目錄迭代 pattern
- `load(path)` 直接拿到 Resource（Godot 自動依 script_class 轉成 CardData）
- 安全性：null check + id 空字串 check + 重複 id warning
- `push_warning` vs `push_error`：warning 是「值得注意但不擋」，error 是「應該不會發生的」

**前端類比**：Webpack `require.context()` 自動 import 整資料夾、Vite glob import。

### `CardDatabase.build_starter_deck` — id → CardData

```gdscript
func build_starter_deck(kit_id: String) -> Array[CardData]:
    if kit_id not in kit_starter_decks:
        push_error("[CardDatabase] 找不到 kit: %s" % kit_id)
        return []
    var deck: Array[CardData] = []
    for card_id in kit_starter_decks[kit_id]:
        var card = get_card(card_id)
        if card != null:
            deck.append(card)
    return deck
```

**Pattern**：id string → CardData ref 的轉換層。

`kit_starter_decks["KIT_SWORD"]` 是 `Array[String]`：
```gdscript
[
    "strike", "strike", "strike", "strike",
    "defend", "defend", "defend",
    "bash",
    "heavy",
    "quick",
]
```

→ 跑完 `build_starter_deck("KIT_SWORD")` 回 10 個 CardData ref。

**為什麼不是 build 後 cache**：
- 起始牌組每 run 重新建（fresh deck，原 list 不變動）
- 玩家在 run 內加卡 / 升卡 / 刪卡只動 GameState.deck，不影響 kit 定義
- 不 cache 邏輯更簡單

### `combat.gd._ready` 用 CardDatabase

```gdscript
func _ready() -> void:
    # ...

    # W9：依 kit 載入起始牌組
    var kit = GameState.current_kit if GameState.current_kit != "" else "KIT_SWORD"
    var starter_deck = CardDatabase.build_starter_deck(kit)
    print("[Combat] 載入 kit=%s，起始牌組 %d 張" % [kit, starter_deck.size()])

    GameState.start_combat_with_deck(starter_deck)
    # ... 剩下同 W8
```

**Fallback `KIT_SWORD`**：標準防禦性程式設計 — 萬一 current_kit 是空字串（直接從 combat F6 跑、不經 main_menu），有預設值能跑不會炸。

---

## 6. 觀念對照（前端視角）

| Godot | 前端 / 一般軟體 |
|---|---|
| CardDatabase autoload | Database service / Repository pattern |
| `_cards: Dictionary{id: CardData}` | In-memory cache / Map |
| `DirAccess` scan + `load()` | `require.context()` (Webpack) / `import.meta.glob()` (Vite) |
| `build_starter_deck(kit_id)` | Factory pattern / `repo.findByKitId(...)` |
| id-based reference | UUID / slug-based foreign key |
| `kit_starter_decks` Dictionary hardcode | JSON config / .env file |
| 之後 KitData.tres 化 | DB table / API schema |

**整體**：W9 把卡牌系統從**「靜態 import」**升級到**「Repository pattern」**。前端對應如 Pinia store 中央管理資料、Repository class 包 DB query。

---

## 7. 擴充方式

### 加新卡（10 秒）

1. 在 `data/cards/` 建新 .tres（複製既有再改值）
2. 設 id 唯一字串
3. 加進 `kit_starter_decks` 對應 kit 的 array（如果要進起始牌組）
4. F5 → CardDatabase 自動載入

**不用碰 combat.tscn / combat.gd / hand.gd**。

### 加新 kit（KIT_SPELL）

```gdscript
# card_database.gd
var kit_starter_decks: Dictionary = {
    "KIT_SWORD": ["strike", "strike", ...],
    "KIT_SPELL": ["zap", "zap", "ice_shield", ...],   ← 新加
}
```

然後 main_menu 開始按鈕 `GameState.reset_for_new_run("KIT_SPELL")`，combat.gd 自動載對應牌組。

### 把 kit 改 .tres 化（W10-W11）

```gdscript
# scripts/data/kit_data.gd
class_name KitData
extends Resource

@export var id: String = ""
@export var kit_name: String = ""
@export var starter_card_ids: Array[String] = []
@export var bonus_max_hp: int = 0
@export var passive_description: String = ""
```

```
# data/kits/kit_sword.tres
[resource]
id = "KIT_SWORD"
kit_name = "劍流"
starter_card_ids = ["strike", "strike", ..., "heavy"]
bonus_max_hp = 0
passive_description = "..."
```

CardDatabase 改：
```gdscript
const KITS_FOLDER = "res://data/kits/"
var _kits: Dictionary = {}    # {id: KitData}

func _ready():
    _load_all_cards()
    _load_all_kits()

func build_starter_deck(kit_id):
    var kit: KitData = _kits.get(kit_id)
    if kit == null:
        return []
    return kit.starter_card_ids.map(get_card)
```

**移除** `kit_starter_decks` hardcode dictionary，**完全 data-driven**。

### 加 card pool（reward 三選一）

```gdscript
# card_database.gd
@export var common_pool: Array[String] = ["strike", "defend", ...]
@export var rare_pool: Array[String] = ["bash", "heavy", ...]

func get_random_rewards(rarity: String, count: int) -> Array[CardData]:
    var pool = common_pool if rarity == "common" else rare_pool
    pool.shuffle()
    var rewards: Array[CardData] = []
    for i in range(min(count, pool.size())):
        rewards.append(get_card(pool[i]))
    return rewards
```

W14 整合戰鬥獎勵時用。

### 加搜尋 / 過濾

```gdscript
func find_cards_by_cost(target_cost: int) -> Array[CardData]:
    var result: Array[CardData] = []
    for card in _cards.values():
        if card.cost == target_cost:
            result.append(card)
    return result

func find_cards_with_damage() -> Array[CardData]:
    return _cards.values().filter(func(c): return c.damage > 0)
```

之後圖鑑 UI / 牌組編輯器用。

---

## 8. 常見錯誤 / debug 指南

### CardDatabase 載 0 張卡

**症狀**：`[CardDatabase] autoload ready，載入 0 張卡`
**根因**：
1. `data/cards/` 沒 .tres
2. 所有 .tres 都缺 id 欄位 → 全被 skip
3. CARDS_FOLDER 路徑寫錯
**修法**：看 push_warning 訊息；確認 .tres 有 `id = "..."` 行

### 重複 id 卡互相覆蓋

**症狀**：`[CardDatabase] 重複 id: strike（覆蓋）`
**根因**：兩個 .tres 用同 id 值
**修法**：改其中一個 id 唯一
**預防**：未來可加單元測試掃 .tres 確保 id 唯一

### `find_cards_by_cost` 回空

**症狀**：`CardDatabase.find_cards_by_cost(2)` 回 `[]`
**根因**：
1. id 字串大小寫不對
2. 卡 cost 設值不對
**檢查**：print `CardDatabase.get_all_cards()` 全部 dump

### kit_starter_decks 內 id 找不到

**症狀**：`[CardDatabase] 找不到 card id: xxx`
**根因**：dictionary 內 id 拼錯（例：寫 "Strike" 但 .tres 是 "strike"）
**修法**：對齊大小寫；統一用 snake_case

### combat.gd 報「current_kit 為空」

**症狀**：直接 F6 combat.tscn 跑，current_kit 是 ""
**根因**：沒經 main_menu 的 reset_for_new_run
**修法**：W9 已加 fallback `if current_kit == "": kit = "KIT_SWORD"`
**長期**：W10+ 加 KitSelectScene，玩家明確選 kit

---

## 9. 下游依賴

| 之後階段 | 怎麼用 W9 成果 |
|---|---|
| W10 Status effect | CardData 加 status_effects: Array[StatusData]；CardDatabase 一樣 scan |
| W11 Combo Lock + 卡牌測試 | CardDatabase.get_all_cards() 提供測試 input；測 combo 邏輯 |
| W11 card_type enum 重構 | CardData 加 enum CardType / TargetType；CardDatabase 不變 |
| W12-W13 棋盤 | 棋盤 reward 格用 `CardDatabase.get_random_rewards(rarity, 3)` |
| W14-W15 整合 | KitData.tres 化；CardDatabase 加 `_load_all_kits` |
| W17 SaveManager | 序列化 deck 為 id strings；讀回時 CardDatabase.get_card(id) |
| W18-W19 內容填充 | 加 10+ 張新卡到 data/cards/（不動 code） |
| W20 美術 | CardData 加 illustration: Texture2D；CardDatabase load |
| 圖鑑 menu | `CardDatabase.get_all_cards()` 展示 + lock/unlock 過濾 |

---

## 10. 面試 talking point

> 「W9 我抽 CardDatabase autoload — 中央卡牌 registry。從 W7-W8 的『Inspector 拖檔配置 5 張』升級到『data/cards/ 自動掃描載入 10+ 張』。
>
> 用 `DirAccess` 啟動時掃資料夾 + `load()` 載入 .tres → 全進 `Dictionary{id: CardData}`。提供 `get_card(id)` / `build_starter_deck(kit_id)` 中央 API。
>
> 設計重點：
>
> 一、**id-based reference 而非 ref**：dict key 是字串。序列化簡單（存檔寫 id strings 不寫 path），未來 .tres 重命名只改 id 不影響其他地方。
>
> 二、**漸進式抽象**：kit 定義先用 hardcode Dictionary，**之後**確定需要才 .tres 化成 KitData.tres。**沒 YAGNI 過早抽象**。
>
> 三、**動態 scan 而非 hardcode load list**：未來加新卡只要丟 .tres 進資料夾、code 不用動。**設計師（自己）友善**。
>
> 對應前端的 Repository pattern / Pinia store / Vite import.meta.glob。
>
> 之後 W14 加 reward 系統會用 `CardDatabase.get_random_rewards(rarity, 3)`；W17 SaveManager 序列化 deck 為 id strings 讀回時 lookup。**現在抽好 CardDatabase 之後所有功能順手加**。」
