# Development Roadmap

## Overview

18-week development plan to reach Alpha release, organized into 9 phases.

---

## Phase Summary

| Phase | Duration | Deliverable | Status |
|-------|----------|-------------|--------|
| 0 | - | Architecture design | ✅ Complete |
| 1 | Week 1-2 | Domain Layer | 📋 Next |
| 2 | Week 3-4 | Application Layer | ⏳ Planned |
| 3 | Week 5-6 | Infrastructure Layer | ⏳ Planned |
| 4 | Week 7-8 | Presentation MVP | ⏳ Planned |
| 5 | Week 9-10 | Vertical Slice | ⏳ Planned |
| 6 | Week 11-12 | Narrative Tooling | ⏳ Planned |
| 7 | Week 13-16 | Content Expansion | ⏳ Planned |
| 8 | Week 17-18 | Polish & Release | ⏳ Planned |

---

## Phase 0: Architecture Design ✅

**Status:** Complete  
**Duration:** Ongoing design  
**Deliverables:**
- [x] Complete game mechanics definition
- [x] Domain model design
- [x] Use cases specification
- [x] Narrative system design
- [x] Development roadmap
- [x] All documentation artifacts

---

## Phase 1: Domain Layer (Week 1-2)

**Goal:** Implement pure business logic with zero external dependencies

### Week 1: Core Value Objects & Entities

**Tasks:**
- [ ] Setup project structure
```bash
flutter create life_sim_game
mkdir -p lib/domain/{entities,value_objects,services,events,repositories}
mkdir -p test/domain/{entities,value_objects,services}
```

- [ ] Implement Value Objects
  - [ ] Stats (with validation, clamping)
  - [ ] TimePoint (with advancement logic)
  - [ ] Relationship (with level computation)
  - [ ] Position (x, y coordinates)
  
- [ ] Unit tests for value objects (>90% coverage)

**Code Example:**
```dart
// lib/domain/value_objects/stats.dart
class Stats {
  final int stamina;
  final int charm;
  final int corruption;
  final int intelligence;
  final int cleanliness;
  
  const Stats._({
    required this.stamina,
    required this.charm,
    required this.corruption,
    required this.intelligence,
    required this.cleanliness,
  });
  
  factory Stats.initial() => const Stats._(
    stamina: 100,
    charm: 50,
    corruption: 0,
    intelligence: 50,
    cleanliness: 100,
  );
  
  factory Stats.create({
    required int stamina,
    required int charm,
    required int corruption,
    required int intelligence,
    required int cleanliness,
  }) {
    return Stats._(
      stamina: stamina.clamp(0, 100),
      charm: charm.clamp(0, 100),
      corruption: corruption.clamp(0, 100),
      intelligence: intelligence.clamp(0, 100),
      cleanliness: cleanliness.clamp(0, 100),
    );
  }
  
  Stats modify(StatType type, int delta) {
    switch (type) {
      case StatType.stamina:
        return _copyWith(stamina: (stamina + delta).clamp(0, 100));
      // ... other cases
    }
  }
  
  // Equality, hashCode, copyWith, toJson, fromJson...
}
```

### Week 2: Player Aggregate & Domain Services

**Tasks:**
- [ ] Implement Player (Aggregate Root)
  - [ ] Basic state (stats, time, location, money)
  - [ ] Behaviors (modifyStat, advanceTime, rest)
  - [ ] InteractionHistory management
  - [ ] Domain event emission
  
- [ ] Implement Domain Services
  - [ ] EventTriggerService
  - [ ] RelationshipService
  - [ ] ExplorationService
  
- [ ] Comprehensive unit tests (>80% coverage)
- [ ] Documentation of all domain rules

**Acceptance Criteria:**
- All domain logic testable without UI/persistence
- No imports from Flutter framework
- No imports from infrastructure layer
- Test coverage >= 80%
- All invariants documented and enforced

---

## Phase 2: Application Layer (Week 3-4)

**Goal:** Orchestrate domain logic through use cases

### Week 3: Core Use Cases

**Tasks:**
- [ ] Setup application layer structure
- [ ] Define Result type for error handling
- [ ] Implement use cases:
  - [ ] EnterLocationUseCase
  - [ ] InteractWithPointUseCase
  - [ ] ExecuteEventUseCase
  - [ ] ExecuteActionUseCase
  
- [ ] Integration tests with mocked repositories

### Week 4: Supporting Use Cases

**Tasks:**
- [ ] Implement remaining use cases:
  - [ ] RestUseCase
  - [ ] DiscoverInteractionPointUseCase
  - [ ] ProcessNPCDialogueUseCase
  - [ ] SaveGameUseCase
  - [ ] LoadGameUseCase
  
- [ ] Complete integration test suite
- [ ] Document use case flows

**Acceptance Criteria:**
- All use cases tested with mocks
- No business logic in use cases (only coordination)
- Test scenarios cover happy path + error cases
- Clear separation from domain layer

---

## Phase 3: Infrastructure Layer (Week 5-6)

**Goal:** Implement persistence and data loading

### Week 5: Repository Implementations

**Tasks:**
- [ ] Implement PlayerRepositoryImpl
  - [ ] JSON serialization/deserialization
  - [ ] SharedPreferences integration
  - [ ] Version migration mechanism
  - [ ] InteractionHistory persistence ⭐
  
