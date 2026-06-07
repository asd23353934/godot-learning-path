# W21：Phase 8 美術收尾 — 音效系統 + 動畫探索

> M5 Phase 8 後半。核心成果：**AudioManager 從 W5 stub 升級為完整音效系統**（EventBus 自動播放 + 三軌 bus + id 約定）。附帶：Aria 立繪動畫的深度探索 → 定案走 Live2D。

---

## 1. 目標

- AudioManager（W5 起的 stub）→ 完整 BGM / SFX 系統
- 音效**零場景改動**自動播放（靠 EventBus 解耦）
- 三軌音量（master / bgm / sfx）串接 SettingsManager
- （探索）讓 Aria 立繪動起來 → 結論：Godot 程式變形是死路，改 Live2D

---

## 2. 設計決策

| 決策 | 理由 |
|---|---|
| **程式動態建 audio bus**（不靠編輯器 bus layout） | 不用手拉 GUI，純 code 可版控；`AudioServer.add_bus()` + `set_bus_send` |
| **id 約定載入**（`play_sfx("hit")` → `sfx/hit.ogg`） | 跟美術 id→png 一致；沒檔靜默跳過 → 漸進式填內容 |
| **EventBus 監聽播 SFX** | AudioManager 訂閱信號，發送方完全不知情 → 零場景改動加音效 |
| **SFX player pool（8 個）** | 同時多音效不互相切斷（單一 player 會被新音效搶走） |
| **placeholder 音效**（Python 合成） | 先驗證架構，內容後補；**架構 ≠ 素材** |
| **動畫走 Live2D 而非程式變形** | 6 種程式法全撞 Godot 限制（見下方 retro），Live2D 是 VN 業界正規解 |

---

## 3. 檔案

- `scripts/autoload/audio_manager.gd` — stub → 完整實作（+207 行）
- `scripts/autoload/settings_manager.gd` — `set_bgm/sfx_volume` 串接 AudioManager
- `assets/audio/` — `sfx/`（7 個 placeholder .wav）+ `README.md`（id 約定 + 資源）
- `experiments/_gen_sfx.py` — placeholder 音效合成器
- `docs/04-art/live2d-roadmap.md`（DFS repo）— Live2D 三階段路線圖

---

## 4. 音效資料流（自動播放）

```
玩家出牌 → enemy.gd: EventBus.card_played.emit()
                          ↓（AudioManager 在 _ready 已 connect）
            AudioManager._on_card_played() → play_sfx("card_play")
                          ↓
            找閒置 SFX player → 載入 sfx/card_play.ogg → play()
```

擲骰(dice) / 打擊(hit) / 治療(heal) / 勝負(victory·defeat) / 獎勵(reward) 同理 —— **combat.gd、board.gd、enemy.gd 完全沒改一行**。

---

## 5. code 解析

**動態建 bus**（`_setup_buses`）：
```gdscript
_bgm_bus_idx = AudioServer.bus_count   # bus 0 是 Master
AudioServer.add_bus()                  # 新增 BGM bus
AudioServer.set_bus_name(_bgm_bus_idx, "BGM")
AudioServer.set_bus_send(_bgm_bus_idx, "Master")  # BGM → Master
```

**id 約定 + 快取**（`_get_sfx`）：
```gdscript
if _sfx_cache.has(id): return _sfx_cache[id]   # 含 null（記住找不到）
for ext in [".ogg", ".wav", ".mp3"]:
    if ResourceLoader.exists(SFX_DIR + id + ext):
        stream = load(...); break
_sfx_cache[id] = stream  # 沒檔也快取 null，避免每次重試
```

**音量套 bus**：`linear_to_db(max(vol, 0.0001))` — `linear_to_db(0)` 是 `-inf` 會無聲不可逆，必 clamp。

---

## 6. Vue / 前端對照

- **EventBus 播音效** ≈ RxJS `Subject` 全域訂閱：emit 方不 import 訂閱方
- **audio bus（Master←BGM/SFX）** ≈ DAW / mixer 的音軌分組 + master fader
- **id 約定 + fallback** ≈ 動態 `import()` 找不到模組時的 graceful degradation
- **player pool** ≈ object pool 重用（避免每次 new AudioStreamPlayer）

---

## 7. 擴充方式

- **換真音效**：找 CC0 音效，**檔名一樣**覆蓋 `assets/audio/sfx/`，程式零改動
- **加新音效**：`play_sfx("new_id")` + 接對應 EventBus signal（或場景直接呼叫）
- **接 BGM**：場景 `_ready` 呼叫 `AudioManager.play_bgm("combat")`（自動循環 + crossfade）

---

## 8. 常見錯誤

- `linear_to_db(0)` = `-inf` → 音量 0 變不可逆無聲，要 `max(v, 0.0001)`
- autoload 順序：AudioManager 必須在 EventBus / SettingsManager **之後**（_ready 要 connect signal + 讀音量）
- 單一 AudioStreamPlayer 播 SFX 會互相切斷 → 用 pool
- `.wav` 沒設 loop → BGM 不循環（AudioManager 依格式設 loop flag）

---

## 9. 下游依賴 / 待辦

- ⏳ **真音效素材**（placeholder 暫定，待 CC0 覆蓋）
- ⏳ **BGM**：API ready，待素材 + 場景接 `play_bgm`
- ⏳ **Aria Live2D**：平行軌進行（見 `live2d-roadmap.md`）

---

## 10. 面試 talking point

- **「如何在不動既有戰鬥邏輯下加音效系統？」** → EventBus 解耦：AudioManager 訂閱信號，發送方零感知。展示 observer pattern 的實戰價值。
- **「素材還沒好怎麼先開發？」** → id 約定 + fallback：架構先行、內容漸進填。架構正確性不被素材阻塞。
- **「遇到做不出來的功能怎麼辦？」** → Aria 動畫：系統性試 6 種方法、讀引擎原始碼定位限制、認清 Godot Polygon2D 對單張立繪的本質限制、選擇業界正規解（Live2D）。展現**除錯深度 + 知道何時止損 + 選對工具**，而非硬幹。

---

## 附：Aria 動畫探索 retro（工程判斷記錄）

目標：讓單張 NovelAI 立繪「分部位動」（待機 / 攻擊）。依序試了 **6 種**：

| # | 方法 | 失敗原因 |
|---|---|---|
| 1 | 單圖 Tween（整體呼吸/位移） | 使用者拒收「同一張圖假動」 |
| 2 | GDCubism Live2D（當下） | 以為要編譯 C++，當下沒驗成 |
| 3 | Godot 原生 Skeleton2D + weight paint | Bone2D det==0 錯 + weight paint GUI 地獄 |
| 4 | Cutout 切圖分層 | 自動分層工具產出混亂，切圖瓶頸 |
| 5 | 程式生成 rigged Polygon2D（mesh+骨+weight） | Polygon2D skeleton deform 官方公認 buggy，讀原始碼修了 path + force rebind 仍無效 |
| 6 | 程式直接推 mesh internal vertex | **Polygon2D 內部頂點單獨改不渲染**（只配合骨架才動）→ 證實死路 |

**結論**：單張立繪在 Godot 硬做骨架動畫是引擎本質限制，正規解是 **Live2D**（對口 WorkNite VN/H game 即戰力）。決策：**平行進行** —— 遊戲開發不停，有空學 Cubism，Godot 整合（GDCubism 編譯）由 AI 代勞。詳見 `docs/04-art/live2d-roadmap.md`。

> 這 6 次撞牆不是浪費 —— 它把「為什麼不能用程式硬幹」徹底搞懂了，這種「窮盡後的篤定」比一開始就放棄更有說服力。
