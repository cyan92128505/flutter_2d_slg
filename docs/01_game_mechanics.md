# Game Mechanics - Complete Definition

## Core Loop

```
Wake Up → Choose Location → Explore Interaction Points → Trigger Events
→ Make Choices → Affect Stats/Relationships → Time Advances → Loop
→ Weekend Events → Seasonal Transitions
```

---

## Time System

### Structure
- **1 Day = 4 Time Slots**: Morning, Afternoon, Evening, Night
- **1 Week = 7 Days**: Monday through Sunday
- **Week Counter**: Tracks game progression (Week 1, Week 2, ...)
- **4 Seasons**: Spring, Summer, Autumn, Winter (event-triggered)
- **Actions consume time slots** (major events only)
- **Sleep** transitions to next day (morning)

### Weekly Cycle
- **Weekdays (Mon-Fri)**: Work available, school events
- **Weekends (Sat-Sun)**: Special events, character dates
- **Week transitions**: Sunday night → Monday morning (week++)

### Seasonal System
- **Trigger Method**: Special events cause season changes
  - Example: Valentine's event → Spring begins
  - Example: Beach party → Summer starts
- **Not time-based**: Seasons don't auto-advance with weeks
- **Effects**:
  - Unlock seasonal locations (beach in summer)
  - Change NPC availability
  - Affect event triggers
  - Modify scene backgrounds

### Stamina Mechanics
- **Range**: 0-100 (cannot be negative)
- **At 0 stamina**: Forced to go home and rest
- **Special items**: Can refill stamina + grant temporary immunity
- **Exploration**: Does NOT consume stamina ⭐

---

## Stat System

### Five Core Attributes

| Attribute        | Range | Purpose                           |
| ---------------- | ----- | --------------------------------- |
| **Stamina**      | 0-100 | Action currency; 0 = must rest    |
| **Charm**        | 0-100 | First impressions; unlock events  |
| **Corruption**   | 0-100 | Route branching (pure/corrupt)    |
| **Intelligence** | 0-100 | Work efficiency; skill checks     |
| **Cleanliness**  | 0-100 | Dating requirement; NPC reactions |

### Stat Modifications

**Increases:**
- Stamina: Sleep, rest, special items
- Charm: Exercise, buy clothes, special items
- Corruption: Bad actions, refuse help, special items
- Intelligence: Work, study, special items
- Cleanliness: Shower

**Decreases:**
- Stamina: Work, dating, exercise
- Charm: Time passage (neglect)
- Intelligence: Failed skill checks (being deceived)
- Cleanliness: Time passage, work, exercise

---

## Exploration System

### Interaction Points

**Definition**: Clickable objects in scenes that trigger events

**Key Features:**
- ⭐ No stamina cost to explore
- Player actively clicks objects
- Events triggered based on conditions
- **Dialogue changes with interaction count per point**

### Hierarchical Locations

**Structure:**
```
Root Location (Home)
├─ Sub-location (Kitchen)
│  ├─ Interaction Point (Stove)
│  └─ Interaction Point (Fridge)
└─ Sub-location (Bedroom)
   ├─ Interaction Point (Bed)
   └─ Interaction Point (Desk)
```

**Navigation:**
- Click on sub-locations to enter
- Interact with specific points within sub-locations
- Parent locations accessible from sub-locations

### Discovery Mechanism

**Initial Visibility:**
- Functional locations (counter, seats) visible
- NPCs always visible when present
- Hidden locations (trash cans, bookshelves) require hints

**Hint Sources:**
- NPC dialogue mentions
- Item descriptions
- Story event rewards
- Conditional reveals (stat thresholds)

### Interaction States

```
Undiscovered → (Hint Unlocks) → Available → (Player Clicks) → Event
```

**Post-Event Behavior:**
- Major Story Event + No other events → Point exhausted
- Major Story Event + Other events → Point remains available
- Minor Branch Event → Always remains available

### Interaction Count Tracking

**Critical Feature:** Each interaction point tracks total clicks

**Example:**
```
Player clicks "Cafe Counter" 7 times total:
  - 1st click: "Welcome! First time here?"
  - 2nd-4th clicks: "Back again? The usual?"
  - 5th+ clicks: "You're a regular now! Here's a discount."
```

**Implementation:**
- Count stored per interaction point (not per event)
- Persisted permanently in save files
- YAML events define dialogue variants by count
- Writers control breakpoints (1, 5, 10, etc.)

---

## Event System

### Event Types

**Major Story Events:**
- Consume 1 time slot
- Force scene exit after completion
- Usually one-time or daily repeatability
- Drive main/character storylines

