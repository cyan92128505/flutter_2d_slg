# Life Simulation Romance Game - AI Development Context

> **專案本質**：Clean Architecture + DDD 的長期 2D 戀愛模擬遊戲，劇本與程式碼分離，支援非技術人員編寫 YAML 場景。

**目前階段**: Phase 0 完成，Phase 1 開始（Domain Layer 實作）  
**技術棧**: Flutter 3.x + Dart 3.x + Riverpod 2.x

---

## 架構決策（ADR）

### 為什麼用 Clean Architecture？
- 專案會持續開發 1-2 年
- 劇本內容會頻繁變動，不能影響程式碼
- 未來可能加入多平台（iOS/Android/Web/Desktop）

### 為什麼用 Domain Events？
- 需要追蹤玩家行為（統計、成就系統）
- UI 更新與業務邏輯解耦
- 是的，一開始就設計，避免日後重構地獄

### 為什麼用 Repository 介面？
- SharedPreferences → SQLite 的遷移計畫
- 測試時需要 Mock
- 是的，我知道這是抽象層，我願意付出代價

### 為什麼用 YAML 不用 JSON？
- 劇本寫手不是工程師，減少括號地獄
- 支援註解（JSON 不支援）
- 更接近自然語言

### Condition 系統設計原則
- **簡化優先**：只支援 4 種常見 pattern（stat/relationship/flag/time）
- **正則表示式解析**：避免複雜 parser
- **複雜邏輯用 Dart DSL**：享受編譯時檢查
- **YAGNI 原則**：不要提前設計 20 種 condition type

---

## 開發原則（Linus Style）

### 程式碼風格
- 函式 < 20 行
- 不寫思考註解（程式碼自解釋）
- 消除特殊情況（good taste）
- Commit message 英文，格式：`feat/fix/chore: description`

### 測試策略
- 先實作，再補測試
- 只測關鍵路徑（Domain Layer 必測）
- 不追求 100% 覆蓋率，追求 0 regression

### 複雜度控制
- 懷疑每一個抽象層
- 如果 3 個月內不會用到，就不要做
- 能用 Map 就不要用 class
- 避免重新發明程式語言（condition parser 陷阱）

---

## 快速指令（Copy & Paste）

### 實作 Value Object
```
我要實作 [ValueObjectName] value object。
根據 `02_domain_model.md` 的定義：
- 不可變（immutable）
- 欄位：[field1, field2]
- 驗證規則：[rule1, rule2]

請提供實作，並說明設計決策。
風格：Linus style（簡潔、消除特殊情況）
```

### 實作 Use Case
```
我要實作 [UseCaseName] use case。
根據 `03_application_layer.md` 的定義：
- 輸入：[Input]
- 輸出：[Output]
- 流程：[Step1, Step2]

請提供實作，並說明協調邏輯。
風格：協調不實作（業務邏輯在 Domain）
```

### 實作 Repository
```
我要實作 [RepositoryName] repository。
根據 `04_infrastructure_layer.md` 的定義：
- 介面：[methods]
- 持久化：SharedPreferences / SQLite
- 序列化：JSON

請提供實作，並處理錯誤情況。
風格：Fail fast（資料損壞直接拋錯）
```

### YAML 場景設計
```
我要設計 [EventName] 事件。
根據 `05_narrative_system.md` 的 STAR 格式：
- Situation: [context]
- Task: [trigger]
- Action: [choices]
- Result: [consequences]

請提供簡化的 YAML 結構（只用 4 種 condition pattern）。
```

### 重構程式碼
```
這段程式碼有 [problem]：
[code snippet]

請用 Linus style 重構：
- 消除特殊情況
- 減少縮排層級
- 函式拆小（< 20 行）
```

### Debug 問題
```
我遇到 [error message]。
相關程式碼：[snippet]
預期行為：[expected]
實際行為：[actual]

請分析根本原因，並提供修復方案。
風格：最小改動（不要順便重構）
```

---

## 文件索引

### 必讀（開發前）
- `readme.md` - 專案概覽
- `01_game_mechanics.md` - 遊戲機制
- `02_domain_model.md` - Domain 設計

### 按需查閱（開發中）
- `03_application_layer.md` - Use Case 定義
- `04_infrastructure_layer.md` - 持久化實作
- `05_narrative_system.md` - YAML 劇本規範

### 進階參考（擴充時）
- `06_development_roadmap.md` - 開發計畫
- `07_collaboration_workflow.md` - 團隊協作

---

## 核心資料結構（速查）

