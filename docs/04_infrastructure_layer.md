# Infrastructure Layer - Implementation Guide

## Purpose

Infrastructure Layer handles:
- Persistence (save/load files)
- External data loading (YAML scenarios)
- Third-party integrations
- Technical implementation details

**Does NOT contain business logic** - that lives in Domain Layer.

---

## Architecture Overview

```
Infrastructure Layer
├── persistence/
│   ├── player_repository_impl.dart
│   ├── save_system.dart
│   └── json_serializer.dart
├── scenario_loader/
│   ├── location_repository_impl.dart
│   ├── event_repository_impl.dart
│   ├── character_repository_impl.dart
│   ├── yaml_parser.dart
│   └── condition_parser.dart
└── event_bus/
    └── event_bus_impl.dart
```

---

## Persistence System

### Save File Format

**Location:** `SharedPreferences` (mobile/desktop) or `localStorage` (web)

**Key Pattern:** `save_slot_{0-9}`

**JSON Structure:**
```json
{
  "version": 1,
  "saved_at": "2025-10-16T10:30:00.000Z",
  "player": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "stats": {
      "stamina": 60,
      "charm": 55,
      "corruption": 10,
      "intelligence": 48,
      "cleanliness": 70
    },
    "personality": "neutral",
    "current_time": {
      "day": 5,
      "time_slot": "afternoon"
    },
    "current_location": "cafe",
    "money": 850,
    "inventory": {
      "coffee": 2,
      "coupon": 1,
      "mysterious_letter": 1
    },
    "relationships": {
      "alex": {
        "affection": 45,
        "level": "friend",
        "status": "normal",
        "leverage_count": 0
      },
      "bob": {
        "affection": 20,
        "level": "acquaintance",
        "status": "normal",
        "leverage_count": 0
      }
    },
    "story_flags": [
      "met_alex",
      "met_bob",
      "regular_customer",
      "found_secret_note"
    ],
    "active_buffs": [
      {
        "id": "energy_drink_buff",
        "type": "stamina_immunity",
        "duration": {
          "type": "time_based",
          "expires_at": {
            "day": 5,
            "time_slot": "night"
          }
        },
        "source_item": "energy_drink"
      }
    ],
    "discovered_interaction_points": {
      "cafe": ["counter", "window_seat", "trash_can"],
      "home": ["bed", "shower", "fridge"],
      "office": ["desk", "break_room"]
    },
    "interaction_histories": {
      "cafe_counter": {
        "total_interactions": 7,
        "completed_events": ["buy_coffee"],
        "last_interaction_time": {
          "day": 5,
          "time_slot": "morning"
        }
      },
      "cafe_window_seat": {
        "total_interactions": 3,
        "completed_events": ["meet_alex_first_time", "alex_date"],
        "last_interaction_time": {
          "day": 4,
          "time_slot": "afternoon"
        }
      }
    },
    "pending_hints": [
      {
        "location_id": "park",
        "point_id": "pond",
        "hint_text": "Alex mentioned a quiet spot by the pond",
        "hint_source": {
          "type": "npc_dialogue",
          "npc_id": "alex"
        },
        "unlock_immediately": false,
        "timestamp": {
          "day": 4,
          "time_slot": "evening"
        }
      }
    ]
  }
}
```

---

## PlayerRepositoryImpl

### Implementation

