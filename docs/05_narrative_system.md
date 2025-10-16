# Narrative System - STAR Format & Writing Guide

## Overview

The narrative system separates **story content** from **game code**, enabling:
- Non-technical writers to create scenarios
- Easy content iteration without code changes
- Clear separation of narrative and mechanics

---

## STAR Format

### Why STAR?

**STAR = Situation, Task, Action, Result**

This format mirrors natural storytelling structure:
- **Situation**: Set the scene (when, where, who)
- **Task**: Present the conflict or trigger
- **Action**: Player chooses response
- **Result**: Consequences of the choice

### Advantages Over BDD (Given-When-Then)

| Aspect | BDD Format | STAR Format |
|--------|------------|-------------|
| Audience | Engineers | Writers |
| Focus | Technical conditions | Narrative context |
| Example | `player.charm >= 30` | "Feeling confident" |
| Readability | Low for non-technical | High for everyone |

---

## STAR Template

```markdown
# Scene: [Scene Name]

## Interaction Point: [Point Name]

### Event: [Event Name]

**Event Type:** Major Story / Minor Branch
**Time Consumption:** None / 1 Time Slot
**Can Repeat:** Yes / No / Daily
**Force Leave Scene:** Yes / No

#### Situation (Context)
- **Time:** [Day X, Morning/Afternoon/Evening/Night]
- **Location:** [Scene - Point]
- **Prerequisites:** [What must have happened before]
- **Character State:** [Describe using natural language]

> [Opening narrative text]

#### Task (Trigger)
> [What happens to start this event]

**NPC Introduction:** [If applicable]
- Name: [Character]
- Expression: [Emotion]
- Sprite: [sprite_file]

> [NPC dialogue or event description]

#### Action (Player Choices)

##### Option A: [Choice Name]
**Personality Trait:** [kind/assertive/cold/etc.]
**Dialogue:** "[What player says]"

**Inner Thought:**
> [Player's internal monologue]

**NPC Reaction:**
- Expression: [Changed emotion]
- Sprite: [new_sprite]

> [NPC's response]

##### Option B: [Alternative Choice]
[Same structure as Option A]

#### Result (Consequences)

##### If Option A:
**Relationship Changes:**
- [Character] affection +X / -X
- [Another character] affected

**Stat Changes:**
- [Stat]: +X / -X reason

**Story Flags:**
- Set: [flag_name]
- Remove: [flag_name]
- Unlock: [location/point/event]

**Future Impact:**
> [Long-term narrative consequences]

##### If Option B:
[Same structure for alternative outcome]
```

---

## Example Scenario

### Complete STAR Format Example

