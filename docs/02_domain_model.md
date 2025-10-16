# Domain Model - Core Business Logic

## Architecture Overview

```
┌─────────────────────────────────────────────────┐
│         Domain Layer (No Dependencies)          │
│                                                 │
│  Aggregates (Entity Clusters)                  │
│    • Player (Root)                              │
│    • Location                                   │
│    • Character                                  │
│                                                 │
│  Entities (Identity-based)                     │
│    • InteractionPoint                           │
│    • GameEvent                                  │
│                                                 │
│  Value Objects (Immutable)                     │
│    • Stats                                      │
│    • TimePoint                                  │
│    • Relationship                               │
│    • InteractionHistory                         │
│    • Buff                                       │
│    • DialogueVariant                            │
│                                                 │
│  Domain Services (Complex Logic)               │
│    • ExplorationService                         │
│    • EventTriggerService                        │
│    • RelationshipService                        │
│                                                 │
│  Domain Events (State Changes)                 │
│    • StatChanged                                │
│    • RelationshipUpdated                        │
│    • NewDayStarted                              │
└─────────────────────────────────────────────────┘
```

---

## Player (Aggregate Root)

### Structure
```
Player
├── id: PlayerId
├── stats: Stats
├── personality: Personality (mutable)
├── currentTime: TimePoint
├── currentLocation: LocationId
├── money: int
├── inventory: Inventory
├── relationships: Map<CharacterId, Relationship>
├── storyFlags: Set<String>
├── activeBuffs: List<Buff>
├── discoveredInteractionPoints: Map<LocationId, Set<InteractionPointId>>
├── interactionHistories: Map<InteractionPointId, InteractionHistory>
└── pendingHints: List<InteractionHint>
```

### Key Behaviors

**executeAction(action)**
- Check stamina (unless buff immunity)
- If stamina < cost → return must_rest error
- Execute action effects
- Deduct stamina
- Emit domain events
- Return result

**rest()**
- Forced behavior when stamina = 0
- stamina = 100
- currentTime = next day morning
- Apply time passage effects
- Emit NewDayStarted event

**modifyStat(type, delta)**
- Clamp to (0, 100)
- If stamina becomes 0 → set must_rest flag
- Emit StatChanged event if significant (|delta| >= 10)
- Return new Player instance (immutable pattern)

**advanceTime()**
- Move to next time slot
- If new day → apply cleanliness decay (-5)
- Emit NewDayStarted if day changed
- Return new Player instance

**discoverInteractionPoint(locationId, pointId)**
- Add to discoveredInteractionPoints
- Used when hints unlock hidden locations
- Return new Player instance

**recordInteraction(pointId, eventId)**
- Increment interactionHistories[pointId].totalInteractions
- Add eventId to completedEvents
- Update lastInteractionTime
- **Critical: totalInteractions persisted in save file**

### Domain Rules (Invariants)

- Stamina always >= 0
- All stats clamped to 0-100
- Cannot execute actions when stamina = 0 (except rest)
- StaminaImmunity buff bypasses stamina checks
- Personality can change via corruption events or courses

---

## Value Objects

### Stats

```
Stats (Immutable)
├── stamina: int (0-100)
├── charm: int (0-100)
├── corruption: int (0-100)
├── intelligence: int (0-100)
└── cleanliness: int (0-100)

Factory Methods:
  • Stats.initial() → Default starting values
  • Stats.create(...) → Validated creation
  • modify(type, delta) → New Stats instance
  • getValue(type) → int
```

**Validation Rules:**
- All values automatically clamped to 0-100
- No negative values allowed
- Immutable (every modification returns new instance)

---

### TimePoint

```
TimePoint (Immutable)
├── day: int
└── timeSlot: TimeSlot (enum)

TimeSlot Enum:
  • morning
  • afternoon
  • evening
  • night

Methods:
  • next() → TimePoint (advance one slot)
  • isNewDay → bool (check if transitioning to morning)
```

---

### Relationship

```
Relationship (Immutable)
├── characterId: CharacterId
├── affection: int (0-100)
├── level: RelationshipLevel (computed from affection)
├── status: RelationshipStatus
└── leverageCount: int

RelationshipLevel (Computed):
  • Stranger: 0-20
  • Acquaintance: 21-40
  • Friend: 41-60
  • Close: 61-80
  • Lover: 81-100

RelationshipStatus (Enum):
  • Normal: Default state
  • Dating: Confirmed relationship
  • Dominated: Via leverage items, no jealousy
  • EndingReached: Character route completed

Methods:
  • modify(delta) → Relationship
  • addLeverage() → Relationship (leverageCount++)
  • checkDomination() → bool (leverageCount >= 3)
```

---

### InteractionHistory