```dart
// infrastructure/persistence/player_repository_impl.dart

import 'package:shared_preferences/shared_preferences.dart';
import 'dart:convert';

class PlayerRepositoryImpl implements PlayerRepository {
  final SharedPreferences _prefs;
  static const String _currentPlayerKey = 'current_player';
  static const String _saveSlotPrefix = 'save_slot_';
  static const int _maxSlots = 10;
  
  PlayerRepositoryImpl(this._prefs);
  
  @override
  Future<Player> get() async {
    final json = _prefs.getString(_currentPlayerKey);
    if (json == null) {
      return Player.newGame(); // Default new game
    }
    
    try {
      final data = jsonDecode(json) as Map<String, dynamic>;
      return _deserializePlayer(data);
    } catch (e) {
      throw SaveDataCorruptedException('Failed to load player: $e');
    }
  }
  
  @override
  Future<void> save(Player player) async {
    final json = jsonEncode(_serializePlayer(player));
    await _prefs.setString(_currentPlayerKey, json);
  }
  
  @override
  Future<List<SaveSlot>> getSaveSlots() async {
    final slots = <SaveSlot>[];
    
    for (int i = 0; i < _maxSlots; i++) {
      final key = '$_saveSlotPrefix$i';
      final json = _prefs.getString(key);
      final timestampKey = '${key}_timestamp';
      final timestamp = _prefs.getInt(timestampKey);
      
      if (json != null) {
        try {
          final data = jsonDecode(json) as Map<String, dynamic>;
          final playerData = data['player'] as Map<String, dynamic>;
          
          slots.add(SaveSlot(
            slot: i,
            day: playerData['current_time']['day'] as int,
            money: playerData['money'] as int,
            savedAt: timestamp != null
                ? DateTime.fromMillisecondsSinceEpoch(timestamp)
                : null,
            isEmpty: false,
          ));
        } catch (e) {
          // Corrupted save
          slots.add(SaveSlot(
            slot: i,
            isEmpty: false,
            isCorrupted: true,
          ));
        }
      } else {
        slots.add(SaveSlot(slot: i, isEmpty: true));
      }
    }
    
    return slots;
  }
  
  @override
  Future<Player?> loadFromSlot(int slot) async {
    if (slot < 0 || slot >= _maxSlots) {
      throw ArgumentError('Invalid slot number: $slot');
    }
    
    final key = '$_saveSlotPrefix$slot';
    final json = _prefs.getString(key);
    
    if (json == null) return null;
    
    try {
      final data = jsonDecode(json) as Map<String, dynamic>;
      
      // Version migration
      final version = data['version'] as int? ?? 0;
      final migratedData = _migrateVersion(data, version);
      
      return _deserializePlayer(migratedData['player']);
    } catch (e) {
      throw SaveDataCorruptedException('Failed to load slot $slot: $e');
    }
  }
  
  @override
  Future<void> saveToSlot(int slot, Player player) async {
    if (slot < 0 || slot >= _maxSlots) {
      throw ArgumentError('Invalid slot number: $slot');
    }
    
    // Backup existing save
    final key = '$_saveSlotPrefix$slot';
    final existing = _prefs.getString(key);
    if (existing != null) {
      await _prefs.setString('${key}_backup', existing);
    }
    
    // Save new data
    final saveData = {
      'version': SaveSystem.currentVersion,
      'saved_at': DateTime.now().toIso8601String(),
      'player': _serializePlayer(player),
    };
    
    final json = jsonEncode(saveData);
    await _prefs.setString(key, json);
    await _prefs.setInt(
      '${key}_timestamp',
      DateTime.now().millisecondsSinceEpoch,
    );
  }
  
  Map<String, dynamic> _serializePlayer(Player player) {
    return {
      'id': player.id.value,
      'stats': {
        'stamina': player.stats.stamina,
        'charm': player.stats.charm,
        'corruption': player.stats.corruption,
        'intelligence': player.stats.intelligence,
        'cleanliness': player.stats.cleanliness,
      },
      'personality': player.personality.name,
      'current_time': {
        'day': player.currentTime.day,
        'time_slot': player.currentTime.timeSlot.name,
      },
      'current_location': player.currentLocation.value,
      'money': player.money,
      'inventory': player.inventory.items,
      'relationships': player.relationships.map(
        (charId, rel) => MapEntry(
          charId.value,
          {
            'affection': rel.affection,
            'level': rel.level.name,
            'status': rel.status.name,
            'leverage_count': rel.leverageCount,
          },
        ),
      ),
      'story_flags': player.storyFlags.toList(),
      'active_buffs': player.activeBuffs.map((buff) => {
        'id': buff.id.value,
        'type': buff.type.name,
        'duration': _serializeBuffDuration(buff.duration),
        'source_item': buff.sourceItem?.value,
      }).toList(),
      'discovered_interaction_points': player.discoveredInteractionPoints.map(
        (locId, points) => MapEntry(
          locId.value,
          points.map((p) => p.value).toList(),
        ),
      ),
      'interaction_histories': player.interactionHistories.map(
        (pointId, history) => MapEntry(
          pointId.value,
          {
            'total_interactions': history.totalInteractions,
            'completed_events': history.completedEvents.map((e) => e.value).toList(),
            'last_interaction_time': history.lastInteractionTime != null
                ? {
                    'day': history.lastInteractionTime!.day,
                    'time_slot': history.lastInteractionTime!.timeSlot.name,
                  }
                : null,
          },
        ),
      ),
      'pending_hints': player.pendingHints.map((hint) => {
        'location_id': hint.locationId.value,
        'point_id': hint.pointId.value,
        'hint_text': hint.hintText,
        'hint_source': {
          'type': hint.hintSource.type.name,
          'npc_id': hint.hintSource.npcId?.value,
        },
        'unlock_immediately': hint.unlockImmediately,
        'timestamp': {
          'day': hint.timestamp.day,
          'time_slot': hint.timestamp.timeSlot.name,
        },
      }).toList(),
    };
  }
  
  Player _deserializePlayer(Map<String, dynamic> data) {
    // Reverse of serialization
    // Implementation details...
    return Player(/* ... */);
  }
  
  Map<String, dynamic> _migrateVersion(
    Map<String, dynamic> data,
    int fromVersion,
  ) {
    var migratedData = Map<String, dynamic>.from(data);
    
    // v0 → v1: Added corruption stat
    if (fromVersion < 1) {
      migratedData['player']['stats']['corruption'] = 0;
    }
    
    // v1 → v2: Added interaction histories
    if (fromVersion < 2) {
      migratedData['player']['interaction_histories'] = {};
    }
    
    // v2 → v3: Added pending hints
    if (fromVersion < 3) {
      migratedData['player']['pending_hints'] = [];
    }
    
    migratedData['version'] = SaveSystem.currentVersion;
    return migratedData;
  }
}

class SaveSystem {
  static const int currentVersion = 3;
}

class SaveSlot {
  final int slot;
  final bool isEmpty;
  final bool isCorrupted;
  final int? day;
  final int? money;
  final DateTime? savedAt;
  
  SaveSlot({
    required this.slot,
    this.isEmpty = false,
    this.isCorrupted = false,
    this.day,
    this.money,
    this.savedAt,
  });
}

class SaveDataCorruptedException implements Exception {
  final String message;
  SaveDataCorruptedException(this.message);
}
```

