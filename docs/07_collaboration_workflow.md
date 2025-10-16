# Collaboration Workflow - Team Processes

## Overview

This document defines how engineers, writers, and PMs collaborate effectively on this project.

---

## Team Roles & Responsibilities

### Engineer

**Primary Responsibilities:**
- Implement game engine (Domain, Application, Infrastructure layers)
- Define domain model and business rules
- Create and maintain narrative_rules.yaml
- Build validation tools (linter, test generator)
- Review and merge code changes
- Optimize performance
- Fix bugs

**Deliverables:**
- Working game engine
- Unit and integration tests
- Technical documentation
- Developer tools

---

### Writer

**Primary Responsibilities:**
- Create character profiles and backstories
- Write scenarios in STAR format
- Design dialogue variants for repeatable events
- Ensure narrative consistency
- Balance relationship progression
- Create emotional impact through storytelling

**Deliverables:**
- Character profiles (.md files)
- Scenario files (.md files in STAR format)
- Dialogue scripts
- Event dependency documentation

**No Coding Required!** Writers work in plain Markdown using templates.

---

### PM (Product Manager / Game Designer)

**Primary Responsibilities:**
- Define game mechanics and balance
- Plan content roadmap
- Review and approve scenarios
- Playtest and provide feedback
- Prioritize features
- Coordinate between engineers and writers

**Deliverables:**
- Game design documents
- Content schedule
- Playtesting reports
- Feature prioritization

---

## Git Workflow

### Branch Strategy

```
main (protected)
  ↑
  ├── develop (integration branch)
  │     ↑
  │     ├── feature/domain-layer
  │     ├── feature/save-system
  │     ├── content/alex-route
  │     └── content/cafe-events
```

**Branch Types:**
- `main`: Production-ready code, always stable
- `develop`: Integration branch for ongoing work
- `feature/*`: New game engine features
- `content/*`: New scenarios and narrative content
- `bugfix/*`: Bug fixes
- `hotfix/*`: Critical production fixes

---

### Commit Message Convention

**Format:** `type(scope): subject`

**Types:**
- `feat`: New feature
- `fix`: Bug fix
- `docs`: Documentation only
- `style`: Code style (formatting, no logic change)
- `refactor`: Code restructuring (no behavior change)
- `test`: Adding or updating tests
- `chore`: Build process, dependencies

**Examples:**
```bash
feat(domain): implement Player aggregate root
fix(save): handle corrupted save files gracefully
content(cafe): add Alex first meeting event
docs(readme): update installation instructions
test(domain): add Stats value object tests
refactor(use-case): simplify InteractWithPointUseCase
```

**English Only:** All commit messages, code comments, and documentation in English.

**No Emojis:** Keep commits professional and searchable.

---

### Pull Request Process

#### For Engineers

**1. Create Feature Branch**
```bash
git checkout develop
git pull origin develop
git checkout -b feature/interaction-system
```

**2. Develop with Tests**
```bash
# Write tests first (TDD)
# Implement feature
# Ensure tests pass
flutter test
```

**3. Commit Regularly**
```bash
git add lib/domain/entities/interaction_point.dart
git commit -m "feat(domain): add InteractionPoint entity"

git add test/domain/entities/interaction_point_test.dart
git commit -m "test(domain): add InteractionPoint tests"
```

**4. Push and Create PR**
```bash
git push origin feature/interaction-system
# Create PR on GitHub/GitLab
```

**5. PR Template**
```markdown
## Description
Implements the InteractionPoint entity for the exploration system.

## Changes
- Added InteractionPoint entity with state management
- Implemented dialogue variant selection logic
- Added comprehensive unit tests (95% coverage)

## Testing
- [x] Unit tests pass
- [x] Integration tests pass
- [x] Manual testing completed

## Checklist
- [x] Code follows style guide
- [x] Tests added/updated
- [x] Documentation updated
- [x] No breaking changes
```

#### For Writers

**1. Create Content Branch**
```bash
git checkout develop
git pull origin develop
git checkout -b content/alex-cafe-events
```

