# 戀愛模擬遊戲完整開發流程

## 專案概覽

### 技術棧
- **前端框架**: Flutter 3.x + Dart 3.x
- **狀態管理**: Riverpod 2.x
- **架構模式**: Clean Architecture + Domain-Driven Design
- **資料格式**: YAML (場景系統)
- **持久化**: SharedPreferences

### 遊戲類型分析
- **類型**: 日式養成 + 事件驅動 AVG
- **核心機制**: 時間推進系統（周一～周日循環）
- **玩法**: 多角色好感度 + 條件觸發事件 + 探索式玩法
- **視覺呈現**: 2D 側視角（舞台劇式）+ 點擊互動

### 開發時程估算

#### 樂觀估算（一切順利）
```
Phase 0: 1 週
Phase 1: 3.5 週
Phase 2: 2.5 週
Phase 3: 2.5 週
Phase 4: 1.5 週
Phase 5: 2.5 週
Phase 6: 2 週
Phase 7: 1 週
───────────────
總計: 16.5 週 ≈ 4 個月
```

#### 實際估算（含 buffer 和返工）
```
Phase 0: 1.5 週
Phase 1: 4.5 週
Phase 2: 3 週
Phase 3: 3.5 週
Phase 4: 2 週
Phase 5: 3.5 週
Phase 6: 3 週
Phase 7: 1.5 週
───────────────
總計: 22.5 週 ≈ 5.5 個月
```

#### 每週 20 小時投入
```
22.5 週 x 40 小時 = 900 小時
900 小時 / 20 小時/週 = 45 週 ≈ 11 個月
```

---

## Phase 0: 驗證核心概念 (1 週)

**目標**: 確認技術可行性，避免走錯方向

### M0.1 - 技術驗證：視覺呈現 (2 天)

**任務清單**:
```
□ 建立專案結構
  - Flutter 3.x + Riverpod 2.x
  - folder structure (domain/application/infrastructure/presentation)
  
□ 實作 1 個場景的視覺 prototype
  - 背景圖（placeholder）
  - 3 個互動物件（用 Positioned + GestureDetector）
  - 角色立繪（右下角固定位置）
  
□ 驗證點擊互動
  - 點擊物件 → 顯示互動選單
  - 點擊 NPC → 顯示對話框
```

**交付物**:
- 1 個能跑的場景（醜版）
- 確認 UI 佈局可行

### M0.2 - 技術驗證：事件系統 (3 天)

**任務清單**:
```
□ 實作簡單的時間系統
  - GameTime (weekday, period)
  - 推進時間按鈕（測試用）
  
□ 實作簡單的事件觸發
  - 1 個時間觸發事件（周二 → 顯示對話）
  - 1 個地點觸發事件（進入廚房 → 顯示對話）
  
□ 實作簡單的存檔
  - 存儲當前時間 + 地點 + 事件歷史
  - 重啟 app 後能恢復
```

**交付物**:
- 能跑一輪時間循環
- 能觸發 2 個簡單事件
- 能存檔讀檔

**完成標準**:
```
✅ 確認「時間 + 地點 + 事件」系統技術可行
✅ 確認 UI 佈局不需要大改
✅ 沒有 showstopper
```

---

## Phase 1: Domain Layer (3.5 週)

**目標**: 建立穩固的領域模型

### M1.1 - 時間系統 (4 天)

**實作內容**:

```dart
// GameTime Value Object
enum Weekday { monday, tuesday, wednesday, thursday, friday, saturday, sunday }
enum TimeOfDay { morning, afternoon, evening, night }

class GameTime {
  final int week;           // 第幾週
  final Weekday weekday;    // 周一～周日
  final TimeOfDay period;   // 早上/下午/晚上/深夜
  
  GameTime({
    required this.week,
    required this.weekday,
    required this.period,
  });
  
  // 推進到下一個時段
  // morning → afternoon → evening → night → (隔天) morning
  GameTime advance() { /* ... */ }
  
  // 推進到隔天早上
  GameTime advanceToNextDay() { /* ... */ }
  
  // 跳到指定星期
  GameTime advanceToWeekday(Weekday target) { /* ... */ }
  
  // Helper methods
  bool isWeekend() => weekday == Weekday.saturday || weekday == Weekday.sunday;
  bool isWeekday(Weekday day) => weekday == day;
}
```

**測試清單**:
```
├─ test: advance() 正確推進
├─ test: advanceToNextDay() 跳到隔天
├─ test: 周日 night → 周一 morning（week + 1）
├─ test: equality and immutability
└─ test: isWeekend(), isWeekday() helpers
```

**交付物**:
- GameTime Value Object（100% 測試覆蓋）
- 設計文件（為什麼不用 DateTime）

### M1.2 - 地點系統 (3 天)

**實作內容**:

```dart
// Location Value Object（扁平化設計）
class Location {
  final String id;  // home, home_kitchen, club_toilet_stall_2
  
  Location(this.id);
  
  // Helper methods
  bool isHome() => id.startsWith('home');
  bool isOffice() => id.startsWith('office');
}

// LocationDefinition（用於 YAML 載入，Infrastructure 層）
class LocationDefinition {
  final String id;
  final String name;
  final String backgroundImage;
  final List<InteractableObject> interactables;
}

class InteractableObject {
  final String id;
  final String name;
  final ObjectType type;  // npc, item, furniture
  final List<double> clickArea;  // [x, y, w, h] 相對位置 0.0-1.0
}
```

**測試清單**:
```
├─ test: equality
└─ test: helper methods (isHome(), isOffice())
```

**交付物**:
- Location Value Object
- LocationDefinition 資料結構草稿

### M1.3 - 事件歷史系統 (3 天)

**實作內容**:

```dart
// EventHistory Value Object
class EventHistory {
  final Map<String, int> triggerCount;  // eventId → 觸發次數
  
  EventHistory(this.triggerCount);
  
  bool hasTriggered(String eventId) => 
    triggerCount[eventId] != null && triggerCount[eventId]! > 0;
  
  int getTriggerCount(String eventId) => triggerCount[eventId] ?? 0;
  
  EventHistory recordTrigger(String eventId) {
    final newCount = {...triggerCount};
    newCount[eventId] = getTriggerCount(eventId) + 1;
    return EventHistory(newCount);
  }
}
```

**測試清單**:
```
├─ test: 未觸發事件回傳 false/0
├─ test: recordTrigger() 正確增加計數
├─ test: 多次觸發同一事件正確計數
└─ test: immutability
```

**交付物**:
- EventHistory Value Object（100% 測試覆蓋）

### M1.4 - 玩家數值系統 (4 天)

**實作內容**:

```dart
// Stats Value Object
class Stats {
  final int stamina;
  final int charm;
  final int intelligence;
  final int mood;
  
  Stats({
    required this.stamina,
    required this.charm,
    required this.intelligence,
    required this.mood,
  });
  
  // 修改數值（回傳新實例）
  Stats modify(StatType type, int delta) {
    return Stats(
      stamina: type == StatType.stamina ? _clamp(stamina + delta) : stamina,
      charm: type == StatType.charm ? _clamp(charm + delta) : charm,
      intelligence: type == StatType.intelligence ? _clamp(intelligence + delta) : intelligence,
      mood: type == StatType.mood ? _clamp(mood + delta) : mood,
    );
  }
  
  int _clamp(int value) => value.clamp(0, 100);
}

// Relationship Value Object
class Relationship {
  final String characterId;
  final int affection;  // 0-100
  
  Relationship({required this.characterId, required this.affection});
  
  Relationship modify(int delta) {
    return Relationship(
      characterId: characterId,
      affection: (affection + delta).clamp(0, 100),
    );
  }
}

// Relationships (collection)
class Relationships {
  final Map<String, Relationship> _data;
  
  Relationships(this._data);
  
  Relationship? get(String characterId) => _data[characterId];
  
  Relationships modify(String characterId, int delta) {
    final current = get(characterId);
    final newRelationship = current != null
      ? current.modify(delta)
      : Relationship(characterId: characterId, affection: (50 + delta).clamp(0, 100));
    
    return Relationships({..._data, characterId: newRelationship});
  }
}
```

**測試清單**:
```
Stats:
├─ test: clamping (0-100)
├─ test: modification
└─ test: equality

Relationships:
├─ test: 不存在的角色回傳 null
├─ test: modify() 建立新關係或更新現有
└─ test: immutability
```

**交付物**:
- Stats Value Object
- Relationships Value Object

### M1.5 - 道具與狀態系統 (5 天)

**實作內容**:

```dart
// Item Value Object
class Item {
  final String id;
  final String name;
  final ItemType type;  // consumable, equipment, key_item
  
  Item({required this.id, required this.name, required this.type});
}

// Inventory Value Object
class Inventory {
  final Map<String, int> items;  // itemId → count
  
  Inventory(this.items);
  
  bool hasItem(String itemId) => items.containsKey(itemId) && items[itemId]! > 0;
  int getCount(String itemId) => items[itemId] ?? 0;
  
  Inventory addItem(String itemId, int count) {
    final newItems = {...items};
    newItems[itemId] = getCount(itemId) + count;
    return Inventory(newItems);
  }
  
  Inventory removeItem(String itemId, int count) {
    final currentCount = getCount(itemId);
    if (currentCount < count) {
      throw InsufficientItemException(itemId, currentCount, count);
    }
    
    final newItems = {...items};
    final newCount = currentCount - count;
    if (newCount == 0) {
      newItems.remove(itemId);
    } else {
      newItems[itemId] = newCount;
    }
    return Inventory(newItems);
  }
}

// PlayerState Value Object
class PlayerState {
  final Inventory inventory;
  final String? wearing;           // 當前裝備的道具 id
  final Set<String> statusEffects;  // drunk, tired, etc.
  
  PlayerState({
    required this.inventory,
    this.wearing,
    required this.statusEffects,
  });
  
  bool hasItem(String itemId) => inventory.hasItem(itemId);
  bool isWearing(String itemId) => wearing == itemId;
  bool hasStatus(String status) => statusEffects.contains(status);
  
  PlayerState equipItem(String itemId) {
    if (!hasItem(itemId)) {
      throw ItemNotFoundException(itemId);
    }
    return copyWith(wearing: itemId);
  }
  
  PlayerState unequipItem() => copyWith(wearing: null);
  
  PlayerState addStatus(String status) {
    return copyWith(statusEffects: {...statusEffects, status});
  }
  
  PlayerState removeStatus(String status) {
    return copyWith(statusEffects: statusEffects.difference({status}));
  }
}
```

**測試清單**:
```
Inventory:
├─ test: addItem 正確增加數量
├─ test: removeItem 正確減少（不能為負）
├─ test: removeItem 當數量為 0 時移除 entry
└─ test: immutability

PlayerState:
├─ test: equipItem 檢查是否擁有該道具
├─ test: equipItem 會 unequip 前一個
├─ test: status effects 正確新增/移除
└─ test: immutability
```

**交付物**:
- Item, Inventory, PlayerState Value Objects（100% 測試覆蓋）

### M1.6 - Player Aggregate (4 天)

**實作內容**:

```dart
// Player Entity
class Player {
  final String id;
  final String name;
  final Stats stats;
  final Relationships relationships;
  final Map<String, dynamic> flags;
  
  Player({
    required this.id,
    required this.name,
    required this.stats,
    required this.relationships,
    required this.flags,
  });
  
  Player modifyStat(StatType type, int delta) {
    return copyWith(stats: stats.modify(type, delta));
  }
  
  Player modifyRelationship(String characterId, int delta) {
    return copyWith(relationships: relationships.modify(characterId, delta));
  }
  
  Player setFlag(String key, dynamic value) {
    return copyWith(flags: {...flags, key: value});
  }
  
  dynamic getFlag(String key) => flags[key];
}
```

**測試清單**:
```
├─ test: stat modification
├─ test: relationship modification
├─ test: flag operations
└─ test: immutability
```

**交付物**:
- Player Aggregate（100% 測試覆蓋）

### M1.7 - GameState Entity (5 天)

**實作內容**:

```dart
// GameState Entity
class GameState {
  final Player player;
  final PlayerState playerState;
  final GameTime time;
  final Location location;
  final EventHistory eventHistory;
  
  GameState({
    required this.player,
    required this.playerState,
    required this.time,
    required this.location,
    required this.eventHistory,
  });
  
  GameState advanceTime() {
    return copyWith(time: time.advance());
  }
  
  GameState moveTo(Location newLocation) {
    return copyWith(location: newLocation);
  }
  
  GameState recordEvent(String eventId) {
    return copyWith(eventHistory: eventHistory.recordTrigger(eventId));
  }
  
  GameState modifyPlayer(Player newPlayer) {
    return copyWith(player: newPlayer);
  }
  
  GameState modifyPlayerState(PlayerState newState) {
    return copyWith(playerState: newState);
  }
}

// SaveData Value Object
class SaveData {
  final int version;  // 用於未來升級
  final GameState gameState;
  
  SaveData({required this.version, required this.gameState});
  
  Map<String, dynamic> toJson() { /* ... */ }
  factory SaveData.fromJson(Map<String, dynamic> json) { /* ... */ }
}
```

**測試清單**:
```
GameState:
├─ test: 時間推進正確
├─ test: 地點切換正確
├─ test: 事件記錄正確
└─ test: immutability

SaveData:
├─ test: round-trip (save → load)
├─ test: 處理 missing fields（向後相容）
└─ test: 處理 corrupted data (拋出特定錯誤)
```

**交付物**:
- GameState Entity（100% 測試覆蓋）
- SaveData 格式文件（JSON schema）

**Phase 1 完成標準**:
```
✅ 所有 Domain 物件 100% immutable
✅ 所有 public method 有單元測試
✅ 零外部依賴（不依賴 Flutter/Riverpod）
✅ 通過 Code Review
```

---

## Phase 2: Application Layer (2.5 週)

**目標**: 協調 Domain 物件，處理業務流程