---

## Scenario Data Format

### Location YAML

```yaml
# assets/data/locations/cafe.yaml

location:
  id: cafe
  name: "Sunny Cafe"
  background_image: "assets/images/backgrounds/bg_cafe.png"
  unlock_conditions: []  # Empty = unlocked from start
  
  basic_actions:
    - work_part_time
    - rest
    - leave
  
  interaction_points:
    - id: counter
      name: "Counter"
      position: {x: 150, y: 300}
      visibility: always_visible
      fallback_dialogue: "The counter is empty right now."
      discovery_hint: null
      
    - id: window_seat
      name: "Window Seat"
      position: {x: 450, y: 250}
      visibility: always_visible
      fallback_dialogue: "The window seat is empty and quiet."
      discovery_hint:
        hint_text: "I usually sit by the window to read."
        hint_source:
          type: npc_dialogue
          npc_id: alex
        unlock_immediately: true
        
    - id: trash_can
      name: "Trash Can"
      position: {x: 100, y: 450}
      visibility: hidden_until_discovered
      fallback_dialogue: "The trash can is empty."
      discovery_hint:
        hint_text: "People sometimes throw out good stuff in the back..."
        hint_source:
          type: npc_dialogue
          npc_id: barista
        unlock_immediately: true
        
    - id: bookshelf
      name: "Bookshelf"
      position: {x: 50, y: 200}
      visibility: conditional_visible
      visibility_conditions:
        - type: stat
          stat: intelligence
          min: 40
      fallback_dialogue: "You've already looked through all the books."
```

### Event YAML

```yaml
# assets/data/events/cafe_counter_buy_coffee.yaml

event:
  id: buy_coffee
  location_id: cafe
  interaction_point_id: counter
  type: minor_branch
  time_consumption: none
  force_leave_scene: false
  priority: 10
  repeatability: unlimited
  random_chance: null
  
  conditions: []  # No conditions, always available
  
  dialogue_variants:
    1:
      speaker: "Barista"
      text: "Welcome! What can I get you?"
      sprite: "barista_smile"
      inner_thought: null
      effects:
        money: -100
        items:
          coffee: 1
        stats:
          stamina: 10
    
    2:
      speaker: "Barista"
      text: "Back again? Same as before?"
      sprite: "barista_smile"
      effects:
        money: -100
        items:
          coffee: 1
        stats:
          stamina: 10
    
    5:
      speaker: "Barista"
      text: "Regular customer! Here, have a discount."
      sprite: "barista_happy"
      effects:
        money: -80
        items:
          coffee: 1
        stats:
          stamina: 10
        flags:
          add: ["regular_customer"]
        hints:
          - location_id: cafe
            point_id: trash_can
            hint_text: "People throw out good stuff in the back..."
            hint_source:
              type: npc_dialogue
              npc_id: barista
            unlock_immediately: true
```

### Complex Event YAML (with choices)

```yaml
# assets/data/events/cafe_meet_alex.yaml

event:
  id: meet_alex_first_time
  location_id: cafe
  interaction_point_id: window_seat
  type: major_story
  time_consumption: one_slot
  force_leave_scene: true
  priority: 100
  repeatability: once
  
  conditions:
    - type: day
      min: 3
    - type: flag
      flag: met_alex
      must_not_have: true
  
  dialogue_variants:
    -1:  # Default (non-repeating event uses -1)
      phases:
        - type: narration
          text: "You sit by the window, enjoying the peaceful atmosphere."
          
        - type: narration
          text: "Suddenly, someone rushes past and bumps your table. Your coffee spills!"
          
        - type: dialogue
          speaker: "???"
          text: "Oh no! I'm so sorry! I didn't see you there!"
          sprite: "alex_surprised"
          
        - type: choice
          prompt: "How do you respond?"
          options:
            - id: kind_response
              text: "It's okay, accidents happen. Are you alright?"
              personality_trait: empathetic
              inner_thought: "The person looks more upset than I am. It's just coffee."
              next_dialogue:
                - speaker: "Alex"
                  text: "Thank you for being so understanding... I'm Alex."
                  sprite: "alex_smile"
              effects:
                relationships:
                  alex: 5
                stats:
                  charm: 2
                  cleanliness: -5
                flags:
                  add: ["met_alex", "alex_first_impression_good"]
                unlocks:
                  locations: []
                  points:
                    - location: cafe
                      point: alex_usual_spot
                      
            - id: cold_response
              text: "Just watch where you're going next time."
              personality_trait: reserved
              inner_thought: "Annoying, but not worth making a scene."
              next_dialogue:
                - speaker: "Alex"
                  text: "...Right. Sorry."
                  sprite: "alex_sad"
              effects:
                relationships:
                  alex: -1
                stats:
                  corruption: 1
                  cleanliness: -5
                flags:
                  add: ["met_alex", "alex_first_impression_neutral"]
                  
            - id: angry_response
              text: "Are you kidding me?! Look at this mess!"
              personality_trait: hot_tempered
              inner_thought: "My relaxing afternoon is ruined!"
              next_dialogue:
                - speaker: "Alex"
                  text: "I-I'm so sorry! I'll pay for cleaning!"
                  sprite: "alex_scared"
              effects:
                relationships:
                  alex: -5
                stats:
                  corruption: 3
                  intelligence: -1
                  cleanliness: -5
                flags:
                  add: ["met_alex", "alex_first_impression_bad", "alex_avoids_player"]
```

