# Domain Layer 物件設計

## Value Objects

### 1. GameTime（遊戲時間）

```dart
enum Weekday {
  monday,
  tuesday,
  wednesday,
  thursday,
  friday,
  saturday,
  sunday,
}

enum TimeOfDay {
  morning,    // 早上
  afternoon,  // 下午
  evening,    // 晚上
  night,      // 深夜
}

class GameTime {
  final int week;           // 第幾週（1, 2, 3...）
  final Weekday weekday;    // 星期幾
  final TimeOfDay period;   // 時段
  
  const GameTime({
    required this.week,
    required this.weekday,
    required this.period,
  });
  
  // 推進到下一個時段
  // morning → afternoon → evening → night → (隔天) morning
  GameTime advanceToNextPeriod() {
    if (period == TimeOfDay.night) {
      // 深夜 → 隔天早上
      return advanceToNextDay();
    }
    
    // 同一天內推進時段
    final nextPeriodIndex = period.index + 1;
    return GameTime(
      week: week,
      weekday: weekday,
      period: TimeOfDay.values[nextPeriodIndex],
    );
  }
  
  // 推進到隔天早上
  GameTime advanceToNextDay() {
    if (weekday == Weekday.sunday) {
      // 周日 → 下週一
      return GameTime(
        week: week + 1,
        weekday: Weekday.monday,
        period: TimeOfDay.morning,
      );
    }
    
    // 推進到隔天
    final nextWeekdayIndex = weekday.index + 1;
    return GameTime(
      week: week,
      weekday: Weekday.values[nextWeekdayIndex],
      period: TimeOfDay.morning,
    );
  }
  
  // 跳到特定星期
  GameTime advanceToWeekday(Weekday target) {
    if (weekday == target) return this;
    
    var current = this;
    while (current.weekday != target) {
      current = current.advanceToNextDay();
    }
    return current;
  }
  
  // Helper methods
  bool isWeekend() => weekday == Weekday.saturday || weekday == Weekday.sunday;
  bool isWeekday(Weekday day) => weekday == day;
  bool isWorkday() => weekday.index >= 0 && weekday.index <= 4; // 週一～週五
  
  // Equality
  @override
  bool operator ==(Object other) =>
    identical(this, other) ||
    other is GameTime &&
    week == other.week &&
    weekday == other.weekday &&
    period == other.period;
  
  @override
  int get hashCode => Object.hash(week, weekday, period);
  
  // copyWith
  GameTime copyWith({
    int? week,
    Weekday? weekday,
    TimeOfDay? period,
  }) {
    return GameTime(
      week: week ?? this.week,
      weekday: weekday ?? this.weekday,
      period: period ?? this.period,
    );
  }
}
```

**設計決策**：
- ❌ Season 不在 GameTime（透過事件切換，不是時間推進）
- ✅ 簡單的線性時間流動

---

### 2. Season（季節）

```dart
enum Season {
  spring,  // 春
  summer,  // 夏
  autumn,  // 秋
  winter,  // 冬
}
```

**設計決策**：
- Season 是 GameState 的獨立欄位
- 透過事件切換（例如：參加情人節事件後進入春天）
- 不會自動變化

---

### 3. Stats（玩家屬性）

