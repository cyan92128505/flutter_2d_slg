# Application Layer - Use Cases

## Purpose

Application Layer orchestrates domain logic to fulfill user intentions. It:
- Coordinates domain objects
- Manages transactions
- Publishes domain events
- Handles cross-cutting concerns (logging, validation)

**Does NOT contain business rules** - those live in Domain Layer.

---

## Use Case Catalog

### Core Game Loop
1. **EnterLocationUseCase**: Navigate to a scene
2. **InteractWithPointUseCase**: Click an interaction point
3. **ExecuteEventUseCase**: Run an event (dialogue, choices, effects)
4. **ExecuteActionUseCase**: Perform basic actions (work, rest)

### Time Management
5. **ProgressTimeUseCase**: Advance time manually
6. **RestUseCase**: Force rest when stamina = 0

### Exploration
7. **DiscoverInteractionPointUseCase**: Unlock hidden points via hints

### Persistence
8. **SaveGameUseCase**: Save to slot
9. **LoadGameUseCase**: Load from slot

### NPC Interaction
10. **ProcessNPCDialogueUseCase**: Handle NPC conversations with hints

---

## 1. EnterLocationUseCase

### Responsibility
Transition player to a new location and determine what's visible.

### Input
```dart
class EnterLocationInput {
  final Player player;
  final LocationId locationId;
}
```

### Process
```
1. Load Location from repository
2. Check unlock conditions
   IF not unlocked → return Error
3. Update player.currentLocation
4. Compute visible interaction points:
   visiblePoints = location.initialVisiblePoints
                 + player.discoveredInteractionPoints[locationId]
                 + conditionalVisible points (stat-based)
5. Check for auto-trigger events (enter location events)
6. Save player state
7. Return LocationView
```

### Output
```dart
class LocationView {
  final String backgroundImage;
  final List<InteractionPointView> visiblePoints;
  final List<ActionView> basicActions;
  final List<EventId> autoTriggeredEvents;
}

class InteractionPointView {
  final InteractionPointId id;
  final String name;
  final Position position;
  final IconType icon; // magnifying_glass, exclamation, etc.
}
```

### Example
```dart
final useCase = EnterLocationUseCase(
  playerRepo: playerRepository,
  locationRepo: locationRepository,
  eventTriggerService: eventTriggerService,
);

final result = await useCase.execute(
  EnterLocationInput(player: player, locationId: LocationId('cafe'))
);

result.when(
  success: (view) {
    // Display background image
    // Render interaction points at positions
    // Show basic action buttons
  },
  failure: (error) {
    // Show error: "Location locked"
  },
);
```

---

## 2. InteractWithPointUseCase

### Responsibility
Handle player clicking an interaction point.

### Input
```dart
class InteractWithPointInput {
  final Player player;
  final LocationId locationId;
  final InteractionPointId pointId;
}
```

### Process
```
1. Load Location and InteractionPoint
2. Check point is discovered
   IF not discovered → return Error
3. Get interaction count:
   interactionCount = player.getInteractionCount(pointId)
4. Get available event:
   event = point.getAvailableEvent(player, interactionCount)
5. IF event exists:
   a. Execute event (ExecuteEventUseCase)
   b. Record interaction:
      player.recordInteraction(pointId, eventId)
   c. Check time consumption:
      IF event.timeConsumption != None:
        player = player.advanceTime(slots)
   d. Check force leave:
      IF event.forceLeaveScene:
        return EventTriggered(forceLeave: true)
   e. Check for new hints/points unlocked
   f. Save player state
   g. Return EventTriggered
6. ELSE (no event available):
   return FallbackDialogue(point.fallbackDialogue)
```

### Output
```dart
sealed class InteractionResult {}

class EventTriggered extends InteractionResult {
  final EventId eventId;
  final bool forceLeave;
  final List<InteractionPointId> newlyDiscovered;
}

class FallbackDialogue extends InteractionResult {
  final String text;
}

class PointNotDiscovered extends InteractionResult {}
```