---

## LocationRepositoryImpl

```dart
// infrastructure/scenario_loader/location_repository_impl.dart

import 'package:yaml/yaml.dart';
import 'package:flutter/services.dart';

class LocationRepositoryImpl implements LocationRepository {
  final Map<LocationId, Location> _cache = {};
  
  @override
  Future<Location> getById(LocationId id) async {
    if (_cache.containsKey(id)) {
      return _cache[id]!;
    }
    
    final location = await _loadLocation(id);
    _cache[id] = location;
    return location;
  }
  
  @override
  Future<List<Location>> getAll() async {
    // Load all location files
    final locationFiles = [
      'cafe',
      'home',
      'office',
      'park',
      'supermarket',
    ];
    
    final locations = <Location>[];
    for (final file in locationFiles) {
      final location = await getById(LocationId(file));
      locations.add(location);
    }
    
    return locations;
  }
  
  @override
  Future<List<Location>> getUnlocked(Player player) async {
    final all = await getAll();
    return all.where((loc) => _isUnlocked(loc, player)).toList();
  }
  
  bool _isUnlocked(Location location, Player player) {
    for (final condition in location.unlockConditions) {
      if (!condition.evaluate(player)) {
        return false;
      }
    }
    return true;
  }
  
  Future<Location> _loadLocation(LocationId id) async {
    final yamlString = await rootBundle.loadString(
      'assets/data/locations/${id.value}.yaml',
    );
    
    final yaml = loadYaml(yamlString) as Map;
    final locData = yaml['location'] as Map;
    
    return Location(
      id: id,
      name: locData['name'] as String,
      backgroundImage: locData['background_image'] as String,
      interactionPoints: _parseInteractionPoints(
        locData['interaction_points'] as List,
      ),
      initialVisiblePoints: _parseInitialVisible(
        locData['interaction_points'] as List,
      ),
      basicActions: (locData['basic_actions'] as List)
          .map((a) => ActionId(a as String))
          .toList(),
      unlockConditions: _parseConditions(
        locData['unlock_conditions'] as List? ?? [],
      ),
    );
  }
  
  List<InteractionPoint> _parseInteractionPoints(List points) {
    return points.map((p) {
      final pointData = p as Map;
      return InteractionPoint(
        id: InteractionPointId(pointData['id'] as String),
        name: pointData['name'] as String,
        position: Position(
          x: pointData['position']['x'] as int,
          y: pointData['position']['y'] as int,
        ),
        visibility: _parseVisibility(pointData['visibility'] as String),
        fallbackDialogue: Dialogue(
          text: pointData['fallback_dialogue'] as String,
        ),
        discoveryHint: _parseDiscoveryHint(pointData['discovery_hint']),
        events: [],  // Events loaded separately
        state: InteractionState.undiscovered,
      );
    }).toList();
  }
  
  Visibility _parseVisibility(String visibility) {
    switch (visibility) {
      case 'always_visible':
        return Visibility.alwaysVisible;
      case 'hidden_until_discovered':
        return Visibility.hiddenUntilDiscovered;
      case 'conditional_visible':
        return Visibility.conditionalVisible;
      default:
        throw ArgumentError('Unknown visibility: $visibility');
    }
  }
  
  DiscoveryHint? _parseDiscoveryHint(dynamic hintData) {
    if (hintData == null) return null;
    
    final hint = hintData as Map;
    return DiscoveryHint(
      hintText: hint['hint_text'] as String,
      hintSource: _parseHintSource(hint['hint_source'] as Map),
      unlockImmediately: hint['unlock_immediately'] as bool? ?? true,
    );
  }
}
```

---

## Condition Parser

