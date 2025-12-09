# Character Sheet Redesign Proposal

**Date:** 2025-12-08
**Status:** Proposal
**Related Issues:** New epic to be created

---

## Executive Summary

A comprehensive redesign of the character view page (`/characters/[publicId]`) to transform it from a static display into a **living character sheet** optimized for actual D&D gameplay sessions.

### Design Philosophy

> "A character sheet should answer your most urgent question instantly, and your second-most-urgent question within one click."

**Current state:** Clean, organized display of character data
**Target state:** Interactive gameplay companion that adapts to your current context (combat vs exploration)

---

## The DM's Perspective: What Players Need at the Table

### During Combat (Time-Critical)

When the DM says "roll initiative" or "the goblin attacks you," players need:

1. **HP Tracking** - Current/max/temp, with easy adjustment
2. **Death Saves** - Visual tracker when at 0 HP
3. **AC** - Instant answer to "does a 17 hit?"
4. **Spell Slots** - "Do I have any 3rd-level slots left?"
5. **Hit Dice** - For short rests
6. **Conditions** - "Am I still poisoned?"
7. **Initiative** - Where am I in the order?
8. **Attack Options** - What can I do on my turn?

### During Exploration/Social

1. **Passive Scores** - Perception, Investigation, Insight (all three!)
2. **Skills** - Quick lookup for ability checks
3. **Languages** - "Can I read this inscription?"
4. **Features** - "Do I have darkvision?"
5. **Equipment** - "What do I have in my pack?"
6. **Notes** - "What was that NPC's name again?"

### Character Identity (Always Visible)

1. **Portrait** - Visual identity
2. **Name/Race/Class** - Quick reference
3. **Alignment** - Roleplay guide
4. **Level & XP** - Progression tracking

---

## Proposed Improvements

### Priority 1: Combat-Critical Features

#### 1.1 Death Saves Tracker
**Status:** Data available, not displayed
**Implementation:** Visual 3-circle tracker for successes/failures

```
Death Saves
○ ○ ○  Successes
○ ○ ○  Failures
```

When character HP ≤ 0, this becomes prominent. Interactive in future (click to toggle).

#### 1.2 Spell Slots Display
**Status:** Data available (`spell_slots` in stats), not displayed
**Implementation:** Visual slot tracker per level in Spells tab

```
┌─ Spell Slots ─────────────────────────────┐
│ 1st: ● ● ● ○  (3/4)                       │
│ 2nd: ● ● ○    (2/3)                       │
│ 3rd: ● ○ ○    (1/3)                       │
└───────────────────────────────────────────┘
```

**Warlock Pact Slots:** Separate display showing pact slot count and level.

#### 1.3 Hit Dice Counter
**Status:** Data available (`hit_dice` array with type/total/current), not displayed
**Implementation:** Add to combat stats grid or as collapsible section

```
Hit Dice: d10 ● ● ● ○ ○ (3/5)
```

For multiclass: Show each die type separately.

#### 1.4 Active Conditions Panel
**Status:** Data structure exists (`conditions` array), not displayed
**Implementation:** Prominent banner/badge area when conditions are active

```
⚠️ Active Conditions
┌─────────────────────────────────────────┐
│ 🟡 Poisoned - 2 hours remaining        │
│ 🔴 Frightened - Until end of next turn │
└─────────────────────────────────────────┘
```

Color-coded by severity. Links to condition descriptions.

### Priority 2: Information Display

#### 2.1 All Passive Scores
**Status:** Data available (perception, investigation, insight), only perception shown
**Implementation:** Expand Passive Perc stat to show all three

```
┌─ Passive Scores ──┐
│ Perception    14  │
│ Investigation 12  │
│ Insight       15  │
└───────────────────┘
```

#### 2.2 Character Portrait
**Status:** Data available (`portrait.original/thumb/medium`), not displayed
**Implementation:** Show in header or sidebar

Options:
- A) Header left side: 80px circular portrait next to name
- B) Sidebar top: Above ability scores
- C) Floating corner: Fixed position avatar

#### 2.3 Alignment Display
**Status:** Data available, not displayed
**Implementation:** Badge in header near race/class

```
Thorin Ironforge
Dwarf Fighter 5 • Lawful Good
```

