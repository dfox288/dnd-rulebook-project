# Level-Up Wizard Design

**Date:** 2025-12-10
**Status:** Approved
**Phase:** 1 (Core Level-Up Flow)

## Overview

A modal wizard for leveling up existing characters, triggered from the Character Sheet. Follows the same UX patterns as the character creation wizard while being optimized for the typically shorter level-up flow.

## User Journey

### Entry Point

**Location:** Character Sheet Header, next to the class display (e.g., "Fighter 3 (Champion)")

**Button Behavior:**
- Always visible by default (milestone leveling)
- Hidden if character is level 20 (max level)
- Styled as secondary action — discoverable but not overwhelming

### Triggering Level-Up

| Scenario | Behavior |
|----------|----------|
| Single-class, level 2+ | Auto-level that class, wizard opens to first pending choice (HP) |
| Level 1 → 2 | Wizard opens with class selection — continue current class or multiclass |
| Already multiclassed | Wizard opens with class selection — which class to advance? |

**Multiclass Eligibility:**
- Only show classes the character qualifies for (ability score prerequisites)
- Disabled classes show tooltip explaining missing prerequisites
- Prerequisites checked: STR 13 (Barbarian, Fighter, Paladin), DEX 13 (Fighter, Monk, Ranger, Rogue), etc.

### Backend Integration

Class selection triggers: `POST /characters/{id}/classes/{classSlug}/level-up`

Response includes:
- `previous_level`, `new_level`
- `features_gained` (array of new class features)
- `hp_choice_pending`, `asi_pending` (flags for pending choices)
- `spell_slots` (recalculated if applicable)

## Wizard Steps

### Step Sequence

Steps are shown dynamically based on the level-up response:

| Step | When Shown | Component |
|------|------------|-----------|
| Class Selection | Level 1→2, or already multiclassed | `StepClassSelection.vue` (new) |
| Subclass | Reaching subclass level | `StepSubclass.vue` (reuse) |
| HP Increase | Always | `StepHitPoints.vue` (new) |
| ASI / Feat | At ASI levels | `StepFeats.vue` (reuse) |
| Spells | Spellcaster gaining new spells | `StepSpells.vue` (reuse) |
| Proficiencies | Multiclass proficiency gains | `StepProficiencies.vue` (reuse) |
| Features with Choices | Features with options | Unified choices UI |
| Summary | Always (final) | `StepLevelUpSummary.vue` (new) |

### Example Flows

**Simple level-up (Fighter 4→5):**
1. HP Increase
2. Summary

**ASI level (Rogue 3→4):**
1. HP Increase
2. ASI/Feat Selection
3. Summary

**Complex multiclass (Wizard 1→2, multiclassing into Cleric):**
1. Class Selection
2. HP Increase
3. Spells (Cleric prepared spells)
4. Proficiencies
5. Summary

## HP Increase Step

### Design

New component with two options:

```
┌─────────────────────────────────────────────────┐
│  Hit Point Increase                             │
│                                                 │
│  Your hit die: d10 (Fighter)                    │
│  Constitution modifier: +2                      │
│                                                 │
│  ┌─────────────┐     ┌─────────────┐           │
│  │   🎲 ROLL   │     │ TAKE AVERAGE │           │
│  │     d10     │     │      6       │           │
│  └─────────────┘     └─────────────┘           │
│                                                 │
│  Total HP gained: [result] + 2 (CON) = [total]  │
└─────────────────────────────────────────────────┘
```

### Roll Interaction

1. User clicks "Roll"
2. Animated dice roll (~1 second)
3. Result revealed with highlight animation
4. Total calculated: "You rolled 7 + 2 (CON) = **9 HP gained**"
5. "Keep this roll" button to confirm

### Take Average

- Instantly shows calculated total
- No animation — it's the "safe" choice

## Summary & Celebration Step

### Design

```
┌─────────────────────────────────────────────────┐
│  🎉 Level Up Complete!                          │
│                                                 │
│  Fighter 3 → Fighter 4                          │
│                                                 │
│  ┌─────────────────────────────────────────┐   │
│  │ HP Increased        +9 (now 38 max)     │   │
│  │ Ability Score       +2 Strength (18)    │   │
│  │ New Feature         Extra Attack        │   │
│  └─────────────────────────────────────────┘   │
│                                                 │
│           [ View Character Sheet ]              │
└─────────────────────────────────────────────────┘
```