### M2.1 - Repository Interfaces (2 天)

**實作內容**:

```dart
// 放在 Domain 層
abstract class GameStateRepository {
  Future<GameState?> load();
  Future<void> save(GameState state);
  Future<void> delete();
}

abstract class ScenarioRepository {
  Future<LocationDefinition> getLocation(String locationId);
  Future<EventDefinition> getEvent(String eventId);
  Future<List<EventDefinition>> getAllEvents();
}
```

**交付物**:
- Repository interfaces
- 職責文件

### M2.2 - Core Use Cases (5 天)

**實作內容**:

```dart
// StartNewGameUseCase
class StartNewGameUseCase {
  final GameStateRepository _repository;
  
  StartNewGameUseCase(this._repository);
  
  Future<GameState> execute() async {
    final initialState = GameState(
      player: Player(
        id: 'player_1',
        name: 'Player',
        stats: Stats(stamina: 50, charm: 50, intelligence: 50, mood: 50),
        relationships: Relationships({}),
        flags: {},
      ),
      playerState: PlayerState(
        inventory: Inventory({}),
        wearing: null,
        statusEffects: {},
      ),
      time: GameTime(week: 1, weekday: Weekday.monday, period: TimeOfDay.morning),
      location: Location('home'),
      eventHistory: EventHistory({}),
    );
    
    await _repository.save(initialState);
    return initialState;
  }
}

// LoadGameUseCase
class LoadGameUseCase {
  final GameStateRepository _repository;
  
  LoadGameUseCase(this._repository);
  
  Future<GameState?> execute() async {
    return await _repository.load();
  }
}

// SaveGameUseCase
class SaveGameUseCase {
  final GameStateRepository _repository;
  
  SaveGameUseCase(this._repository);
  
  Future<void> execute(GameState state) async {
    await _repository.save(state);
  }
}

// DeleteSaveUseCase
class DeleteSaveUseCase {
  final GameStateRepository _repository;
  
  DeleteSaveUseCase(this._repository);
  
  Future<void> execute() async {
    await _repository.delete();
  }
}
```

**測試清單**:
```
├─ test: StartNewGameUseCase 建立正確的初始狀態
├─ test: LoadGameUseCase 成功載入現有存檔
├─ test: LoadGameUseCase 處理不存在的存檔
├─ test: SaveGameUseCase 成功儲存
├─ test: SaveGameUseCase 處理寫入失敗
└─ test: DeleteSaveUseCase 成功刪除
```

**交付物**:
- 4 個基礎 UseCase（80% 測試覆蓋）

### M2.3 - Time & Location Use Cases (4 天)

**實作內容**:

```dart
// AdvanceTimeUseCase
class AdvanceTimeUseCase {
  final GameStateRepository _repository;
  final CheckEventTriggerUseCase _checkEventTrigger;
  
  AdvanceTimeUseCase(this._repository, this._checkEventTrigger);
  
  Future<GameState> execute(GameState currentState) async {
    final newState = currentState.advanceTime();
    
    // 檢查是否觸發時間相關事件
    final triggeredEvents = await _checkEventTrigger.execute(newState);
    
    // 自動儲存
    await _repository.save(newState);
    
    return newState;
  }
}

// MoveToLocationUseCase
class MoveToLocationUseCase {
  final GameStateRepository _repository;
  final CheckEventTriggerUseCase _checkEventTrigger;
  
  MoveToLocationUseCase(this._repository, this._checkEventTrigger);
  
  Future<GameState> execute(GameState currentState, Location newLocation) async {
    final newState = currentState.moveTo(newLocation);
    
    // 檢查是否觸發地點相關事件
    final triggeredEvents = await _checkEventTrigger.execute(newState);
    
    // 自動儲存（optional）
    await _repository.save(newState);
    
    return newState;
  }
}

// CheckEventTriggerUseCase
class CheckEventTriggerUseCase {
  final ScenarioRepository _scenarioRepository;
  final ConditionEvaluator _conditionEvaluator;
  
  CheckEventTriggerUseCase(this._scenarioRepository, this._conditionEvaluator);
  
  Future<List<EventDefinition>> execute(GameState state) async {
    final allEvents = await _scenarioRepository.getAllEvents();
    
    return allEvents.where((event) {
      // 過濾已觸發且不可重複的事件
      if (!event.repeatable && state.eventHistory.hasTriggered(event.id)) {
        return false;
      }
      
      // 檢查所有觸發條件
      return event.triggers.conditions.every((condition) {
        return _conditionEvaluator.evaluate(condition, state);
      });
    }).toList();
  }
}
```

**測試清單**:
```
├─ test: AdvanceTimeUseCase 正確推進時間
├─ test: AdvanceTimeUseCase 檢查事件觸發
├─ test: AdvanceTimeUseCase 自動儲存
├─ test: MoveToLocationUseCase 正確切換地點
├─ test: MoveToLocationUseCase 檢查事件觸發
├─ test: CheckEventTriggerUseCase 過濾邏輯正確
├─ test: CheckEventTriggerUseCase 檢查 repeatable 欄位
└─ test: CheckEventTriggerUseCase 條件判斷正確
```

**交付物**:
- 3 個 UseCase（80% 測試覆蓋）

### M2.4 - Event Execution Use Cases (5 天)

**實作內容**:

```dart
// ExecuteEventUseCase
class ExecuteEventUseCase {
  final ScenarioRepository _scenarioRepository;
  
  ExecuteEventUseCase(this._scenarioRepository);
  
  Future<EventDefinition> execute(String eventId) async {
    return await _scenarioRepository.getEvent(eventId);
  }
}

// SelectChoiceUseCase
class SelectChoiceUseCase {
  final GameStateRepository _repository;
  final ConditionEvaluator _conditionEvaluator;
  
  SelectChoiceUseCase(this._repository, this._conditionEvaluator);
  
  Future<GameState> execute(
    GameState currentState,
    String eventId,
    ChoiceOption choice,
  ) async {
    // 1. 驗證選項可選
    final canSelect = choice.conditions.every((condition) {
      return _conditionEvaluator.evaluate(condition, currentState);
    });
    
    if (!canSelect) {
      throw ChoiceNotAvailableException(choice.text);
    }
    
    // 2. 執行 effects
    var newState = currentState;
    for (final effect in choice.effects) {
      newState = _applyEffect(newState, effect);
    }
    
    // 3. 記錄事件觸發
    newState = newState.recordEvent(eventId);
    
    // 4. 自動儲存
    await _repository.save(newState);
    
    return newState;
  }
  
  GameState _applyEffect(GameState state, String effect) {
    // 解析 effect 語法並執行
    // "relationship.eva += 5"
    // "stat.charm += 10"
    // "flag:met_alice"
    // "unlock_event:eva_office_2"
    // "item:red_dress += 1"
    // ...
  }
}

// GetAvailableChoicesUseCase
class GetAvailableChoicesUseCase {
  final ConditionEvaluator _conditionEvaluator;
  
  GetAvailableChoicesUseCase(this._conditionEvaluator);
  
  List<ChoiceOption> execute(
    GameState state,
    EventDefinition event,
  ) {
    return event.choices.where((choice) {
      return choice.conditions.every((condition) {
        return _conditionEvaluator.evaluate(condition, state);
      });
    }).toList();
  }
}
```

**測試清單**:
```
├─ test: ExecuteEventUseCase 正確取得事件內容
├─ test: SelectChoiceUseCase 驗證條件
├─ test: SelectChoiceUseCase 執行各種 effects
├─ test: SelectChoiceUseCase 記錄事件
├─ test: SelectChoiceUseCase 自動儲存
├─ test: GetAvailableChoicesUseCase 過濾邏輯正確
└─ test: GetAvailableChoicesUseCase 顯示/隱藏不可選項
```

**交付物**:
- 3 個 UseCase（80% 測試覆蓋）

### M2.5 - Player Action Use Cases (3 天)

**實作內容**:

```dart
// ModifyStatUseCase
class ModifyStatUseCase {
  final GameStateRepository _repository;
  
  ModifyStatUseCase(this._repository);
  
  Future<GameState> execute(
    GameState currentState,
    StatType type,
    int delta,
  ) async {
    final newPlayer = currentState.player.modifyStat(type, delta);
    final newState = currentState.modifyPlayer(newPlayer);
    
    await _repository.save(newState);
    return newState;
  }
}

// EquipItemUseCase
class EquipItemUseCase {
  final GameStateRepository _repository;
  
  EquipItemUseCase(this._repository);
  
  Future<GameState> execute(
    GameState currentState,
    String itemId,
  ) async {
    final newPlayerState = currentState.playerState.equipItem(itemId);
    final newState = currentState.modifyPlayerState(newPlayerState);
    
    await _repository.save(newState);
    return newState;
  }
}

// UseItemUseCase
class UseItemUseCase {
  final GameStateRepository _repository;
  
  UseItemUseCase(this._repository);
  
  Future<GameState> execute(
    GameState currentState,
    String itemId,
  ) async {
    // 1. 減少 Inventory 數量
    final newInventory = currentState.playerState.inventory.removeItem(itemId, 1);
    var newState = currentState.modifyPlayerState(
      currentState.playerState.copyWith(inventory: newInventory),
    );
    
    // 2. 執行道具效果（根據 itemId）
    newState = _applyItemEffect(newState, itemId);
    
    // 3. 自動儲存
    await _repository.save(newState);
    
    return newState;
  }
  
  GameState _applyItemEffect(GameState state, String itemId) {
    // 根據道具定義執行效果
    // 例如：紅酒 → stamina -= 10, mood += 20, add status:drunk
  }
}
```

**測試清單**:
```
├─ test: ModifyStatUseCase 正確修改數值
├─ test: EquipItemUseCase 成功裝備
├─ test: EquipItemUseCase 未擁有時拋錯
├─ test: UseItemUseCase 成功使用
└─ test: UseItemUseCase 數量不足時拋錯
```

**交付物**:
- 3 個 UseCase（80% 測試覆蓋）

**Phase 2 完成標準**:
```
✅ UseCase 只協調，不實作業務邏輯
✅ 關鍵路徑有測試（happy path + 1-2 個 error case）
✅ 事務邊界清楚（哪些操作會自動存檔）
```

---

## Phase 3: Infrastructure Layer (2.5 週)

**目標**: 實作資料存取、YAML 解析

### M3.1 - Persistence: SharedPreferences (4 天)

**實作內容**:

```dart
// GameStateRepository 實作
class SharedPrefsGameStateRepository implements GameStateRepository {
  final SharedPreferences _prefs;
  static const String _saveKey = 'game_save_v1';
  
  SharedPrefsGameStateRepository(this._prefs);
  
  @override
  Future<GameState?> load() async {
    try {
      final jsonString = _prefs.getString(_saveKey);
      if (jsonString == null) return null;
      
      final json = jsonDecode(jsonString);
      final saveData = SaveData.fromJson(json);
      
      // 版本遷移（如果需要）
      if (saveData.version < 1) {
        return _migrateV0ToV1(saveData);
      }
      
      return saveData.gameState;
    } catch (e) {
      // Fail fast: 資料損壞直接拋錯
      throw CorruptedSaveDataException(e.toString());
    }
  }
  
  @override
  Future<void> save(GameState state) async {
    try {
      final saveData = SaveData(version: 1, gameState: state);
      final jsonString = jsonEncode(saveData.toJson());
      await _prefs.setString(_saveKey, jsonString);
    } catch (e) {
      throw SaveFailedException(e.toString());
    }
  }
  
  @override
  Future<void> delete() async {
    await _prefs.remove(_saveKey);
  }
  
  GameState _migrateV0ToV1(SaveData oldData) {
    // 版本升級邏輯
    // 例如：v0 沒有 playerState，需要建立預設值
  }
}
```

**測試清單**:
```
├─ test: save and load (round-trip)
├─ test: 處理 corrupted JSON (拋錯，fail fast)
├─ test: 處理不存在的 key (回傳 null)
└─ test: 版本 migration (模擬 v0 → v1)
```

**交付物**:
- GameStateRepository 實作（80% 測試覆蓋）
- 版本遷移文件

### M3.2 - Scenario Loading: YAML (5 天)

**YAML 格式設計**:

```yaml
# locations.yaml
locations:
  - id: home
    name: 家
    background: assets/images/bg_home.png
    interactables:
      - id: kitchen
        name: 廚房
        type: location
        click_area: [0.1, 0.3, 0.3, 0.6]  # [x, y, w, h]
      
      - id: bedroom
        name: 臥室
        type: location
        click_area: [0.6, 0.3, 0.3, 0.6]

  - id: office
    name: 公司
    background: assets/images/bg_office.png
    interactables:
      - id: eva
        name: Eva
        type: npc
        click_area: [0.4, 0.2, 0.2, 0.6]

# events.yaml
events:
  - id: eva_office_1
    repeatable: false
    triggers:
      time:
        weekday: tuesday
      location: office
      conditions:
        - "!event:eva_office_1"
    
    content:
      - type: dialogue
        speaker: eva
        text: "嗨，新來的？"
      
      - type: choice
        options:
          - text: "是的"
            conditions: []
            effects:
              - "relationship.eva += 5"
              - "unlock_event:eva_office_2"
          
          - text: "忽略她"
            conditions: []
            effects: []

  - id: eva_office_2
    repeatable: false
    triggers:
      time:
        weekday: friday
      location: office
      conditions:
        - "event:eva_office_1"
        - "charm >= 30"
    
    content:
      - type: dialogue
        speaker: eva
        text: "週末有空嗎？"
      
      - type: choice
        options:
          - text: "有啊，怎麼了？"
            conditions:
              - "charm >= 50"
            effects:
              - "relationship.eva += 10"
              - "unlock_event:eva_date_1"
          
          - text: "抱歉，我很忙"
            conditions: []
            effects:
              - "relationship.eva -= 5"
```

**實作內容**:

```dart
// YamlScenarioRepository 實作
class YamlScenarioRepository implements ScenarioRepository {
  final Map<String, LocationDefinition> _locationCache = {};
  final Map<String, EventDefinition> _eventCache = {};
  bool _isLoaded = false;
  
  YamlScenarioRepository();
  
  Future<void> loadAll() async {
    if (_isLoaded) return;
    
    try {
      // 載入所有 YAML 檔案（啟動時）
      await _loadLocations();
      await _loadEvents();
      
      _isLoaded = true;
    } catch (e) {
      // Fail fast: YAML 格式錯誤直接拋錯
      throw MalformedScenarioException(e.toString());
    }
  }
  
  Future<void> _loadLocations() async {
    final yamlString = await rootBundle.loadString('assets/data/locations.yaml');
    final yaml = loadYaml(yamlString);
    
    for (final locationData in yaml['locations']) {
      final location = LocationDefinition.fromYaml(locationData);
      _locationCache[location.id] = location;
    }
  }
  
  Future<void> _loadEvents() async {
    final yamlString = await rootBundle.loadString('assets/data/events.yaml');
    final yaml = loadYaml(yamlString);
    
    for (final eventData in yaml['events']) {
      final event = EventDefinition.fromYaml(eventData);
      _eventCache[event.id] = event;
    }
  }
  
  @override
  Future<LocationDefinition> getLocation(String locationId) async {
    await loadAll();
    
    final location = _locationCache[locationId];
    if (location == null) {
      throw LocationNotFoundException(locationId);
    }
    
    return location;
  }
  
  @override
  Future<EventDefinition> getEvent(String eventId) async {
    await loadAll();
    
    final event = _eventCache[eventId];
    if (event == null) {
      throw EventNotFoundException(eventId);
    }
    
    return event;
  }
  
  @override
  Future<List<EventDefinition>> getAllEvents() async {
    await loadAll();
    return _eventCache.values.toList();
  }
}
```

**測試清單**:
```
├─ test: 載入 valid YAML
├─ test: 處理 malformed YAML (拋錯，fail fast)
└─ test: caching 正常運作
```

**交付物**:
- YAML 格式文件 + 範例檔案
- YamlScenarioRepository 實作（80% 測試覆蓋）

### M3.3 - Condition Evaluator (6 天)

**支援的 Condition 語法**:

```
1. Stat: "charm >= 50", "stamina > 30"
2. Relationship: "eva.affection >= 50"
3. Event: "event:eva_office_1", "!event:nina_rent"
4. Flag: "flag:met_alice", "!flag:drunk"
5. Time: "time:tuesday", "time:weekend", "time:morning"
6. Location: "location:home", "location:home_kitchen"
7. Item: "item:red_dress", "wearing:red_dress"
8. Status: "status:drunk"
9. 複合條件: "charm >= 50 && event:eva_office_1"
```

**實作內容**:

```dart
// ConditionEvaluator 實作
class ConditionEvaluator {
  // Pattern matching 用正則表示式
  static final _patterns = {
    r'^(\w+)\s*([><=]+)\s*(\d+): _evalStat,
    r'^(\w+)\.(\w+)\s*([><=]+)\s*(\d+): _evalRelationship,
    r'^(!)?event:(\w+): _evalEvent,
    r'^(!)?flag:(\w+): _evalFlag,
    r'^time:(\w+): _evalTime,
    r'^location:([\w_]+): _evalLocation,
    r'^(!)?item:(\w+): _evalItem,
    r'^(!)?wearing:(\w+): _evalWearing,
    r'^(!)?status:(\w+): _evalStatus,
  };
  
  bool evaluate(String condition, GameState state) {
    // 處理複合條件（AND）
    if (condition.contains('&&')) {
      final parts = condition.split('&&').map((s) => s.trim());
      return parts.every((part) => _evaluateSingle(part, state));
    }
    
    return _evaluateSingle(condition, state);
  }
  
  bool _evaluateSingle(String condition, GameState state) {
    for (final entry in _patterns.entries) {
      final regex = RegExp(entry.key);
      final match = regex.firstMatch(condition);
      
      if (match != null) {
        return entry.value(match, state);
      }
    }
    
    // 無效語法
    throw InvalidConditionException(condition);
  }
  
  static bool _evalStat(RegExpMatch match, GameState state) {
    final statName = match.group(1)!;
    final operator = match.group(2)!;
    final value = int.parse(match.group(3)!);
    
    final statValue = _getStatValue(state.player.stats, statName);
    return _compare(statValue, operator, value);
  }
  
  static bool _evalRelationship(RegExpMatch match, GameState state) {
    final characterId = match.group(1)!;
    final field = match.group(2)!; // 應該是 "affection"
    final operator = match.group(3)!;
    final value = int.parse(match.group(4)!);
    
    if (field != 'affection') {
      throw InvalidConditionException('Only "affection" field supported');
    }
    
    final relationship = state.player.relationships.get(characterId);
    final affection = relationship?.affection ?? 0;
    
    return _compare(affection, operator, value);
  }
  
  static bool _evalEvent(RegExpMatch match, GameState state) {
    final negate = match.group(1) == '!';
    final eventId = match.group(2)!;
    
    final hasTriggered = state.eventHistory.hasTriggered(eventId);
    return negate ? !hasTriggered : hasTriggered;
  }
  
  static bool _evalFlag(RegExpMatch match, GameState state) {
    final negate = match.group(1) == '!';
    final flagKey = match.group(2)!;
    
    final hasFlag = state.player.getFlag(flagKey) == true;
    return negate ? !hasFlag : hasFlag;
  }
  
  static bool _evalTime(RegExpMatch match, GameState state) {
    final timeType = match.group(1)!;
    
    // time:monday, time:tuesday, ...
    if (_isWeekday(timeType)) {
      return state.time.weekday.name == timeType;
    }
    
    // time:weekend
    if (timeType == 'weekend') {
      return state.time.isWeekend();
    }
    
    // time:morning, time:afternoon, ...
    if (_isTimeOfDay(timeType)) {
      return state.time.period.name == timeType;
    }
    
    throw InvalidConditionException('Unknown time type: $timeType');
  }
  
  static bool _evalLocation(RegExpMatch match, GameState state) {
    final locationId = match.group(1)!;
    return state.location.id == locationId;
  }
  
  static bool _evalItem(RegExpMatch match, GameState state) {
    final negate = match.group(1) == '!';
    final itemId = match.group(2)!;
    
    final hasItem = state.playerState.hasItem(itemId);
    return negate ? !hasItem : hasItem;
  }
  
  static bool _evalWearing(RegExpMatch match, GameState state) {
    final negate = match.group(1) == '!';
    final itemId = match.group(2)!;
    
    final isWearing = state.playerState.isWearing(itemId);
    return negate ? !isWearing : isWearing;
  }
  
  static bool _evalStatus(RegExpMatch match, GameState state) {
    final negate = match.group(1) == '!';
    final status = match.group(2)!;
    
    final hasStatus = state.playerState.hasStatus(status);
    return negate ? !hasStatus : hasStatus;
  }
  
  static bool _compare(int actual, String operator, int expected) {
    switch (operator) {
      case '>': return actual > expected;
      case '>=': return actual >= expected;
      case '<': return actual < expected;
      case '<=': return actual <= expected;
      case '==': return actual == expected;
      default: throw InvalidConditionException('Unknown operator: $operator');
    }
  }
  
  static int _getStatValue(Stats stats, String statName) {
    switch (statName) {
      case 'stamina': return stats.stamina;
      case 'charm': return stats.charm;
      case 'intelligence': return stats.intelligence;
      case 'mood': return stats.mood;
      default: throw InvalidConditionException('Unknown stat: $statName');
    }
  }
  
  static bool _isWeekday(String s) {
    return ['monday', 'tuesday', 'wednesday', 'thursday', 'friday', 'saturday', 'sunday']
      .contains(s);
  }
  
  static bool _isTimeOfDay(String s) {
    return ['morning', 'afternoon', 'evening', 'night'].contains(s);
  }
}
```

**測試清單**:
```
每種 pattern 都要測（9 種 x 2-3 case = 20+ tests）:
├─ test: stat condition (charm >= 50)
├─ test: relationship condition (eva.affection >= 50)
├─ test: event condition (event:xxx, !event:xxx)
├─ test: flag condition (flag:xxx, !flag:xxx)
├─ test: time condition (time:tuesday, time:weekend, time:morning)
├─ test: location condition (location:home)
├─ test: item condition (item:xxx, !item:xxx)
├─ test: wearing condition (wearing:xxx, !wearing:xxx)
├─ test: status condition (status:drunk)
├─ test: AND 邏輯 (charm >= 50 && event:xxx)
├─ test: 無效語法拋錯
└─ test: edge cases (空字串、特殊字元)
```

**交付物**:
- ConditionEvaluator 實作（100% 測試覆蓋）
- Condition 語法文件

**Phase 3 完成標準**:
```
✅ 能載入所有 YAML 檔案
✅ Condition 解析正確率 100%（所有測試通過）
✅ 損壞資料 fail fast（不嘗試修復）
```

---

## Phase 4: Presentation Layer (1.5 週)

**目標**: 建立 ViewModel 層

### M4.1 - Riverpod Provider 架構 (2 天)

**Provider 結構設計**:

```dart
// ========== Repositories (Singleton) ==========
final sharedPreferencesProvider = Provider<SharedPreferences>((ref) {
  throw UnimplementedError('Initialize in main()');
});

final gameStateRepositoryProvider = Provider<GameStateRepository>((ref) {
  final prefs = ref.watch(sharedPreferencesProvider);
  return SharedPrefsGameStateRepository(prefs);
});

final scenarioRepositoryProvider = Provider<ScenarioRepository>((ref) {
  return YamlScenarioRepository();
});

// ========== Use Cases (Singleton) ==========
final loadGameUseCaseProvider = Provider<LoadGameUseCase>((ref) {
  return LoadGameUseCase(ref.read(gameStateRepositoryProvider));
});

final saveGameUseCaseProvider = Provider<SaveGameUseCase>((ref) {
  return SaveGameUseCase(ref.read(gameStateRepositoryProvider));
});

final startNewGameUseCaseProvider = Provider<StartNewGameUseCase>((ref) {
  return StartNewGameUseCase(ref.read(gameStateRepositoryProvider));
});

final advanceTimeUseCaseProvider = Provider<AdvanceTimeUseCase>((ref) {
  return AdvanceTimeUseCase(
    ref.read(gameStateRepositoryProvider),
    ref.read(checkEventTriggerUseCaseProvider),
  );
});

// ... 其他 UseCases

// ========== State (AsyncNotifier) ==========
final gameStateProvider = AsyncNotifierProvider<GameStateNotifier, GameState?>(
  () => GameStateNotifier(),
);

// ========== Derived State (computed) ==========
final currentLocationProvider = Provider<Location?>((ref) {
  final gameState = ref.watch(gameStateProvider).value;
  return gameState?.location;
});

final currentTimeProvider = Provider<GameTime?>((ref) {
  final gameState = ref.watch(gameStateProvider).value;
  return gameState?.time;
});

final playerStatsProvider = Provider<Stats?>((ref) {
  final gameState = ref.watch(gameStateProvider).value;
  return gameState?.player.stats;
});

final availableEventsProvider = FutureProvider<List<EventDefinition>>((ref) async {
  final gameState = ref.watch(gameStateProvider).value;
  if (gameState == null) return [];
  
  final useCase = ref.read(checkEventTriggerUseCaseProvider);
  return useCase.execute(gameState);
});
```

**GameStateNotifier 實作**:

```dart
class GameStateNotifier extends AsyncNotifier<GameState?> {
  @override
  Future<GameState?> build() async {
    // 啟動時自動載入存檔
    final loadUseCase = ref.read(loadGameUseCaseProvider);
    return await loadUseCase.execute();
  }
  
  Future<void> startNewGame() async {
    state = const AsyncLoading();
    
    state = await AsyncValue.guard(() async {
      final useCase = ref.read(startNewGameUseCaseProvider);
      return await useCase.execute();
    });
  }
  
  Future<void> saveGame() async {
    final currentState = state.value;
    if (currentState == null) return;
    
    final useCase = ref.read(saveGameUseCaseProvider);
    await useCase.execute(currentState);
  }
  
  Future<void> deleteSave() async {
    state = const AsyncLoading();
    
    state = await AsyncValue.guard(() async {
      final useCase = ref.read(deleteGameUseCaseProvider);
      await useCase.execute();
      return null;
    });
  }
  
  Future<void> advanceTime() async {
    final currentState = state.value;
    if (currentState == null) return;
    
    state = const AsyncLoading();
    
    state = await AsyncValue.guard(() async {
      final useCase = ref.read(advanceTimeUseCaseProvider);
      return await useCase.execute(currentState);
    });
  }
  
  Future<void> moveTo(Location location) async {
    final currentState = state.value;
    if (currentState == null) return;
    
    state = const AsyncLoading();
    
    state = await AsyncValue.guard(() async {
      final useCase = ref.read(moveToLocationUseCaseProvider);
      return await useCase.execute(currentState, location);
    });
  }
  
  Future<void> modifyStat(StatType type, int delta) async {
    final currentState = state.value;
    if (currentState == null) return;
    
    final useCase = ref.read(modifyStatUseCaseProvider);
    final newState = await useCase.execute(currentState, type, delta);
    state = AsyncData(newState);
  }
}
```

**交付物**:
- Provider 架構文件
- GameStateNotifier 實作

### M4.2 - Screen ViewModels (3 天)

**HomeScreenViewModel**:

```dart
@riverpod
class HomeScreenViewModel extends _$HomeScreenViewModel {
  @override
  HomeScreenState build() {
    final gameState = ref.watch(gameStateProvider);
    
    return HomeScreenState(
      hasSave: gameState.value != null,
      isLoading: gameState.isLoading,
      error: gameState.error?.toString(),
    );
  }
  
  Future<void> startNewGame() async {
    await ref.read(gameStateProvider.notifier).startNewGame();
  }
  
  Future<void> continueGame() async {
    // 已經載入了，直接導航
  }
  
  Future<void> deleteSave() async {
    await ref.read(gameStateProvider.notifier).deleteSave();
  }
}

class HomeScreenState {
  final bool hasSave;
  final bool isLoading;
  final String? error;
  
  HomeScreenState({
    required this.hasSave,
    required this.isLoading,
    this.error,
  });
}
```

**LocationScreenViewModel**:

```dart
@riverpod
class LocationScreenViewModel extends _$LocationScreenViewModel {
  @override
  Future<LocationScreenState> build() async {
    final gameState = ref.watch(gameStateProvider).requireValue;
    
    final scenarioRepo = ref.read(scenarioRepositoryProvider);
    final location = await scenarioRepo.getLocation(gameState.location.id);
    
    final availableEvents = await ref.watch(availableEventsProvider.future);
    
    return LocationScreenState(
      location: location,
      interactables: location.interactables,
      gameState: gameState,
      availableEvents: availableEvents,
    );
  }
  
  Future<void> onInteract(String objectId) async {
    final state = this.state.requireValue;
    
    if (state.availableEvents.isNotEmpty) {
      // 觸發第一個符合條件的事件
      ref.read(currentDialogueProvider.notifier).startEvent(
        state.availableEvents.first.id,
      );
    }
  }
}

class LocationScreenState {
  final LocationDefinition location;
  final List<InteractableObject> interactables;
  final GameState gameState;
  final List<EventDefinition> availableEvents;
  
  LocationScreenState({
    required this.location,
    required this.interactables,
    required this.gameState,
    required this.availableEvents,
  });
}
```

**DialogueOverlay 佈局**:
```
┌─────────────────────────────────┐
│                                 │
│        [背景圖 + 半透明遮罩]      │
│                                 │
│  ┌───────────────────────────┐  │
│  │  [Speaker Name]            │  │ ← 對話框
│  │  對話內容文字...            │  │
│  │                            │  │
│  │  [ 選項 1 ]                │  │
│  │  [ 選項 2 ]                │  │
│  └───────────────────────────┘  │
└─────────────────────────────────┘
```

**交付物**:
- LocationScreen（美化版）
- DialogueOverlay（美化版）

### M5.5 - Production Screens: 數值畫面 (2 天)

**StatsScreen 佈局**:
```
┌─────────────────────────────────┐
│  [< Back]              [Player] │
├─────────────────────────────────┤
│  === 基本屬性 ===               │
│  體力    ████████░░  80/100    │
│  魅力    ██████░░░░  60/100    │
│  智力    ███████░░░  70/100    │
│                                 │
│  === 人際關係 ===               │
│  Eva     💖💖💖💖🤍  80/100   │
│  Nina    💖💖💖🤍🤍  60/100   │
│                                 │
│  === 持有道具 ===               │
│  🍷 紅酒 x2                     │
│  👗 紅色連衣裙 (已裝備)          │
└─────────────────────────────────┘
```

**交付物**:
- StatsScreen（美化版）

**Phase 5 完成標準**:
```
✅ 能在手機上流暢運行（60 FPS）
✅ 所有畫面美化完成（不醜）
✅ 無明顯 UI bug
✅ 基本動畫完成（轉場、對話框淡入淡出）
```

---

## Phase 6: Content & Polish (2 週)

**目標**: 修 bug、補測試、優化體驗

### M6.1 - 製作遊戲內容 (5 天)

**撰寫劇本**:
```
□ 至少 10 個事件（完整可玩的一週）
□ 3-5 個角色（有完整的好感度線）
□ 2-3 個結局（根據數值或選擇）

內容檢查清單：
- [ ] 每個星期都有至少 1 個事件可觸發
- [ ] 每個角色都有至少 3 個事件
- [ ] 有明確的「目標」
- [ ] 有「失敗」條件
```

**製作 YAML 檔案**:
```
□ events.yaml（10+ 事件）
□ locations.yaml（5+ 地點）
□ characters.yaml（3-5 角色）

品質檢查：
- [ ] 所有 condition 語法正確
- [ ] 所有 effect 邏輯正確
- [ ] 沒有拼寫錯誤
- [ ] 沒有死循環
```

**準備美術資源**:
```
□ 背景圖（5+ 張）
□ 角色立繪（3-5 個角色）
□ UI 圖示

美術資源來源：
- AI 生成：Stable Diffusion, Midjourney
- 免費素材：itch.io, OpenGameArt
- 委託：Fiverr（$50-200 USD/張）
```

**交付物**:
- 完整劇本（YAML 檔案）
- 所有美術資源
- 內容測試報告

### M6.2 - 整合測試 (3 天)

**完整遊戲流程測試**:
```
□ 新遊戲 → 玩到結局 → 存檔
□ 讀檔 → 繼續玩 → 另一個結局
□ 測試所有分支選項

測試清單：
- [ ] 所有事件都能正確觸發
- [ ] 所有選項都能正確執行 effects
- [ ] 存檔/讀檔不會丟失資料
- [ ] 時間推進正確
- [ ] 數值變化正確
- [ ] 沒有 crash
```

**邊界條件測試**:
```
□ 數值 = 0, 100（極值）
□ 道具數量 = 0, 999（極值）
□ 連續快速點擊（防呆）
□ 記憶體測試（玩 1 小時不 crash）
```

**多裝置測試**:
```
□ Android（至少 2 台不同機型）
□ iOS（至少 2 台不同機型）
□ 不同螢幕尺寸
```

**交付物**:
- 測試報告（bug 清單 with priority）
- 錄影（展示完整遊戲流程）

### M6.3 - Bug Fixing (4 天)

**Bug 優先級**:
```
P0（遊戲無法進行）：
- Crash
- 卡住無法繼續
- 存檔損壞

P1（邏輯錯誤）：
- 數值計算錯誤
- 事件觸發錯誤
- 選項顯示錯誤

P2（UI 問題）：
- 文字溢出
- 按鈕太小
- 圖片變形

P3（優化，optional）：
- 動畫優化
- 音效
- 特效
```

**交付物**:
- 零 P0/P1 bug
- P2 bug 修復 80% 以上

### M6.4 - UX 改善 (2 天)

**減少等待感**:
```
□ 啟動畫面（如果啟動 > 1 秒）
□ Loading indicator（如果載入 > 0.5 秒）
□ 場景切換動畫（淡入淡出）
```

**防呆機制**:
```
□ 退出時提示「未存檔」
□ 覆蓋存檔時二次確認
□ 刪除存檔時二次確認
□ 關鍵選擇時提示
```

**輔助功能（optional）**:
```
□ 文字速度調整
□ 自動前進
□ 跳過已讀對話
□ 回顧歷史對話
```

**交付物**:
- UX 改善清單（before/after 對比）

**Phase 6 完成標準**:
```
✅ 零 P0/P1 bug
✅ 能完整玩通（至少 1 小時遊戲時間）
✅ 朋友能玩懂（不需要你解釋規則）
✅ 基本防呆機制完整
```

---

## Phase 7: Release Preparation (1 週)

**目標**: 準備上架資料

### M7.1 - App Store Metadata (2 天)

**基本資訊**:
```
□ App 名稱（唯一，檢查是否被佔用）
□ Bundle ID / Package Name（唯一）
□ 版本號：1.0.0
□ 分級：根據內容決定（12+? 17+?）
□ 類別：Games > Simulation / Adventure
```

**文案撰寫**:
```
□ App 描述（英文 + 繁中）
  - 3-5 句話說明遊戲內容
  - 強調特色
  - 避免誇大

□ 關鍵字（10 個）
  - 英文：visual novel, dating sim, life simulation
  - 繁中：戀愛模擬, 養成遊戲, 劇情遊戲

□ 更新說明
  - v1.0.0: "初次發布"
```

**截圖準備**:
```
iOS 要求（5-10 張）：
- 6.5" (iPhone 14 Pro Max): 1284 x 2778
- 5.5" (iPhone 8 Plus): 1242 x 2208

Android 要求（2-8 張）：
- Phone: 1080 x 1920 (最小)
- Tablet: 1200 x 1920 (optional)

截圖內容：
- 主畫面
- 對話畫面
- 選項畫面
- 數值畫面
- 結局畫面（optional）
```

**App Icon 設計**:
```
□ 尺寸：1024 x 1024
□ 格式：PNG, 無透明背景
□ 設計原則：
  - 簡潔（小圖示也能辨識）
  - 代表遊戲主題
  - 配色鮮豔

製作方式：
- 自己設計（Figma, Canva）
- AI 生成（Midjourney, DALL-E）
- 委託設計（Fiverr, $5-50）
```

**Feature Graphic (Android)**:
```
□ 尺寸：1024 x 500
□ 內容：遊戲標題 + 主視覺 + 標語
```

**交付物**:
- App Store Connect 草稿
- Google Play Console 草稿
- 所有截圖 + Icon + Feature Graphic

### M7.2 - 法律文件 (1 天)

**隱私政策（必要）**:

必須回答的問題：
```
1. 收集什麼資料？
   - 遊戲存檔（存在本地，不上傳）
   - Crash logs（如果有整合 Firebase Crashlytics）
   - 廣告 ID（如果有廣告）

2. 如何使用資料？
   - 存檔：儲存遊戲進度
   - Crash logs：修復 bug
   - 廣告 ID：投放個人化廣告

3. 是否分享給第三方？
   - 否（如果沒有廣告/分析）
   - 是（說明分享給誰：Google Analytics, AdMob）

4. 如何刪除資料？
   - 刪除 App 即可刪除所有本地資料
```

**隱私政策範本**:
- 用產生器：https://www.privacypolicygenerator.info/
- 或參考其他獨立遊戲的隱私政策

**語言**：英文 + 繁中（兩個版本）

**託管**：
- GitHub Pages（免費）
- Google Sites（免費）
- 自己的網站

**服務條款（optional，建議有）**:

內容：
```
- 使用限制（禁止逆向工程、作弊）
- 免責聲明（遊戲內容為虛構，不代表真實觀點）
- 著作權聲明（遊戲內容受著作權保護）
- 終止條款（開發者保留終止服務的權利）
```

範本：https://www.termsofservicegenerator.net/

**版權聲明**:

需要聲明的內容：
```
1. 自己的原創內容
   - "© 2025 [Your Name/Studio]. All rights reserved."

2. 第三方資源
   - 音樂：標註作者 + 授權方式（CC BY, CC0）
   - 圖片：標註來源 + 授權
   - 字型：檢查授權（商業使用需付費?）
   - 程式庫：檢查 License（MIT, Apache 2.0 通常無問題）
```

**檢查清單**:
```
- [ ] 所有美術資源都有合法授權
- [ ] 所有音樂/音效都有合法授權
- [ ] 所有字型都允許商業使用
- [ ] 所有第三方程式庫的 License 都相容
```

**交付物**:
- 隱私政策網頁（含 URL）
- 服務條款網頁（含 URL）
- 版權聲明清單

### M7.3 - Build & Submit (4 天)

**iOS Build 準備**:

```bash
1. 設定 Bundle ID
   - 格式：com.yourname.gamename
   - 必須在 Apple Developer 帳號註冊（$99/年）

2. 設定 Signing
   - 使用 Xcode 自動管理（推薦）
   - 或手動建立 Certificate + Provisioning Profile

3. 設定版本號
   - Version: 1.0.0
   - Build: 1

4. 產生 .ipa
   flutter build ios --release
   open ios/Runner.xcworkspace
   # 在 Xcode: Product > Archive > Distribute App

5. 上傳到 App Store Connect
   - 使用 Xcode 內建上傳工具
   - 或使用 Transporter app

6. 內部測試（TestFlight）
   - 邀請自己的 Apple ID
   - 在真實裝置上測試
   - 確認沒有 crash、功能正常
```

**常見問題**:
- Missing compliance: 在 App Store Connect 設定「No encryption」
- Invalid binary: 檢查 Info.plist 設定

**Android Build 準備**:

```bash
1. 設定 Package Name
   - 格式：com.yourname.gamename
   - 在 android/app/build.gradle 修改

2. 產生 Keystore（簽名檔）
   keytool -genkey -v -keystore ~/upload-keystore.jks \
     -keyalg RSA -keysize 2048 -validity 10000 \
     -alias upload
   
   ⚠️ 重要：備份 keystore，遺失無法再上傳更新！

3. 設定 Signing
   - 建立 android/key.properties
   - 修改 android/app/build.gradle 加入 signing config

4. 設定版本號
   - versionName: "1.0.0"
   - versionCode: 1

5. 產生 .aab
   flutter build appbundle --release

6. 上傳到 Google Play Console
   - 選擇 Internal Testing track
   - 上傳 .aab 檔案
   - 填寫 Release notes

7. 內部測試
   - 加入測試者 email
   - 在真實裝置上透過 Play Store 安裝
   - 確認沒有 crash、功能正常
```

**常見問題**:
- 64-bit compliance: 確保有編譯 arm64-v8a（Flutter 預設有）
- Target API level: 必須符合 Google Play 要求（目前是 API 33+）

**正式送審**:

iOS 送審流程：
```
1. 在 App Store Connect 建立「App 版本」
2. 填寫所有 Metadata（描述、截圖、分級）
3. 選擇 TestFlight 的 build
4. 送交審核（Submit for Review）
5. 等待審核（通常 1-3 天）

審核重點：
- App 功能完整（不能是 demo 或 beta）
- 沒有 crash
- 隱私政策連結有效
- 分級正確
```

Android 送審流程：
```
1. 在 Google Play Console 填寫所有 Metadata
2. 設定定價（免費 or 付費）
3. 選擇發布國家/地區
4. 從 Internal Testing → Production
5. 提交審核
6. 等待審核（通常數小時到 1 天）

審核重點：
- 內容分級問卷（誠實填寫）
- 隱私政策
- 目標對象（兒童 or 成人？）
- 廣告（如果有）
```