**2. Write Scenarios**
```bash
# Create/edit .md files in scenarios/
# Use STAR template
# Run validator
dart run scenario_validator scenarios/cafe/
```

**3. Commit Scenarios**
```bash
git add scenarios/cafe/meet_alex.md
git commit -m "content(cafe): add Alex first meeting event"

git add scenarios/cafe/alex_date.md
git commit -m "content(cafe): add Alex cafe date event"
```

**4. Create PR**
```markdown
## Description
Adds first meeting and date events for Alex at the cafe.

## Content Summary
- First meeting: 3 dialogue options (kind/cold/angry)
- Cafe date: Repeatable event for relationship building

## Checklist
- [x] Follows STAR format
- [x] Validator passes with no errors
- [x] Character consistent with profile
- [x] Consequences clear and balanced
```

---

## Code Review Guidelines

### For Reviewers

**Check:**
- [ ] Code follows architecture (Clean Architecture + DDD)
- [ ] Tests present and passing
- [ ] No business logic in wrong layers
- [ ] Documentation updated
- [ ] Performance acceptable
- [ ] No breaking changes (or properly documented)

**Focus Areas:**
- **Domain Layer**: Pure logic, no dependencies
- **Application Layer**: No business logic, just coordination
- **Infrastructure**: Proper error handling
- **Tests**: >80% coverage for domain

**Feedback Style:**
```markdown
# ✅ Good feedback
The InteractionPoint entity looks solid, but consider:
- Line 45: Should `getAvailableEvent` return Result<Event> instead of Event? for better error handling
- Line 78: Extract magic number 999 into a constant

# ❌ Poor feedback
This code is wrong. Fix it.
```

---

## Scenario Review Guidelines

### For PM/Lead Writer

**Check:**
- [ ] STAR structure complete
- [ ] Character personality consistent
- [ ] Consequences feel fair
- [ ] Dialogue quality good
- [ ] No plot holes
- [ ] Pacing appropriate

**Feedback Style:**
```markdown
# ✅ Good feedback
The Alex meeting event is great! A few suggestions:
- The "angry response" consequences feel too harsh (-5 affection). Consider -3?
- Add an inner thought for Option B to clarify the player's mindset
- The "Future Impact" section could be more specific about unlocked content

# ❌ Poor feedback
This doesn't make sense. Rewrite it.
```

---

## Communication Channels

### Synchronous (Real-time)

**Use Discord/Slack for:**
- Quick questions
- Clarifications
- Unblocking issues
- Daily standups (if team > 1)

**Channels:**
- `#general`: General discussion
- `#engineering`: Technical questions
- `#narrative`: Story and character discussion
- `#playtesting`: Bug reports and feedback

---

### Asynchronous

**Use GitHub/GitLab Issues for:**
- Feature requests
- Bug reports
- Content planning
- Design discussions

**Issue Templates:**

**Bug Report:**
```markdown
## Bug Description
Game crashes when loading save slot 5

## Steps to Reproduce
1. Start new game
2. Play to day 10
3. Save to slot 5
4. Quit game
5. Load slot 5

## Expected Behavior
Game loads successfully

## Actual Behavior
App crashes with error: "Failed to parse JSON"

## Environment
- Platform: Android
- Flutter version: 3.16.0
- Device: Pixel 6
```

**Feature Request:**
```markdown
## Feature Description
Add achievement system for tracking player milestones

## Use Case
Players want to see what they've accomplished and have goals to work towards

## Proposed Implementation
- Track achievements in player state
- Display achievement popups when unlocked
- Achievement gallery screen

## Priority
Low (post-Alpha feature)
```

---

## Content Creation Workflow

### Phase 1: Planning (PM + Writer)

