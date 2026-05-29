# 07-implementation：實作流程教材

> 對應 PROGRESS.md 每週的詳細「**這週做了什麼 + 為什麼 + 怎麼擴充**」教材。
> 配合 AI 加速開發的學習模式：AI 寫 code、user 透過這份 guide review 並學會擴充。

## 用法

- 每週 commit + push 後，這個資料夾會新增一份 `WX-主題.md`
- 想複習某週實作時 → 看對應 .md
- 想加新功能時 → 看 .md「擴充方式」段
- 想面試講某個技術點時 → 看 .md「設計決策」段 + 對應 `06-interview-prep.md` Q

## 索引

| W / Phase | 主題 | 對應 repo |
|---|---|---|
| [W5 / Phase 0](W5-phase-0-autoloads.md) | DFS 5 個 autoload + 資料夾骨架 | [dice-fate-survivor](https://github.com/asd23353934/dice-fate-survivor) |
| [W6 / Phase 1](W6-phase-1-walking-skeleton.md) | SceneRouter fade + 4 場景 walking skeleton | dice-fate-survivor |
| [W7 / Phase 2 上](W7-phase-2-combat-prototype-upper.md) | 戰鬥 prototype 上半 — 移植 W4 卡牌 + drag-drop + EventBus 整合 | dice-fate-survivor |
| [W8 / Phase 2 下](W8-phase-2-combat-prototype-lower.md) | 戰鬥 prototype 下半 — 抽棄牌堆 + Shield + End Turn + 敵人攻擊 + 勝負 | dice-fate-survivor |
| [W9 / Phase 3 (1/3)](W9-phase-3-card-database.md) | 卡牌資料化 — CardDatabase autoload + id-based lookup + 10 張卡池 | dice-fate-survivor |
| [W10 / Phase 3 (2/3)](W10-phase-3-status-effects.md) | Status Effect 系統 — poison / weak / vulnerable / strength + 修飾鏈分輸出/受傷端 | dice-fate-survivor |
| [W11 / Phase 3 (3/3)](W11-phase-3-combo-lock.md) | **Combo Lock 招牌** — 1.0/1.2/1.5 multiplier + CardType enum + 11 個 combat-formulas tests | dice-fate-survivor |

## 每篇 guide 的固定結構

1. **目標 / 不目標**：明確 scope
2. **設計決策**：選 X 不選 Y 的理由
3. **檔案清單**：每個 .gd / .tscn 角色
4. **實作流程**：建構順序的邏輯
5. **關鍵 code 解析**：逐段註解
6. **觀念對照**：前端 / Vue / Angular 類比
7. **擴充方式**：之後加 X 怎麼動
8. **常見錯誤**：踩雷 + 修法
9. **下游依賴**：之後哪些功能會 build on this

## 寫作品質要求

- **能讓你不開 IDE 就能講解這套 code 給面試官**
- **能讓你或別人 2 個月後翻回來還能擴充**
- **每個「為什麼」都要答得出**
- **code 範例用實際 repo 內的程式碼**