```dart
class Stats {
  final Map<String, int> _values;
  
  const Stats(this._values);
  
  // 預設初始值（所有屬性 50，除了 corruption 從 0 開始）
  factory Stats.initial() {
    return Stats({
      'stamina': 50,
      'charm': 50,
      'intelligence': 50,
      'corruption': 0,        // 從 0 開始（代表純潔）
      'cleanliness': 50,
    });
  }
  
  // 取得屬性值（不存在回傳 0）
  int get(String statName) => _values[statName] ?? 0;
  
  // 修改屬性（自動 clamp 到 0-100）
  Stats modify(String statName, int delta) {
    final current = get(statName);
    final newValue = (current + delta).clamp(0, 100);
    final newValues = Map<String, int>.from(_values);
    newValues[statName] = newValue;
    return Stats(newValues);
  }
  
  // 設定屬性（直接設定值，也會 clamp）
  Stats set(String statName, int value) {
    final newValue = value.clamp(0, 100);
    final newValues = Map<String, int>.from(_values);
    newValues[statName] = newValue;
    return Stats(newValues);
  }
  
  // 取得所有屬性
  Map<String, int> getAll() => Map.unmodifiable(_values);
  
  // 檢查是否有某個屬性
  bool has(String statName) => _values.containsKey(statName);
  
  // Equality
  @override
  bool operator ==(Object other) =>
    identical(this, other) ||
    other is Stats &&
    _mapsEqual(_values, other._values);
  
  @override
  int get hashCode => Object.hashAll(_values.entries);
  
  static bool _mapsEqual(Map<String, int> a, Map<String, int> b) {
    if (a.length != b.length) return false;
    for (var key in a.keys) {
      if (a[key] != b[key]) return false;
    }
    return true;
  }
}
```

**設計決策**：
- ✅ 完全彈性（可以新增任意 stat）
- ✅ 提供預設初始值
- ✅ 提供常數避免拼字錯誤
- ✅ 允許 YAML 定義新的 stat（例如 cooking, courage）

---

### 4. Relationship（單一角色關係）

```dart
class Relationship {
  final String characterId;
  final int affection;    // 好感度 0-100
  final int domination;   // 支配度 0-100
  
  const Relationship({
    required this.characterId,
    required this.affection,
    this.domination = 0,  // 預設 0（無支配）
  });
  
  // 修改好感度
  Relationship modifyAffection(int delta) {
    final newAffection = (affection + delta).clamp(0, 100);
    return Relationship(
      characterId: characterId,
      affection: newAffection,
      domination: domination,
    );
  }
  
  // 修改支配度
  Relationship modifyDomination(int delta) {
    final newDomination = (domination + delta).clamp(0, 100);
    return Relationship(
      characterId: characterId,
      affection: affection,
      domination: newDomination,
    );
  }
  
  // Equality
  @override
  bool operator ==(Object other) =>
    identical(this, other) ||
    other is Relationship &&
    characterId == other.characterId &&
    affection == other.affection &&
    domination == other.domination;
  
  @override
  int get hashCode => Object.hash(characterId, affection, domination);
}

```

---

### 5. Relationships（關係集合）

```dart
class Relationships {
  final Map<String, Relationship> _data;
  
  const Relationships(this._data);
  
  factory Relationships.empty() => const Relationships({});
  
  Relationship? get(String characterId) => _data[characterId];
  
  // 修改好感度（如果不存在則建立新關係，預設 affection=50, domination=0）
  Relationships modifyAffection(String characterId, int delta) {
    final current = get(characterId);
    final newRelationship = current != null
      ? current.modifyAffection(delta)
      : Relationship(
          characterId: characterId,
          affection: (50 + delta).clamp(0, 100),
          domination: 0,
        );
    
    final newData = Map<String, Relationship>.from(_data);
    newData[characterId] = newRelationship;
    return Relationships(newData);
  }
  
  // 修改支配度（如果不存在則建立新關係）
  Relationships modifyDomination(String characterId, int delta) {
    final current = get(characterId);
    final newRelationship = current != null
      ? current.modifyDomination(delta)
      : Relationship(
          characterId: characterId,
          affection: 50,
          domination: delta.clamp(0, 100),
        );
    
    final newData = Map<String, Relationship>.from(_data);
    newData[characterId] = newRelationship;
    return Relationships(newData);
  }
  
  List<Relationship> getAll() => _data.values.toList();
  
  bool has(String characterId) => _data.containsKey(characterId);
  
  @override
  bool operator ==(Object other) =>
    identical(this, other) ||
    other is Relationships &&
    _mapsEqual(_data, other._data);
  
  @override
  int get hashCode => Object.hashAll(_data.entries);
  
  static bool _mapsEqual(Map<String, Relationship> a, Map<String, Relationship> b) {
    if (a.length != b.length) return false;
    for (var key in a.keys) {
      if (a[key] != b[key]) return false;
    }
    return true;
  }
}
```