**1. Create Content Brief**
```markdown
# Alex Romance Route - Content Brief

## Character Summary
- Name: Alex
- Personality: Thoughtful, gentle, bookworm
- Corruption Range: 0-40 (pure love route)
- Occupation: Librarian

## Story Arc
- Chapter 1 (Day 3-7): Meeting and first impressions
- Chapter 2 (Day 8-14): Building friendship
- Chapter 3 (Day 15-25): Romance development
- Chapter 4 (Day 26+): Commitment

## Key Events
1. First meeting at cafe (accidental coffee spill)
2. Library encounter
3. First intentional hangout
4. Deep conversation about past
5. First date
6. Confession
7. Relationship decision
```

**2. Get Engineer Approval**
- Verify all mechanics exist
- Confirm conditions are implementable
- Check for missing narrative states

---

### Phase 2: Writing (Writer)

**1. Use STAR Template**
```bash
cp templates/star_event_template.md scenarios/cafe/meet_alex.md
```

**2. Write Event**
- Fill in all STAR sections
- Write dialogue variants for repeatable events
- Describe consequences clearly
- Use natural language for conditions

**3. Self-Review Checklist**
- [ ] All STAR sections present
- [ ] At least 2 choices for player
- [ ] Consequences described for each choice
- [ ] Character voice consistent
- [ ] No technical jargon (use natural language)
- [ ] Inner thoughts add character depth

**4. Validate**
```bash
dart run scenario_validator scenarios/cafe/meet_alex.md
```

Fix any errors or warnings.

---

### Phase 3: Review (PM)

**PM Reviews For:**
- Narrative quality
- Character consistency
- Game balance (affection changes, stat changes)
- Player agency (choices feel meaningful)
- Pacing

**Provide Feedback:**
```markdown
## Feedback: meet_alex.md

### Strengths
- Opening scene is vivid and engaging
- Dialogue feels natural
- Consequences are clear

### Suggestions
- Line 45: The angry option feels TOO harsh. Maybe tone down to -3 affection?
- Line 67: Add an inner thought for the cold response to show why player chooses this
- Consider: Should this event unlock Alex's apartment immediately, or after another encounter?

### Approval Status
Approved with minor revisions
```

---

### Phase 4: Implementation (Engineer)

**Engineer Converts to YAML**
```bash
# Manual conversion (Phase 1-5)
# Or use compiler (Phase 9, optional)
```

**Add Mapping for New States**
```yaml
# narrative_rules.yaml

# Writer used: "feeling confident and well-rested"
feeling_confident_and_well_rested:
  conditions:
    - stats.charm >= 40
    - stats.stamina >= 60
  description: "Player has high charm and good energy"
```

**Create YAML Event File**
```yaml
# assets/data/events/cafe/meet_alex.yaml
# (See infrastructure_layer.md for format)
```

---

### Phase 5: Testing (PM + Writer)

**Playtest Checklist:**
- [ ] Event triggers at correct time/location
- [ ] All dialogue displays correctly
- [ ] Choices lead to correct outcomes
- [ ] Stats change as expected
- [ ] Flags set correctly
- [ ] Unlocks work (new locations/events)
- [ ] Repeatable events show variant dialogue

**Bug Report if Issues:**
```markdown
## Bug: Alex meeting event

**Issue:** Angry response doesn't decrease affection

**Expected:** alex_affection should be -5
**Actual:** alex_affection stays at 0

**Location:** scenarios/cafe/meet_alex.md, Line 89
```

---

## Conflict Resolution

### Scenario: Writer wants mechanic that doesn't exist

**Writer:** "I want an event where Alex gives the player a key item that permanently increases charm by 10."

**Engineer:** "Items currently don't have permanent stat effects. That would require new domain logic."

**Resolution Options:**

**Option A: Engineer Implements**
- Engineer adds "Equipment" item type to domain
- PM approves scope increase
- Timeline adjusts

**Option B: Writer Adjusts**
- Use temporary buff instead (charm +10 for 1 day)
- Or item provides one-time stat boost when used

**Option C: Defer**
- Add to "future features" backlog
- Implement in later phase

**Decision Process:**
1. PM evaluates priority
2. Engineer estimates effort
3. Team decides together

---

### Scenario: Engineer refactors, breaks content