#### 2.4 Experience Points & Level Progress
**Status:** Data available (`experience_points`), not displayed
**Implementation:** Progress bar showing XP toward next level

```
Level 5 ───────────●─────── Level 6
         6,500 / 14,000 XP
```

### Priority 3: Enhanced Functionality

#### 3.1 Notes Tab
**Status:** Backend endpoint exists (`/characters/{id}/notes`), never fetched
**Implementation:** New tab with categorized notes

Categories:
- Session Notes
- NPCs
- Locations
- Quests
- Custom

#### 3.2 Improved Spells Tab Organization
**Status:** Current flat list, missing key info
**Implementation:** Group by level with slot tracking

```
┌─ Spells ──────────────────────────────────────┐
│ [Spell Slots: see 1.2 above]                  │
│                                               │
│ ▼ Cantrips (4 known)                          │
│   Fire Bolt, Light, Mage Hand, Prestidigitation│
│                                               │
│ ▼ 1st Level (4/4 slots) - 6 prepared          │
│   ✓ Magic Missile    ✓ Shield                 │
│   ✓ Detect Magic     ✓ Mage Armor            │
│   ○ Sleep            ○ Charm Person           │
│                                               │
│ ▼ 2nd Level (2/3 slots) - 3 prepared          │
│   ✓ Misty Step       ✓ Hold Person           │
│   ○ Invisibility                              │
└───────────────────────────────────────────────┘

Legend: ✓ = prepared, ○ = known but not prepared
```

#### 3.3 Preparation Limit Display
**Status:** Data available (`preparation_limit`, `prepared_spell_count`), not displayed
**Implementation:** Show in Spells tab header

```
Prepared: 8/11 spells (INT + Wizard level)
```

#### 3.4 Carrying Capacity
**Status:** Data available (`carrying_capacity`, `push_drag_lift`), not displayed
**Implementation:** In Equipment tab footer

```
Carrying: ~45 lbs / 150 lbs capacity
Push/Drag/Lift: 300 lbs
```

### Priority 4: Future Enhancements

#### 4.1 Combat Mode Toggle
A "Combat Mode" view that reorganizes the sheet for initiative order:

- HP/AC/Initiative prominent
- Quick attack reference
- Condition tracking
- Death saves (when relevant)
- Spell slot tracking

#### 4.2 Quick Actions Panel
Common combat actions with precalculated bonuses:

```
Quick Actions
├─ Longsword: +7 to hit, 1d8+4 slashing
├─ Longbow: +5 to hit, 1d8+2 piercing
├─ Fire Bolt: +6 to hit, 2d10 fire (60 ft)
└─ Eldritch Blast: +7 to hit, 1d10+4 force (120 ft)
```

This requires parsing equipped weapons and attack cantrips.

