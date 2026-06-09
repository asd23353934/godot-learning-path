# Godot → Unity 轉換路徑

> **決策日**：2026-06-07（原 6 月計畫 W22）
> **原因**：目標公司（WorkNite）用 Unity + 台灣遊戲業 Unity 職缺明顯較多 → 就業導向換引擎
> **你的起點**：前端（會 TypeScript）+ 4 個月 Godot（懂遊戲開發核心、已做出接近完整的 DFS）

---

## 核心定位：你不是從零，比較像「W8 換工具」

你這 4 個月學的是**遊戲開發**，Godot 只是載體。設計思維幾乎全部可轉移到 Unity：

- 場景組合、狀態管理、事件解耦、data-driven 資料化、回合制戰鬥 / Combo / 狀態系統設計、存檔、設定、i18n、美術 / 音效 pipeline、Live2D 評估

**要重學的只有兩塊**：① GDScript → C#（你會 TS，超快）② Godot 編輯器 → Unity 編輯器。

---

## 時程（誠實）+ 雙軌策略

換引擎會讓「純 Unity 作品集」延後約 **6-9 週**。但你**不用空等才投履歷** —— 用雙軌：

- **軌 A（即刻）**：Godot DFS 已是完整作品集 → 現在就能放履歷，證明「能獨立做完一款有戰鬥 / 棋盤 / Meta / 存檔的遊戲」。面試講「正在用 Unity 重製以對齊貴司技術棧」= 展示學習力 + 誠意。
- **軌 B（進修）**：同時學 Unity + 做一個對口 WorkNite 的 **Unity 戰鬥垂直切片 + Live2D 角色** demo。

> 估時：C# + Unity 基礎 **2-3 週** → ScriptableObject + 戰鬥切片 **3-4 週** → Live2D **1-2 週**。

---

## Step 1：C# 速成（你會 TS → 直覺）

C# 跟 TypeScript 像近親，你 90% 看得懂。重點對照：

| 概念 | TypeScript | C# |
|---|---|---|
| 變數 | `let x: number` | `int x` / `var x` |
| 函式 | `function f(): void` | `void F()` |
| class / interface | 一樣 | 一樣（PascalCase 命名） |
| 非同步 | `async/await` | `async/await`（也有 Coroutine） |
| 陣列處理 | `.filter().map()` | **LINQ** `.Where().Select()` |
| 屬性 | getter/setter | `public int Hp { get; set; }` |
| 字串模板 | `` `${x}` `` | `$"{x}"` |
| 選擇性鏈 | `a?.b` | `a?.b`（一樣） |

**新東西**：`namespace`、強型別更嚴、`struct` vs `class`（值/參考）、`[Attribute]` 標註（如 `[SerializeField]`）。
**資源**：Microsoft Learn「C#」、邊寫邊查即可。

---

## Step 2：Unity 核心對應（對照你的 DFS）

**這張表是轉換關鍵 —— 你 DFS 的每個系統都能對上：**

| Godot（你做過的） | Unity | 備註 |
|---|---|---|
| Node | GameObject + Component | Unity 是組件組合，你 DFS 已用 composition 思維 |
| `extends Node` + script | **MonoBehaviour**（C# class） | `class X : MonoBehaviour` |
| Scene `.tscn` | Scene + **Prefab** | Prefab = 可重用 GameObject（≈ 你的 sub-scene 嵌入） |
| Resource `.tres`（CardData/EnemyData） | **ScriptableObject** | data-driven，**直接對應**，比 .tres 還好用 |
| autoload 單例（GameState/CardDatabase） | 單例 MonoBehaviour / static / ScriptableObject | |
| signal（EventBus 41 個） | **C# `event` / `Action` / UnityEvent** | EventBus pattern 一模一樣 |
| `_ready` | `Awake()` / `Start()` | Awake 最早、Start 次之 |
| `_process(delta)` | `Update()` | `Time.deltaTime` |
| `@export var` | `[SerializeField] private` | Inspector 可編 |
| `@onready $Node` | `GetComponent<T>()` / SerializeField 拖 | |
| `create_tween()`（juice/hit pause） | **DOTween**（套件）/ Animator | 補間動畫業界標配 |
| `await signal` | **Coroutine**（`yield return`）/ async | QTE / 過場 |
| `SceneRouter`（change_scene） | `SceneManager.LoadScene` | |
| `SettingsManager`（ConfigFile） | `PlayerPrefs` / `JsonUtility` | |
| `TranslationServer`（i18n） | Localization Package / I2 Localization | |
| `AudioManager`（bus + player） | `AudioSource` + `AudioMixer` | |