**Situation:** Engineer refactors event system, old YAML format incompatible.

**Process:**
1. Engineer creates migration script
2. Engineer runs script on all content
3. Engineer tests all scenarios still work
4. Engineer notifies team of changes
5. Writer reviews migrated content
6. PM approves if no narrative changes

**Communication:**
```markdown
## Breaking Change: Event format updated

**What changed:**
Event YAML now requires `type` field (major_story | minor_branch)

**Action required:**
All existing events need `type` field added

**Migration:**
I've run a migration script on all scenarios/ files.
Please review your events to ensure they still work correctly.

**Timeline:**
- Today: Migration complete
- Tomorrow: Testing
- Day 3: Merge to develop
```

---

## Release Process

### Pre-Release Checklist

**Code:**
- [ ] All tests passing
- [ ] No critical bugs
- [ ] Performance targets met
- [ ] Documentation updated

**Content:**
- [ ] All scenarios validated
- [ ] Playtest complete (3+ testers)
- [ ] Typos fixed
- [ ] Character routes completable

**Build:**
- [ ] Version number incremented
- [ ] CHANGELOG.md updated
- [ ] Release notes written
- [ ] Builds created (Android/iOS/Web)

---

### Version Numbering

**Format:** `MAJOR.MINOR.PATCH`

**Examples:**
- `1.0.0`: First public release (Alpha)
- `1.1.0`: Added Bob character route
- `1.1.1`: Fixed save corruption bug
- `2.0.0`: Major engine refactor (breaking changes)

**When to increment:**
- **MAJOR**: Breaking changes, major features
- **MINOR**: New content, non-breaking features
- **PATCH**: Bug fixes, small improvements

---

### Release Notes Template

```markdown
# Version 1.1.0 - "New Beginnings"

## Release Date
2025-02-15

## New Content
- Added Bob character route (10 new events)
- New location: City Park (5 interaction points)
- 15+ new dialogue variants for existing events

## Features
- Implemented domination mechanic (leverage items)
- Added personality change events based on corruption
- New achievement system (20 achievements)

## Improvements
- Save/load performance improved by 50%
- UI polish: smoother animations
- Better hint system feedback

## Bug Fixes
- Fixed crash when loading saves from v1.0.0
- Fixed Alex relationship not progressing after day 20
- Fixed dialogue variants not changing after 10th interaction
- Fixed stat bars displaying incorrect values after rest

## Known Issues
- Web version: Audio may not play on first load (refresh fixes)
- Android: Minor UI clipping on some devices

## Breaking Changes
None - all v1.0.0 saves compatible

## Credits
- Writing: [Writer Names]
- Engineering: [Engineer Names]
- Playtesting: [Tester Names]

## Download
- Android: [Link]
- iOS: [Link]
- Web: [Link]
```

---

## Playtesting Process

### Recruiting Playtesters

**Criteria:**
- Interested in romance/simulation games
- Can provide detailed feedback
- Available for 2-3 hour session
- Mix of experienced and casual gamers

**Compensation:**
- Free copy of full game on release
- Credits in game
- Optional: Small payment ($20-50)

---

### Playtest Session Structure

**1. Pre-Session (15 min)**
- Brief introduction to game
- Explain feedback process
- No detailed instructions (test discoverability)