**Minor Branch Events:**
- No time consumption
- Can continue exploring after completion
- Unlimited repeatability
- Dialogue changes with interaction count
- Provide flavor and small rewards

### Event Conditions

Events trigger when ALL conditions met:
- Time: Week range, weekday, specific time slot
- Season: Spring/Summer/Autumn/Winter requirement
- Location: Specific scene + interaction point
- Stats: Minimum attribute values
- Flags: Story progression markers
- Relationships: Character affection/domination thresholds
- Random: Optional probability (0-1)

### Dialogue Variants

Events can have different dialogue based on **interaction count**:

```yaml
event_cafe_counter:
  dialogue_variants:
    1: "Welcome! First time here?"
    2: "Back again? The usual?"
    5: "Regular customer! Here's a discount."
```

**Selection Logic:**
- Find largest key ≤ current count
- Example: 7th interaction uses variant "5"
- **Stored permanently in save files** ⭐

### Fallback Dialogue

When all events at a point are exhausted:
- Display generic message
- Example: "Nothing interesting here anymore"
- Point remains clickable (not hidden)

---

## Relationship System

### Structure

**Per Character:**
- Affection: 0-100 (numeric value)
- Domination: 0-100 (numeric value)
- Computed Status: Normal / Dating / Dominated / EndingReached

### Affection Changes

**Increases:**
- Dating events
- Gift giving
- Correct dialogue choices
- Character-specific actions

**Decreases:**
- Rejecting dates
- Wrong dialogue choices
- Ignoring character
- Cheating (being caught)

### Multi-Character Romance

**Allowed:** Simultaneous pursuit of multiple characters  
**Risk:** Jealousy events if caught cheating  
**Solution:** Domination mechanic

### Domination Mechanic

**Purpose:** Eliminate jealousy risk

**Implementation:** YAML-driven numeric system

**How It Works:**
```yaml
# Example 1: Item gives domination
item_compromising_photo:
  on_use:
    effects:
      - type: modify_domination
        character: alice
        delta: 20  # Photo gives +20 domination

# Example 2: Event gives domination
event_blackmail_success:
  effects:
    - type: modify_domination
      character: bob
      delta: 30  # Blackmail gives +30 domination

# Example 3: Different items = different power
item_secret_video:
  on_use:
    effects:
      - type: modify_domination
        character: alice
        delta: 50  # Video is more powerful
```

**Threshold:**
- Domination >= 80 → Character accepts multi-romance
- No jealousy events triggered
- Unlocks domination-specific routes

**Design Benefits:**
- Writers control power of each item/event
- No code changes needed for balancing
- Different characters can have different items
- Flexible progression paths

---

## Corruption System

### Single Value Design
- **Range**: 0-100
- **Not typed** (no pure/assertive/dominant subtypes)

### Effects

**Route Locking:**
- Pure characters: Require corruption < 30
- Neutral characters: Accept corruption 20-70
- Corrupt characters: Require corruption >= 60

**Personality Transformation:**
- High corruption triggers personality change events
- Can be reversed via school courses
- Affects available dialogue options

---

## Money & Items

### Money Sources
- Working (job/part-time)
- Event rewards
- Found items (exploration)

### Money Uses
- Buying clothes (permanent charm boost)
- Buying gifts (relationship boost)
- Dating expenses (some dates cost money)
- Food/consumables (stamina recovery)

### Item Categories

**Equipment:**
- Clothes: Permanent +charm
- Accessories: Various stat boosts

**Consumables:**
- Coffee: +stamina
- Energy drinks: +stamina + immunity
- Food: Various effects

**Special Items:**
- Heart's Eye: See cards in minigames
- Leverage items: Increase character domination
- Discount coupons: Save money

---

## Minigame System

### Purpose
Drive story progression at key moments

### Types
- Beer sliding (timing-based)
- Memory matching (card pairs)
- Rock-paper-scissors (strategic)

### Mechanics
- **Difficulty**: Fixed (not stat-based)
- **Rewards**: Scale with player stats
  - Example: Intelligence >= 60 → +50% money reward
- **Cheating**: Special items allow advantages
  - Example: "Heart's Eye" reveals cards
- **Retry**: Depends on event design
  - Some allow retries
  - Others are one-shot

### Failure Consequences
- Lose money
- Lose items
- Stat decrease
- Relationship decrease

---

## Location System

### Initial Locations (5)
- Home: Sleep, shower, rest
- Office: Work, meet coworkers
- Cafe: Socialize, part-time work
- Park: Exercise, random encounters
- Supermarket: Shopping