### Example
```dart
final result = await interactUseCase.execute(
  InteractWithPointInput(
    player: player,
    locationId: LocationId('cafe'),
    pointId: InteractionPointId('window_seat'),
  ),
);

result.when(
  eventTriggered: (eventId, forceLeave, discovered) {
    // Show event dialogue
    if (forceLeave) navigateToMap();
    if (discovered.isNotEmpty) showDiscoveryNotification();
  },
  fallbackDialogue: (text) {
    // Show simple message
  },
  error: (message) {
    // Handle error
  },
);
```

---

## 3. ExecuteEventUseCase

### Responsibility
Run an event through all its phases (dialogue, choices, minigames, effects).

### Input
```dart
class ExecuteEventInput {
  final Player player;
  final EventId eventId;
  final int interactionCount; // For dialogue variant selection
}
```

### Process
```
1. Load GameEvent from repository
2. Get dialogue variant:
   dialogue = event.getDialogue(interactionCount)
3. Execute phases sequentially:
   FOR each phase in event.phases:
     CASE phase.type:
       Dialogue → Display text, wait for continue
       Choice → Display options, wait for selection
       Minigame → Execute minigame, get result
       SkillCheck → Perform check, branch on result
4. Apply effects based on choices/results:
   player = applyEffects(player, selectedChoice.effects)
5. Emit domain events:
   FOR event in player.domainEvents:
     eventBus.publish(event)
   player.clearEvents()
6. Save player state
7. Return EventResult
```

### Output
```dart
class EventResult {
  final bool success;
  final Map<String, dynamic> effects; // For UI display
  final List<String> unlockedFlags;
  final List<InteractionPointId> discoveredPoints;
}
```

### Phase Types
```dart
sealed class EventPhase {}

class DialoguePhase extends EventPhase {
  final String speaker;
  final String text;
  final String? sprite;
}

class ChoicePhase extends EventPhase {
  final String prompt;
  final List<Choice> options;
}

class MinigamePhase extends EventPhase {
  final MinigameType type;
  final Minigame config;
}

class SkillCheckPhase extends EventPhase {
  final StatType stat;
  final int requiredValue;
  final EventOutcome successOutcome;
  final EventOutcome failureOutcome;
}
```

---

## 4. ExecuteActionUseCase

### Responsibility
Execute basic scene actions (work, rest, shop, etc.).

### Input
```dart
class ExecuteActionInput {
  final Player player;
  final GameAction action;
}
```

### Process
```
1. Check action preconditions:
   IF !action.canExecute(player):
     return Failure(action.getFailureReason())
2. Execute action:
   result = action.execute(player)
3. IF result.success:
   a. Apply effects to player
   b. Check for triggered events (post-action)
   c. Emit domain events
   d. Save player state
4. Return ActionResult
```

### Example Actions
```dart
class WorkAction extends GameAction {
  bool canExecute(Player player) {
    return player.stats.stamina >= 20 
        && player.currentTime.timeSlot.isWorkHours;
  }
  
  ActionResult execute(Player player) {
    return ActionResult(
      success: true,
      effects: {
        'money': +200,
        'stamina': -20,
        'intelligence': +3,
        'cleanliness': -10,
      },
      timeAdvance: 1,
    );
  }
}
```

---

## 5. ProgressTimeUseCase

### Responsibility
Manually advance time (for "wait" or "do nothing" actions).

### Input
```dart
class ProgressTimeInput {
  final Player player;
  final int slots; // How many time slots to advance
}
```

### Process
```
1. FOR i in 1..slots:
   player = player.advanceTime()
2. Check for time-triggered events (e.g., "nightfall event")
3. Emit domain events
4. Save player state
5. Return TimeProgressResult
```

---

## 6. RestUseCase

### Responsibility
Force rest when stamina reaches 0.

### Input
```dart
class RestInput {
  final Player player;
}
```

### Process
```
1. Verify stamina = 0 (or close to it)
2. player = player.rest()
   → stamina = 100
   → time = next day morning
   → apply overnight effects (cleanliness decay)
3. Check for "new day" events
4. Emit NewDayStarted event
5. Save player state
6. Return RestResult
```

### Output
```dart
class RestResult {
  final int newDay;
  final List<EventId> newDayEvents;
  final Map<String, int> overnightChanges; // cleanliness: -5
}
```

---

## 7. DiscoverInteractionPointUseCase

### Responsibility
Unlock hidden interaction points via hints.