---

## Step 3：Unity 必學特定（Godot 沒有或不同的）

1. **MonoBehaviour 生命週期**：`Awake → OnEnable → Start → Update → FixedUpdate → OnDestroy`
2. **Prefab**：可重用 GameObject + **Prefab Variant / override**（比 Godot 繼承場景更強）
3. **ScriptableObject**：你的 CardData / EnemyData / EventData 全搬這（資料 asset，Inspector 編）
4. **Coroutine**：`StartCoroutine` + `yield return new WaitForSeconds()`（≈ await signal / timer）
5. **uGUI**（Canvas / RectTransform）：VN / 卡牌 UI 用這個較成熟（UI Toolkit 較新但 VN 生態少）
6. **Input System**（新）：比舊 `Input` 強

---

## Step 4：Live2D in Unity（WorkNite 核心，重點投入）

- **Cubism SDK for Unity**（Live2D 官方、免費）—— 比 Godot 的 GDCubism **順太多**（官方維護 + 文件齊 + 不用自己編譯 C++）
- 這是 VN/H game 工作室的核心技能，**對 WorkNite 最直接**
- 流程：Cubism Editor 做模型（跟 Godot 路線一樣）→ 匯出 → 官方 SDK 直接拖進 Unity → C# 控參數（眨眼 / 嘴型 / 表情 / lip sync）
- 之前研究的 `docs/04-art/live2d-roadmap.md` 的「分層 + Cubism rig」階段照樣有用，只有「進引擎」那步從 GDCubism 換成官方 Unity SDK（更簡單）

---

## Step 5：用 Unity 重建 DFS（複用設計，只翻譯不重想）

你**不用重新設計**，只把已驗證的架構翻成 Unity 實作。建議順序（先做最能展示的）：

1. **C# + Unity 基礎**：Unity Learn 跑一個小範例（hello + 移動方塊）
2. **ScriptableObject**：把 CardData / EnemyData 搬過去（你的資料層幾乎照抄）
3. **GameState 單例 + C# event bus**：狀態 + 事件骨架
4. **戰鬥垂直切片**：手牌 UI（uGUI）+ 拖拉打卡 + 敵人 HP + 回合制（你最熟的系統，先做這個最有成就感）
5. **Live2D Aria**：接 Cubism SDK，角色會動（對口亮點）
6. （行有餘力）漸進補棋盤 / Hub

---

## Step 6：作品集策略

- **Godot DFS**：保留 + 整理 README / 截圖 → 證明「能完成完整遊戲」
- **Unity 作品**：戰鬥垂直切片 + Live2D 角色 → 證明「Unity 即戰力 + 對齊 VN/H 技術棧」
- 兩個作品互補（一個完整、一個對口），比單一更有說服力

---

## 資源清單

- **Unity 官方**：[Unity Learn](https://learn.unity.com/)（免費官方路徑）
- **C#**：[Microsoft Learn - C#](https://learn.microsoft.com/dotnet/csharp/)
- **YouTube**：Tarodev、Code Monkey、Git-Amend（Unity 架構 / C# 最佳實踐）
- **Live2D**：[Cubism SDK for Unity 官方文件](https://docs.live2d.com/en/cubism-sdk-tutorials/getting-started/)
- **補間動畫**：DOTween（Asset Store 免費）
- **本地化**：Unity Localization Package（官方）

---

## 下一步

1. 裝 **Unity Hub + Unity 6 LTS**（最新 LTS）+ **Visual Studio / VS Code + C# Dev Kit**
2. 跑一個 Unity Learn 入門範例（熟悉編輯器，1-2 天）
3. 回來找 AI，開始 Step 2：把 DFS 的 CardData 用 ScriptableObject 重建（第一個「啊，原來能直接搬」的爽點）