```dart
// infrastructure/scenario_loader/condition_parser.dart

class ConditionParser {
  static List<Condition> parse(List<dynamic> conditionsData) {
    return conditionsData.map((c) => _parseCondition(c as Map)).toList();
  }
  
  static Condition _parseCondition(Map<String, dynamic> data) {
    final type = data['type'] as String;
    
    switch (type) {
      case 'day':
        return DayCondition(
          min: data['min'] as int?,
          max: data['max'] as int?,
        );
        
      case 'time_slot':
        return TimeSlotCondition(
          timeSlot: TimeSlot.values.byName(data['time_slot'] as String),
        );
        
      case 'location':
        return LocationCondition(
          locationId: LocationId(data['location_id'] as String),
        );
        
      case 'stat':
        return StatCondition(
          stat: StatType.values.byName(data['stat'] as String),
          min: data['min'] as int?,
          max: data['max'] as int?,
        );
        
      case 'flag':
        return FlagCondition(
          flag: data['flag'] as String,
          mustHave: data['must_have'] as bool? ?? true,
          mustNotHave: data['must_not_have'] as bool? ?? false,
        );
        
      case 'relationship':
        return RelationshipCondition(
          characterId: CharacterId(data['character_id'] as String),
          affectionMin: data['affection_min'] as int?,
          affectionMax: data['affection_max'] as int?,
          status: data['status'] != null
              ? RelationshipStatus.values.byName(data['status'] as String)
              : null,
        );
        
      default:
        throw ArgumentError('Unknown condition type: $type');
    }
  }
}
```

---

## Error Handling

### Custom Exceptions

```dart
// infrastructure/exceptions.dart

class InfrastructureException implements Exception {
  final String message;
  InfrastructureException(this.message);
  
  @override
  String toString() => 'InfrastructureException: $message';
}

class SaveDataCorruptedException extends InfrastructureException {
  SaveDataCorruptedException(String message) : super(message);
}

class DataLoadException extends InfrastructureException {
  DataLoadException(String message) : super(message);
}

class YamlParseException extends InfrastructureException {
  final String fileName;
  final int? line;
  
  YamlParseException(String message, this.fileName, [this.line])
      : super('$fileName${line != null ? ":$line" : ""}: $message');
}
```

---

## Testing Strategy

### Repository Tests

```dart
// test/infrastructure/persistence/player_repository_impl_test.dart

void main() {
  late MockSharedPreferences mockPrefs;
  late PlayerRepositoryImpl repository;
  
  setUp(() {
    mockPrefs = MockSharedPreferences();
    repository = PlayerRepositoryImpl(mockPrefs);
  });
  
  group('PlayerRepositoryImpl', () {
    test('save() persists player data', () async {
      final player = Player.newGame();
      
      await repository.save(player);
      
      verify(mockPrefs.setString('current_player', any)).called(1);
    });
    
    test('get() returns default player when no save exists', () async {
      when(mockPrefs.getString(any)).thenReturn(null);
      
      final player = await repository.get();
      
      expect(player, isA<Player>());
      expect(player.stats.stamina, 100);
    });
    
    test('saveToSlot() creates backup before overwriting', () async {
      final player = Player.newGame();
      when(mockPrefs.getString('save_slot_0'))
          .thenReturn('{"old": "data"}');
      
      await repository.saveToSlot(0, player);
      
      verify(mockPrefs.setString('save_slot_0_backup', '{"old": "data"}'))
          .called(1);
    });
    
    test('loadFromSlot() migrates old versions', () async {
      final oldSave = jsonEncode({
        'version': 1,  // Old version
        'player': {
          'stats': {
            'stamina': 60,
            // corruption missing in v1
          },
        },
      });
      when(mockPrefs.getString('save_slot_0')).thenReturn(oldSave);
      
      final player = await repository.loadFromSlot(0);
      
      expect(player, isNotNull);
      expect(player!.stats.corruption, 0);  // Default value added
    });
  });
}
```

---

## Performance Optimization

### Asset Preloading

```dart
// infrastructure/asset_manager.dart

class AssetManager {
  final Map<String, String> _imageCache = {};
  final Map<String, dynamic> _dataCache = {};
  
  Future<void> preloadCriticalAssets() async {
    // Preload common locations
    await Future.wait([
      _preloadImage('assets/images/backgrounds/bg_cafe.png'),
      _preloadImage('assets/images/backgrounds/bg_home.png'),
      _preloadImage('assets/images/backgrounds/bg_office.png'),
    ]);
    
    // Preload common character sprites
    await Future.wait([
      _preloadImage('assets/images/sprites/alex_normal.png'),
      _preloadImage('assets/images/sprites/alex_smile.png'),
      _preloadImage('assets/images/sprites/alex_sad.png'),
    ]);
  }
  
  Future<void> _preloadImage(String path) async {
    final data = await rootBundle.load(path);
    _imageCache[path] = path;  // Mark as cached
  }
}
```

### Event Indexing

```dart
// infrastructure/scenario_loader/event_index.dart

class EventIndex {
  final Map<LocationId, List<EventId>> _locationIndex = {};
  final Map<InteractionPointId, List<EventId>> _pointIndex = {};
  
  void buildIndex(List<GameEvent> events) {
    for (final event in events) {
      // Index by location
      _locationIndex
          .putIfAbsent(event.locationId, () => [])
          .add(event.id);
      
      // Index by interaction point
      _pointIndex
          .putIfAbsent(event.interactionPointId, () => [])
          .add(event.id);
    }
  }
  
  List<EventId> getEventsForPoint(InteractionPointId pointId) {
    return _pointIndex[pointId] ?? [];
  }
}
```

---

## Best Practices