#### 4.3 Resource Tracking
Track limited-use abilities (relates to GitHub Issue #256):

```
Resources (Long Rest)
├─ Second Wind: ○ (0/1)
├─ Action Surge: ● (1/1)
└─ Superiority Dice: ● ● ● ○ (3/4)
```

#### 4.4 Interactive HP/Slot Adjustment
Click to adjust HP, use spell slots, track hit dice. Would require PATCH endpoints.

---

## Proposed Layout

### Desktop (lg+)

```
┌─────────────────────────────────────────────────────────────────┐
│ [Portrait] THORIN IRONFORGE                    [Inspiration ★]  │
│            Dwarf Fighter 5 • Soldier • Lawful Good              │
│            XP: ████████░░ 6,500/14,000                          │
├─────────────────────────────────────────────────────────────────┤
│ ⚠️ Conditions: Poisoned (2 hr)                                  │
├────────────┬────────────────────────────────────────────────────┤
│            │  ┌─────┬─────┬─────┐                               │
│  ABILITIES │  │ HP  │ AC  │Init │                               │
│ ┌────────┐ │  │45/52│ 18  │ +2  │                               │
│ │STR 18+4│ │  │+5tmp│     │     │                               │
│ │DEX 14+2│ │  └─────┴─────┴─────┘                               │
│ │CON 16+3│ │  ┌─────┬─────┬─────┐                               │
│ │INT 10+0│ │  │Speed│Prof │Passv│                               │
│ │WIS 12+1│ │  │30 ft│ +3  │Perc:14 Inv:10 Ins:11               │
│ │CHA  8-1│ │  └─────┴─────┴─────┘                               │
│ └────────┘ │                                                    │
│            │  Hit Dice: d10 ●●●○○ (3/5)                         │
│  DEATH     │                                                    │
│  SAVES     │  ┌─ Saves ────┬─ Skills ─────────────────────────┐ │
│  ○ ○ ○ ✓   │  │ STR +7 ◆   │ Acrobatics (DEX)        +2      │ │
│  ○ ○ ○ ✗   │  │ DEX +2     │ Animal Handling (WIS)   +1      │ │
│            │  │ CON +6 ◆   │ Athletics (STR)         +7 ◆◆   │ │
│            │  │ INT +0     │ ...                              │ │
│            │  │ WIS +1     │                                  │ │
│            │  │ CHA -1     │                                  │ │
│            │  └────────────┴──────────────────────────────────┘ │
├────────────┴────────────────────────────────────────────────────┤
│ [Features] [Proficiencies] [Equipment] [Spells] [Languages] [Notes]│
│ ┌───────────────────────────────────────────────────────────────┐│
│ │ Tab content here                                              ││
│ └───────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────┘
```

### Key Layout Changes

1. **Portrait in header** - Visual identity
2. **XP progress bar** - Level progression at a glance
3. **Conditions banner** - Only shows when active, prominent placement
4. **Death saves in sidebar** - Always visible, becomes critical at 0 HP
5. **Hit dice counter** - Between combat grid and saves/skills
6. **Expanded passive scores** - All three in one stat card
7. **Notes tab added** - For session tracking

---

## Implementation Plan

### Phase 1: Quick Wins (Low Effort, High Impact)
1. Add alignment display to header
2. Expand passive scores (perception → all three)
3. Add portrait display (if available)
4. Add death saves visual tracker

### Phase 2: Spell Management
1. Add spell slots display to Spells tab
2. Add preparation limit counter
3. Group spells by level
4. Add warlock pact slot handling

### Phase 3: Combat Readiness
1. Add hit dice counter
2. Add conditions display (when populated)
3. Add carrying capacity to Equipment tab

### Phase 4: Notes Integration
1. Create Nitro route for notes endpoint
2. Add Notes tab component
3. Implement category-based organization

### Phase 5: Future (Requires Backend)
1. Interactive HP adjustment
2. Spell slot usage tracking
3. Resource/ability tracking (#256)
4. Combat mode view toggle

---

## Data Availability Summary

| Feature | API Field | Currently Displayed |
|---------|-----------|---------------------|
| Death Saves | `death_save_successes/failures` | ❌ |
| Spell Slots | `stats.spell_slots` | ❌ |
| Hit Dice | `stats.hit_dice[]` | ❌ |
| Conditions | `character.conditions[]` | ❌ |
| Passive Investigation | `stats.passive_investigation` | ❌ |
| Passive Insight | `stats.passive_insight` | ❌ |
| Portrait | `character.portrait` | ❌ |
| Alignment | `character.alignment` | ❌ |
| XP | `character.experience_points` | ❌ |
| Preparation Limit | `stats.preparation_limit` | ❌ |
| Prepared Count | `stats.prepared_spell_count` | ❌ |
| Carrying Capacity | `stats.carrying_capacity` | ❌ |
| Notes | `/characters/{id}/notes` endpoint | ❌ (not fetched) |

All Priority 1-3 features have **backend support already** - this is purely frontend work.

---

## Success Metrics

1. **Completeness:** All available API data is displayed meaningfully
2. **Usability:** Players can find combat-critical info within 2 seconds
3. **Responsiveness:** Works well on tablet (common at gaming tables)
4. **Maintainability:** New components follow existing patterns

---

## Questions for Discussion

1. **Portrait placement:** Header, sidebar, or floating?
2. **Combat mode:** Worth the complexity, or is better organization enough?
3. **Interactive elements:** Should we plan PATCH endpoints for HP/slots/dice tracking?
4. **Notes:** Full rich text editor or simple text areas?

---

*This proposal represents the ideal character sheet from a DM/player perspective, prioritized by impact and implementation complexity.*