**交付物**:
- iOS .ipa 檔案（已上傳到 TestFlight）
- Android .aab 檔案（已上傳到 Internal Testing）
- 審核中的 App Store / Play Store 連結

**Phase 7 完成標準**:
```
✅ TestFlight / Internal Testing 能正常安裝
✅ 所有必要文件齊全
✅ 送審成功（等待審核通過）
```

---

## 追蹤工具與範本

### progress.md（推薦使用）

```markdown
# Game Development Progress

Last Updated: 2025-10-20

## Current Status
- **Current Phase**: Phase 1 - Domain Layer
- **Current Milestone**: M1.3 - Event History System
- **Progress**: 30% (Phase 1: 60%)
- **Estimated Completion**: 2025-07-01

## Phase 0: Validation ✅ (Completed: 2025-10-15)
- [x] M0.1 - Visual Prototype (2 days)
- [x] M0.2 - Event System Validation (3 days)

## Phase 1: Domain Layer 🚧 (In Progress, Target: 2025-02-10)
- [x] M1.1 - Time System (4 days) ✅ 2025-10-18
- [x] M1.2 - Location System (3 days) ✅ 2025-10-20
- [ ] M1.3 - Event History System (3 days) 🚧 WIP
- [ ] M1.4 - Player Stats System (4 days)
- [ ] M1.5 - Inventory & State System (5 days)
- [ ] M1.6 - Player Aggregate (4 days)
- [ ] M1.7 - GameState Entity (5 days)

## Phase 2-7: ...

## Blockers
- [ ] 需要確認 Condition 語法複雜度
- [ ] 需要找美術資源

## Next Actions
1. 完成 EventHistory Value Object
2. 寫 EventHistory 測試
3. 開始 M1.4 Player Stats System

## Time Tracking
- Week 1 (2025-10-06): 25 hours (Phase 0)
- Week 2 (2025-10-13): 22 hours (Phase 1.1, 1.2)
- Week 3 (2025-10-20): 18 hours (Phase 1.3) 🚧

## Decisions Log
見 decisions.md

## Lessons Learned
- GameTime 不用 DateTime 是對的
- Location 扁平化比層級好
- 測試時間總是被低估（要加 50% buffer）
```

### decisions.md

```markdown
# decisions.md

## D001: Condition 語法設計 (2025-10-15)

### 問題
要支援多複雜的 Condition？

### 選項
1. 自創 DSL（支援 and/or/not）
2. 用正則表示式（只支援 4 種 pattern）
3. 用 Dart code（完全不用 DSL）

### 決定
選 2（正則表示式 + 4 種 pattern）

### 原因
- 90% 情況夠用
- 避免重新發明程式語言
- 複雜邏輯用 Dart DSL（compile-time check）

### 何時重新評估
- 發現 10+ 個場景需要複雜 Condition
- YAML 可讀性變差

---

## D002: Repository 抽象層 (2025-10-16)

### 問題
要不要為 Repository 加 interface？

### 選項
1. 加 abstract class（預留擴充性）
2. 直接用 concrete class（YAGNI）

### 決定
選 2（concrete class）

### 原因
- 3 個月內不會換 DB
- SharedPreferences 夠用
- 真的要換時再抽象（成本不高）

### 何時重新評估
- 需要多端同步（Firebase）
- 需要離線優先（SQLite）
```

### tech-debt.md

```markdown
# tech-debt.md

## High Priority (Blocking)
無

## Medium Priority (應該修但不急)

### TD001: Condition Evaluator 錯誤訊息不明確
- **問題**：當 condition 語法錯誤時，只回傳 "Invalid condition"
- **影響**：寫劇本時難以 debug
- **建議解決**：加入詳細錯誤訊息
- **估計時間**：2 小時
- **記錄日期**：2025-10-22

### TD002: GameState 序列化沒有版本檢查
- **問題**：SaveData 沒有 version field
- **影響**：將來難以升級
- **建議解決**：加入 version field + migration 機制
- **估計時間**：4 小時
- **記錄日期**：2025-10-23

## Low Priority (Nice to have)

### TD003: Stats 只支援 4 個屬性
- **問題**：Stats 寫死 stamina, charm, intelligence, mood
- **影響**：如果要加新屬性需要改 Domain Layer
- **建議解決**：改用 Map<String, int>（但會失去類型安全）
- **估計時間**：6 小時
- **記錄日期**：2025-10-24
- **備註**：先不做，等真的需要 5 種以上屬性再說

## Resolved

### TD000: Location 用層級結構難以序列化 ✅
- **解決方案**：改用扁平化 id（home_kitchen）
- **解決日期**：2025-10-20
- **花費時間**：3 小時
```

---

## 開發建議

### 每日工作節奏（每天 2-3 小時）

```
第 1 小時：實作
- 專注寫 code
- 不看手機、不查資料
- 目標：完成 1 個小任務

第 2 小時：測試 + 重構
- 寫測試
- 跑測試
- 小範圍重構

第 3 小時（optional）：學習 + 規劃
- 查資料
- 更新 progress.md
- 規劃明天的任務

週末（4-6 小時）：
- 完成 1 個 Milestone
- 整合測試
- Code Review
```

### 避免常見陷阱

**陷阱 1：過度設計**
```
症狀：
- 「這個以後可能要改，先預留擴充性」
- 「要不要加個 interface？」

解藥：
- 問自己：「3 個月內會用到嗎？」
- 答案是 No → 不做
```

**陷阱 2：完美主義**
```
症狀：
- 重構同一段 code 5 次
- 糾結變數命名 30 分鐘

解藥：
- 設定時間上限
- 用 TODO 標註「以後改善」
- 80% 就夠好
```

**陷阱 3：跳過測試**
```
症狀：
- 「先把功能做出來，測試以後補」

後果：
- Phase 6 發現一堆 bug
- 不敢重構

解藥：
- 關鍵路徑一定要測
- 寫測試比 debug 快
```

**陷阱 4：忽略 MVP**
```
症狀：
- 「要做就做完整版」
- 「先做完所有功能再測試」

後果：
- 開發 6 個月發現核心玩法不好玩

解藥：
- 先做最小可玩版本
- 讓朋友試玩
- 根據回饋決定要不要繼續
```

### Milestone 完成檢查清單

每完成一個 Milestone，檢查：

```markdown
## Milestone 完成檢查

### 功能完整性
- [ ] 核心功能能正常運作
- [ ] 沒有 placeholder 或 TODO
- [ ] 通過手動測試

### 程式碼品質
- [ ] 關鍵路徑有單元測試
- [ ] 測試都通過（flutter test）
- [ ] 沒有 lint warning（flutter analyze）
- [ ] 程式碼格式化（flutter format）

### 文件
- [ ] 更新 progress.md
- [ ] 如有重要決策，記錄在 decisions.md
- [ ] 如有 API 變更，更新相關文件

### Git
- [ ] Commit message 清楚
- [ ] 每個 Milestone 至少 1 個 commit
- [ ] 推送到遠端（備份）

### 技術債
- [ ] 記錄已知問題（tech-debt.md）
- [ ] 評估影響（blocking? 可延後?）
```

### 週回顧範本

```markdown
# Week X Review (2025-MM-DD)

## 完成的工作
- ✅ M1.1 - Time System
- ✅ M1.2 - Location System
- 🔄 M1.3 - Event History (80% 完成)

## 時間統計
- 計畫：20 小時
- 實際：18 小時
- 差異原因：週三加班，沒時間寫 code

## 遇到的問題
1. GameTime 的週數計算邏輯比預期複雜
   - 解決：改用 (totalDays / 7) + 1
   - 學到：邊界條件要仔細考慮

## 下週計畫
- [ ] 完成 M1.3
- [ ] 完成 M1.4
- [ ] 開始 M1.5

## 需要調整的地方
- 測試時間被低估，下次要加 30% buffer
- 每天開工前先看 progress.md

## 士氣 😊😐😞
😊 進度順利，對架構設計滿意
```

---

## 緊急情況處理

### 情況 1：進度落後 2 週以上

**診斷**：
- 哪個 Phase 落後？
- 為什麼落後？

**對策**：
1. Scope Cut（砍功能）
   - 先做 MVP（10 個事件 → 5 個事件）
   - 延後非核心功能

2. 尋求幫助
   - 委託美術
   - 問社群

3. 調整時間投入
   - 週末多投入 2 小時

### 情況 2：發現架構設計錯誤

**診斷**：
- 錯誤在哪一層？
- 影響範圍多大？
- 修正成本多高？

**對策**：
1. 評估是否真的要改
   - 能用 workaround 繞過嗎？
   - 不改會有什麼後果？

2. 如果必須改
   - 寫下重構計畫
   - 確保測試覆蓋
   - 分批重構

### 情況 3：失去動力

**症狀**：
- 不想打開專案
- 覺得永遠做不完

**對策**：
1. 休息 1-2 天
2. 回顧初衷
3. 降低目標（今天只做 1 小時）
4. 慶祝小成就
5. 尋求回饋（給朋友試玩）

---

## 行動清單

### 今天（2 小時）
```
□ 複製 progress.md 範本到專案
□ 複製 decisions.md 範本到專案
□ 設定 Git repository
□ 確認開發環境正常（flutter doctor）
□ 決定要做「完整版」還是「簡化 MVP」
```

### 本週（10-15 小時）
```
□ 完成 Phase 0（驗證核心概念）
  - 做出 1 個能跑的場景
  - 確認技術可行性
  
□ 開始 Phase 1.1（Time System）
  - 實作 GameTime Value Object
  - 寫測試
```

### 本月（40-60 小時）
```
□ 完成 Phase 1（Domain Layer）
  - 所有 ViewModel**:

```dart
@riverpod
class CurrentDialogue extends _$CurrentDialogue {
  @override
  DialogueState? build() {
    return null;
  }
  
  Future<void> startEvent(String eventId) async {
    final scenarioRepo = ref.read(scenarioRepositoryProvider);
    final event = await scenarioRepo.getEvent(eventId);
    
    state = DialogueState(
      eventId: eventId,
      event: event,
      currentNodeIndex: 0,
    );
  }
  
  void advance() {
    if (state == null) return;
    
    final newIndex = state!.currentNodeIndex + 1;
    
    if (newIndex >= state!.event.content.length) {
      // 對話結束
      state = null;
    } else {
      state = state!.copyWith(currentNodeIndex: newIndex);
    }
  }
  
  Future<void> selectChoice(ChoiceOption choice) async {
    if (state == null) return;
    
    final gameState = ref.read(gameStateProvider).requireValue;
    final useCase = ref.read(selectChoiceUseCaseProvider);
    
    await useCase.execute(gameState, state!.eventId, choice);
    
    // 對話結束
    state = null;
  }
}

class DialogueState {
  final String eventId;
  final EventDefinition event;
  final int currentNodeIndex;
  
  DialogueState({
    required this.eventId,
    required this.event,
    required this.currentNodeIndex,
  });
  
  ContentNode get currentNode => event.content[currentNodeIndex];
  bool get isEnd => currentNodeIndex >= event.content.length - 1;
}
```

**StatsScreenViewModel**:

```dart
@riverpod
class StatsScreenViewModel extends _$StatsScreenViewModel {
  @override
  StatsScreenState build() {
    final gameState = ref.watch(gameStateProvider).requireValue;
    
    return StatsScreenState(
      playerName: gameState.player.name,
      week: gameState.time.week,
      weekday: gameState.time.weekday.name,
      stamina: gameState.player.stats.stamina,
      charm: gameState.player.stats.charm,
      intelligence: gameState.player.stats.intelligence,
      mood: gameState.player.stats.mood,
      relationships: _buildRelationships(gameState.player.relationships),
      inventory: _buildInventory(gameState.playerState.inventory),
      flags: gameState.player.flags,
    );
  }
  
  List<RelationshipDisplay> _buildRelationships(Relationships relationships) {
    // 轉換成 UI 需要的格式
  }
  
  List<ItemDisplay> _buildInventory(Inventory inventory) {
    // 轉換成 UI 需要的格式
  }
  
  Future<void> equipItem(String itemId) async {
    final gameState = ref.read(gameStateProvider).requireValue;
    final useCase = ref.read(equipItemUseCaseProvider);
    
    await useCase.execute(gameState, itemId);
  }
}

class StatsScreenState {
  final String playerName;
  final int week;
  final String weekday;
  final int stamina;
  final int charm;
  final int intelligence;
  final int mood;
  final List<RelationshipDisplay> relationships;
  final List<ItemDisplay> inventory;
  final Map<String, dynamic> flags;
  
  StatsScreenState({...});
}
```

**交付物**:
- 4 個 ViewModel（80% 測試覆蓋）

### M4.3 - Error Handling & Loading States (2 天)

**統一錯誤處理**:

```dart
// AppError sealed class
sealed class AppError {
  final String message;
  AppError(this.message);
  
  String toUserFriendlyMessage();
}

class SaveError extends AppError {
  SaveError(String message) : super(message);
  
  @override
  String toUserFriendlyMessage() => '存檔失敗，請檢查儲存空間';
}

class LoadError extends AppError {
  LoadError(String message) : super(message);
  
  @override
  String toUserFriendlyMessage() => '讀檔失敗，存檔可能已損壞';
}

class InvalidDataError extends AppError {
  InvalidDataError(String message) : super(message);
  
  @override
  String toUserFriendlyMessage() => '資料格式錯誤';
}
```

**交付物**:
- 錯誤處理文件
- AppError 定義

**Phase 4 完成標準**:
```
✅ ViewModel 不包含業務邏輯（只呼叫 UseCase）
✅ 錯誤處理一致
✅ Loading 狀態清楚（不會卡 UI）
```

---

## Phase 5: UI Layer (2.5 週)

**目標**: 從醜版進化到可用版

### M5.1 - Debug Screens（醜版）(4 天)

詳細實作內容請參考完整文件中的 Debug Screens 部分。

**交付物**:
- DebugHomeScreen
- DebugLocationScreen
- DebugDialogueScreen
- DebugStatsScreen

### M5.2 - 視覺設計與參考 (2 天)

**任務清單**:
```
□ 找 3 款參考遊戲
  - 截圖 5-10 個關鍵畫面
  - 標註佈局結構
  - 分析配色、字體、間距
  