**2. Play Session (90-120 min)**
- Tester plays independently
- Observer takes notes (don't interrupt)
- Record screen if possible

**3. Post-Session Interview (30 min)**
- What did you enjoy?
- What was confusing?
- Any bugs encountered?
- Would you continue playing?

---

### Feedback Form Template

```markdown
## Playtester Feedback Form

**Tester ID:** PT-001
**Date:** 2025-01-16
**Build Version:** 1.0.0-alpha
**Play Duration:** 120 minutes

### Overall Experience (1-5)
- Fun Factor: 4/5
- Clarity: 3/5 (some mechanics unclear)
- Polish: 4/5
- Would Recommend: Yes

### What You Liked
- Character dialogue feels natural
- Stat management is engaging
- Discovery of hidden events was satisfying

### What Confused You
- Didn't understand corruption system at first
- Unclear how to unlock new locations
- Not sure if choices mattered early on

### Bugs Encountered
1. Game crashed when clicking save slot 3
2. Alex sprite didn't change during angry dialogue
3. Stats bar showed "NaN" after rest

### Suggestions
- Add tutorial for first playthrough
- Show corruption value more prominently
- Hint system could be more visible

### Additional Comments
Really enjoyed the writing! Alex feels like a real person.
Would love to see more character routes.
```

---

## Communication Best Practices

### Writing Good Issues

**❌ Bad Issue:**
```markdown
Title: Bug

Description: The game doesn't work
```

**✅ Good Issue:**
```markdown
Title: [Bug] Game crashes when loading save slot with interaction count > 100

Description:
## Problem
Game crashes with error "Integer overflow" when loading any save 
where an interaction point has been visited more than 100 times.

## Reproduction
1. Start new game
2. Visit cafe counter 101 times
3. Save game
4. Quit and reload

## Environment
- Version: 1.0.0-alpha
- Platform: Android 12
- Device: Samsung Galaxy S21

## Expected Behavior
Game loads normally regardless of interaction count

## Actual Behavior
App crashes with stack trace:
```
[Error trace here]
```

## Suggested Fix
Change interaction count from int32 to int64 in save format
```

---

### Asking Questions

**❌ Bad Question:**
```
How do I make events?
```

**✅ Good Question:**
```
I'm trying to create an event where the player finds a hidden item, 
but only if they've explored the cafe 5+ times. I read the STAR 
format docs, but I'm unsure how to express "visited location X times" 
as a condition.

Should I use a story flag that increments, or is there a better way?
```

---

### Giving Updates

**Weekly Update Template:**
```markdown
## Week of 2025-01-13 Progress Update

### Completed
- ✅ Implemented InteractionHistory persistence
- ✅ Created cafe location with 5 interaction points
- ✅ Wrote 3 events for Alex route

### In Progress
- 🔄 Testing save/load with interaction counts
- 🔄 Writing Bob character profile

### Blocked
- ⛔ Need narrative_rules.yaml update for "feeling_betrayed" state
  - Waiting on: Engineer to define conditions
  - Impact: Can't write Chapter 3 events for Alex

### Next Week Plan
- Complete Alex Chapter 1 events (2 remaining)
- Start Bob Chapter 1
- Playtest current content

### Questions/Concerns
- Should corruption affect Alex route progression? 
  Currently writing assuming it doesn't.
```

---

## Onboarding New Team Members

### Engineer Onboarding (Day 1-3)

**Day 1: Setup & Architecture**
- [ ] Clone repository
- [ ] Run `flutter doctor`
- [ ] Setup IDE (VS Code/Android Studio)
- [ ] Read README.md
- [ ] Read 02_domain_model.md
- [ ] Run existing tests (`flutter test`)

**Day 2: Codebase Tour**
- [ ] Walk through domain layer with senior engineer
- [ ] Understand use cases
- [ ] Review one PR to learn code style
- [ ] Make first small change (fix typo, add test)

**Day 3: First Task**
- [ ] Pick up "good first issue"
- [ ] Implement with TDD
- [ ] Submit PR
- [ ] Learn PR review process

---

### Writer Onboarding (Day 1-2)

**Day 1: Understanding the Game**
- [ ] Play current build (if available)
- [ ] Read 01_game_mechanics.md
- [ ] Read 05_narrative_system.md
- [ ] Review existing character profiles

**Day 2: First Scenario**
- [ ] Choose a simple event (e.g., "buy coffee")
- [ ] Use STAR template
- [ ] Write event
- [ ] Run validator
- [ ] Submit PR with guidance

---

## Tools & Resources

### For Engineers

**Must Have:**
- Flutter SDK
- Git
- IDE with Dart support
- shared_preferences package
- yaml package

**Recommended:**
- Riverpod DevTools
- Flutter DevTools (profiling)
- mockito (testing)

**Documentation:**
- Flutter docs: https://flutter.dev/docs
- Riverpod: https://riverpod.dev
- Clean Architecture: https://blog.cleancoder.com

---

### For Writers

**Must Have:**
- Markdown editor (VS Code, Typora, etc.)
- Git basics (clone, commit, push)
- Access to STAR templates

**Recommended:**
- Grammarly (grammar check)
- Character profile templates
- Story planning tools (Miro, Notion)

**Resources:**
- STAR format guide: `05_narrative_system.md`
- Character profiles: `scenarios/characters/`
- Examples: `scenarios/examples/`

---

### For Everyone

**Project Tools:**
- Git repository (GitHub/GitLab)
- Issue tracker
- Discord/Slack (communication)
- Google Docs/Notion (planning)

---

## Conflict & Disagreement Resolution

### Decision-Making Framework

**Level 1: Individual Decisions**
- Code style within accepted patterns
- Variable naming
- Test structure
- Dialogue phrasing (within character voice)

**Level 2: Team Discussion**
- New narrative states
- Character personality details
- Event balance (stat changes)
- UI/UX improvements

**Level 3: PM Decision**
- Feature prioritization
- Scope changes
- Release timing
- Breaking changes

**Level 4: Team Vote**
- Major architectural changes
- Large scope additions
- Controversial features

---

### Handling Disagreements

**Process:**
1. **State positions clearly**
   - Each person explains their view
   - Focus on reasoning, not emotion

2. **Identify common ground**
   - What do we agree on?
   - What's the real point of disagreement?

3. **Explore options**
   - Are there alternatives?
   - Can we compromise?
   - What are trade-offs?

4. **Time-box discussion**
   - 30 minutes max
   - If no consensus, escalate

5. **Escalate if needed**
   - PM makes final call
   - Document decision and reasoning
   - Team commits to decision

**Example:**
```markdown
## Disagreement: Should dialogue history be unlimited?

**Engineer's View:**
Unlimited history could cause performance issues with 1000+ messages.
Propose: Limit to last 100 messages.

**Writer's View:**
Players should be able to review full story. Limiting breaks immersion.
Propose: Keep all history.

**Discussion:**
- Common ground: Both want good player experience
- Key issue: Performance vs features
- Options:
  A) Limit to 100 (engineer's proposal)
  B) Unlimited with pagination (compromise)
  C) Unlimited with compression (compromise)

**Decision (PM):**
Go with Option B - unlimited history with pagination.
- Addresses performance concern (only load visible messages)
- Maintains full story access
- Engineer estimates 2 days implementation

**Result:**
Team commits. Engineer implements, writer tests.
```

---

## Metrics & KPIs

### Engineering Metrics

**Code Quality:**
- Test coverage: >80% (domain layer)
- Build success rate: >95%
- PR review time: <24 hours
- Bug escape rate: <5 per release

**Performance:**
- Save/load: <500ms
- Event loading: <200ms
- FPS: >30 stable
- Crash rate: <0.1%

---

### Content Metrics

**Volume:**
- Events per character: 10+ main story
- Dialogue variants: 3+ per repeatable event
- Locations: Target 8-10 for Alpha

**Quality:**
- Validator pass rate: >95%
- Playtest satisfaction: >4/5
- Typo rate: <1 per 1000 words

---

### Project Metrics

**Velocity:**
- Features completed per week
- Content created per week
- Issues closed per week

**Quality:**
- Bug resolution time: <7 days (non-critical)
- PR merge time: <2 days
- Playtest feedback incorporation: <2 weeks

---

## Emergency Procedures

### Critical Bug in Production

**Process:**
1. **Identify** (user report or monitoring)
2. **Confirm** (reproduce bug)
3. **Assess impact** (how many users affected?)
4. **Create hotfix branch** from main
5. **Fix quickly** (skip normal PR process if needed)
6. **Test** (critical path only)
7. **Deploy hotfix**
8. **Post-mortem** (how did this slip through?)

**Communication:**
```markdown
## Critical Bug Alert

**Issue:** Game crashes on launch for Android users

**Impact:** ~50% of Android users cannot play

**Status:** Hotfix in progress

**ETA:** Fix deployed within 2 hours

**Workaround:** Use web version temporarily

**Root Cause:** [To be determined after fix]
```

---

### Data Loss / Save Corruption

**Prevention:**
- Always backup before overwrite
- Version all save formats
- Test migration thoroughly

**Recovery:**
- Attempt to restore from backup
- If impossible, clearly communicate to user
- Provide "sorry" bonus (in-game items/currency)

---

### Contributor Unavailability

**Single Engineer:**
- Document all in-progress work
- Ensure code in main/develop is stable
- Provide access credentials to PM

**Single Writer:**
- Leave notes on unfinished scenarios
- Share character profiles and outlines
- Brief PM on story direction

**Mitigation:**
- Cross-train when possible
- Documentation is critical
- Regular backups

---

## Celebration & Recognition

### Milestones

**Celebrate:**
- ✨ First successful test
- ✨ First complete scenario
- ✨ First playable demo
- ✨ Phase completions
- ✨ Alpha release
- ✨ Positive player feedback

**How:**
- Team shout-out in Discord/Slack
- Credits in release notes
- Optional: Team lunch/gathering (if local)

---

### Credits

**In-Game Credits Structure:**
```
Life Simulation Game

Created by [Studio Name]

--- Engineering ---
Lead Engineer: [Name]
Additional Engineering: [Names]

--- Writing ---
Lead Writer: [Name]
Additional Writing: [Names]

--- Design ---
Game Design: [Name]

--- Special Thanks ---
Playtesters: [Names]
Community Feedback: [Names]

Built with Flutter
Music: [Sources]
Fonts: [Sources]

© 2025 [Studio Name]
```

---

## Appendix: Templates

### Quick Access

All templates available in:
```
templates/
├── star_event_template.md
├── character_profile_template.md
├── location_config_template.md
├── pr_template.md
├── bug_report_template.md
└── feature_request_template.md
```

---

### STAR Event Template

```markdown
# Event: [Event Name]

**Event Type:** Major Story / Minor Branch
**Time Consumption:** None / 1 Time Slot
**Can Repeat:** Yes / No / Daily
**Force Leave Scene:** Yes / No

## Situation (Context)
- **Time:** [Day X, Time Slot]
- **Location:** [Scene - Point]
- **Prerequisites:** [What must have happened]
- **Character State:** [Natural language description]

> [Opening narrative]

## Task (Trigger)
> [What happens to start this event]

**NPC Introduction:** [If applicable]
- Name: [Character]
- Expression: [Emotion]
- Sprite: [sprite_file]

> [NPC dialogue or event description]

## Action (Player Choices)

### Option A: [Choice Name]
**Personality Trait:** [kind/assertive/cold/etc.]
**Dialogue:** "[What player says]"

**Inner Thought:**
> [Player's internal monologue]

**NPC Reaction:**
- Expression: [Changed emotion]
- Sprite: [new_sprite]

> [NPC's response]

### Option B: [Alternative Choice]
[Same structure]

## Result (Consequences)

### If Option A:
**Relationship Changes:**
- [Character] affection +X / -X

**Stat Changes:**
- [Stat]: +X / -X (reason)

**Story Flags:**
- Set: [flag_name]
- Remove: [flag_name]
- Unlock: [location/point/event]

**Future Impact:**
> [Long-term narrative consequences]

### If Option B:
[Same structure]
```

---

## Final Notes

### Remember

- **Communication is key** - Over-communicate rather than under
- **Document everything** - Future you will thank present you
- **Ask questions** - No question is stupid
- **Respect expertise** - Engineers know code, writers know story
- **Stay organized** - Use tools, follow processes
- **Have fun!** - We're making a game people will enjoy

---

**Last Updated**: 2025-01-16  
**Version**: 1.0