### Player (Aggregate Root)
```dart
class Player {
  final PlayerId id;
  final Stats stats;              // 5 attributes: 0-100
  final TimePoint currentTime;    // day + timeSlot
  final Map<CharacterId, Relationship> relationships;
  final Set<String> storyFlags;
  final Map<LocationId, Set<InteractionPointId>> discoveredPoints;
  final Map<InteractionPointId, InteractionHistory> interactionHistories;
}
```

### InteractionHistory (Critical)
```dart
class InteractionHistory {
  final int totalInteractions;    // ⭐ Persisted forever
  final Set<EventId> completedEvents;
  final TimePoint? lastInteractionTime;
}
```

### ConditionalEvent
```dart
class ConditionalEvent {
  final EventType type;           // major_story / minor_branch
  final List<Condition> conditions;
  final Map<int, DialogueVariant> dialogueVariants;  // key = interaction count
}
```

---

## 簡化的 Condition 系統

### 支援的 4 種 Pattern

#### 1. Stat Condition
```yaml
conditions:
  - "charm >= 30"
  - "corruption < 50"
  - "intelligence >= 60"
```

#### 2. Relationship Condition
```yaml
conditions:
  - "alex.affection >= 50"
  - "bob.affection < 20"
```

#### 3. Flag Condition
```yaml
conditions:
  - "flag:met_alex"           # has flag
  - "!flag:alex_angry"        # not has flag
```

#### 4. Time Condition
```yaml
conditions:
  - "time:afternoon"
  - "day >= 3"
```

### 複雜邏輯用 Dart DSL
```dart
// 當 YAML 寫不出來時，用 Dart
final condition = Conditions.custom((player) {
  return player.stats.charm >= 30 
      && (player.relationships['alex']?.affection >= 50 ||
          player.hasFlag('alex_forced_route'));
});
```

---

## 常見問題（FAQ）

### Q: 為什麼 `totalInteractions` 要永久儲存？
**A**: 對話變體依賴它（第 1 次、第 2-4 次、第 5 次+ 不同台詞）

### Q: 為什麼 `stamina` 不能是負數？
**A**: 遊戲設計：0 體力時強制休息，避免玩家卡死

### Q: 為什麼用 immutable pattern？
**A**: 
1. 狀態變化可追蹤（time travel debugging）
2. 避免意外修改（特別是多執行緒）
3. 測試更簡單（pure function）

### Q: Domain Events 會影響效能嗎？
**A**: 不會。平均每次操作 < 5 個 events，序列化成本可忽略

### Q: YAML 解析會很慢嗎？
**A**: 不會。啟動時載入並快取，runtime 不重新解析

### Q: 為什麼不用完整的 condition parser？
**A**: 
1. 避免重新發明程式語言
2. 簡單的 4 種 pattern 涵蓋 80% 需求
3. 複雜邏輯用 Dart DSL（編譯時檢查）

---

## 開發檢查清單（Phase 1）

### Domain Layer - Value Objects
- [ ] `Stats` value object + 測試
- [ ] `TimePoint` value object + 測試
- [ ] `Relationship` value object + 測試
- [ ] `InteractionHistory` value object + 測試

### Domain Layer - Entities
- [ ] `Player` aggregate（核心方法）
  - [ ] `modifyStat()`
  - [ ] `advanceTime()`
  - [ ] `rest()`
  - [ ] `recordInteraction()`
- [ ] `InteractionPoint` entity
- [ ] `ConditionalEvent` entity

### Domain Layer - Services
- [ ] `ExplorationService`
- [ ] `EventTriggerService`

### Infrastructure Layer - Condition System
- [ ] 簡化版 `ConditionParser`（4 種 pattern）
- [ ] 正則表示式驗證
- [ ] Dart DSL 支援（複雜條件）

---

## Linus 的忠告

> "Talk is cheap. Show me the code."  
> 不要過度設計。先把核心跑起來，再優化。

> "Good taste" means eliminating special cases.  
> 如果你的函式有 3 層 if-else，重新思考資料結構。

> "Never break userspace."  
> Save file 格式一旦釋出，就要永久支援（版本遷移）。

> "Avoiding complexity is not the same as being simple."  
> 不要為了「可能的需求」提前設計複雜系統（YAGNI）。

---

## 風險警示

### 🔴 高風險區域（必須做對）
1. **InteractionHistory 持久化** - 對話變體的核心
2. **Save 版本遷移** - 影響玩家體驗
3. **Stats clamping** - 必須永遠在 0-100

### 🟡 中風險區域（謹慎設計）
1. **Condition 系統** - 避免過度設計
2. **Domain Events** - 確認真的需要
3. **YAML 格式** - 保持簡單

### 🟢 低風險區域（可以快速迭代）
1. **UI 層** - 隨時可以重構
2. **對話文本** - 純資料修改
3. **數值調整** - 遊戲平衡

---

**最後更新**: 2025-10-18  
**版本**: 1.0