**設計決策**：
- ✅ 首次互動自動建立關係（預設 50）

---

### 6. Inventory（道具庫存）

```dart
class Inventory {
  final Map<String, int> _items;  // itemId → count
  
  const Inventory(this._items);
  
  // 空庫存
  factory Inventory.empty() => const Inventory({});
  
  // 檢查是否擁有道具
  bool hasItem(String itemId) => _items.containsKey(itemId) && _items[itemId]! > 0;
  
  // 取得道具數量
  int getCount(String itemId) => _items[itemId] ?? 0;
  
  // 新增道具
  Inventory addItem(String itemId, int count) {
    if (count <= 0) return this;
    
    final newItems = Map<String, int>.from(_items);
    final currentCount = getCount(itemId);
    newItems[itemId] = currentCount + count;
    return Inventory(newItems);
  }
  
  // 移除道具
  Inventory removeItem(String itemId, int count) {
    if (count <= 0) return this;
    
    final currentCount = getCount(itemId);
    if (currentCount < count) {
      throw InsufficientItemException(itemId, currentCount, count);
    }
    
    final newItems = Map<String, int>.from(_items);
    final newCount = currentCount - count;
    
    if (newCount == 0) {
      newItems.remove(itemId);
    } else {
      newItems[itemId] = newCount;
    }
    
    return Inventory(newItems);
  }
  
  // 取得所有道具
  Map<String, int> getAll() => Map.unmodifiable(_items);
  
  // Equality
  @override
  bool operator ==(Object other) =>
    identical(this, other) ||
    other is Inventory &&
    _mapsEqual(_items, other._items);
  
  @override
  int get hashCode => Object.hashAll(_items.entries);
  
  static bool _mapsEqual(Map<String, int> a, Map<String, int> b) {
    if (a.length != b.length) return false;
    for (var key in a.keys) {
      if (a[key] != b[key]) return false;
    }
    return true;
  }
}
```

---

### 7. PlayerState（玩家狀態）

```dart
class PlayerState {
  final Inventory inventory;
  final String? wearing;           // 當前裝備的道具 ID
  final Set<String> statusEffects; // 狀態效果
  
  const PlayerState({
    required this.inventory,
    this.wearing,
    required this.statusEffects,
  });
  
  // 初始狀態
  factory PlayerState.initial() {
    return PlayerState(
      inventory: Inventory.empty(),
      wearing: null,
      statusEffects: {},
    );
  }
  
  // === 道具相關 ===
  
  bool hasItem(String itemId) => inventory.hasItem(itemId);
  
  PlayerState addItem(String itemId, int count) {
    return copyWith(inventory: inventory.addItem(itemId, count));
  }
  
  PlayerState removeItem(String itemId, int count) {
    return copyWith(inventory: inventory.removeItem(itemId, count));
  }
  
  // === 裝備相關 ===
  
  bool isWearing(String itemId) => wearing == itemId;
  
  PlayerState equipItem(String itemId) {
    if (!hasItem(itemId)) {
      throw ItemNotFoundException(itemId);
    }
    return copyWith(wearing: itemId);
  }
  
  PlayerState unequipItem() {
    return copyWith(wearing: null);
  }
  
  // === 狀態效果相關 ===
  
  bool hasStatus(String status) => statusEffects.contains(status);
  
  PlayerState addStatus(String status) {
    return copyWith(statusEffects: {...statusEffects, status});
  }
  
  PlayerState removeStatus(String status) {
    final newEffects = Set<String>.from(statusEffects);
    newEffects.remove(status);
    return copyWith(statusEffects: newEffects);
  }
  
  // copyWith
  PlayerState copyWith({
    Inventory? inventory,
    String? wearing,
    Set<String>? statusEffects,
  }) {
    return PlayerState(
      inventory: inventory ?? this.inventory,
      wearing: wearing ?? this.wearing,
      statusEffects: statusEffects ?? this.statusEffects,
    );
  }
  
  // Equality
  @override
  bool operator ==(Object other) =>
    identical(this, other) ||
    other is PlayerState &&
    inventory == other.inventory &&
    wearing == other.wearing &&
    _setsEqual(statusEffects, other.statusEffects);
  
  @override
  int get hashCode => Object.hash(inventory, wearing, Object.hashAll(statusEffects));
  
  static bool _setsEqual(Set<String> a, Set<String> b) {
    if (a.length != b.length) return false;
    return a.containsAll(b);
  }
}
```