```markdown
# Scene: Sunny Cafe

## Scene Configuration
Background: bg_cafe.png
Unlock Conditions: None (initial location)

### Initial Visible Points
- counter (Counter)
- window_seat (Window Seat)
- alex_npc (Alex, if present)

### Hidden Points
- bathroom (Bathroom) - Always discoverable
- trash_can (Trash Can) - Unlocked by barista hint
- bookshelf (Bookshelf) - intelligence >= 40

---

## Interaction Point: Window Seat

### Point Configuration
Position: (450, 250)
Visibility: Always Visible
Fallback Dialogue: "The window seat is empty and quiet."

---

### Event: First Meeting with Alex

**Event Type:** Major Story
**Time Consumption:** 1 Time Slot
**Can Repeat:** No (one-time only)
**Force Leave Scene:** Yes

#### Situation
- **Time:** Day 3 or later, Afternoon
- **Location:** Cafe - Window Seat
- **Prerequisites:** Haven't met Alex yet
- **Character State:** Feeling relaxed (stamina >= 50)

> You notice the window seat is empty. Sunlight streams through
> the glass, creating a warm, inviting atmosphere. You decide to
> sit down with your coffee.

#### Task
**Trigger:** Player sits at window seat

> Just as you settle in, someone rushes past your table.
> CRASH! Your coffee cup tips over, spilling across the table.

**NPC Introduction:** Alex
- Expression: Shocked, apologetic
- Sprite: alex_surprised

> "Oh no! I'm so sorry! I didn't see you there!"
> The person looks genuinely distressed, frantically looking
> around for napkins.

#### Action

##### Option A: Kind Response
**Personality Trait:** Empathetic
**Dialogue:** "It's okay, accidents happen. Are you alright?"

**Inner Thought:**
> The person looks more upset about it than you are.
> It's just coffee - no big deal.

**NPC Reaction:**
- Expression: Relieved, grateful
- Sprite: alex_smile

> "Thank you for being so understanding... I'm Alex, by the way."
> Alex extends a hand, smiling warmly.
> "Please, let me buy you another coffee to make up for it."

##### Option B: Cold Response
**Personality Trait:** Independent, Reserved
**Dialogue:** "Just watch where you're going next time."

**Inner Thought:**
> You're annoyed, but it's not worth making a scene over.

**NPC Reaction:**
- Expression: Embarrassed, disappointed
- Sprite: alex_sad

> "...Right. I'm really sorry."
> The person quickly walks away, leaving you alone.

##### Option C: Angry Response
**Personality Trait:** Hot-tempered
**Dialogue:** "Are you kidding me?! Look at this mess!"

**Inner Thought:**
> Your relaxing afternoon is ruined. This is unacceptable.

**NPC Reaction:**
- Expression: Frightened, backing away
- Sprite: alex_scared

> "I-I'm so sorry! I'll pay for dry cleaning, I promise!"
> The person looks terrified and fumbles for their wallet.

#### Result

##### If Option A (Kind Response)
**Relationship Changes:**
- Alex affection +5
- Unlock: "Cafe Date" event (requires affection >= 40)

**Stat Changes:**
- Charm +2 (showed warmth and grace)
- Cleanliness -5 (coffee stain on clothes)

**Story Flags:**
- Set: met_alex
- Set: alex_first_impression_good
- Unlock Location: alex_apartment (when affection >= 20)

**Future Impact:**
> Alex becomes your first friend in the city.
> You'll often run into each other at the cafe, and Alex
> will remember your kindness. This opens up a friendship
> route that could develop into something more.

##### If Option B (Cold Response)
**Relationship Changes:**
- Alex affection -1
- Delayed progression (need to repair relationship)

**Stat Changes:**
- Corruption +1 (showed coldness)
- Cleanliness -5 (coffee stain)

**Story Flags:**
- Set: met_alex
- Set: alex_first_impression_neutral

**Future Impact:**
> Your first impression wasn't great, but it's not irreparable.
> Alex will be more cautious around you. You'll need to make
> an effort to improve the relationship through other encounters.

##### If Option C (Angry Response)
**Relationship Changes:**
- Alex affection -5
- Locked: Friendship route (can only pursue conflict/domination path)

**Stat Changes:**
- Corruption +3 (showed aggression)
- Intelligence -1 (lost emotional control)
- Cleanliness -5 (coffee stain)

**Story Flags:**
- Set: met_alex
- Set: alex_first_impression_bad
- Trigger: alex_avoids_player (Alex will actively avoid you)

**Future Impact:**
> Alex is now intimidated by you and will try to stay away.
> The only way forward is through manipulation or domination.
> Repairing this relationship naturally will be extremely difficult.

---

### Event: Daily Interaction (Fallback)

**Event Type:** Minor Branch
**Time Consumption:** None
**Can Repeat:** Yes
**Force Leave Scene:** No

#### Dialogue Variants

**First Time (Interaction #1):**
> You sit by the window, watching people pass by outside.
> It's peaceful here.

**Result:** Stamina +5

---

**Second-Third Time (Interaction #2-3):**
> You return to your favorite spot by the window.
> The familiar view is comforting.

**Result:** Stamina +5

---

**Fourth Time Onward (Interaction #4+):**
> You find yourself back at the window seat again.
> Maybe you should try exploring other parts of the cafe?

**Result:** Stamina +3 (diminishing returns, hint to explore more)

---

## Interaction Point: Counter

### Event: Buy Coffee

**Event Type:** Minor Branch
**Time Consumption:** None
**Can Repeat:** Yes
**Force Leave Scene:** No

#### Dialogue Variants

**First Time (Interaction #1):**

#### Situation
- **Location:** Cafe - Counter
- **Prerequisites:** None

> Behind the counter stands a friendly-looking barista.

#### Action
> You order a latte.

#### Result
- Money -100
- Item: Coffee +1
- Stamina +10

---

**Second-Fourth Time (Interaction #2-4):**
> "Back again? Same as before?"
> You nod and order your usual latte.

**Result:**
- Money -100
- Item: Coffee +1
- Stamina +10

---

**Fifth Time Onward (Interaction #5+):**
> "Regular customer! Here, have a discount."
> The barista smiles and charges you less.

**Result:**
- Money -80 (discount applied!)
- Item: Coffee +1
- Stamina +10
- Set Flag: regular_customer
- Unlock Hint: "By the way, people sometimes throw out good stuff in the trash can out back..." → Unlocks trash_can interaction point
```

