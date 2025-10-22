# Life Simulation Romance Game - AI Development Context

> **專案本質**：Clean Architecture + DDD 的長期 2D 戀愛模擬遊戲，劇本與程式碼分離，支援非技術人員編寫 YAML 場景。

**目前階段**: Phase 1 開始（Domain Layer 實作）  
**技術棧**: Flutter 3.x + Dart 3.x + Riverpod 2.x  
**Domain 版本**: v2.0 (2025-10-23)

---

## 架構決策（ADR）

### 為什麼用 Clean Architecture？
- 專案會持續開發 1-2 年
- 劇本內容會頻繁變動，不能影響程式碼
- 未來可能加入多平台（iOS/Android/Web/Desktop）

### 為什麼 GameState 是 Aggregate Root（不是 Player）？
- **舊設計問題**：Player 太肥大（時間、地點、庫存混在一起）
- **新設計優勢**：
  - Player 只管玩家資料（name, stats, relationships, flags）
  - GameState 協調所有狀態（player, time, season, location）
  - 更容易擴充（世界狀態、NPC 狀態）
- **結論**：符合 DDD 原則，職責分離清楚

### 為什麼 InteractionHistory 按點記錄（不是按事件）？
- **遊戲需求**：「第 7 次來咖啡廳」的對話變化
- **不是**：「第 3 次觸發 meet_alex 事件」
- **實作**：pointId → InteractionRecord(totalInteractions, completedEvents)
- **結論**：符合遊戲設計，支援對話變體

### 為什麼 Domination 用數值（不是道具計數）？
- **YAML 驅動**：不同道具/事件給不同增量
  - Photo +20, Video +50, Blackmail +30
- **劇本彈性**：寫手自由調整，不需要改 code
- **避免硬編碼**：不是「收集 3 個道具」，而是「domination >= 80」
- **結論**：實用主義 > 純粹 DDD

### 為什麼用 Domain Events？
- 需要追蹤玩家行為（統計、成就系統）
- UI 更新與業務邏輯解耦
- **Phase 2 才實作**（Phase 1 保持簡單）

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
根據 `domain_objects.md` 的定義：
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
包含 domination 增量設定（如果需要）。
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
- `01_game_mechanics.md` - 遊戲機制（v1.1 - 含週/季節）
- `02_domain_model.md` - Domain 設計（v1.1 - GameState Aggregate）
- `domain_objects.md` - Dart 實作細節（v2.0 - InteractionHistory）

### 按需查閱（開發中）
- `03_application_layer.md` - Use Case 定義
- `04_infrastructure_layer.md` - 持久化實作
- `05_narrative_system.md` - YAML 劇本規範

### 進階參考（擴充時）
- `06_development_roadmap.md` - 開發計畫
- `07_collaboration_workflow.md` - 團隊協作

---

## 核心資料結構（速查）

### GameState (Aggregate Root)
```dart
class GameState {
  final Player player;
  final PlayerState playerState;
  final GameTime time;              // week + weekday + period
  final Season season;              // spring/summer/autumn/winter
  final String currentLocationId;   // 只存 ID
  final InteractionHistory interactionHistory;
  
  // 快捷方法
  GameState modifyStat(String statName, int delta);
  GameState modifyAffection(String characterId, int delta);
  GameState modifyDomination(String characterId, int delta);
  GameState recordInteraction(String pointId, String eventId);
  int getInteractionCount(String pointId);
}
```

### Player (Entity)
```dart
class Player {
  final String id;
  final String name;
  final Stats stats;                // 5 attributes: stamina, charm, intelligence, corruption, cleanliness
  final Relationships relationships;
  final Map<String, dynamic> flags;
}
```

### GameTime (Value Object)
```dart
class GameTime {
  final int week;           // 1, 2, 3, ...
  final Weekday weekday;    // monday, tuesday, ..., sunday
  final TimeOfDay period;   // morning, afternoon, evening, night
  
  GameTime advance();       // 推進到下一時段
  GameTime advanceToNextDay();  // 推進到隔天早上
}
```

### Relationship (Value Object)
```dart
class Relationship {
  final String characterId;
  final int affection;    // 0-100
  final int domination;   // 0-100 (YAML-driven)
  
  Relationship modifyAffection(int delta);
  Relationship modifyDomination(int delta);
}
```

### InteractionHistory (Value Object - Critical)
```dart
class InteractionHistory {
  final Map<String, InteractionRecord> _records;  // pointId → record
  
  int getInteractionCount(String pointId);
  InteractionHistory recordInteraction(String pointId, String eventId);
}

class InteractionRecord {
  final String pointId;
  final int totalInteractions;    // ⭐ 按點計數，永久儲存
  final Set<String> completedEvents;
}
```

---

## 簡化的 Condition 系統

### 支援的 4 種 Pattern

#### 1. Stat Condition
```yaml
conditions:
  - "stamina >= 30"
  - "corruption < 50"
  - "intelligence >= 60"
```

#### 2. Relationship Condition
```yaml
conditions:
  - "alex.affection >= 50"
  - "bob.domination >= 80"  # 支配度檢查
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
  - "weekday:saturday"        # 週六
  - "season:spring"           # 春天
  - "week >= 3"
```

### 複雜邏輯用 Dart DSL
```dart
// 當 YAML 寫不出來時，用 Dart
final condition = Conditions.custom((state) {
  return state.player.stats.get('charm') >= 30 
      && (state.player.relationships.get('alex')?.affection >= 50 ||
          state.player.hasFlag('alex_forced_route'));
});
```

---

## YAML Domination 設定範例