### 1. Always Handle Errors

```dart
// ❌ Bad: Silent failure
Future<Location> loadLocation(LocationId id) async {
  final yaml = await rootBundle.loadString('assets/locations/${id.value}.yaml');
  return parseLocation(yaml);  // Could throw
}

// ✅ Good: Explicit error handling
Future<Location> loadLocation(LocationId id) async {
  try {
    final yaml = await rootBundle.loadString(
      'assets/locations/${id.value}.yaml',
    );
    return parseLocation(yaml);
  } on FlutterError catch (e) {
    throw DataLoadException('Location file not found: ${id.value}');
  } on YamlException catch (e) {
    throw YamlParseException(
      'Invalid YAML syntax',
      '${id.value}.yaml',
      e.line,
    );
  }
}
```

### 2. Validate All External Data

```dart
// Always validate data loaded from files

void _validateLocationData(Map data) {
  if (!data.containsKey('id')) {
    throw YamlParseException('Missing required field: id', 'location.yaml');
  }
  
  if (!data.containsKey('name')) {
    throw YamlParseException('Missing required field: name', 'location.yaml');
  }
  
  // Validate interaction points
  final points = data['interaction_points'] as List?;
  if (points == null || points.isEmpty) {
    throw YamlParseException(
      'Location must have at least one interaction point',
      'location.yaml',
    );
  }
}
```

### 3. Cache Aggressively

```dart
// Cache parsed data to avoid repeated parsing

class LocationRepositoryImpl implements LocationRepository {
  final Map<LocationId, Location> _cache = {};
  
  @override
  Future<Location> getById(LocationId id) async {
    // Check cache first
    if (_cache.containsKey(id)) {
      return _cache[id]!;
    }
    
    // Load and cache
    final location = await _loadLocation(id);
    _cache[id] = location;
    return location;
  }
  
  // Invalidate cache if needed (e.g., hot reload in dev)
  void clearCache() {
    _cache.clear();
  }
}
```

### 4. Version Everything

```dart
// Always include version in save files

class SaveSystem {
  static const int currentVersion = 3;
  
  static Map<String, dynamic> createSaveData(Player player) {
    return {
      'version': currentVersion,  // Critical!
      'saved_at': DateTime.now().toIso8601String(),
      'player': serializePlayer(player),
    };
  }
  
  static Player loadSaveData(Map<String, dynamic> data) {
    final version = data['version'] as int? ?? 0;
    
    // Migrate if necessary
    if (version < currentVersion) {
      data = migrateVersion(data, version);
    }
    
    return deserializePlayer(data['player']);
  }
}
```

### 5. Fail Fast on Corruption

```dart
// Don't try to fix corrupted data - fail immediately

Future<Player> loadPlayer() async {
  try {
    final json = await loadSaveFile();
    return deserializePlayer(json);
  } catch (e) {
    // Log error for debugging
    print('Save file corrupted: $e');
    
    // Throw specific exception
    throw SaveDataCorruptedException(
      'Save file is corrupted and cannot be loaded. '
      'Please start a new game or load a different save.',
    );
  }
}
```

---

## Migration Examples

### Version 0 → 1: Adding Corruption Stat

```dart
Map<String, dynamic> migrateV0toV1(Map<String, dynamic> data) {
  final playerData = data['player'] as Map<String, dynamic>;
  final stats = playerData['stats'] as Map<String, dynamic>;
  
  // Add corruption with default value
  stats['corruption'] = 0;
  
  print('Migrated save from v0 to v1: Added corruption stat');
  return data;
}
```

### Version 1 → 2: Adding Interaction Histories

```dart
Map<String, dynamic> migrateV1toV2(Map<String, dynamic> data) {
  final playerData = data['player'] as Map<String, dynamic>;
  
  // Add empty interaction histories
  playerData['interaction_histories'] = <String, dynamic>{};
  
  print('Migrated save from v1 to v2: Added interaction histories');
  return data;
}
```

### Version 2 → 3: Restructuring Relationships

```dart
Map<String, dynamic> migrateV2toV3(Map<String, dynamic> data) {
  final playerData = data['player'] as Map<String, dynamic>;
  final oldRelationships = playerData['relationships'] as Map<String, dynamic>;
  
  // Old format: just affection value
  // New format: object with affection, level, status
  
  final newRelationships = <String, dynamic>{};
  for (final entry in oldRelationships.entries) {
    final affection = entry.value as int;
    newRelationships[entry.key] = {
      'affection': affection,
      'level': _computeLevel(affection),
      'status': 'normal',
      'leverage_count': 0,
    };
  }
  
  playerData['relationships'] = newRelationships;
  
  print('Migrated save from v2 to v3: Restructured relationships');
  return data;
}

String _computeLevel(int affection) {
  if (affection >= 81) return 'lover';
  if (affection >= 61) return 'close';
  if (affection >= 41) return 'friend';
  if (affection >= 21) return 'acquaintance';
  return 'stranger';
}
```

---

## File Loading Utilities

### Batch Loading