---

## Writing Guidelines

### 1. Natural Language for Conditions

**❌ Don't Write:**
```markdown
Situation:
- player.stats.charm >= 30 AND player.stats.stamina >= 50
```

**✅ Do Write:**
```markdown
Situation:
- Character State: Feeling confident and energetic
```

**Mapping (Engineers Define):**
```yaml
# narrative_rules.yaml
narrative_states:
  "feeling_confident_and_energetic":
    conditions:
      - stats.charm >= 30
      - stats.stamina >= 50
```

---

### 2. Describe Effects, Not Code

**❌ Don't Write:**
```markdown
Result:
- relationships['alex'].affection += 5
- stats.charm += 2
```

**✅ Do Write:**
```markdown
Result:
- Alex likes you more
- You feel more confident after the positive interaction
```

**Mapping (Engineers Define):**
```yaml
effects:
  "alex_likes_you_more":
    type: modify_relationship
    character: alex
    delta: 5
  "feel_more_confident":
    type: modify_stat
    stat: charm
    delta: 2
```

---

### 3. Show Character Reactions

**❌ Flat Dialogue:**
```markdown
Alex: "Thank you."
```

**✅ Rich Description:**
```markdown
NPC Reaction:
- Expression: Relieved, grateful
- Sprite: alex_smile

> "Thank you for being so understanding..."
> Alex's voice is warm, and a genuine smile
> lights up their face.
```

---

### 4. Provide Player Inner Thoughts

**Why?** Helps players understand character personality.

```markdown
**Inner Thought:**
> You feel a pang of sympathy. Everyone has bad days,
> and taking it out on this person won't help anything.
```

This guides players to understand the "kind" choice aligns with empathy.

---

### 5. Explain Long-Term Consequences

**❌ Vague:**
```markdown
Future Impact:
> Things changed.
```

**✅ Clear:**
```markdown
Future Impact:
> Alex becomes your first friend in the city. You'll
> often run into each other at the cafe. This opens
> the friendship route, which could develop into
> romance if you continue to be kind and supportive.
```

---

## Narrative State Definitions

### Common Narrative States

Engineers maintain this file, writers reference it.

```yaml
# config/narrative_rules.yaml

narrative_states:
  # Energy levels
  "feeling_energetic":
    conditions:
      - stats.stamina >= 60
    description: "Player has good energy"
  
  "exhausted":
    conditions:
      - stats.stamina < 30
    description: "Player is tired"
  
  # Appearance
  "looking_good":
    conditions:
      - stats.charm >= 50
      - stats.cleanliness >= 60
    description: "Well-groomed and attractive"
  
  "unkempt":
    conditions:
      - stats.cleanliness < 40
    description: "Appearance neglected"
  
  # Personality states
  "feeling_confident":
    conditions:
      - stats.charm >= 40
    description: "Self-assured"
  
  "feeling_sharp":
    conditions:
      - stats.intelligence >= 50
    description: "Mentally alert"
  
  "morally_conflicted":
    conditions:
      - stats.corruption >= 30
      - stats.corruption <= 60
    description: "Between pure and corrupt"
  
  # Relationship contexts
  "close_with_alex":
    conditions:
      - relationships.alex.affection >= 60
    description: "Good relationship with Alex"
  
  "alex_is_dominated":
    conditions:
      - relationships.alex.status == dominated
    description: "Alex is under player's control"
```

---

## Discovery Hints

### How to Write Hints

**In NPC Dialogue:**
```markdown
### Event: Casual Conversation with Alex

> "I usually come here in the afternoons. I like to sit
> by the window and read. The light is perfect there."

**Hidden Effect:**
- Unlock Hint: window_seat
  - Location: cafe
  - Point: window_seat
  - Unlock Immediately: Yes
```

**In Item Descriptions:**
```markdown
### Item: Crumpled Note

Description:
> A note you found on the ground. It reads:
> "Meet me behind the cafe at midnight. -J"

**Hidden Effect:**
- Unlock Hint: alley_entrance
  - Location: cafe
  - Point: back_alley
  - Unlock Immediately: No (player must go there at night)
```