- [ ] Implement save system
  - [ ] Multiple save slots (10)
  - [ ] Save metadata (timestamp, day, money)
  - [ ] Backup before overwrite
  
- [ ] Persistence integration tests

**Save File Structure:**
```json
{
  "version": 1,
  "player": {
    "stats": {...},
    "currentTime": {"day": 5, "timeSlot": "afternoon"},
    "money": 850,
    "discoveredInteractionPoints": {
      "cafe": ["counter", "window_seat", "trash_can"]
    },
    "interactionHistories": {
      "cafe_counter": {
        "totalInteractions": 7,
        "completedEvents": ["buy_coffee"],
        "lastInteractionTime": {"day": 5, "timeSlot": "morning"}
      }
    }
  },
  "saved_at": "2025-01-16T10:30:00Z"
}
```

### Week 6: Data Loaders

**Tasks:**
- [ ] Implement LocationRepositoryImpl
  - [ ] YAML parser for locations
  - [ ] InteractionPoint loading
  
- [ ] Implement EventRepositoryImpl
  - [ ] YAML parser for events
  - [ ] Condition parsing
  - [ ] Dialogue variant loading
  
- [ ] Create initial data files (placeholder content)
  ```
  assets/data/locations/cafe.yaml
  assets/data/events/cafe_events.yaml
  ```

**Acceptance Criteria:**
- Save/load works correctly
- InteractionHistory persisted and restored
- Version migration tested (v0 → v1)
- Corrupted save handling graceful
- Data files load without errors

---

## Phase 4: Presentation Layer MVP (Week 7-8)

**Goal:** Playable single-scene demo

### Week 7: Basic UI Components

**Tasks:**
- [ ] Setup Riverpod providers
  - [ ] PlayerProvider
  - [ ] CurrentLocationProvider
  - [ ] VisiblePointsProvider
  
- [ ] Implement core screens:
  - [ ] HomeScreen (stats display, map)
  - [ ] LocationScreen (background, interaction points)
  - [ ] DialogueScreen (text, choices)
  
- [ ] Implement widgets:
  - [ ] StatBar (progress bars with animations)
  - [ ] InteractionPointIcon (clickable markers)
  - [ ] DialogueBox (text display, continue button)
  - [ ] ChoiceButtons (option selection)

**UI Mockup:**
```
┌──────────────────────────────────────┐
│  ☀️ Day 5 - Afternoon    💰 $850    │
├──────────────────────────────────────┤
│  [Cafe Background Image]             │
│                                      │
│     🔍 Counter                       │
│                                      │
│              ❗ Window Seat          │
│                                      │
│  🔍 Trash Can                        │
│                                      │
├──────────────────────────────────────┤
│  [Work] [Rest] [Leave]               │
│  ❤️ 60  ✨ 55  😈 10  🧠 48  🚿 70  │
└──────────────────────────────────────┘
```

### Week 8: Integration & Testing

**Tasks:**
- [ ] Connect UI to use cases via Riverpod
- [ ] Implement navigation flow
- [ ] Add placeholder assets:
  - [ ] Background images (colored placeholders with text)
  - [ ] Character sprites (simple colored shapes)
  - [ ] UI icons
  
- [ ] Implement SaveLoadScreen
- [ ] Widget tests for critical UI components
- [ ] End-to-end test: Full game loop

**Test Scenario:**
```
1. Start new game
2. Enter cafe location
3. Click window seat → Trigger event
4. Make choice → See stats change
5. Time advances
6. Save game
7. Load game → Verify state restored
```

**Acceptance Criteria:**
- Can play through cafe scene completely
- Interaction count correctly increments
- Dialogue variants change based on count
- Stats update visibly
- Save/load preserves all state
- Stamina = 0 forces rest

---

## Phase 5: Vertical Slice (Week 9-10)

**Goal:** Complete 1-character route playable in 14 days

### Content Scope

**Locations:** 3
- Home (bed, shower, fridge)
- Office (desk, break room)
- Cafe (counter, window seat, bathroom, trash can, bookshelf)

**Characters:** 1
- Alex (full romance route)

**Events:** 15
- Main story: 5 (Alex romance progression)
- Side content: 10 (daily interactions, discoveries)

**Game Length:** 14 in-game days

### Week 9: Content Creation

**Tasks:**
- [ ] Write Alex character profile
  - Personality, background, preferences
  - Corruption range: 0-40 (pure love route)
  - Affection milestones: 20, 40, 60, 80
  
- [ ] Write main story events (STAR format):
  - [ ] Day 3: First meeting (cafe)
  - [ ] Day 5: Office encounter (if applicable)
  - [ ] Day 7: First date
  - [ ] Day 10: Relationship deepens
  - [ ] Day 14: Confession ending
  
- [ ] Write side events:
  - [ ] Cafe daily interactions (counter, window seat)
  - [ ] Office work routine
  - [ ] Home solitary events
  - [ ] Random discoveries
  
- [ ] Write dialogue variants (3+ per repeatable event)

### Week 10: Implementation & Testing

**Tasks:**
- [ ] Convert STAR scenarios to YAML
- [ ] Implement all events in game
- [ ] Create NPC hints system
  - Alex mentions window seat
  - Barista mentions trash can
  
- [ ] Playtesting:
  - [ ] Pure love route (corruption < 30)
  - [ ] Cold route (low affection)
  - [ ] Interaction count verification
  - [ ] Hint unlock verification
  
- [ ] Balance adjustment based on feedback

**