---

### 8. EventHistory（事件歷史）

```dart
class EventHistory {
  final Map<String, int> _interactionCount;  // eventId → 互動次數
  
  const EventHistory(this._interactionCount);
  
  factory EventHistory.empty() => const EventHistory({});
  
  // 檢查事件是否已觸發
  bool hasTriggered(String eventId) {
    return _interactionCount.containsKey(eventId) && _interactionCount[eventId]! > 0;
  }
  
  // 取得互動次數
  int getInteractionCount(String eventId) => _interactionCount[eventId] ?? 0;
  
  // 記錄互動
  EventHistory recordInteraction(String eventId) {
    final newCount = Map<String, int>.from(_interactionCount);
    final currentCount = getInteractionCount(eventId);
    newCount[eventId] = currentCount + 1;
    return EventHistory(newCount);
  }
  
  // 取得所有已觸發事件
  List<String> getTriggeredEvents() {
    return _interactionCount.entries
      .where((e) => e.value > 0)
      .map((e) => e.key)
      .toList();
  }
  
  @override
  bool operator ==(Object other) =>
    identical(this, other) ||
    other is EventHistory &&
    _mapsEqual(_interactionCount, other._interactionCount);
  
  @override
  int get hashCode => Object.hashAll(_interactionCount.entries);
  
  static bool _mapsEqual(Map<String, int> a, Map<String, int> b) {
    if (a.length != b.length) return false;
    for (var key in a.keys) {
      if (a[key] != b[key]) return false;
    }
    return true;
  }
}
```

---

## Entities

### 9. Location（地點 - Entity）

```dart
class Location {
  final String id;        // kitchen_stove
  final String? parentId; // kitchen
  
  const Location({
    required this.id,
    this.parentId,
  });
  
  // 檢查是否為某地點的子層級
  bool isChildOf(String locationId) => parentId == locationId;
  
  // 檢查是否為根層級
  bool isRoot() => parentId == null;
  
  // Entity 用 id 比較（不比較 parentId）
  @override
  bool operator ==(Object other) =>
    identical(this, other) ||
    other is Location && id == other.id;
  
  @override
  int get hashCode => id.hashCode;
}
```

**設計決策**：
- ✅ Location 是 Entity（有 parent-child 關係）
- ✅ GameState 只儲存 `currentLocationId`（字串），不儲存整個 Location
- ✅ 名稱、背景圖等資訊放在 Infrastructure 的 `LocationDefinition`

---

### 10. Player（玩家 Aggregate Root）

```dart
class Player {
  final String id;
  final String name;
  final Stats stats;
  final Relationships relationships;
  final Map<String, dynamic> flags;  // 自訂旗標
  
  const Player({
    required this.id,
    required this.name,
    required this.stats,
    required this.relationships,
    required this.flags,
  });
  
  // 建立新玩家
  factory Player.create({
    required String id,
    required String name,
  }) {
    return Player(
      id: id,
      name: name,
      stats: Stats.initial(),
      relationships: Relationships.empty(),
      flags: {},
    );
  }
  
  // === 屬性相關 ===
  
  Player modifyStat(String statName, int delta) {
    return copyWith(stats: stats.modify(statName, delta));
  }
  
  Player setStat(String statName, int value) {
    return copyWith(stats: stats.set(statName, value));
  }
  
  // === 關係相關 ===

  // 修改好感度
  Player modifyAffection(String characterId, int delta) {
    return copyWith(relationships: relationships.modifyAffection(characterId, delta));
  }
  
  // 修改支配度
  Player modifyDomination(String characterId, int delta) {
    return copyWith(relationships: relationships.modifyDomination(characterId, delta));
  }
  
  // === 旗標相關 ===
  
  Player setFlag(String key, dynamic value) {
    final newFlags = Map<String, dynamic>.from(flags);
    newFlags[key] = value;
    return copyWith(flags: newFlags);
  }
  
  dynamic getFlag(String key) => flags[key];
  
  bool hasFlag(String key) => flags.containsKey(key);
  
  Player removeFlag(String key) {
    final newFlags = Map<String, dynamic>.from(flags);
    newFlags.remove(key);
    return copyWith(flags: newFlags);
  }
  
  // copyWith
  Player copyWith({
    String? id,
    String? name,
    Stats? stats,
    Relationships? relationships,
    Map<String, dynamic>? flags,
  }) {
    return Player(
      id: id ?? this.id,
      name: name ?? this.name,
      stats: stats ?? this.stats,
      relationships: relationships ?? this.relationships,
      flags: flags ?? this.flags,
    );
  }
  
  // Equality (Entity 用 id 比較)
  @override
  bool operator ==(Object other) =>
    identical(this, other) ||
    other is Player && id == other.id;
  
  @override
  int get hashCode => id.hashCode;
}
```