---

## File Organization

### Recommended Structure

```
scenarios/
├── locations/
│   ├── cafe/
│   │   ├── _config.md              (Scene setup)
│   │   ├── counter_events.md       (All counter events)
│   │   ├── window_seat_events.md
│   │   ├── bathroom_events.md
│   │   └── trash_can_events.md
│   ├── home/
│   ├── office/
│   └── park/
├── characters/
│   ├── alex/
│   │   ├── _profile.md             (Character bio)
│   │   ├── first_meeting.md        (Already in cafe/)
│   │   ├── date_events.md
│   │   └── romance_route.md
│   └── bob/
└── special_events/
    ├── festivals.md
    └── random_encounters.md
```

---

## Validation Checklist

Before submitting a scenario, check:

- [ ] **STAR Structure Complete**
  - Situation clearly set
  - Task/conflict described
  - At least 2 action options
  - Results for each option defined

- [ ] **Event Type Specified**
  - Major Story or Minor Branch
  - Time consumption noted
  - Force leave behavior set

- [ ] **Consequences Clear**
  - Relationship changes specified
  - Stat changes with reasons
  - Flags set/removed listed
  - Future impact described

- [ ] **Dialogue Variants (if repeatable)**
  - At least 3 variants (1st, 2nd-4th, 5th+)
  - Diminishing returns if applicable
  - Hints for player to try other content

- [ ] **No Technical Jargon**
  - Use natural language for conditions
  - Describe effects narratively
  - No code syntax

- [ ] **Character Consistency**
  - Personality matches character profile
  - Reactions feel authentic
  - Sprite changes make sense

---

## Linter Output Example

When you run the validation tool:

```bash
$ dart run scenario_validator scenarios/cafe/

✓ scenarios/cafe/counter_events.md
  - 2 events found
  - All STAR sections present
  - Dialogue variants: 1st, 2nd-4th, 5th+ ✓
  
✗ scenarios/cafe/window_seat_events.md
  Line 45: Warning - "feeling_super_confident" not defined in narrative_rules.yaml
  Line 78: Error - Result section missing for Option C
  Line 92: Warning - Consider adding inner thought for Option B
  
✓ scenarios/cafe/bathroom_events.md
  - 3 events found
  - All conditions valid

Summary:
  Files checked: 3
  Passed: 2
  Warnings: 3
  Errors: 1
```

---

## Writer Workflow

### Step 1: Plan the Event

Before writing, answer:
- What is the core dramatic moment?
- What choices make sense here?
- What are the meaningful consequences?
- Does this advance character/plot?

### Step 2: Use the Template

Copy the STAR template and fill it in:
```bash
cp templates/star_event_template.md scenarios/cafe/my_new_event.md
```

### Step 3: Write Naturally

Focus on storytelling, not technical details:
- Describe scenes vividly
- Give characters personality
- Make choices feel meaningful
- Show consequences clearly

### Step 4: Validate

Run the linter:
```bash
dart run scenario_validator scenarios/cafe/my_new_event.md
```

Fix any errors or warnings.

### Step 5: Test

If possible, play through the event:
- Does dialogue flow naturally?
- Are choices interesting?
- Do consequences feel fair?
- Is pacing good?

### Step 6: Submit

Commit to version control:
```bash
git add scenarios/cafe/my_new_event.md
git commit -m "feat: Add new cafe event - accidental meeting"
git push
```

---

## Collaboration with Engineers

### When to Ask Engineers

**You need an engineer when:**
- Adding a new narrative state ("feeling_betrayed")
- Requesting a new game mechanic (new stat, new system)
- Unsure how to express a complex condition
- Need a new item or location

**You don't need an engineer when:**
- Writing new dialogue variants
- Creating new events with existing conditions
- Adjusting stat change values (within reason)
- Rewriting narrative text

### How to Request New Features

**❌ Vague Request:**
> "I need Alex to react differently in some situations"

**✅ Clear Request:**
> "I need a new narrative state: 'alex_is_jealous'
> 
> Condition: Player is dating both Alex and Bob
> 
> Usage: In future events, Alex's dialogue should be
> colder when this state is true. Can we add this to
> narrative_rules.yaml?"

---

## Advanced Techniques

### Conditional Branching in Events

Sometimes an event's outcome depends on past choices:

```markdown
### Event: Alex Confrontation

#### Situation
- Alex found out you've been seeing Bob

#### Task
> Alex confronts you at the cafe, looking hurt and angry.

#### Action

##### Option A: Honest Confession
> "You're right, I've been seeing Bob too. I'm sorry."

**Result Depends on Past:**
- IF alex_first_impression_good:
  → Alex is hurt but willing to talk
  → Unlock: poly_route_negotiation
  
- IF alex_first_impression_neutral:
  → Alex walks away angry
  → affection -20
  
- IF player has leverage items >= 3:
  → Alex reluctantly accepts
  → Status becomes Dominated
```

**Note for Engineers:** This requires event logic to check multiple flags.

### Multi-Event Arcs

Plan event sequences:

```markdown
## Alex Romance Arc

### Chapter 1: Meeting (Day 3-7)
1. First Meeting (cafe - window seat)
2. Casual Encounter (office - if same workplace)
3. Exchange Numbers (cafe - counter)

### Chapter 2: Friendship (Day 8-15)
4. Coffee Chat (cafe - window seat, affection >= 20)
5. Lunch Together (office - cafeteria)
6. Learn About Past (park - bench, affection >= 40)

### Chapter 3: Romance (Day 16-25)
7. First Date (restaurant - unlocked location)
8. Confession (park - evening, affection >= 60)
9. First Kiss (alex_apartment, affection >= 80)

### Chapter 4: Commitment (Day 26+)
10. Meet Friends (cafe - group event)
11. Serious Talk (alex_apartment, affection >= 90)
12. Proposal (special_location, player initiated)
```

---

## Tips for Great Narrative

### 1. Show, Don't Tell

**❌ Telling:**
> Alex is sad.

**✅ Showing:**
> Alex's smile fades. They look down at their coffee,
> stirring it absently without saying a word.

### 2. Varied Sentence Structure

Mix short and long sentences for rhythm:
> You freeze. The envelope is addressed to Alex. Should you open it? No - that would be wrong. But curiosity gnaws at you.

### 3. Sensory Details

Engage multiple senses:
> The cafe smells of fresh-ground coffee and cinnamon.
> Soft jazz plays in the background. Sunlight warms your face.

### 4. Character Voice

Each character should sound distinct:
- Alex: Thoughtful, articulate, uses longer sentences
- Bob: Casual, slang, shorter sentences with humor
- Charlie: Mysterious, cryptic, leaves things unsaid

### 5. Meaningful Choices

Avoid "fake" choices:

**❌ Fake Choice:**
- Option A: "Yes" → Same result
- Option B: "Okay" → Same result

**✅ Real Choice:**
- Option A: "I'll help" → +relationship, -time, unlock side quest
- Option B: "I can't" → -relationship, +time, different story path

---

## Common Mistakes

### Mistake 1: Over-Explaining

**❌ Too Much:**
> You feel 50% more charming after this interaction,
> which will help you in future encounters where
> charisma is tested above the threshold of 60.

**✅ Just Right:**
> You feel more confident after the positive exchange.

### Mistake 2: Breaking Character

**❌ Inconsistent:**
> [Alex is established as gentle and thoughtful]
> Alex: "Whatever, I don't care." [Out of character]

**✅ Consistent:**
> Alex: "I... I understand. Take your time." [In character]

### Mistake 3: Unclear Consequences

**❌ Vague:**
> Things will be different now.

**✅ Clear:**
> Alex will remember this. Future conversations will
> be more intimate, and new events will unlock.

### Mistake 4: Dialogue Variant Confusion

**❌ Wrong:**
```markdown
Interaction #5:
> "Hello again!" [Treats like second meeting]
```

**✅ Right:**
```markdown
Interaction #5:
> "You're here all the time now! Regular customer, huh?"
[Acknowledges many visits]
```

---

## Resources for Writers

### Templates
- `templates/star_event_template.md`
- `templates/location_config_template.md`
- `templates/character_profile_template.md`

### Reference
- `config/narrative_rules.yaml` (available conditions)
- `docs/character_profiles/` (character bios)
- `docs/world_lore/` (setting information)

### Examples
- `scenarios/examples/simple_event.md`
- `scenarios/examples/branching_event.md`
- `scenarios/examples/multi_part_event.md`

---

**Last Updated**: 2025-01-16  
**Version**: 1.0