### 道具使用增加支配度
```yaml
item_compromising_photo:
  name: "Compromising Photo"
  description: "A photo that could be used as leverage"
  on_use:
    effects:
      - type: modify_domination
        character: alex
        delta: 20  # 照片 +20

item_secret_video:
  name: "Secret Video"
  description: "More powerful leverage material"
  on_use:
    effects:
      - type: modify_domination
        character: alex
        delta: 50  # 影片 +50（更強）
```

### 事件增加支配度
```yaml
event_blackmail_success:
  conditions:
    - "alex.affection >= 30"
  dialogue: "I have evidence of what you did..."
  choices:
    - text: "Use it as leverage"
      effects:
        - type: modify_domination
          character: alex
          delta: 30
        - type: modify_affection
          character: alex
          delta: -10  # 好感度下降
```

### 檢查支配度條件
```yaml
event_domination_route:
  conditions:
    - "alex.domination >= 80"  # 支配度夠高才能觸發
  dialogue: "She has no choice but to accept..."
```

---

## 常見問題（FAQ - Updated）

### Q: 為什麼 GameState 是 Aggregate Root 而不是 Player？
**A**: 
1. **職責分離**：Player 只管玩家資料，GameState 管全局狀態
2. **擴充性**：未來加入世界狀態（天氣、NPC）更容易
3. **符合 DDD**：Aggregate Root 應該是最高層的協調者

### Q: 為什麼 InteractionHistory 按點記錄而不是按事件？
**A**: 
1. **遊戲需求**：「第 7 次來咖啡廳」（不是「第 3 次觸發某事件」）
2. **對話變體**：YAML 根據總互動次數選擇對話
3. **永久儲存**：totalInteractions 永不清除

### Q: 為什麼 Domination 是數值而不是道具計數？
**A**: 
1. **彈性**：不同道具給不同增量（Photo +20, Video +50）
2. **YAML 驅動**：劇本設計者自由調整，不改 code
3. **多種來源**：道具、事件都能增加 domination

### Q: 週和季節系統如何運作？
**A**: 
1. **週**：7 天循環，Sunday night → Monday morning (week++)
2. **季節**：不自動變化，由特殊事件觸發（Valentine → Spring）
3. **分離**：Season 不在 GameTime 裡（獨立欄位）

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
**A**: 不會。Phase 1 不實作，Phase 2 再加（實用主義）

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
  - [ ] Map 實作（支援 YAML 擴充）
  - [ ] 5 個預設屬性
  - [ ] Clamp 到 0-100
- [ ] `GameTime` value object + 測試
  - [ ] week + weekday + period
  - [ ] advance() 處理週轉換
- [ ] `Relationship` value object + 測試
  - [ ] affection + domination 雙數值
  - [ ] 分別修改方法
- [ ] `InteractionHistory` + `InteractionRecord` + 測試
  - [ ] 按點記錄（pointId → record）
  - [ ] totalInteractions 計數

### Domain Layer - Entities
- [ ] `Player` entity（核心方法）
  - [ ] `modifyStat()`
  - [ ] `modifyAffection()`
  - [ ] `modifyDomination()`
  - [ ] `setFlag()`, `getFlag()`, `hasFlag()`
- [ ] `Location` entity
  - [ ] parent-child 關係
  - [ ] `isChildOf()`, `isRoot()`

### Domain Layer - Aggregate Root
- [ ] `GameState` aggregate
  - [ ] `advanceTime()` 處理週轉換
  - [ ] `changeSeason()` 切換季節
  - [ ] `recordInteraction()` 記錄互動
  - [ ] `getInteractionCount()` 查詢次數
  - [ ] 快捷方法（modifyStat, modifyAffection, etc.）

### Domain Layer - Exceptions
- [ ] `InsufficientItemException`
- [ ] `ItemNotFoundException`
- [ ] `InvalidStatValueException`
- [ ] `LocationNotFoundException`

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
1. **InteractionHistory 持久化** - 按點記錄，永久儲存
2. **Save 版本遷移** - 影響玩家體驗
3. **Stats clamping** - 必須永遠在 0-100
4. **GameState 序列化** - Aggregate Root 的完整狀態

### 🟡 中風險區域（謹慎設計）
1. **Condition 系統** - 避免過度設計（4 種 pattern）
2. **YAML Domination 設定** - 確保劇本設計者理解
3. **週/季節轉換** - 邊界處理（Sunday → Monday, week++）

### 🟢 低風險區域（可以快速迭代）
1. **UI 層** - 隨時可以重構
2. **對話文本** - 純資料修改
3. **數值調整** - 遊戲平衡（domination 閾值）

---

## Phase 1 vs Phase 2

### Phase 1（當前 - 最小可行 Domain）
- ✅ GameState as Aggregate Root
- ✅ GameTime with week/weekday/period
- ✅ Season (event-driven)
- ✅ Relationship with affection + domination
- ✅ InteractionHistory by point
- ✅ Stats with Map (5 default attributes)
- ✅ PlayerState with Inventory
- ❌ Domain Events (延後)
- ❌ Domain Services (延後)
- ❌ Complex Buff System (延後)

### Phase 2（未來 - 進階功能）
- Domain Events (StatChanged, RelationshipChanged, etc.)
- Domain Services (EventTriggerService, RelationshipService)
- Complex Buff System (duration, stacking)
- Computed Properties (RelationshipLevel from affection)
- Jealousy Mechanics

---

**最後更新**: 2025-10-23  
**版本**: 2.0 (Phase 1 Domain Finalized)  
**重大變更**: GameState as Aggregate Root, InteractionHistory by point, Domination as numeric value