---

### 11. GameState（遊戲狀態 Aggregate Root）

```dart
class GameState {
  final Player player;
  final PlayerState playerState;
  final GameTime time;
  final Season season;              // 獨立的季節欄位
  final String currentLocationId;   // 只存 location id
  final EventHistory eventHistory;
  
  const GameState({
    required this.player,
    required this.playerState,
    required this.time,
    required this.season,
    required this.currentLocationId,
    required this.eventHistory,
  });
  
  // 建立初始遊戲狀態
  factory GameState.initial({
    required String playerId,
    required String playerName,
  }) {
    return GameState(
      player: Player.create(id: playerId, name: playerName),
      playerState: PlayerState.initial(),
      time: GameTime(week: 1, weekday: Weekday.monday, period: TimeOfDay.morning),
      season: Season.spring,
      currentLocationId: 'home',
      eventHistory: EventHistory.empty(),
    );
  }
  
  // === 時間相關 ===
  
  // 推進時間（由事件觸發）
  GameState advanceTime() {
    return copyWith(time: time.advanceToNextPeriod());
  }
  
  // 推進到隔天
  GameState advanceToNextDay() {
    return copyWith(time: time.advanceToNextDay());
  }
  
  // === 季節相關 ===
  
  // 切換季節（由事件觸發）
  GameState changeSeason(Season newSeason) {
    return copyWith(season: newSeason);
  }
  
  // === 地點相關 ===
  
  // 移動地點（不推進時間）
  GameState moveTo(String locationId) {
    return copyWith(currentLocationId: locationId);
  }
  
  // === 事件相關 ===
  
  // 記錄互動
  GameState recordInteraction(String eventId) {
    return copyWith(eventHistory: eventHistory.recordInteraction(eventId));
  }
  
  // === 玩家相關 ===
  
  // 修改玩家
  GameState modifyPlayer(Player newPlayer) {
    return copyWith(player: newPlayer);
  }
  
  // 修改玩家狀態
  GameState modifyPlayerState(PlayerState newPlayerState) {
    return copyWith(playerState: newPlayerState);
  }
  
  // === 快捷方法（避免過度串接）===
  
  GameState modifyStat(String statName, int delta) {
    return copyWith(player: player.modifyStat(statName, delta));
  }

  GameState modifyAffection(String characterId, int delta) {
    return copyWith(player: player.modifyAffection(characterId, delta));
  }
  
  GameState modifyDomination(String characterId, int delta) {
    return copyWith(player: player.modifyDomination(characterId, delta));
  }
  
  GameState setFlag(String key, dynamic value) {
    return copyWith(player: player.setFlag(key, value));
  }
  
  GameState addItem(String itemId, int count) {
    return copyWith(playerState: playerState.addItem(itemId, count));
  }
  
  GameState removeItem(String itemId, int count) {
    return copyWith(playerState: playerState.removeItem(itemId, count));
  }
  
  GameState equipItem(String itemId) {
    return copyWith(playerState: playerState.equipItem(itemId));
  }
  
  GameState addStatus(String status) {
    return copyWith(playerState: playerState.addStatus(status));
  }
  
  // copyWith
  GameState copyWith({
    Player? player,
    PlayerState? playerState,
    GameTime? time,
    Season? season,
    String? currentLocationId,
    EventHistory? eventHistory,
  }) {
    return GameState(
      player: player ?? this.player,
      playerState: playerState ?? this.playerState,
      time: time ?? this.time,
      season: season ?? this.season,
      currentLocationId: currentLocationId ?? this.currentLocationId,
      eventHistory: eventHistory ?? this.eventHistory,
    );
  }
  
  // Equality (比較所有欄位)
  @override
  bool operator ==(Object other) =>
    identical(this, other) ||
    other is GameState &&
    player == other.player &&
    playerState == other.playerState &&
    time == other.time &&
    season == other.season &&
    currentLocationId == other.currentLocationId &&
    eventHistory == other.eventHistory;
  
  @override
  int get hashCode => Object.hash(
    player,
    playerState,
    time,
    season,
    currentLocationId,
    eventHistory,
  );
}
```

