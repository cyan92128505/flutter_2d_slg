# Life Simulation Romance Game - Development Documentation

## Project Vision

A long-term 2D life simulation + romance game built with Flutter, designed with **Clean Architecture + DDD** principles.

### Core Philosophy

- **Separation of Concerns**: Game engine (code) vs. game content (narrative data)
- **Domain-Driven Design**: Business logic independent of infrastructure
- **Narrative-First**: Enable non-technical writers to create scenarios using STAR format
- **No Big Ball of Mud**: Strict architectural boundaries prevent technical debt

---

## Quick Start

### Project Status
- **Current Phase**: Phase 0 - Architecture Design ✅ Complete
- **Next Phase**: Phase 1 - Domain Layer Implementation

### Documentation Structure

```
docs/
├── 01_game_mechanics.md          # 完整遊戲機制定義
├── 02_domain_model.md            # Domain Layer 設計
├── 03_application_layer.md       # Use Cases 定義
├── 04_infrastructure_layer.md    # 持久化與資料載入
├── 05_narrative_system.md        # STAR 格式與劇本規範
├── 06_development_roadmap.md     # 開發路線圖
└── 07_collaboration_workflow.md  # 團隊協作流程
```

---

## Core Concepts

### Game Loop
```
玩家起床 → 選擇場景 → 探索互動點 → 觸發事件 
→ 做出選擇 → 影響數值/關係 → 時間推進 → 循環
```

### Key Features
- **Interactive Exploration**: Click objects in scenes to discover events
- **Relationship Building**: Multiple romance routes with jealousy/domination mechanics
- **Stat Management**: 5 attributes affecting gameplay (stamina, charm, corruption, intelligence, cleanliness)
- **Dynamic Dialogue**: Conversations change based on interaction count
- **Multiple Endings**: Non-exclusive endings for each character

---

## Technology Stack

### Core
- **Framework**: Flutter 3.x
- **Language**: Dart 3.x
- **State Management**: Riverpod 2.x

### Architecture
- **Pattern**: Clean Architecture + DDD
- **Testing**: >80% domain layer coverage target

### Persistence
- **Local Storage**: shared_preferences
- **Format**: JSON with versioning

---

## Development Phases

| Phase | Duration | Deliverable | Status |
|-------|----------|-------------|--------|
| 0. Architecture | - | Complete design docs | ✅ Done |
| 1. Domain Layer | Week 1-2 | Core business logic | 📋 Next |
| 2. Application Layer | Week 3-4 | Use cases | ⏳ Planned |
| 3. Infrastructure | Week 5-6 | Save system | ⏳ Planned |
| 4. Presentation MVP | Week 7-8 | Playable demo | ⏳ Planned |
| 5. Vertical Slice | Week 9-10 | 1 character route | ⏳ Planned |
| 6. Tooling | Week 11-12 | Validation tools | ⏳ Planned |
| 7. Expansion | Week 13-16 | 3 characters, 8 scenes | ⏳ Planned |
| 8. Polish | Week 17-18 | Alpha release | ⏳ Planned |

---

## Team Roles

### Engineer
- Define Domain Model
- Implement game engine
- Create mapping rules (narrative_rules.yaml)
- Maintain tooling

### Writer
- Create scenarios in STAR format
- Write character dialogues
- Design event branches
- No coding required

### PM
- Review narrative quality
- Define game balance
- Request new features
- Approve content

---

## Getting Started

### For Engineers
1. Read: `02_domain_model.md`
2. Setup: Run `flutter create life_sim_game`
3. Start: Implement `Stats` value object (TDD)

### For Writers
1. Read: `05_narrative_system.md`
2. See: Example scenarios in `/scenarios/examples/`
3. Write: Use STAR template

### For PMs
1. Read: `01_game_mechanics.md`
2. Review: `06_development_roadmap.md`
3. Plan: Content schedule

---

## Key Decisions Log

### Exploration System
- ✅ Click-based interaction (not random encounters)
- ✅ No stamina cost for exploration
- ✅ NPC hints unlock hidden interaction points
- ✅ Interaction count persisted in save files

### Time System
- ✅ 4 time slots per day
- ✅ Stamina cannot go negative (forced rest at 0)
- ✅ Minor events don't consume time
- ✅ Major events consume time and force scene exit

### Relationship System
- ✅ Multiple simultaneous romances allowed
- ✅ Domination mechanic via leverage items
- ✅ Corruption as single value (0-100)
- ✅ Characters have corruption range requirements

### Ending System
- ✅ Multiple non-exclusive endings
- ✅ Player triggers endings manually
- ✅ No game over condition
- ✅ Routes lock individually after ending

---

## Contact & Resources

- **Documentation**: See `/docs` folder
- **Examples**: See `/scenarios/examples` folder
- **Architecture Diagrams**: See `02_domain_model.md`

---

**Last Updated**: 2025-01-16  
**Version**: 1.0 (Initial Architecture)