```dart
// infrastructure/scenario_loader/batch_loader.dart

class BatchLoader {
  final LocationRepository _locationRepo;
  final EventRepository _eventRepo;
  final CharacterRepository _characterRepo;
  
  BatchLoader(this._locationRepo, this._eventRepo, this._characterRepo);
  
  /// Load all game data at startup
  Future<void> loadAllData() async {
    await Future.wait([
      _loadAllLocations(),
      _loadAllEvents(),
      _loadAllCharacters(),
    ]);
  }
  
  Future<void> _loadAllLocations() async {
    final locations = await _locationRepo.getAll();
    print('Loaded ${locations.length} locations');
  }
  
  Future<void> _loadAllEvents() async {
    final events = await _eventRepo.getAll();
    print('Loaded ${events.length} events');
  }
  
  Future<void> _loadAllCharacters() async {
    final characters = await _characterRepo.getAll();
    print('Loaded ${characters.length} characters');
  }
}
```

### Lazy Loading

```dart
// infrastructure/scenario_loader/lazy_loader.dart

class LazyEventLoader {
  final Map<EventId, GameEvent> _cache = {};
  
  Future<GameEvent> getEvent(EventId id) async {
    if (_cache.containsKey(id)) {
      return _cache[id]!;
    }
    
    // Only load when needed
    final event = await _loadEventFromFile(id);
    _cache[id] = event;
    return event;
  }
  
  Future<GameEvent> _loadEventFromFile(EventId id) async {
    final yaml = await rootBundle.loadString(
      'assets/data/events/${id.value}.yaml',
    );
    return parseEvent(yaml);
  }
}
```

---

## Dependency Injection Setup

### Using get_it

```dart
// infrastructure/dependency_injection.dart

import 'package:get_it/get_it.dart';

final getIt = GetIt.instance;

Future<void> setupDependencies() async {
  // SharedPreferences (singleton)
  final prefs = await SharedPreferences.getInstance();
  getIt.registerSingleton<SharedPreferences>(prefs);
  
  // Repositories
  getIt.registerLazySingleton<PlayerRepository>(
    () => PlayerRepositoryImpl(getIt<SharedPreferences>()),
  );
  
  getIt.registerLazySingleton<LocationRepository>(
    () => LocationRepositoryImpl(),
  );
  
  getIt.registerLazySingleton<EventRepository>(
    () => EventRepositoryImpl(),
  );
  
  getIt.registerLazySingleton<CharacterRepository>(
    () => CharacterRepositoryImpl(),
  );
  
  // Use Cases
  getIt.registerFactory(
    () => InteractWithPointUseCase(
      playerRepo: getIt<PlayerRepository>(),
      locationRepo: getIt<LocationRepository>(),
      eventRepo: getIt<EventRepository>(),
    ),
  );
  
  // ... other use cases
}
```

---

## Asset Management

### Directory Structure

```
assets/
├── data/
│   ├── locations/
│   │   ├── cafe.yaml
│   │   ├── home.yaml
│   │   ├── office.yaml
│   │   ├── park.yaml
│   │   └── supermarket.yaml
│   ├── events/
│   │   ├── cafe/
│   │   │   ├── counter_buy_coffee.yaml
│   │   │   ├── window_seat_meet_alex.yaml
│   │   │   └── trash_can_find_coupon.yaml
│   │   ├── home/
│   │   └── office/
│   ├── characters/
│   │   ├── alex.yaml
│   │   ├── bob.yaml
│   │   └── charlie.yaml
│   └── narrative_rules.yaml
├── images/
│   ├── backgrounds/
│   │   ├── bg_cafe.png
│   │   ├── bg_home.png
│   │   └── bg_office.png
│   ├── sprites/
│   │   ├── alex_normal.png
│   │   ├── alex_smile.png
│   │   ├── alex_sad.png
│   │   └── alex_surprised.png
│   └── ui/
│       ├── icon_magnifying_glass.png
│       ├── icon_exclamation.png
│       └── icon_question.png
└── audio/
    ├── bgm/
    │   ├── cafe_bgm.mp3
    │   ├── home_bgm.mp3
    │   └── office_bgm.mp3
    └── sfx/
        ├── click.wav
        ├── stat_increase.wav
        └── stat_decrease.wav
```

### pubspec.yaml Configuration

```yaml
flutter:
  assets:
    - assets/data/locations/
    - assets/data/events/cafe/
    - assets/data/events/home/
    - assets/data/events/office/
    - assets/data/characters/
    - assets/data/narrative_rules.yaml
    - assets/images/backgrounds/
    - assets/images/sprites/
    - assets/images/ui/
    - assets/audio/bgm/
    - assets/audio/sfx/
```

---

## Logging & Debugging

### Structured Logging