### Celebration Animation

1. Confetti burst (2-3 seconds, subtle)
2. Level badge animates in with glow
3. Summary items fade in with staggered reveal

### Content Shown

- Level change (including multiclass format)
- HP gained and new max
- ASI/Feat chosen (if applicable)
- Features gained with descriptions
- Spell slot changes
- Spells learned

## Modal Layout

### Structure

Full-page modal matching character creation wizard:
- **Left sidebar:** Step navigation with status indicators
- **Center:** Scrollable content area for current step
- **Bottom:** Navigation buttons (Back / Continue)

### Sidebar Navigation

- ✓ Completed steps (clickable to revisit)
- → Current step (highlighted)
- ○ Upcoming steps (visible, not clickable)

## Cancellation & Saving

### Behavior

**Auto-save progress** — Level-up commits when class is selected, choices save as completed.

- If user closes modal mid-wizard, character level is already incremented
- Pending choices remain pending (visible on character sheet)
- User can reopen level-up wizard to continue, or resolve via sheet

### Why This Approach

- Matches character creation wizard behavior
- Backend already supports `pending_choices` system
- No complex rollback logic needed
- User can always complete choices later

## Technical Architecture

### New Components

```
app/components/character/levelup/
├── LevelUpWizard.vue          # Main wizard container (modal)
├── LevelUpSidebar.vue         # Step navigation
├── StepClassSelection.vue     # Multiclass choice
├── StepHitPoints.vue          # HP roll/average
├── StepLevelUpSummary.vue     # Celebration & summary
└── HitDieRoller.vue           # Animated dice component
```

### Reused Components

Import directly from wizard:
- `StepSubclass.vue`
- `StepFeats.vue`
- `StepSpells.vue`
- `StepProficiencies.vue`

### State Management

**New store:** `app/stores/characterLevelUp.ts`

Responsibilities:
- Level-up session state (characterId, selectedClass, currentStep)
- Level-up response data (features gained, pending choices)
- Step visibility logic

Separate from `characterWizard.ts` because:
- Different lifecycle (modal vs. page)
- Cleaner separation of concerns
- Can import shared utilities

### Composable

**New:** `app/composables/useLevelUpWizard.ts`

Responsibilities:
- Step definitions and conditional visibility
- Navigation between steps
- Backend calls for level-up and choice submission
- Integration with unified choices system

### Nitro Routes

**New route:**
```
server/api/characters/[id]/classes/[classSlug]/level-up.post.ts
```

**Existing routes (already work):**
- `GET /characters/{id}/pending-choices`
- `POST /characters/{id}/choices/{choiceId}`

### API Flow

```
1. POST /api/characters/{id}/classes/fighter/level-up
   → Returns level-up result with pending choice flags

2. POST /api/characters/{id}/choices/{hpChoiceId}
   → Submit HP choice (roll result or average)

3. POST /api/characters/{id}/choices/{asiChoiceId}
   → Submit ASI/Feat choice (if applicable)

4. GET /api/characters/{id}/stats
   → Refresh character data for summary
```

## Error Handling

| Error | Handling |
|-------|----------|
| Max level (20) | Hide level-up button entirely |
| Multiclass prerequisites not met | Disable class option with tooltip |
| Network error mid-wizard | Choices saved incrementally, resume later |
| Backend validation error | Show error message, allow retry |

## Phase 2 (Deferred)

**XP Tracking:**
- Per-character toggle: Milestone vs. XP progression
- XP bar display on character sheet header
- Level-up button conditional on XP threshold
- "Add XP" functionality after sessions

## Success Criteria

- [ ] Level-up button visible on character sheet header
- [ ] Single-class characters can level up with minimal steps
- [ ] Multiclass flow asks which class to advance
- [ ] HP step offers roll vs. average with dice animation
- [ ] ASI/Feat step appears at correct levels
- [ ] Spellcasters can select new spells
- [ ] Summary shows all gains with celebration animation
- [ ] Pending choices persist if wizard closed early
- [ ] All existing wizard step components work in level-up context