**設計決策**：
- ✅ Season 獨立欄位，透過事件切換
- ✅ 只儲存 `currentLocationId`，不儲存整個 Location
- ✅ 提供快捷方法避免過度串接

---

## Domain Exceptions

```dart
// 道具不足
class InsufficientItemException implements Exception {
  final String itemId;
  final int has;
  final int needs;
  
  InsufficientItemException(this.itemId, this.has, this.needs);
  
  @override
  String toString() => 'Insufficient item: $itemId (has: $has, needs: $needs)';
}

// 道具不在庫存
class ItemNotFoundException implements Exception {
  final String itemId;
  
  ItemNotFoundException(this.itemId);
  
  @override
  String toString() => 'Item not found in inventory: $itemId';
}

// 無效的數值
class InvalidStatValueException implements Exception {
  final String statName;
  final int value;
  
  InvalidStatValueException(this.statName, this.value);
  
  @override
  String toString() => 'Invalid stat value: $statName = $value';
}

// 無效的時間
class InvalidGameTimeException implements Exception {
  final String message;
  
  InvalidGameTimeException(this.message);
  
  @override
  String toString() => 'Invalid game time: $message';
}

// 地點不存在
class LocationNotFoundException implements Exception {
  final String locationId;
  
  LocationNotFoundException(this.locationId);
  
  @override
  String toString() => 'Location not found: $locationId';
}
```

---

## 設計原則總結

### ✅ 最終確認的設計

1. **GameTime**：week + weekday + period（季節分離）
2. **Season**：獨立欄位，透過事件切換
3. **Location**：Entity with parent-child（linking object）
4. **Stats**：完全彈性 Map + 預設初始值
5. **GameState**：只存 locationId，不存整個 Location

### ✅ Immutability（不可變性）
- 所有物件都是 immutable
- 所有修改都回傳新實例
- 使用 copyWith 模式

### ✅ Rich Domain Model（豐富的領域模型）
- 包含業務邏輯方法
- 不是貧血模型

### ✅ Zero Dependencies（零依賴）
- 不依賴 Flutter、Riverpod
- 只使用 Dart 標準庫

### ✅ Validation（驗證）
- 數值自動 clamp（0-100）
- 操作前檢查
- 無效操作拋出 Domain Exception

---

## 不在 Domain 的內容

這些放在 Infrastructure 層：

```dart
// LocationDefinition (DTO)
class LocationDefinition {
  final String id;
  final String? parentId;
  final String name;
  final String backgroundImage;
  final List<InteractableObject> interactables;
}

// CharacterDefinition (DTO)
class CharacterDefinition {
  final String id;
  final String name;
  final String description;
  final String portraitPath;
}

// EventDefinition (DTO)
class EventDefinition {
  final String id;
  final bool repeatable;
  final bool consumesTime;  // 是否消耗時間
  final TriggerConditions triggers;
  final List<ContentNode> content;
}
```

---