```
InteractionHistory (Value Object)
├── pointId: InteractionPointId
├── totalInteractions: int ⭐ Persisted in save
├── completedEvents: Set<EventId>
├── lastInteractionTime: TimePoint?
└── (optional) interactionTimeline: List<InteractionRecord>

Methods:
  • recordInteraction(eventId, time) → InteractionHistory
  • canRepeat(event, currentTime) → bool
  • getInteractionCount() → int
```

**Critical Design Decision:**
- `totalInteractions` stored permanently in save files
- Enables dialogue variants based on total interaction count
- Example: 7th visit to cafe counter shows "regular customer" dialogue

---

### Buff

```
Buff (Value Object)
├── id: BuffId
├── type: BuffType
├── duration: BuffDuration
└── sourceItem: ItemId?

BuffType (Enum):
  • StaminaImmunity: Ignore stamina costs
  • SeeCards: Cheat in minigames
  • DoubleReward: 2x event rewards
  • ImmuneToJealousy: No multi-romance penalties
  • AttributeProtection: Prevent stat decreases

BuffDuration (Enum):
  • TimeBasedDuration(expiresAt: TimePoint)
  • EventBasedDuration(remainingEvents: int)
  • Permanent

Methods:
  • isActive(currentTime) → bool
  • consume() → Buff? (for event-based, null if exhausted)
```

---

## Entities

### InteractionPoint

```
InteractionPoint (Entity)
├── id: InteractionPointId
├── name: String
├── position: Position (x, y)
├── visibility: Visibility
├── discoveryHint: DiscoveryHint?
├── events: List<ConditionalEvent>
├── fallbackDialogue: Dialogue
└── state: InteractionState

Visibility (Enum):
  • AlwaysVisible: Functional points (counter, NPC)
  • HiddenUntilDiscovered: Hidden points (trash can)
  • ConditionalVisible: Stat threshold reveals

InteractionState (Enum):
  • Undiscovered: Player hasn't found yet
  • Available: Can interact
  • (Exhausted state removed - use fallback instead)

Methods:
  • isVisibleTo(player) → bool
  • getAvailableEvent(player, interactionCount) → EventId?
  • afterEventCompleted(eventId) → void
  • getFallbackDialogue() → Dialogue
```

---

### ConditionalEvent

```
ConditionalEvent (Value Object)
├── eventId: EventId
├── type: EventType
├── timeConsumption: TimeConsumption
├── forceLeaveScene: bool
├── conditions: List<Condition>
├── priority: int
├── repeatability: Repeatability
├── randomChance: float?
└── dialogueVariants: Map<int, DialogueVariant>

EventType (Enum):
  • MajorStory: Consumes time, forces exit
  • MinorBranch: No time, can continue exploring

TimeConsumption:
  • None: 0 time slots
  • OneSlot: 1 time slot
  • Custom(int): N time slots

Repeatability (Enum):
  • Once: One-time event
  • Daily: Once per day
  • Unlimited: No restrictions
  • ConditionalRepeat: Check conditions each time

Methods:
  • canTrigger(player, interactionCount) → bool
  • getDialogue(interactionCount) → Dialogue
```

**Dialogue Variant Selection:**
```
interactionCount = 7
dialogueVariants = {1: "first", 2: "second", 5: "fifth+"}

Selection logic:
  - Exact match? Return 7 if exists
  - Otherwise: Find largest key <= 7
  - Result: Use key 5 ("fifth+")
```

---

### Location

```
Location (Entity)
├── id: LocationId
├── name: String
├── backgroundImage: String
├── interactionPoints: List<InteractionPoint>
├── initialVisiblePoints: Set<InteractionPointId>
├── basicActions: List<ActionId>
└── unlockConditions: List<Condition>

Methods:
  • getVisiblePoints(player) → List<InteractionPoint>
    Returns: initialVisiblePoints 
           + player.discoveredInteractionPoints
           + conditionalVisible (if conditions met)
  
  • getInteractablePoints(player) → List<InteractionPoint>
    Filter out Undiscovered state
```

---

### Character

```
Character (Entity)
├── id: CharacterId
├── name: String
├── personality: CharacterPersonality
├── corruptionRange: Range<int>
├── dominationRequirement: DominationRequirement
└── endingConditions: List<EndingCondition>

DominationRequirement:
  • leverageItemCount: int (e.g., 3)
  • OR eventCompleted: EventId

CorruptionRange Examples:
  • Pure character: (0, 30)
  • Neutral character: (20, 70)
  • Corrupt character: (60, 100)

Methods:
  • isRouteAccessible(playerCorruption) → bool
  • canBeDominated(player) → bool
```

---

## Domain Services

### ExplorationService