□ 選定設計方向
  - 選 1 款作為主要參考
  - 列出要借鑑的元素
  
□ 繪製 Wireframe（紙筆或 Figma）
  - 主畫面
  - 地點畫面
  - 對話框
  - 數值畫面
```

**交付物**:
- 參考遊戲分析文件
- Wireframe 草稿

### M5.3 - Production Screens: 主畫面 (2 天)

**HomeScreen 美化**:
- 背景圖
- 標題設計
- 按鈕美化（圓角、陰影、漸層）
- hover/press 效果
- 角色立繪（optional）

### M5.4 - Production Screens: 地點畫面 (4 天)

**LocationScreen 佈局**:
```
┌─────────────────────────────────┐
│  [Time] [Stats]          [Menu] │ ← Top bar
├─────────────────────────────────┤
│                                 │
│        [背景圖]                  │
│                                 │
│    💼   🪴   👩‍💼              │ ← 互動物件
│                                 │
│                     🧍‍♀️        │ ← 玩家立繪
│                                 │
└─────────────────────────────────┘
```

**DialogueOverlay 佈局**:
```
┌─────────────────────────────────┐
│                                 │
│        [背景圖 + 半透明遮罩]      │
│                                 │
│  ┌───────────────────────────┐  │
│  │  [Speaker Name]            │  │ ← 對話框
│  │  對話內容文字...            │  │
│  │                            │  │
│  │  [ 選項 1 ]                │  │
│  │  [ 選項 2 ]                │  │
│  └───────────────────────────┘  │
└─────────────────────────────────┘
```

**實作內容**:

```dart
// LocationScreen 實作
class LocationScreen extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final location = ref.watch(currentLocationProvider);
    final gameState = ref.watch(gameStateProvider).value!;
    
    return Scaffold(
      body: Stack(
        fit: StackFit.expand,
        children: [
          // 1. 背景圖
          Image.asset(
            location.backgroundImage,
            fit: BoxFit.cover,
          ),
          
          // 2. 互動物件（可點擊區域）
          ..._buildInteractables(context, ref, location),
          
          // 3. 玩家立繪（optional）
          Positioned(
            right: 20,
            bottom: 20,
            child: Image.asset(
              'assets/images/player_idle.png',
              height: 200,
            ),
          ),
          
          // 4. Top bar（時間、數值、選單）
          _buildTopBar(context, ref, gameState),
          
          // 5. 對話框（如果有觸發事件）
          if (ref.watch(currentDialogueProvider) != null)
            _buildDialogueOverlay(context, ref),
        ],
      ),
    );
  }
  
  List<Widget> _buildInteractables(
    BuildContext context,
    WidgetRef ref,
    LocationDefinition location,
  ) {
    return location.interactables.map((obj) {
      // click_area: [x, y, w, h] (相對螢幕的百分比)
      final area = obj.clickArea;
      
      return Positioned(
        left: MediaQuery.of(context).size.width * area[0],
        top: MediaQuery.of(context).size.height * area[1],
        width: MediaQuery.of(context).size.width * area[2],
        height: MediaQuery.of(context).size.height * area[3],
        child: GestureDetector(
          onTap: () => _onInteract(context, ref, obj),
          child: Container(
            // Debug 模式顯示邊框
            decoration: BoxDecoration(
              border: kDebugMode 
                ? Border.all(color: Colors.red, width: 2)
                : null,
            ),
            child: Center(
              // 顯示物件名稱（hover 時）或圖示
              child: Text(
                obj.name,
                style: TextStyle(
                  color: Colors.white,
                  fontSize: 14,
                  backgroundColor: Colors.black54,
                ),
              ),
            ),
          ),
        ),
      );
    }).toList();
  }
  
  Widget _buildTopBar(BuildContext context, WidgetRef ref, GameState state) {
    return Positioned(
      top: 0,
      left: 0,
      right: 0,
      child: Container(
        padding: EdgeInsets.symmetric(horizontal: 16, vertical: 8),
        decoration: BoxDecoration(
          gradient: LinearGradient(
            begin: Alignment.topCenter,
            end: Alignment.bottomCenter,
            colors: [
              Colors.black.withOpacity(0.7),
              Colors.transparent,
            ],
          ),
        ),
        child: SafeArea(
          child: Row(
            children: [
              // 時間顯示
              _buildTimeDisplay(state.time),
              
              SizedBox(width: 16),
              
              // 數值顯示（簡化版）
              _buildStatsDisplay(state.player.stats),
              
              Spacer(),
              
              // 選單按鈕
              IconButton(
                icon: Icon(Icons.menu, color: Colors.white),
                onPressed: () => _showMenu(context, ref),
              ),
            ],
          ),
        ),
      ),
    );
  }
  
  Widget _buildTimeDisplay(GameTime time) {
    return Container(
      padding: EdgeInsets.symmetric(horizontal: 12, vertical: 6),
      decoration: BoxDecoration(
        color: Colors.black54,
        borderRadius: BorderRadius.circular(20),
      ),
      child: Row(
        children: [
          Icon(Icons.calendar_today, size: 16, color: Colors.white),
          SizedBox(width: 4),
          Text(
            '${_weekdayName(time.weekday)} ${_periodName(time.period)}',
            style: TextStyle(color: Colors.white, fontSize: 12),
          ),
        ],
      ),
    );
  }
  
  Widget _buildStatsDisplay(Stats stats) {
    return Row(
      children: [
        _buildStatIcon(Icons.favorite, stats.stamina, Colors.red),
        SizedBox(width: 8),
        _buildStatIcon(Icons.auto_awesome, stats.charm, Colors.pink),
      ],
    );
  }
  
  Widget _buildStatIcon(IconData icon, int value, Color color) {
    return Container(
      padding: EdgeInsets.symmetric(horizontal: 8, vertical: 4),
      decoration: BoxDecoration(
        color: Colors.black54,
        borderRadius: BorderRadius.circular(15),
      ),
      child: Row(
        children: [
          Icon(icon, size: 16, color: color),
          SizedBox(width: 4),
          Text(
            '$value',
            style: TextStyle(color: Colors.white, fontSize: 12),
          ),
        ],
      ),
    );
  }
  
  Widget _buildDialogueOverlay(BuildContext context, WidgetRef ref) {
    final dialogue = ref.watch(currentDialogueProvider)!;
    
    return GestureDetector(
      onTap: () {
        // 點擊空白處關閉（如果是可跳過的對話）
      },
      child: Container(
        color: Colors.black.withOpacity(0.5),
        child: Align(
          alignment: Alignment.bottomCenter,
          child: Container(
            width: double.infinity,
            margin: EdgeInsets.all(16),
            padding: EdgeInsets.all(20),
            decoration: BoxDecoration(
              color: Colors.white,
              borderRadius: BorderRadius.circular(16),
              boxShadow: [
                BoxShadow(
                  color: Colors.black26,
                  blurRadius: 10,
                  offset: Offset(0, 5),
                ),
              ],
            ),
            child: Column(
              mainAxisSize: MainAxisSize.min,
              crossAxisAlignment: CrossAxisAlignment.start,
              children: [
                // Speaker 名稱
                if (dialogue.speaker != null)
                  Padding(
                    padding: EdgeInsets.only(bottom: 8),
                    child: Text(
                      dialogue.speaker!,
                      style: TextStyle(
                        fontSize: 18,
                        fontWeight: FontWeight.bold,
                        color: Colors.blue.shade700,
                      ),
                    ),
                  ),
                
                // 對話文字
                Text(
                  dialogue.text,
                  style: TextStyle(fontSize: 16),
                ),
                
                SizedBox(height: 16),
                
                // 選項或繼續按鈕
                if (dialogue.choices.isNotEmpty)
                  ...dialogue.choices.map((choice) => Padding(
                    padding: EdgeInsets.only(bottom: 8),
                    child: SizedBox(
                      width: double.infinity,
                      child: ElevatedButton(
                        onPressed: choice.isAvailable
                          ? () => _selectChoice(ref, choice)
                          : null,
                        child: Text(choice.text),
                      ),
                    ),
                  )).toList()
                else
                  Align(
                    alignment: Alignment.centerRight,
                    child: TextButton(
                      onPressed: () => ref.read(currentDialogueProvider.notifier).advance(),
                      child: Text('繼續 ▶'),
                    ),
                  ),
              ],
            ),
          ),
        ),
      ),
    );
  }
  
  void _onInteract(BuildContext context, WidgetRef ref, InteractableObject obj) {
    // 檢查是否有可觸發的事件
    final availableEvents = ref.read(availableEventsProvider).value ?? [];
    
    if (availableEvents.isNotEmpty) {
      // 觸發第一個符合條件的事件
      ref.read(currentDialogueProvider.notifier).startEvent(availableEvents.first.id);
    } else {
      // 沒有事件，顯示預設訊息
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text('沒什麼特別的...')),
      );
    }
  }
  
  void _selectChoice(WidgetRef ref, ChoiceOption choice) {
    final gameState = ref.read(gameStateProvider).value!;
    final useCase = ref.read(selectChoiceUseCaseProvider);
    
    // 執行選擇
    useCase.execute(gameState, currentEventId, choice);
    
    // 關閉對話框
    ref.read(currentDialogueProvider.notifier).close();
  }
  
  void _showMenu(BuildContext context, WidgetRef ref) {
    showModalBottomSheet(
      context: context,
      builder: (context) => Container(
        padding: EdgeInsets.all(16),
        child: Column(
          mainAxisSize: MainAxisSize.min,
          children: [
            ListTile(
              leading: Icon(Icons.person),
              title: Text('數值'),
              onTap: () {
                Navigator.pop(context);
                Navigator.push(context, MaterialPageRoute(
                  builder: (_) => StatsScreen(),
                ));
              },
            ),
            ListTile(
              leading: Icon(Icons.save),
              title: Text('存檔'),
              onTap: () {
                Navigator.pop(context);
                ref.read(gameStateProvider.notifier).saveGame();
                ScaffoldMessenger.of(context).showSnackBar(
                  SnackBar(content: Text('存檔成功')),
                );
              },
            ),
            ListTile(
              leading: Icon(Icons.settings),
              title: Text('設定'),
              onTap: () {
                Navigator.pop(context);
                // TODO: 設定畫面
              },
            ),
            ListTile(
              leading: Icon(Icons.exit_to_app),
              title: Text('返回主選單'),
              onTap: () {
                Navigator.pop(context);
                _confirmExit(context, ref);
              },
            ),
          ],
        ),
      ),
    );
  }
  
  void _confirmExit(BuildContext context, WidgetRef ref) {
    showDialog(
      context: context,
      builder: (context) => AlertDialog(
        title: Text('確認離開'),
        content: Text('尚未存檔的進度將會遺失，確定要離開嗎？'),
        actions: [
          TextButton(
            onPressed: () => Navigator.pop(context),
            child: Text('取消'),
          ),
          TextButton(
            onPressed: () {
              Navigator.pop(context);
              Navigator.pushAndRemoveUntil(
                context,
                MaterialPageRoute(builder: (_) => HomeScreen()),
                (route) => false,
              );
            },
            child: Text('確定', style: TextStyle(color: Colors.red)),
          ),
        ],
      ),
    );
  }
  
  String _weekdayName(Weekday weekday) {
    const names = {
      Weekday.monday: '周一',
      Weekday.tuesday: '周二',
      Weekday.wednesday: '周三',
      Weekday.thursday: '周四',
      Weekday.friday: '周五',
      Weekday.saturday: '周六',
      Weekday.sunday: '周日',
    };
    return names[weekday]!;
  }
  
  String _periodName(TimeOfDay period) {
    const names = {
      TimeOfDay.morning: '早上',
      TimeOfDay.afternoon: '下午',
      TimeOfDay.evening: '晚上',
      TimeOfDay.night: '深夜',
    };
    return names[period]!;
  }
}
```

**交付物**:
- LocationScreen（美化版）
- DialogueOverlay（美化版）

### M5.5 - Production Screens: 數值畫面 (2 天)

**StatsScreen 佈局**:
```
┌─────────────────────────────────┐
│  [< Back]              [Player] │
├─────────────────────────────────┤
│  === 基本屬性 ===               │
│  體力    ████████░░  80/100    │
│  魅力    ██████░░░░  60/100    │
│  智力    ███████░░░  70/100    │
│                                 │
│  === 人際關係 ===               │
│  Eva     💖💖💖💖🤍  80/100   │
│  Nina    💖💖💖🤍🤍  60/100   │
│                                 │
│  === 持有道具 ===               │
│  🍷 紅酒 x2                     │
│  👗 紅色連衣裙 (已裝備)          │
└─────────────────────────────────┘
```

**實作內容**:

```dart
class StatsScreen extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final state = ref.watch(statsScreenProvider);
    
    return Scaffold(
      appBar: AppBar(
        title: Text('角色資訊'),
        actions: [
          // 可選：匯出存檔資訊（debug 用）
        ],
      ),
      body: ListView(
        padding: EdgeInsets.all(16),
        children: [
          // 基本資訊
          _buildSection(
            title: '基本資訊',
            children: [
              _buildInfoRow('姓名', state.playerName),
              _buildInfoRow('遊戲時間', 'Week ${state.week}, ${state.weekday}'),
            ],
          ),
          
          SizedBox(height: 24),
          
          // 基本屬性
          _buildSection(
            title: '基本屬性',
            children: [
              _buildStatBar('體力', state.stamina, Colors.red),
              SizedBox(height: 12),
              _buildStatBar('魅力', state.charm, Colors.pink),
              SizedBox(height: 12),
              _buildStatBar('智力', state.intelligence, Colors.blue),
            ],
          ),
          
          SizedBox(height: 24),
          
          // 人際關係
          _buildSection(
            title: '人際關係',
            children: state.relationships.map((rel) => Padding(
              padding: EdgeInsets.only(bottom: 12),
              child: _buildRelationshipRow(rel),
            )).toList(),
          ),
          
          SizedBox(height: 24),
          
          // 持有道具
          _buildSection(
            title: '持有道具',
            children: state.inventory.isEmpty
              ? [Text('沒有道具', style: TextStyle(color: Colors.grey))]
              : state.inventory.map((item) => _buildItemRow(item, ref)).toList(),
          ),
          
          SizedBox(height: 24),
          
          // 旗標（Debug 用，可選）
          if (kDebugMode)
            _buildSection(
              title: 'DEBUG: Flags',
              children: state.flags.entries.map((e) => 
                Text('${e.key}: ${e.value}', style: TextStyle(fontSize: 12))
              ).toList(),
            ),
        ],
      ),
    );
  }
  
  Widget _buildSection({required String title, required List<Widget> children}) {
    return Column(
      crossAxisAlignment: CrossAxisAlignment.start,
      children: [
        Text(
          title,
          style: TextStyle(
            fontSize: 20,
            fontWeight: FontWeight.bold,
            color: Colors.blue.shade700,
          ),
        ),
        SizedBox(height: 12),
        Container(
          width: double.infinity,
          padding: EdgeInsets.all(16),
          decoration: BoxDecoration(
            color: Colors.grey.shade100,
            borderRadius: BorderRadius.circular(12),
          ),
          child: Column(
            crossAxisAlignment: CrossAxisAlignment.start,
            children: children,
          ),
        ),
      ],
    );
  }
  
  Widget _buildInfoRow(String label, String value) {
    return Row(
      mainAxisAlignment: MainAxisAlignment.spaceBetween,
      children: [
        Text(label, style: TextStyle(fontSize: 16)),
        Text(value, style: TextStyle(fontSize: 16, fontWeight: FontWeight.bold)),
      ],
    );
  }
  
  Widget _buildStatBar(String label, int value, Color color) {
    return Column(
      crossAxisAlignment: CrossAxisAlignment.start,
      children: [
        Row(
          mainAxisAlignment: MainAxisAlignment.spaceBetween,
          children: [
            Text(label, style: TextStyle(fontSize: 16)),
            Text('$value/100', style: TextStyle(fontSize: 14, color: Colors.grey)),
          ],
        ),
        SizedBox(height: 6),
        ClipRRect(
          borderRadius: BorderRadius.circular(10),
          child: LinearProgressIndicator(
            value: value / 100,
            minHeight: 20,
            backgroundColor: Colors.grey.shade300,
            valueColor: AlwaysStoppedAnimation<Color>(color),
          ),
        ),
      ],
    );
  }
  
  Widget _buildRelationshipRow(RelationshipDisplay rel) {
    // 用愛心表示好感度（每 20 點一顆愛心）
    final hearts = (rel.affection / 20).floor();
    final emptyHearts = 5 - hearts;
    
    return Row(
      mainAxisAlignment: MainAxisAlignment.spaceBetween,
      children: [
        Text(rel.characterName, style: TextStyle(fontSize: 16)),
        Row(
          children: [
            ...List.generate(hearts, (_) => Text('💖', style: TextStyle(fontSize: 16))),
            ...List.generate(emptyHearts, (_) => Text('🤍', style: TextStyle(fontSize: 16))),
            SizedBox(width: 8),
            Text('${rel.affection}/100', style: TextStyle(fontSize: 14, color: Colors.grey)),
          ],
        ),
      ],
    );
  }
  
  Widget _buildItemRow(ItemDisplay item, WidgetRef ref) {
    return ListTile(
      contentPadding: EdgeInsets.zero,
      leading: Text(item.icon, style: TextStyle(fontSize: 24)),
      title: Text(item.name),
      trailing: Row(
        mainAxisSize: MainAxisSize.min,
        children: [
          if (item.count > 1)
            Text('x${item.count}', style: TextStyle(color: Colors.grey)),
          
          if (item.isEquipped)
            Padding(
              padding: EdgeInsets.only(left: 8),
              child: Chip(
                label: Text('已裝備', style: TextStyle(fontSize: 12)),
                backgroundColor: Colors.green.shade100,
              ),
            )
          else if (item.isEquippable)
            IconButton(
              icon: Icon(Icons.check_circle_outline),
              onPressed: () => ref.read(statsScreenProvider.notifier).equipItem(item.id),
            ),
        ],
      ),
    );
  }
}
```

**交付物**:
- StatsScreen（美化版）

**Phase 5 完成標準**:
```
✅ 能在手機上流暢運行（60 FPS）
✅ 所有畫面美化完成（不醜）
✅ 無明顯 UI bug（按鈕能按、文字能讀、不會溢出）
✅ 基本動畫完成（轉場、對話框淡入淡出）
```

---

## Phase 6: Content & Polish (2 週)

**目標**: 修 bug、補測試、優化體驗

### M6.1 - 製作遊戲內容 (5 天)

**撰寫劇本**:
```
□ 至少 10 個事件（完整可玩的一週）
□ 3-5 個角色（有完整的好感度線）
□ 2-3 個結局（根據數值或選擇）

