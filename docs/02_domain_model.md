# Domain Model - Core Business Logic

## Architecture Overview

```
┌─────────────────────────────────────────────────┐
│         Domain Layer (No Dependencies)          │
│                                                 │
│  Aggregate Root                                 │
│    • GameState (Root)                           │
│                                                 │
│  Entities                                       │
│    • Player                                     │
│    • Location                                   │
│                                                 │
│  Value Objects (Immutable)                     │
│    • Stats                                      │
│    • GameTime                                   │
│    • Relationships                              │
│    • Relationship                               │
│    • InteractionHistory                         │
│    • InteractionRecord                          │
│    • PlayerState                                │
│    • Inventory                                  │
│                                                 │
│  Domain Events (Phase 2)                       │
│    • StatChangedSignificantly                   │
│    • RelationshipUpdated                        │
│    • NewDayStarted                              │
└─────────────────────────────────────────────────┘
```

---

## GameState (Aggregate Root)

### Structure
```
GameState (Aggregate Root)
├── player: Player
├── playerState: PlayerState
├── time: GameTime
├── season: Season
├── currentLocationId: String
└── interactionHistory: InteractionHistory
```

### Key Behaviors

**advanceTime()**
- Move to next time period
- If night → advance to next day morning
- If Sunday night → increment week
- Apply time passage effects (cleanliness decay)
- Emit NewDayStarted if day changed
- Return new GameState instance

**changeSeason(season)**
- Triggered by special events (not automatic)
- Update season enum
- Affects location availability
- Triggers seasonal event checks
- Return new GameState instance

**moveTo(locationId)**
- Change current location
- No time consumption for movement
- Location validation in Application Layer
- Return new GameState instance

**recordInteraction(pointId, eventId)**
- Record interaction at specific point
- Increment totalInteractions for point
- Add eventId to completedEvents
- Used for dialogue variant selection
- Return new GameState instance

**modifyStat/modifyAffection/modifyDomination**
- Shortcut methods to avoid deep nesting
- Delegate to Player methods
- Return new GameState instance

### Domain Rules (Invariants)

- GameState is the only entry point for state changes
- All child entities/value objects modified through GameState
- Season only changes via explicit changeSeason() call
- currentLocationId always valid (checked in Application Layer)
- InteractionHistory never cleared (permanent record)

---

## Value Objects

### GameTime

```
GameTime (Immutable)
├── week: int (1, 2, 3, ...)
├── weekday: Weekday (enum)
└── period: TimeOfDay (enum)

Weekday Enum:
  • monday, tuesday, wednesday, thursday
  • friday, saturday, sunday

TimeOfDay Enum:
  • morning
  • afternoon
  • evening
  • night

Methods:
  • advance() → GameTime (next period)
  • advanceToNextDay() → GameTime (next day morning)
  • advanceToWeekday(target) → GameTime (skip to weekday)
  • isWeekend() → bool
  • isWorkday() → bool
```

**Design Decision:**
- Season NOT in GameTime (separate field in GameState)
- Week counter tracks game progression
- Simple linear time flow

---

### Season

```
Season (Enum)
  • spring
  • summer
  • autumn
  • winter
```