```dart
// infrastructure/logging/logger.dart

enum LogLevel { debug, info, warning, error }

class GameLogger {
  static void log(String message, {LogLevel level = LogLevel.info}) {
    final timestamp = DateTime.now().toIso8601String();
    final prefix = _getLevelPrefix(level);
    print('[$timestamp] $prefix $message');
  }
  
  static void debug(String message) => log(message, level: LogLevel.debug);
  static void info(String message) => log(message, level: LogLevel.info);
  static void warning(String message) => log(message, level: LogLevel.warning);
  static void error(String message) => log(message, level: LogLevel.error);
  
  static String _getLevelPrefix(LogLevel level) {
    switch (level) {
      case LogLevel.debug: return '[DEBUG]';
      case LogLevel.info: return '[INFO]';
      case LogLevel.warning: return '[WARN]';
      case LogLevel.error: return '[ERROR]';
    }
  }
}

// Usage
GameLogger.info('Loading location: cafe');
GameLogger.error('Failed to parse event: invalid YAML syntax');
```

### Debug Tools

```dart
// infrastructure/debug/save_inspector.dart

class SaveInspector {
  static void printSaveData(Map<String, dynamic> data) {
    print('=== Save Data Inspector ===');
    print('Version: ${data['version']}');
    print('Saved at: ${data['saved_at']}');
    
    final player = data['player'] as Map<String, dynamic>;
    print('\nPlayer:');
    print('  Day: ${player['current_time']['day']}');
    print('  Money: ${player['money']}');
    print('  Location: ${player['current_location']}');
    
    print('\nStats:');
    final stats = player['stats'] as Map<String, dynamic>;
    stats.forEach((key, value) {
      print('  $key: $value');
    });
    
    print('\nFlags: ${player['story_flags'].length}');
    print('Relationships: ${player['relationships'].length}');
    print('Interaction histories: ${player['interaction_histories'].length}');
  }
}
```

---

## Common Issues & Solutions

### Issue 1: Save File Too Large

**Problem:** Save file exceeds reasonable size (>1MB)

**Solution:** Compress before saving
```dart
import 'dart:convert';
import 'dart:io';
import 'package:archive/archive.dart';

String compressSaveData(Map<String, dynamic> data) {
  final jsonString = jsonEncode(data);
  final bytes = utf8.encode(jsonString);
  final compressed = GZipEncoder().encode(bytes);
  return base64Encode(compressed!);
}

Map<String, dynamic> decompressSaveData(String compressed) {
  final bytes = base64Decode(compressed);
  final decompressed = GZipDecoder().decodeBytes(bytes);
  final jsonString = utf8.decode(decompressed);
  return jsonDecode(jsonString);
}
```

### Issue 2: Slow Event Loading

**Problem:** Loading 50+ events takes >5 seconds

**Solution:** Implement event indexing
```dart
class EventRepositoryImpl {
  final Map<LocationId, List<GameEvent>> _locationIndex = {};
  
  Future<void> buildIndex() async {
    final allEvents = await _loadAllEvents();
    
    for (final event in allEvents) {
      _locationIndex
          .putIfAbsent(event.locationId, () => [])
          .add(event);
    }
  }
  
  @override
  Future<List<GameEvent>> getEventsForLocation(LocationId id) async {
    return _locationIndex[id] ?? [];
  }
}
```

### Issue 3: YAML Parse Errors

**Problem:** Cryptic YAML parsing errors

**Solution:** Better error messages
```dart
GameEvent parseEvent(String yaml) {
  try {
    final data = loadYaml(yaml) as Map;
    return GameEvent.fromYaml(data);
  } on YamlException catch (e) {
    throw YamlParseException(
      'Syntax error: ${e.message}',
      'event.yaml',
      e.line,
    );
  } on TypeError catch (e) {
    throw YamlParseException(
      'Type mismatch: Expected different data type. ${e.toString()}',
      'event.yaml',
    );
  } on ArgumentError catch (e) {
    throw YamlParseException(
      'Invalid value: ${e.message}',
      'event.yaml',
    );
  }
}
```

---

## Testing Checklist

### Repository Tests
- [ ] Save/load player data
- [ ] Multiple save slots
- [ ] Save slot preview
- [ ] Version migration (v0→v1, v1→v2, v2→v3)
- [ ] Corrupted save handling
- [ ] Backup before overwrite

### Data Loading Tests
- [ ] Load all locations
- [ ] Load events by location
- [ ] Parse complex events with choices
- [ ] Handle missing files gracefully
- [ ] Validate YAML structure
- [ ] Cache performance

### Integration Tests
- [ ] Full save → quit → load cycle
- [ ] Interaction count persistence
- [ ] Discovered points persistence
- [ ] Story flags persistence
- [ ] Relationship data integrity

---

## Performance Targets

| Metric | Target | Acceptable | Unacceptable |
|--------|--------|------------|--------------|
| Save time | <100ms | <500ms | >1s |
| Load time | <200ms | <1s | >2s |
| Event loading | <50ms | <200ms | >500ms |
| YAML parsing | <10ms | <50ms | >100ms |
| Cache hit rate | >90% | >70% | <50% |

---

## Next Steps

After implementing Infrastructure Layer:

1. **Write integration tests** for all repositories
2. **Create sample YAML files** for development
3. **Test migration** from empty save → v1 → v2 → v3
4. **Verify performance** with large data sets (100+ events)
5. **Move to Presentation Layer** (UI implementation)

---

**Last Updated**: 2025-10-16  
**Version**: 1.0