內容檢查清單：
- [ ] 每個星期都有至少 1 個事件可觸發
- [ ] 每個角色都有至少 3 個事件
- [ ] 有明確的「目標」（玩家知道要做什麼）
- [ ] 有「失敗」條件（體力歸零、時間到）
```

**製作 YAML 檔案**:
```
□ events.yaml（10+ 事件）
□ locations.yaml（5+ 地點）
□ characters.yaml（3-5 角色）

品質檢查：
- [ ] 所有 condition 語法正確
- [ ] 所有 effect 邏輯正確
- [ ] 沒有拼寫錯誤
- [ ] 沒有死循環（事件 A 觸發 B，B 觸發 A）
```

**準備美術資源**:
```
□ 背景圖（5+ 張，可用 AI 生成或免費素材）
□ 角色立繪（3-5 個角色，可用 AI 生成）
□ UI 圖示（按鈕、圖示等）

美術資源來源：
- AI 生成：Stable Diffusion, Midjourney
- 免費素材：itch.io, OpenGameArt
- 委託：Fiverr, 巴哈姆特接案版（預算 $50-200 USD/張）
```

**交付物**:
- 完整劇本（YAML 檔案）
- 所有美術資源
- 內容測試報告（玩過一輪，記錄 bug）

### M6.2 - 整合測試 (3 天)

**完整遊戲流程測試**:
```
□ 新遊戲 → 玩到結局 → 存檔
□ 讀檔 → 繼續玩 → 另一個結局
□ 測試所有分支選項

測試清單：
- [ ] 所有事件都能正確觸發
- [ ] 所有選項都能正確執行 effects
- [ ] 存檔/讀檔不會丟失資料
- [ ] 時間推進正確（不會卡住）
- [ ] 數值變化正確（有 clamp）
- [ ] 沒有 crash
```

**邊界條件測試**:
```
□ 數值 = 0, 100（極值）
□ 道具數量 = 0, 999（極值）
□ 連續快速點擊（防呆）
□ 記憶體測試（玩 1 小時不 crash）
```

**多裝置測試**:
```
□ Android（至少 2 台不同機型）
□ iOS（至少 2 台不同機型）
□ 不同螢幕尺寸（小螢幕、大螢幕、平板）
```

**交付物**:
- 測試報告（bug 清單 with priority）
- 錄影（展示完整遊戲流程）

### M6.3 - Bug Fixing (4 天)

**修復 P0 bug（遊戲無法進行）**:
```
- Crash
- 卡住無法繼續
- 存檔損壞
```

**修復 P1 bug（邏輯錯誤）**:
```
- 數值計算錯誤
- 事件觸發錯誤
- 選項顯示錯誤
```

**修復 P2 bug（UI 問題）**:
```
- 文字溢出
- 按鈕太小
- 圖片變形
```

**P3 優化（optional）**:
```
- 動畫優化
- 音效（如果有）
- 特效
```

**交付物**:
- 零 P0/P1 bug
- P2 bug 修復 80% 以上

### M6.4 - UX 改善 (2 天)

**減少等待感**:
```
□ 啟動畫面（如果啟動 > 1 秒）
□ Loading indicator（如果載入 > 0.5 秒）
□ 場景切換動畫（淡入淡出）
```

**防呆機制**:
```
□ 退出時提示「未存檔」（如果有未儲存的進度）
□ 覆蓋存檔時二次確認
□ 刪除存檔時二次確認
□ 關鍵選擇時提示（「此選擇無法撤回」）
```

**輔助功能（optional）**:
```
□ 文字速度調整（對話顯示速度）
□ 自動前進（自動播放對話）
□ 跳過已讀對話
□ 回顧歷史對話（backlog）
```

**交付物**:
- UX 改善清單（before/after 對比）

**Phase 6 完成標準**:
```
✅ 零 P0/P1 bug
✅ 能完整玩通（至少 1 小時遊戲時間）
✅ 朋友能玩懂（不需要你解釋規則）
✅ 基本防呆機制完整
```

---

## Phase 7: Release Preparation (1 週)

**目標**: 準備上架資料

### M7.1 - App Store Metadata (2 天)

**基本資訊**:
```
□ App 名稱（唯一，檢查是否被佔用）
□ Bundle ID / Package Name（唯一）
□ 版本號：1.0.0
□ 分級：根據內容決定（12+? 17+?）
□ 類別：Games > Simulation / Adventure
```

**文案撰寫**:
```
□ App 描述（英文 + 繁中）
  - 3-5 句話說明遊戲內容
  - 強調特色（"multiple endings", "choices matter"）
  - 避免誇大（不要說「最好玩」「最刺激」）

□ 關鍵字（10 個，每個 < 100 字元）
  - 英文：visual novel, dating sim, life simulation, choice-based
  - 繁中：戀愛模擬, 養成遊戲, 劇情遊戲

□ 更新說明（What's New）
  - v1.0.0: "初次發布"
```

**截圖準備**:
```
iOS 要求（5-10 張）：
- 6.5" (iPhone 14 Pro Max): 1284 x 2778
- 5.5" (iPhone 8 Plus): 1242 x 2208

Android 要求（2-8 張）：
- Phone: 1080 x 1920 (最小)
- Tablet: 1200 x 1920 (optional)

截圖內容：
- 主畫面
- 對話畫面（顯示有趣的對話）
- 選項畫面（顯示選擇的重要性）
- 數值畫面
- 結局畫面（optional）

截圖技巧：
- 用真實裝置截圖（不要用模擬器）
- 選擇視覺吸引力高的場景
- 避免文字過小（放大重要文字）
- 可以加邊框/說明文字（但不要擋住遊戲內容）
```

**App Icon 設計**:
```
□ 尺寸：1024 x 1024 (iOS/Android 通用)
□ 格式：PNG, 無透明背景
□ 設計原則：
  - 簡潔（在小圖示上也能辨識）
  - 代表遊戲主題（角色剪影? 關鍵物件?）
  - 配色鮮豔（在 App Store 中顯眼）

製作方式：
- 自己設計（Figma, Canva）
- AI 生成（Midjourney, DALL-E）
- 委託設計（Fiverr, $5-50）
```

**Feature Graphic (Android 限定)**:
```
□ 尺寸：1024 x 500
□ 用途：Google Play 首頁展示
□ 內容：遊戲標題 + 主視覺 + 標語
```

**交付物**:
- App Store Connect 草稿（填完所有資訊）
- Google Play Console 草稿（填完所有資訊）
- 所有截圖 + Icon + Feature Graphic

### M7.2 - 法律文件 (1 天)

**隱私政策（必要）**:

必須回答的問題：
```
1. 收集什麼資料？
   - 遊戲存檔（存在本地，不上傳）
   - Crash logs（如果有整合 Firebase Crashlytics）
   - 廣告 ID（如果有廣告）

2. 如何使用資料？
   - 存檔：儲存遊戲進度
   - Crash logs：修復 bug
   - 廣告 ID：投放個人化廣告

3. 是否分享給第三方？
   - 否（如果沒有廣告/分析）
   - 是（說明分享給誰：Google Analytics, AdMob）

4. 如何刪除資料？
   - 刪除 App 即可刪除所有本地資料
   - 聯絡開發者刪除雲端資料（如果有）
```

**隱私政策範本**:
- 用產生器：https://www.privacypolicygenerator.info/
- 或參考其他獨立遊戲的隱私政策

**語言**：英文 + 繁中（兩個版本）

**託管**：
- GitHub Pages（免費）
- Google Sites（免費）
- 自己的網站

**服務條款（optional，建議有）**:

內容：
```
- 使用限制（禁止逆向工程、作弊）
- 免責聲明（遊戲內容為虛構，不代表真實觀點）
- 著作權聲明（遊戲內容受著作權保護）
- 終止條款（開發者保留終止服務的權利）
```

範本：https://www.termsofservicegenerator.net/

**版權聲明**:

需要聲明的內容：
```
1. 自己的原創內容
   - "© 2025 [Your Name/Studio]. All rights reserved."

2. 第三方資源
   - 音樂：標註作者 + 授權方式（CC BY, CC0）
   - 圖片：標註來源 + 授權
   - 字型：檢查授權（商業使用需付費?）
   - 程式庫：檢查 License（MIT, Apache 2.0 通常無問題）
```

**檢查清單**:
```
- [ ] 所有美術資源都有合法授權
- [ ] 所有音樂/音效都有合法授權
- [ ] 所有字型都允許商業使用
- [ ] 所有第三方程式庫的 License 都相容
```

**交付物**:
- 隱私政策網頁（含 URL）
- 服務條款網頁（含 URL）
- 版權聲明清單（列出所有第三方資源）

### M7.3 - Build & Submit (4 天)

**iOS Build 準備**:

```bash
1. 設定 Bundle ID
   - 格式：com.yourname.gamename
   - 必須在 Apple Developer 帳號註冊（需加入 $99/年 計畫）