### Input
```dart
class DiscoverPointInput {
  final Player player;
  final LocationId locationId;
  final InteractionPointId pointId;
  final HintSource source; // npc_dialogue, item, tutorial
}
```

### Process
```
1. Load Location and InteractionPoint
2. Check point exists
3. Check not already discovered
4. Add to player.discoveredInteractionPoints
5. Trigger UI notification: "Discovered: {point.name}"
6. Log discovery source (for analytics)
7. Save player state
8. Return DiscoveryResult
```

### Example
```dart
// Triggered after NPC dialogue mentions a location
await discoverUseCase.execute(
  DiscoverPointInput(
    player: player,
    locationId: LocationId('cafe'),
    pointId: InteractionPointId('trash_can'),
    source: HintSource.npcDialogue('barista'),
  ),
);

// UI shows: "💡 You remember the barista mentioning the trash can..."
```

---

## 8. SaveGameUseCase

### Responsibility
Persist player state to a save slot.

### Input
```dart
class SaveGameInput {
  final Player player;
  final int slot; // 0-9
}
```

### Process
```
1. Validate slot number (0-9)
2. Serialize player to JSON:
   json = player.toJson()
   - Include ALL fields
   - Especially: discoveredInteractionPoints
   - Especially: interactionHistories (with totalInteractions)
3. Add metadata:
   json['saved_at'] = DateTime.now()
   json['version'] = SAVE_VERSION
4. Save to repository:
   await playerRepo.saveToSlot(slot, json)
5. Update slot preview (day, money, timestamp)
6. Return SaveResult
```

### Critical Fields in Save
```json
{
  "version": 1,
  "player": {
    "stats": {...},
    "currentTime": {...},
    "money": 850,
    "discoveredInteractionPoints": {
      "cafe": ["counter", "window_seat", "trash_can"]
    },
    "interactionHistories": {
      "cafe_counter": {
        "totalInteractions": 7,  // ⭐ Critical: Persisted
        "completedEvents": ["buy_coffee", "part_time_work"],
        "lastInteractionTime": {"day": 5, "timeSlot": "morning"}
      }
    },
    "relationships": {...},
    "storyFlags": [...]
  },
  "saved_at": "2025-10-16T10:30:00Z"
}
```

---

## 9. LoadGameUseCase

### Responsibility
Restore player state from a save slot.

### Input
```dart
class LoadGameInput {
  final int slot;
}
```

### Process
```
1. Load JSON from repository
2. Check version:
   IF json['version'] < CURRENT_VERSION:
     json = migrateVersion(json)
3. Deserialize to Player:
   player = Player.fromJson(json)
4. Validate integrity:
   - All stats in 0-100 range
   - All referenced locations/characters exist
   - No corrupted data
5. IF validation fails:
   return LoadError("Save file corrupted")
6. Return LoadResult(player)
```

### Version Migration Example
```dart
SaveData migrateVersion(Map<String, dynamic> json) {
  final version = json['version'] as int;
  
  if (version < 2) {
    // v1 → v2: Added corruption stat
    json['player']['stats']['corruption'] = 0;
  }
  
  if (version < 3) {
    // v2 → v3: Added interactionHistories
    json['player']['interactionHistories'] = {};
  }
  
  json['version'] = CURRENT_VERSION;
  return json;
}
```

---

## 10. ProcessNPCDialogueUseCase

### Responsibility
Handle NPC conversations that may contain hints.

### Input
```dart
class ProcessDialogueInput {
  final Player player;
  final DialogueId dialogueId;
}
```

### Process
```
1. Load Dialogue from repository
2. Display dialogue to player
3. Process embedded hints:
   FOR each hint in dialogue.hints:
     IF hint.unlockImmediately:
       await DiscoverInteractionPointUseCase(...)
     ELSE:
       player.pendingHints.add(hint)
4. Apply dialogue effects (affection, flags)
5. Emit domain events
6. Save player state
7. Return DialogueResult
```

### Example
```dart
// Alex says: "I usually sit by the window to read"
// Dialogue definition includes:
{
  "hints": [{
    "location": "cafe",
    "point": "window_seat",
    "unlockImmediately": true
  }]
}

// After dialogue completes:
// → DiscoverInteractionPointUseCase auto-called
// → UI shows: "💡 You now know where Alex likes to sit"
```