```
ExplorationService
└── explore(player, location) → ExplorationResult

Process:
  1. Get visible interaction points
  2. For each point: check available events
  3. Return list of interactable points
  
Not responsible for:
  • Executing events (that's ExecuteEventUseCase)
  • Just identifies what's available
```

---

### EventTriggerService

```
EventTriggerService
└── checkTriggers(player, location) → List<GameEvent>

Process:
  1. Get all events in location
  2. Filter by conditions (time, stats, flags)
  3. Sort by priority (high to low)
  4. Return triggerable events

Used by:
  • Auto-trigger systems (enter location, time advance)
  • Not for manual interaction points
```

---

### RelationshipService

```
RelationshipService
├── checkDominationStatus(player, charId) → bool
└── calculateJealousyRisk(player) → List<CharacterId>

Methods:
  • checkDominationStatus: 
    Check leverageCount >= 3 OR special event flag
    Return true if character should be dominated
  
  • calculateJealousyRisk:
    Find all characters with status=Dating
    Return list of characters who might get jealous
```

---

## Domain Events

### Purpose
Domain events notify other parts of the system about state changes without coupling.

### Event Types

**StatChangedSignificantly**
```
StatChangedSignificantly
├── playerId: PlayerId
├── statType: StatType
├── oldValue: int
├── newValue: int
└── timestamp: DateTime
```

**RelationshipLevelChanged**
```
RelationshipLevelChanged
├── playerId: PlayerId
├── characterId: CharacterId
├── oldLevel: RelationshipLevel
├── newLevel: RelationshipLevel
└── timestamp: DateTime
```

**NewDayStarted**
```
NewDayStarted
├── playerId: PlayerId
├── day: int
└── timestamp: DateTime
```

### Usage Pattern
```dart
// Domain layer emits
player._domainEvents.add(StatChangedSignificantly(...));

// Application layer handles
for (final event in player.domainEvents) {
  _eventBus.publish(event);
}
player.clearEvents();
```

---

## Repository Interfaces

Defined in Domain Layer, implemented in Infrastructure Layer.

### PlayerRepository
```
abstract class PlayerRepository {
  Future<Player> get();
  Future<void> save(Player player);
  Future<List<SaveSlot>> getSaveSlots();
  Future<Player?> loadFromSlot(int slot);
  Future<void> saveToSlot(int slot, Player player);
}
```

### LocationRepository
```
abstract class LocationRepository {
  Future<Location> getById(LocationId id);
  Future<List<Location>> getAll();
  Future<List<Location>> getUnlocked(Player player);
}
```

### EventRepository
```
abstract class EventRepository {
  Future<GameEvent> getById(EventId id);
  Future<List<GameEvent>> getTriggerable(Player player, Location location);
}
```

### CharacterRepository
```
abstract class CharacterRepository {
  Future<Character> getById(CharacterId id);
  Future<List<Character>> getAvailableRoutes(Player player);
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
  final stats = Stats.create(stamina: 150, charm: -10, ...);
  expect(stats.stamina, 100);
  expect(stats.charm, 0);
});

test('TimePoint advances correctly', () {
  final time = TimePoint(day: 1, timeSlot: TimeSlot.morning);
  final next = time.next();
  expect(next.timeSlot, TimeSlot.afternoon);
  expect(next.day, 1);
});

test('Relationship level computed from affection', () {
  final rel = Relationship(characterId: 'alex', affection: 55);
  expect(rel.level, RelationshipLevel.friend);
});
```

**Entities:**
```dart
test('Player.modifyStat emits event when significant', () {
  final player = Player.newGame();
  final updated = player.modifyStat(StatType.charm, 15);
  
  expect(updated.domainEvents.length, 1);
  expect(updated.domainEvents.first, isA<StatChangedSignificantly>());
});

test('Player.rest() forces new day', () {
  var player = Player.newGame();
  player = player.modifyStat(StatType.stamina, -100); // stamina = 0
  
  final rested = player.rest();
  
  expect(rested.stats.stamina, 100);
  expect(rested.currentTime.day, 2);
  expect(rested.currentTime.timeSlot, TimeSlot.morning);
});

test('InteractionPoint selects correct dialogue variant', () {
  final point = InteractionPoint(
    dialogueVariants: {
      1: DialogueVariant(text: "First time"),
      5: DialogueVariant(text: "Regular"),
    },
  );
  
  expect(point.getDialogue(1).text, "First time");
  expect(point.getDialogue(3).text, "First time"); // Uses closest
  expect(point.getDialogue(7).text, "Regular");
});
```