2. 設定 Signing
   - 使用 Xcode 自動管理（推薦）
   - 或手動建立 Certificate + Provisioning Profile

3. 設定版本號
   - Version: 1.0.0
   - Build: 1

4. 產生 .ipa
   flutter build ios --release
   open ios/Runner.xcworkspace
   # 在 Xcode: Product > Archive > Distribute App

5. 上傳到 App Store Connect
   - 使用 Xcode 內建上傳工具
   - 或使用 Transporter app

6. 內部測試（TestFlight）
   - 邀請自己的 Apple ID
   - 在真實裝置上測試
   - 確認沒有 crash、功能正常
```

**常見問題**:
- Missing compliance: 在 App Store Connect 設定「No encryption」（如果沒用加密）
- Invalid binary: 檢查 Info.plist 設定（權限描述、Icon）

**Android Build 準備**:

```bash
1. 設定 Package Name
   - 格式：com.yourname.gamename
   - 在 android/app/build.gradle 修改

2. 產生 Keystore（簽名檔）
   keytool -genkey -v -keystore ~/upload-keystore.jks \
     -keyalg RSA -keysize 2048 -validity 10000 \
     -alias upload
   
   ⚠️ 重要：備份 keystore，遺失無法再上傳更新！

3. 設定 Signing
   - 建立 android/key.properties
   ```
   storePassword=<password>
   keyPassword=<password>
   keyAlias=upload
   storeFile=<path to keystore>
   ```
   - 修改 android/app/build.gradle 加入 signing config

4. 設定版本號
   - versionName: "1.0.0"
   - versionCode: 1

5. 產生 .aab
   flutter build appbundle --release

6. 上傳到 Google Play Console
   - 選擇 Internal Testing track
   - 上傳 .aab 檔案
   - 填寫 Release notes

7. 內部測試
   - 加入測試者 email
   - 在真實裝置上透過 Play Store 安裝
   - 確認沒有 crash、功能正常
```

**常見問題**:
- 64-bit compliance: 確保有編譯 arm64-v8a（Flutter 預設有）
- Target API level: 必須符合 Google Play 要求（目前是 API 33+）

**正式送審**:

iOS 送審流程：
```
1. 在 App Store Connect 建立「App 版本」
2. 填寫所有 Metadata（描述、截圖、分級）
3. 選擇 TestFlight 的 build
4. 送交審核（Submit for Review）
5. 等待審核（通常 1-3 天）

審核重點：
- App 功能完整（不能是 demo 或 beta）
- 沒有 crash
- 隱私政策連結有效
- 分級正確（內容符合年齡分級）
```

Android 送審流程：
```
1. 在 Google Play Console 填寫所有 Metadata
2. 設定定價（免費 or 付費）
3. 選擇發布國家/地區
4. 從 Internal Testing → Production
5. 提交審核
6. 等待審核（通常數小時到 1 天）

審核重點：
- 內容分級問卷（誠實填寫）
- 隱私政策
- 目標對象（兒童 or 成人？）
- 廣告（如果有）
```

**交付物**:
- iOS .ipa 檔案（已上傳到 TestFlight）
- Android .aab 檔案（已上傳到 Internal Testing）
- 審核中的 App Store / Play Store 連結

**Phase 7 完成標準**:
```
✅ TestFlight / Internal Testing 能正常安裝
✅ 所有必要文件齊全（隱私政策、服務條款）
✅ 送審成功（等待審核通過）
```

---

## 總時間估算

### 樂觀估算（一切順利）
```
Phase 0: 1 週
Phase 1: 3.5 週
Phase 2: 2.5 週
Phase 3: 2.5 週
Phase 4: 1.5 週
Phase 5: 2.5 週
Phase 6: 2 週
Phase 7: 1 週
───────────────
總計: 16.5 週 ≈ 4 個月
```

### 實際估算（含 buffer 和返工）
```
Phase 0: 1.5 週（技術驗證可能遇到問題）
Phase 1: 4.5 週（Domain 設計會反覆調整）
Phase 2: 3 週（UseCase 測試時間被低估）
Phase 3: 3.5 週（Condition 解析器很複雜）
Phase 4: 2 週（Riverpod 架構需要時間熟悉）
Phase 5: 3.5 週（UI 美化很花時間）
Phase 6: 3 週（內容製作 + bug fixing）
Phase 7: 1.5 週（送審可能被退件）
───────────────
總計: 22.5 週 ≈ 5.5 個月
```

### 如果每週只能投入 20 小時
```
22.5 週 x 40 小時 = 900 小時
900 小時 / 20 小時/週 = 45 週 ≈ 11 個月
```

---

## 最後的行動清單

### 今天（2 小時）
```
□ 複製 progress.md 範本到專案
□ 複製 decisions.md 範本到專案
□ 設定 Git repository（如果還沒有）
□ 確認開發環境正常（flutter doctor）
□ 決定要做「完整版」還是「簡化 MVP」
```

### 本週（10-15 小時）
```
□ 完成 Phase 0（驗證核心概念）
  - 做出 1 個能跑的場景
  - 確認技術可行性
  - 沒有 showstopper
  
□ 開始 Phase 1.1（Time System）
  - 實作 GameTime Value Object
  - 寫測試
```

### 本月（40-60 小時）
```
□ 完成 Phase 1（Domain Layer）
  - 所有 Value Objects
  - Player Aggregate
  - GameState Entity
  
□ 通過 Phase 1 檢查標準
  - 100% immutable
  - 80% 測試覆蓋
  - 零外部依賴
```

### 未來 3 個月
```
□ 完成 Phase 1-4（Domain + Application + Infrastructure + Presentation）
□ 做出醜版但能玩的遊戲（Phase 5.1）
□ 讓 1-2 個朋友試玩
□ 根據回饋決定：繼續完成 or 調整方向
```

---

## 緊急情況處理

### 情況 1：進度落後 2 週以上

**診斷**：
- 哪個 Phase 落後？
- 為什麼落後？（技術問題 or 時間不足）

**對策**：
1. Scope Cut（砍功能）
   - 先做 MVP（10 個事件 → 5 個事件）
   - 延後非核心功能（音效、動畫）
   
2. 尋求幫助
   - 委託美術（如果卡在美術資源）
   - 問社群（如果卡在技術問題）
   
3. 調整時間投入
   - 週末多投入 2 小時
   - 減少其他活動（短期衝刺）

### 情況 2：發現架構設計錯誤

**診斷**：
- 錯誤在哪一層？
- 影響範圍多大？
- 修正成本多高？

**對策**：
1. 評估是否真的要改
   - 能用 workaround 繞過嗎？
   - 不改會有什麼後果？
   
2. 如果必須改
   - 寫下重構計畫（要改哪些檔案）
   - 確保測試覆蓋（改完能驗證）
   - 分批重構（不要一次改太多）
   
3. 從教訓中學習
   - 記錄在 decisions.md
   - 避免重複犯錯

### 情況 3：測試發現重大 bug

**診斷**：
- Bug 在哪一層？
- 何時引入的？（git blame）
- 影響多少功能？

**對策**：
1. 立即修復（如果 blocking）
   - 寫 failing test（重現 bug）
   - 修復 bug
   - 確認測試通過
   
2. 延後修復（如果 non-blocking）
   - 記錄在 tech-debt.md
   - 標註 priority
   - 排入下個 sprint
   
3. 防止再發生
   - 補測試（為什麼沒測到？）
   - Code review（為什麼沒發現？）

### 情況 4：失去動力

**症狀**：
- 不想打開專案
- 覺得永遠做不完
- 開始懷疑為什麼要做這個

**對策**：
1. 休息 1-2 天（真的休息，不要有罪惡感）

2. 回顧初衷
   - 為什麼要做這個遊戲？
   - 做完後會得到什麼？
   
3. 降低目標
   - 今天只做 1 小時
   - 這週只完成 1 個小任務
   
4. 慶祝小成就
   - 完成 1 個 Milestone → 吃頓好的
   - 通過 50 個測試 → 看場電影
   
5. 尋求回饋
   - 給朋友試玩（即使很醜）
   - 發到社群展示進度
   - 正向回饋能提升動力

---

## 常見問題 FAQ

### Q1: 我沒有美術資源怎麼辦？

**答**：
1. 先用 placeholder（灰色方塊）
2. AI 生成（Stable Diffusion, Midjourney）
3. 免費素材（itch.io, OpenGameArt）
4. 委託外包（Fiverr, $50-200 USD）

### Q2: 我不會寫測試怎麼辦？

**答**：
1. 從簡單的開始（equality test）
2. 參考範例（Flutter 官方文件）
3. 只測關鍵路徑（不追求 100%）
4. 測試就是文件（解釋如何使用）

### Q3: 進度落後怎麼辦？

**答**：
1. 砍功能（MVP 優先）
2. 降低品質標準（先能玩，再美化）
3. 尋求幫助（委託、問社群）
4. 調整預期（11 個月 → 18 個月）

### Q4: 如何保持動力？

**答**：
1. 設定小目標（每週 1 個 Milestone）
2. 慶祝小成就（完成 Phase 1 → 吃大餐）
3. 尋求回饋（給朋友試玩）
4. 加入社群（Discord, Reddit）
5. 記錄進度（週回顧、截圖）

### Q5: 需要學什麼前置知識？

**答**：
- **必須**：Dart 基礎、Flutter 基礎
- **建議**：Riverpod、Clean Architecture 概念
- **不需要**：設計模式全部背熟、演算法專家

可以邊做邊學，遇到問題再查。

---

## 成功標準

### MVP（最小可行產品）

```
✅ 能玩完一輪遊戲（5 個事件）
✅ 能存檔讀檔
✅ 沒有 P0/P1 bug
✅ 朋友能玩懂（不需要解釋）

目標：3 個月內完成
```

### v1.0（正式版）

```
✅ 能玩完整遊戲（10+ 個事件）
✅ 多個結局（2-3 個）
✅ 美化完成（不醜）
✅ 通過 App Store / Play Store 審核

目標：6 個月內完成
```

### v1.1+（後續更新）

```
✅ 新增更多事件
✅ 新增更多角色
✅ 加入音效/音樂
✅ 加入成就系統
✅ 加入更多結局

目標：持續更新
```

---

## 資源連結

### 學習資源

**Flutter**:
- 官方文檔：https://docs.flutter.dev/
- Flutter Codelabs：https://codelabs.developers.google.com/
- Flutter by Example：https://flutterbyexample.com/

**Riverpod**:
- 官方文檔：https://riverpod.dev/
- 範例：https://github.com/rrousselGit/riverpod/tree/master/examples
- 教學影片：Andrea Bizzotto YouTube

**Clean Architecture**:
- Uncle Bob 原文：https://blog.cleancoder.com/
- Reso Coder 教學：https://resocoder.com/flutter-clean-architecture-tdd/
- Clean Architecture in Flutter：https://medium.com/flutter-community

### 工具

**設計**:
- Figma：https://www.figma.com/（免費）
- Canva：https://www.canva.com/（免費）
- Adobe XD：https://www.adobe.com/products/xd.html

**美術**:
- AI 生成：
  - Stable Diffusion：https://stability.ai/
  - Midjourney：https://www.midjourney.com/
  - DALL-E：https://openai.com/dall-e-2
- 免費素材：
  - itch.io：https://itch.io/game-assets
  - OpenGameArt：https://opengameart.org/
  - Kenney：https://www.kenney.nl/
- 外包平台：
  - Fiverr：https://www.fiverr.com/
  - Upwork：https://www.upwork.com/

**音效/音樂**:
- 免費音效：https://freesound.org/
- 免費音樂：https://incompetech.com/
- 付費音樂：https://www.epidemicsound.com/

**測試**:
- Flutter 測試文檔：https://docs.flutter.dev/testing
- Mockito：https://pub.dev/packages/mockito

**CI/CD**:
- GitHub Actions：https://github.com/features/actions
- Codemagic：https://codemagic.io/
- Bitrise：https://www.bitrise.io/

### 社群

**Discord**:
- Flutter Community：https://discord.gg/flutter
- Indie Game Devs：搜尋 "indie game dev discord"
- Visual Novel Developers：搜尋 "visual novel discord"

**Reddit**:
- r/FlutterDev：https://www.reddit.com/r/FlutterDev/
- r/gamedev：https://www.reddit.com/r/gamedev/
- r/visualnovels：https://www.reddit.com/r/visualnovels/
- r/IndieDev：https://www.reddit.com/r/IndieDev/

**論壇**:
- Stack Overflow：https://stackoverflow.com/questions/tagged/flutter
- Lemmasoft Forums：https://lemmasoft.renai.us/forums/

**YouTube**:
- Flutter Official：https://www.youtube.com/c/flutterdev
- Reso Coder：https://www.youtube.com/c/ResoCoder
- Andrea Bizzotto：https://www.youtube.com/c/CodeWithAndrea

---

## 最後的話

### Talk is cheap. Show me the code.

你現在有了：
1. **完整的開發地圖**（Phase 0-7，22.5 週）
2. **可追蹤的 Milestone**（每個 2-6 天）
3. **實用的工具**（progress.md, decisions.md）
4. **應對策略**（遇到問題知道怎麼辦）

### 接下來要做什麼？

**不要被計畫嚇到。**

這份文件是「地圖」，不是「監獄」。你可以：
- 調整順序
- 縮減範圍
- 跳過不需要的部分

**重點是開始行動。**

今天就選擇 Phase 0，用 2 小時做出第一個能跑的 prototype。

看到畫面上顯示對話，你就會知道：
**這不是幻想，是可以實現的。**

### Good luck and have fun! 🚀

記住：
- 完成比完美重要
- 進度比計畫重要
- 行動比思考重要

**現在就開始寫 code 吧！**