---

## Use Case Dependencies

### Typical Dependency Injection
```dart
class InteractWithPointUseCase {
  final PlayerRepository _playerRepo;
  final LocationRepository _locationRepo;
  final EventRepository _eventRepo;
  final ExecuteEventUseCase _executeEventUseCase;
  final EventBus _eventBus;
  
  InteractWithPointUseCase({
    required PlayerRepository playerRepo,
    required LocationRepository locationRepo,
    required EventRepository eventRepo,
    required ExecuteEventUseCase executeEventUseCase,
    required EventBus eventBus,
  }) : _playerRepo = playerRepo,
       _locationRepo = locationRepo,
       _eventRepo = eventRepo,
       _executeEventUseCase = executeEventUseCase,
       _eventBus = eventBus;
  
  Future<InteractionResult> execute(InteractWithPointInput input) async {
    // Implementation...
  }
}
```

---

## Error Handling

### Result Type Pattern
```dart
sealed class Result<T> {
  const Result();
}

class Success<T> extends Result<T> {
  final T value;
  const Success(this.value);
}

class Failure<T> extends Result<T> {
  final String error;
  const Failure(this.error);
}

// Usage
Future<Result<EventResult>> execute(ExecuteEventInput input) async {
  try {
    // ... process
    return Success(eventResult);
  } catch (e) {
    return Failure(e.toString());
  }
}
```

---

## Transaction Management

### Pattern: Unit of Work
```dart
Future<Result<T>> executeInTransaction<T>(
  Future<T> Function() work,
) async {
  // In our simple case: just save at end
  try {
    final result = await work();
    await _playerRepo.save(player); // Commit
    return Success(result);
  } catch (e) {
    // Rollback not needed (domain is immutable)
    return Failure(e.toString());
  }
}
```

---

## Testing Use Cases

### Integration Test Example
```dart
void main() {
  late MockPlayerRepository mockPlayerRepo;
  late MockLocationRepository mockLocationRepo;
  late InteractWithPointUseCase useCase;
  
  setUp(() {
    mockPlayerRepo = MockPlayerRepository();
    mockLocationRepo = MockLocationRepository();
    useCase = InteractWithPointUseCase(
      playerRepo: mockPlayerRepo,
      locationRepo: mockLocationRepo,
      // ... other deps
    );
  });
  
  test('First interaction triggers correct event', () async {
    // Arrange
    final player = Player.newGame();
    final location = Location.cafe();
    when(mockPlayerRepo.get()).thenAnswer((_) async => player);
    when(mockLocationRepo.getById(any)).thenAnswer((_) async => location);
    
    // Act
    final result = await useCase.execute(
      InteractWithPointInput(
        player: player,
        locationId: LocationId('cafe'),
        pointId: InteractionPointId('counter'),
      ),
    );
    
    // Assert
    expect(result, isA<EventTriggered>());
    final triggered = result as EventTriggered;
    expect(triggered.eventId, EventId('buy_coffee'));
    
    // Verify save was called
    verify(mockPlayerRepo.save(any)).called(1);
  });
}
```

---

## Common Patterns

### Pattern: Check-Execute-Save
```
1. Load required data (player, location, etc.)
2. Check preconditions (return early if fail)
3. Execute business logic (via domain)
4. Emit domain events
5. Save state
6. Return result
```

### Pattern: Coordination Not Logic
```dart
// ❌ Wrong: Business logic in use case
class WorkUseCase {
  Future<Result> execute(input) async {
    // Don't calculate effects here!
    player.money += 200;
    player.stats.stamina -= 20;
  }
}

// ✅ Correct: Delegate to domain
class WorkUseCase {
  Future<Result> execute(input) async {
    // Domain handles logic
    final action = WorkAction();
    final result = action.execute(player);
    // Use case just coordinates
  }
}
```

---

## Next Steps

After implementing Application Layer:
1. Write integration tests for each use case
2. Verify proper error handling
3. Ensure no business logic in use cases
4. Move to Infrastructure Layer (repositories)

---

**Last Updated**: 2025-10-16  
**Version**: 1.0