**Domain Services:**
```dart
test('EventTriggerService filters by conditions', () {
  final player = Player.newGame().modifyStat(StatType.charm, 30);
  final events = [
    GameEvent(conditions: [CharmCondition(min: 50)]), // Should fail
    GameEvent(conditions: [CharmCondition(min: 20)]), // Should pass
  ];
  
  final service = EventTriggerService();
  final triggerable = service.checkTriggers(player, events);
  
  expect(triggerable.length, 1);
});
```

---

## Domain Rules Summary

### Critical Invariants

1. **Stamina Boundary**: 0 <= stamina <= 100, never negative
2. **Forced Rest**: When stamina = 0, only rest() allowed
3. **Stat Clamping**: All stats automatically clamp to 0-100
4. **Immutability**: All value objects are immutable
5. **Event Ordering**: Priority determines event selection when multiple match
6. **Interaction Counting**: totalInteractions persisted forever
7. **Time Progression**: Only certain events consume time
8. **Scene Exit**: Major story events force leaving scene

### Business Logic Examples

**Corruption Route Locking:**
```
IF player.corruption >= 60 THEN
  pure_characters.forEach(char => char.routeLocked = true)
```

**Domination Check:**
```
IF relationship.leverageCount >= 3 OR hasFlag("dominated_" + charId) THEN
  relationship.status = Dominated
```

**Dialogue Variant Selection:**
```
interactionCount = player.getInteractionCount(pointId)
variant = findClosestVariant(event.dialogueVariants, interactionCount)
```

---

## Design Patterns Used

### Aggregate Pattern
- **Player** is the aggregate root
- All player-related data modified through Player methods
- Ensures consistency across stats, relationships, flags

### Value Object Pattern
- Stats, TimePoint, Relationship are immutable
- Every modification returns new instance
- Prevents accidental mutation bugs

### Domain Events Pattern
- State changes emit events
- Application layer subscribes to events
- Decouples domain from infrastructure

### Repository Pattern
- Interfaces in domain layer
- Implementations in infrastructure layer
- Enables testing without real persistence

### Strategy Pattern
- BuffType determines behavior
- EventType determines time consumption
- Repeatability determines when events can retrigger

---

## Key Design Decisions

### Why Immutable Value Objects?
- **Thread safety**: No race conditions
- **Predictability**: State changes explicit
- **Testing**: Easier to verify state transitions
- **Time travel**: Can snapshot states easily

### Why Domain Events?
- **Decoupling**: UI doesn't depend on domain
- **Extensibility**: Easy to add new event listeners
- **Audit trail**: All state changes recorded
- **Testing**: Can verify events emitted

### Why Aggregate Root?
- **Consistency**: Single point of truth
- **Encapsulation**: Business rules enforced
- **Transactional**: All changes atomic
- **Simplicity**: Clear ownership boundaries

### Why Repository Interfaces?
- **Testability**: Can mock persistence
- **Flexibility**: Can swap storage backend
- **Dependency Inversion**: Domain doesn't depend on infrastructure
- **Clean Architecture**: Maintains layer boundaries

---

## Common Pitfalls to Avoid

### ❌ Don't: Put UI logic in domain
```dart
// Wrong
class Player {
  String getDisplayName() => "Player: $name"; // UI concern
}
```

### ✅ Do: Keep domain pure
```dart
// Correct
class Player {
  String get name => _name; // Just data
}

// In presentation layer:
String displayName = "Player: ${player.name}";
```

---

### ❌ Don't: Return nulls for missing data
```dart
// Wrong
Relationship? getRelationship(CharacterId id) {
  return relationships[id]; // Null if not found
}
```

### ✅ Do: Use Result type or default values
```dart
// Correct
Relationship getRelationship(CharacterId id) {
  return relationships[id] ?? Relationship.stranger(id);
}
```

---

### ❌ Don't: Mutate value objects
```dart
// Wrong
stats.stamina += 10; // Mutation!
```

### ✅ Do: Return new instances
```dart
// Correct
final newStats = stats.modify(StatType.stamina, 10);
```

---

### ❌ Don't: Let domain depend on infrastructure
```dart
// Wrong
import 'package:shared_preferences/shared_preferences.dart';

class Player {
  Future<void> save() async {
    final prefs = await SharedPreferences.getInstance();
    // ...
  }
}
```

### ✅ Do: Use repository interfaces
```dart
// Correct
// Domain defines interface
abstract class PlayerRepository {
  Future<void> save(Player player);
}

// Infrastructure implements
class PlayerRepositoryImpl implements PlayerRepository {
  final SharedPreferences _prefs;
  // ...
}
```

---

## Next Steps

After implementing Domain Layer:
1. Write comprehensive unit tests (target >80% coverage)
2. Verify no dependencies on Flutter/infrastructure
3. Document all domain rules and invariants
4. Move to Application Layer (Use Cases)

---

**Last Updated**: 2025-01-16  
**Version**: 1.0