**Design Decision:**
- Not time-based (won't auto-advance)
- Changed via special events
- Separate from GameTime

---

### Stats

```
Stats (Immutable)
├── _values: Map<String, int>
│
Factory Methods:
  • Stats.initial() → Default 5 attributes
  • get(statName) → int (0 if not exists)
  • modify(statName, delta) → Stats (clamp 0-100)
  • set(statName, value) → Stats (clamp 0-100)
  • has(statName) → bool
  • getAll() → Map<String, int>

Default Attributes:
  • stamina: 50
  • charm: 50
  • intelligence: 50
  • corruption: 0  (starts pure)
  • cleanliness: 50
```

**Validation Rules:**
- All values automatically clamped to 0-100
- Map implementation allows YAML to add new stats
- Immutable (every modification returns new instance)

---

### Relationships & Relationship

```
Relationship (Immutable)
├── characterId: String
├── affection: int (0-100)
└── domination: int (0-100)

Methods:
  • modifyAffection(delta) → Relationship
  • modifyDomination(delta) → Relationship

Computed Properties (Future):
  • level → RelationshipLevel (from affection)
  • isDominated → bool (domination >= 80)
```

```
Relationships (Value Object Collection)
├── _data: Map<String, Relationship>
│
Methods:
  • get(characterId) → Relationship?
  • modifyAffection(characterId, delta) → Relationships
  • modifyDomination(characterId, delta) → Relationships
  • getAll() → List<Relationship>
  • has(characterId) → bool

Initialization:
  • First interaction creates relationship with:
    - affection: 50 + delta (clamped)
    - domination: 0
```

**Design Decision:**
- Dual-value system (affection + domination)
- Domination increased via YAML events/items
- Writers control gain per item (flexible balancing)
- No leverageCount (use domination value instead)

**YAML Integration:**
```yaml
event_get_photo:
  effects:
    - type: modify_domination
      character: alice
      delta: 20  # Photo +20

item_video_evidence:
  on_use:
    - type: modify_domination
      character: bob
      delta: 50  # Video +50 (more powerful)
```

---

### InteractionHistory & InteractionRecord

```
InteractionHistory (Value Object)
├── _records: Map<String, InteractionRecord>
│
Methods:
  • getInteractionCount(pointId) → int
  • hasCompletedEvent(eventId) → bool
  • recordInteraction(pointId, eventId) → InteractionHistory
  • getInteractedPoints() → List<String>
```

```
InteractionRecord (Value Object)
├── pointId: String
├── totalInteractions: int  ⭐ Persisted forever
├── completedEvents: Set<String>
│
Methods:
  • recordEvent(eventId) → InteractionRecord (totalInteractions++)
```

**Critical Design Decision:**
- Tracks interactions **per point**, not per event
- `totalInteractions` stored permanently in save files
- Enables "7th visit to cafe counter" dialogue variants
- Example:
  ```
  Player clicks "cafe_counter" 7 times:
    - InteractionRecord.totalInteractions = 7
    - YAML event uses this for variant selection
  ```

**Usage Pattern:**
```dart
// Application Layer
final count = gameState.getInteractionCount('cafe_counter');
// count = 7

// YAML Definition
event_cafe_counter:
  dialogue_variants:
    1: "Welcome! First time here?"
    2: "Back again?"
    5: "You're a regular now!"  # Used for count >= 5
```

---

### PlayerState

```
PlayerState (Value Object)
├── inventory: Inventory
├── wearing: String? (equipped item ID)
└── statusEffects: Set<String>

Methods:
  • hasItem(itemId) → bool
  • addItem(itemId, count) → PlayerState
  • removeItem(itemId, count) → PlayerState
  • equipItem(itemId) → PlayerState
  • unequipItem() → PlayerState
  • hasStatus(status) → bool
  • addStatus(status) → PlayerState
  • removeStatus(status) → PlayerState
```

```
Inventory (Value Object)
├── _items: Map<String, int>  # itemId → count
│
Methods:
  • hasItem(itemId) → bool
  • getCount(itemId) → int
  • addItem(itemId, count) → Inventory
  • removeItem(itemId, count) → Inventory
  • getAll() → Map<String, int>

Validation:
  • removeItem() throws InsufficientItemException if count < needed
```

---

## Entities

### Player

```
Player (Entity)
├── id: String
├── name: String
├── stats: Stats
├── relationships: Relationships
└── flags: Map<String, dynamic>

Methods:
  • modifyStat(statName, delta) → Player
  • setStat(statName, value) → Player
  • modifyAffection(characterId, delta) → Player
  • modifyDomination(characterId, delta) → Player
  • setFlag(key, value) → Player
  • getFlag(key) → dynamic
  • hasFlag(key) → bool
  • removeFlag(key) → Player

Identity:
  • Equality based on id only
  • Immutable (all modifications return new instance)
```

**Design Decision:**
- Player is an Entity (has identity)
- NOT an Aggregate Root (GameState is)
- Flags use Map<String, dynamic> for YAML flexibility

---

### Location

```
Location (Entity)
├── id: String
└── parentId: String?

Methods:
  • isChildOf(locationId) → bool
  • isRoot() → bool

Identity:
  • Equality based on id only

Hierarchy Example:
  home (root)
  ├─ kitchen (parent: home)
  │  ├─ stove (parent: kitchen)
  │  └─ fridge (parent: kitchen)
  └─ bedroom (parent: home)
```

**Design Decision:**
- Location is Entity (has parent-child relationships)
- GameState stores only currentLocationId (String)
- Full location data (name, background, interaction points) in Infrastructure Layer

---

## Domain Events (Phase 2)

### Purpose
Domain events notify other parts of the system about state changes without coupling.

### Event Types

**StatChangedSignificantly**
```
StatChangedSignificantly
├── playerId: String
├── statName: String
├── oldValue: int
├── newValue: int
└── timestamp: DateTime

Trigger Condition:
  • |newValue - oldValue| >= 10
```

**RelationshipChanged**
```
RelationshipChanged
├── playerId: String
├── characterId: String
├── affectionDelta: int
├── dominationDelta: int
└── timestamp: DateTime

Trigger Condition:
  • Any change to affection or domination
```

**NewDayStarted**
```
NewDayStarted
├── playerId: String
├── week: int
├── weekday: Weekday
└── timestamp: DateTime
```

**NewWeekStarted**
```
NewWeekStarted
├── playerId: String
├── week: int
└── timestamp: DateTime
```

**SeasonChanged**
```
SeasonChanged
├── playerId: String
├── oldSeason: Season
├── newSeason: Season
└── timestamp: DateTime
```

### Usage Pattern (Phase 2)
```dart
// Domain layer emits
player._domainEvents.add(StatChangedSignificantly(...));

// Application layer handles
for (final event in gameState.domainEvents) {
  _eventBus.publish(event);
}
gameState.clearEvents();
```

---

## Repository Interfaces

Defined in Domain Layer, implemented in Infrastructure Layer.

### GameStateRepository
```
abstract class GameStateRepository {
  Future<GameState> get();
  Future<void> save(GameState state);
  Future<List<SaveSlot>> getSaveSlots();
  Future<GameState?> loadFromSlot(int slot);
  Future<void> saveToSlot(int slot, GameState state);
}
```

### LocationRepository
```
abstract class LocationRepository {
  Future<Location> getById(String id);
  Future<List<Location>> getAll();
  Future<List<Location>> getUnlocked(GameState state);
  Future<List<Location>> getByParent(String parentId);
}
```

### EventDefinitionRepository (Infrastructure)
```
abstract class EventDefinitionRepository {
  Future<EventDefinition> getById(String id);
  Future<List<EventDefinition>> getTriggerable(
    GameState state,
    String pointId,
    int interactionCount,
  );
}
```

---

## Testing Strategy

### Unit Tests (Domain Layer)

**Target**: >80% coverage

**Test Categories:**

**Value Objects:**
```dart
test('Stats clamps values to 0-100', () {
  final stats = Stats.initial().modify('stamina', 150);
  expect(stats.get('stamina'), 100);
});

test('GameTime advances correctly through week', () {
  var time = GameTime(week: 1, weekday: Weekday.sunday, period: TimeOfDay.night);
  time = time.advance();
  
  expect(time.week, 2);
  expect(time.weekday, Weekday.monday);
  expect(time.period, TimeOfDay.morning);
});

test('Relationship domination increases correctly', () {
  final rel = Relationship(
    characterId: 'alice',
    affection: 50,
    domination: 0,
  );
  
  final updated = rel.modifyDomination(20);
  
  expect(updated.domination, 20);
  expect(updated.affection, 50); // Unchanged
});

test('InteractionHistory tracks per point', () {
  var history = InteractionHistory.empty();
  
  // Click "cafe_counter" 3 times with different events
  history = history.recordInteraction('cafe_counter', 'event_1');
  history = history.recordInteraction('cafe_counter', 'event_2');
  history = history.recordInteraction('cafe_counter', 'event_1');
  
  expect(history.getInteractionCount('cafe_counter'), 3);
  expect(history.hasCompletedEvent('event_1'), true);
  expect(history.hasCompletedEvent('event_2'), true);
});
```

**Entities:**
```dart
test('GameState.recordInteraction updates history', () {
  final state = GameState.initial(playerId: 'p1', playerName: 'Test');
  
  final updated = state.recordInteraction('cafe_counter', 'meet_alice');
  
  expect(updated.getInteractionCount('cafe_counter'), 1);
  expect(updated.interactionHistory.hasCompletedEvent('meet_alice'), true);
});

test('GameState.advanceTime handles week transition', () {
  var state = GameState.initial(playerId: 'p1', playerName: 'Test');
  state = state.copyWith(
    time: GameTime(week: 1, weekday: Weekday.sunday, period: TimeOfDay.night),
  );
  
  final updated = state.advanceTime();
  
  expect(updated.time.week, 2);
  expect(updated.time.weekday, Weekday.monday);
});

test('Player.modifyDomination creates new instance', () {
  final player = Player.create(id: 'p1', name: 'Test');
  
  final updated = player.modifyDomination('alice', 30);
  
  expect(updated.relationships.get('alice')?.domination, 30);
  expect(player.relationships.get('alice'), isNull); // Original unchanged
});
```

---

## Domain Rules Summary

### Critical Invariants

1. **Stat Boundaries**: 0 <= value <= 100, auto-clamped
2. **Immutability**: All value objects immutable
3. **Aggregate Boundary**: GameState is sole entry point
4. **Season Independence**: Season only changes via events
5. **Interaction Permanence**: InteractionHistory never cleared
6. **Location References**: GameState stores ID only, not object
7. **First Interaction**: Auto-creates relationship with affection=50

### Business Logic Examples

**Domination Threshold:**
```
IF relationship.domination >= 80 THEN
  character accepts multi-romance
  no jealousy events
END IF
```

**Corruption Route Locking:**
```
IF player.corruption >= 60 THEN
  pure_character_routes.forEach(route => route.locked = true)
END IF
```

**Dialogue Variant Selection:**
```
interactionCount = gameState.getInteractionCount(pointId)
variants = event.dialogueVariants
selectedKey = max(variants.keys.where(k => k <= interactionCount))
dialogue = variants[selectedKey]
```

**Week-Specific Events:**
```
IF gameState.time.week >= 5 AND gameState.time.weekday == Weekday.saturday THEN
  unlock_festival_event()
END IF

```

---

## Design Patterns Used

### Aggregate Pattern
- **GameState** is the aggregate root
- All game state modifications through GameState methods
- Ensures consistency across player, time, season, interactions

### Value Object Pattern
- Stats, GameTime, Relationship, InteractionHistory are immutable
- Every modification returns new instance
- Prevents accidental mutation bugs

### Entity Pattern
- Player and Location have identity (compared by ID)
- But still immutable (modifications return new instances)
- Part of GameState aggregate, not independent roots

### Repository Pattern
- Interfaces in domain layer
- Implementations in infrastructure layer
- Enables testing without real persistence

---

## Key Design Decisions

### Why GameState as Aggregate Root (not Player)?

**Old Design (02_domain_model.md v1.0):**
```dart
Player (Aggregate Root)
├── stats, personality, currentTime, currentLocation
├── money, inventory, relationships, flags
└── activeBuffs, discoveredInteractionPoints, interactionHistories
```

**Problems:**
- Player too fat (violates Single Responsibility)
- Time/Season mixed with player data (wrong conceptually)
- Hard to add world state later (weather, NPCs)

**New Design (02_domain_model.md v1.1):**
```dart
GameState (Aggregate Root)
├── player: Player (just player data)
├── playerState: PlayerState (inventory, status)
├── time: GameTime (temporal state)
├── season: Season (world state)
├── currentLocationId: String (spatial state)
└── interactionHistory: InteractionHistory (progress tracking)
```

**Benefits:**
- Clear separation of concerns
- Easy to add world-level state (NPCs, weather)
- Player is focused (name, stats, relationships, flags)
- Better aligns with DDD principles

---

### Why InteractionHistory by Point (not by Event)?

**Requirement from 01_game_mechanics.md:**
```
Dialogue changes with interaction count:
  - 1st click: "Welcome! First time here?"
  - 2nd-4th clicks: "Back again?"
  - 5th+ clicks: "Regular customer!"
```

This is **"Nth time clicking THIS POINT"**, not "Nth time triggering THIS EVENT".

**Alternative Considered:**
```dart
// ❌ By Event (rejected)
class EventHistory {
  Map<String, int> _eventCount;  // eventId → count
}
```

**Problem:** Different events at same point wouldn't share count.

**Chosen Design:**
```dart
// ✅ By Point (correct)
class InteractionHistory {
  Map<String, InteractionRecord> _records;  // pointId → record
}

class InteractionRecord {
  int totalInteractions;  // Total clicks on THIS point
  Set<String> completedEvents;  // Which events triggered
}
```

**Benefits:**
- Matches game design ("7th visit to cafe")
- Supports dialogue progression naturally
- Can still track which events completed

---

### Why Domination as Numeric Value (not Item Count)?

**Alternative Considered:**
```dart
// ❌ Item Counter (rejected)
class Relationship {
  int leverageCount;
  bool isDominated => leverageCount >= 3;
}
```

**Problems:**
- All leverage items equal (photo = video = blackmail)
- No flexibility for writers
- Code changes needed for balancing

**Chosen Design:**
```dart
// ✅ Numeric Value (YAML-driven)
class Relationship {
  int domination;  // 0-100
  bool isDominated => domination >= 80;
}
```

**Benefits:**
- Writers control power per item in YAML
- Photo +20, Video +50, Blackmail +30 (flexible)
- No code changes for balancing
- Can have non-item domination sources (events)

---

### Why Season Separate from GameTime?

**Alternative Considered:**
```dart
// ❌ Season in GameTime (rejected)
class GameTime {
  int week;
  Weekday weekday;
  TimeOfDay period;
  Season season;  // Auto-advances every 13 weeks?
}
```

**Problems:**
- Seasons tied to time (game design wants event-driven)
- What if game ends at week 8? (never see all seasons)
- Time advancement logic becomes complex

**Chosen Design:**
```dart
// ✅ Season separate (event-driven)
class GameState {
  GameTime time;
  Season season;  // Changed via changeSeason()
}
```

**Benefits:**
- Season changes at narrative moments (Valentine → Spring)
- Player can experience all seasons regardless of week count
- Simple time advancement logic

---

## Common Pitfalls to Avoid

### ❌ Don't: Put UI logic in domain
```dart
// Wrong
class Player {
  String getDisplayName() => "Player: $name";  // UI concern
  Color getCorruptionColor() => ...;  // UI concern
}
```

### ✅ Do: Keep domain pure
```dart
// Correct
class Player {
  String get name => _name;  // Just data
}

// In presentation layer:
String displayName = "Player: ${player.name}";
Color color = player.stats.get('corruption') > 50 ? red : blue;
```

---

### ❌ Don't: Return nulls for missing data
```dart
// Wrong
Relationship? getRelationship(String characterId) {
  return relationships.get(characterId);  // Null if not found
}
```

### ✅ Do: Use default values or explicit checks
```dart
// Correct Option 1: Return default
Relationship getRelationship(String characterId) {
  return relationships.get(characterId) 
    ?? Relationship.stranger(characterId);
}

// Correct Option 2: Let caller handle null
Relationship? getRelationship(String characterId) {
  return relationships.get(characterId);
}
// Caller: final rel = getRelationship('alice') ?? Relationship.stranger('alice');
```

---

### ❌ Don't: Mutate value objects
```dart
// Wrong
stats._values['stamina'] = 100;  // Mutation!
player.stats.modify('charm', 10);  // Returns new, but not assigned!
```

### ✅ Do: Reassign to new instances
```dart
// Correct
final newStats = stats.modify('stamina', -10);
player = player.copyWith(stats: newStats);

// Or shortcut
gameState = gameState.modifyStat('stamina', -10);
```

---

### ❌ Don't: Let domain depend on infrastructure
```dart
// Wrong
import 'package:shared_preferences/shared_preferences.dart';

class GameState {
  Future<void> save() async {
    final prefs = await SharedPreferences.getInstance();
    // ...
  }
}
```

### ✅ Do: Use repository interfaces
```dart
// Correct - Domain defines interface
abstract class GameStateRepository {
  Future<void> save(GameState state);
}

// Infrastructure implements
class GameStateRepositoryImpl implements GameStateRepository {
  final SharedPreferences _prefs;
  // ...
}
```

---

## Differences from v1.0

### Major Changes

| Aspect               | v1.0 (Old)                              | v1.1 (New)                                      | Reason                  |
| -------------------- | --------------------------------------- | ----------------------------------------------- | ----------------------- |
| Aggregate Root       | Player                                  | GameState                                       | Separation of concerns  |
| Time Structure       | TimePoint(day, slot)                    | GameTime(week, weekday, period)                 | Added weekly/seasonal   |
| Season               | Not mentioned                           | Separate enum                                   | Event-driven system     |
| Relationship         | affection + leverageCount + status      | affection + domination                          | YAML-driven flexibility |
| Interaction Tracking | InteractionHistory(pointId)             | InteractionHistory(pointId) + InteractionRecord | Same concept, refined   |
| Player Buffs         | List<Buff> in Player                    | Set<String> statusEffects in PlayerState        | Simplified Phase 1      |
| Domain Services      | ExplorationService, EventTriggerService | (Phase 2)                                       | Defer complexity        |

### Migration Guide (if implementing v1.0)

**If you started with v1.0 design:**

1. **Extract time/season from Player**
   ```dart
   // Before
   Player(currentTime, currentLocation, ...)
   
   // After
   GameState(
     player: Player(name, stats, ...),
     time: GameTime(...),
     currentLocationId: ...,
   )
   ```

2. **Change Relationship structure**
   ```dart
   // Before
   Relationship(affection, leverageCount, status)
   
   // After
   Relationship(affection, domination)
   ```

3. **Update time methods**
   ```dart
   // Before
   TimePoint.next() → day + 1
   
   // After
   GameTime.advance() → handles week transitions
   ```

4. **Rename methods**
   ```dart
   // Before
   eventHistory.recordTrigger(eventId)
   
   // After
   interactionHistory.recordInteraction(pointId, eventId)
   ```

---

## Phase 1 vs Phase 2 Features

### Phase 1 (Current - Minimal Viable Domain)

**Implemented:**
- ✅ GameState as Aggregate Root
- ✅ GameTime with week/weekday/period
- ✅ Season enum (event-driven)
- ✅ Stats (5 attributes, Map-based)
- ✅ Relationships with affection + domination
- ✅ InteractionHistory by point
- ✅ PlayerState with Inventory
- ✅ Location with parent-child
- ✅ Basic domain exceptions

**Not Implemented (kept simple):**
- ❌ Domain Events
- ❌ Domain Services
- ❌ Complex Buff system
- ❌ Computed properties on Relationship
- ❌ Personality system

### Phase 2 (Future - Advanced Features)

**To Add:**
- Domain Events (StatChanged, RelationshipChanged, etc.)
- Domain Services (EventTriggerService, RelationshipService)
- Complex Buff system (duration, stacking, effects)
- Computed properties (RelationshipLevel from affection)
- Personality system (affects dialogue choices)
- Jealousy mechanics
- Character route locking

---

## Next Steps

After implementing Domain Layer:

1. **Write comprehensive unit tests**
   - Target >80% coverage
   - Focus on business logic (Stats clamping, time advancement, interaction counting)
   - Test all domain exceptions

2. **Verify zero dependencies**
   - No Flutter imports
   - No Riverpod imports
   - Only Dart standard library

3. **Document all domain rules**
   - Update this document with discovered rules
   - Add examples for complex logic

4. **Move to Application Layer**
   - Define Use Cases
   - Implement Use Cases using domain objects
   - Add integration tests

5. **Plan Infrastructure Layer**
   - Design save file format
   - Plan YAML schema
   - Design repository implementations

---

## References

### Related Documents
- `01_game_mechanics.md` - Game design specification
- `domain_objects.md` - Dart implementation details
- `README.md` - Project overview

### External Resources
- [Domain-Driven Design by Eric Evans](https://www.domainlanguage.com/ddd/)
- [Clean Architecture by Robert Martin](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
- [Value Objects Explained](https://martinfowler.com/bliki/ValueObject.html)
- [Aggregate Pattern](https://martinfowler.com/bliki/DDD_Aggregate.html)

---

**Last Updated**: 2025-10-23  
**Version**: 1.1 (Phase 1 - Aligned with Implementation)  
**Previous Version**: 1.0 (Phase 0 - Initial Design)