### Seasonal Locations
Unlocked by season changes:
- Beach (Summer)
- Ski Resort (Winter)
- Cherry Blossom Park (Spring)
- Festival Grounds (Autumn)

### Unlock Mechanism
- Story progression
- Relationship milestones
- Item acquisition
- Stat thresholds
- Season requirements

### Scene Structure

Each location contains:
- Background image (may change with seasons)
- 3-7 interaction points
- Basic actions (work, rest, leave)
- Unlock conditions (if not initial)
- Sub-locations (hierarchical structure)

### Movement
- **No travel cost** (currently)
- Instant scene transition
- Future: May add time/stamina cost for distant locations

---

## Ending System

### Non-Exclusive Design
- Can achieve multiple character endings
- Each character route locks individually
- Other characters remain available

### Trigger Method
- **Player initiated**: Not automatic at day limit
- Special ending events unlock when conditions met
- Player chooses when to trigger ending event

### Ending Conditions (Example)
```
Alex Good Ending:
  - Affection >= 90
  - Completed 10 key events
  - Corruption < 30
  - Week >= 12
  - Player triggers "Proposal" event
```

### Post-Ending Behavior
- Character route locked (no new main story events)
- Repeatable daily events still available
- Other characters unaffected
- DLC can add new content for ended routes

---

## Skill Check System

### Mechanism
Events can require stat checks:

```
Example: Scam Event
  Check: Intelligence >= 60
  Success: Recognize scam, gain $500
  Failure: Lose $1000, intelligence -5
```

### Protection
- Special items can prevent failure
- Example: "Lawyer Card" prevents scam penalties

### All Stats Applicable
- Intelligence: Recognize deception
- Charm: Persuasion checks
- Stamina: Physical challenges
- Others as needed per event

---

## Special Mechanics

### Buff System

**Temporary Effects:**
- Stamina immunity (energy drinks)
- Doubled rewards (lucky items)
- Card vision (cheating items)
- Jealousy immunity (special circumstances)

**Duration Types:**
- Time-based: Until next day
- Event-based: For next N events
- Permanent: Equipment effects

### Flag System

**Story Flags:**
- Binary (true/false) markers
- Track story progression
- Control event availability
- Example: "met_alex", "regular_customer"

**Uses:**
- Event conditions
- Dialogue branching
- Unlock mechanics

---

## Game Flow Example

### Week 2, Wednesday Afternoon - Cafe Visit

```
1. Player: "Go to Cafe"
   → EnterLocationUseCase
   → Display: Background + Visible interaction points
   → Current: Week 2, Wednesday, Afternoon, Spring

2. Player clicks: "Window Seat" (7th time total)
   → InteractWithPointUseCase
   → Check interaction count: 7
   → Trigger: "Regular Customer" event (Minor Branch)

3. Event executes:
   → Dialogue displays: "You're a regular now!"
   → Player chooses: "Accept discount"
   → Effects: alex_affection +5, money +50
   → Time: No consumption (Minor Event)
   → Can continue exploring

4. Player continues exploring
   → Clicks "Counter"
   → Triggers another event
```

---

## Design Principles

### Player Agency
- No forced bad outcomes
- Always provide choices
- Consequences clear but not punishing
- Multiple valid playstyles

### Replayability
- Dialogue variants encourage revisits
- Multiple character routes
- Different corruption paths
- Hidden events to discover
- Seasonal content variations

### Narrative Focus
- Mechanics serve story
- No grind required
- Events feel natural, not mechanical
- Character interactions prioritized

---

## Constraints & Limits

### Hard Limits
- Stamina cannot be negative
- Stats clamped to 0-100
- Cannot act when stamina = 0
- Must sleep to advance day

### Soft Limits
- Recommended 3 actions per day
- Money doesn't cap (but rare to accumulate too much)
- Relationship count unlimited (but jealousy risk)

### Time Limits
- No hard week/day limit (game doesn't end)
- Endings available when conditions met
- Player decides when to finish
- Seasonal events may have week requirements

---

## Summary of Time Progression

```
Granular → Abstract

Time Slot:  Morning → Afternoon → Evening → Night
            (4 per day)
            
Day:        Monday → Tuesday → ... → Sunday
            (7 per week)
            
Week:       Week 1 → Week 2 → Week 3 → ...
            (unlimited)
            
Season:     Spring → Summer → Autumn → Winter
            (event-triggered, not time-based)
```

---

**Last Updated**: 2025-10-23  
**Version**: 1.1 (Added Weekly & Seasonal Systems)