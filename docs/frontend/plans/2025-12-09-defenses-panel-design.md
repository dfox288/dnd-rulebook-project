# DefensesPanel Component Design

**Date:** 2025-12-09
**Status:** Approved
**Issue:** [#432](https://github.com/dfox288/ledger-of-heroes/issues/432)

## Overview

Display defensive traits (damage resistances, immunities, vulnerabilities, condition advantages/immunities) on the character sheet.

## Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| **Location** | New panel below CombatStatsGrid | Clean separation, easy to hide when empty |
| **Layout** | Horizontal badges grouped by category | Scannable, collapses well with few items |
| **Empty state** | Hide entirely | Reduces noise, appearance becomes meaningful |
| **Source display** | Inline parenthetical | Always visible, mobile-friendly |

## Scope

### In Scope

- New `DefensesPanel.vue` component
- Display damage resistances, immunities, vulnerabilities
- Display condition advantages (with `condition !== null` only)
- Display condition immunities
- Integration into character sheet page
- Component tests

### Out of Scope

- SavingThrowsList condition advantage indicators (follow-up)
- Skill advantages display (#433)

## Component Design

### Props

```typescript
interface Props {
  damageResistances: { type: string; condition: string | null; source: string }[]
  damageImmunities: { type: string; condition: string | null; source: string }[]
  damageVulnerabilities: { type: string; condition: string | null; source: string }[]
  conditionAdvantages: { condition: string; effect: string; source: string }[]
  conditionImmunities: { condition: string; effect: string; source: string }[]
}
```

### Behavior

- Renders nothing when all arrays empty
- Each category only renders if it has items
- Categories displayed in order: Resistances, Immunities, Vulnerabilities, Save Advantages, Condition Immunities

### Badge Colors

| Category | Color | Rationale |
|----------|-------|-----------|
| Resistances | `neutral` | Defensive, passive |
| Immunities | `success` | Strong positive |
| Vulnerabilities | `error` | Dangerous weakness |
| Save Advantages | `info` | Situational benefit |
| Condition Immunities | `success` | Strong positive |

### Badge Format

Standard: `[Type (Source)]`
With condition: `[Type (Source) - condition text]`

Examples:
- `[Poison (Dwarf)]`
- `[Fire (Red Dragon) - from nonmagical attacks]`
- `[vs Poisoned (Dwarf)]`

### Visual Layout

```
┌─────────────────────────────────────────────────────────────┐
│ Resistances: [Poison (Dwarf)]                               │
│ Save Advantages: [vs Poisoned (Dwarf)]                      │
└─────────────────────────────────────────────────────────────┘
```

## Files

### Create

| File | Purpose |
|------|---------|
| `app/components/character/sheet/DefensesPanel.vue` | New component |
| `tests/components/character/sheet/DefensesPanel.test.ts` | Component tests |

### Modify

| File | Change |
|------|--------|
| `app/pages/characters/[publicId]/index.vue` | Add DefensesPanel after CombatStatsGrid |
| `app/types/character.ts` | Add defensive trait types to CharacterStats |

## Test Cases

| Test | Description |
|------|-------------|
| Empty state | Returns nothing when all arrays empty |
| Resistances display | Shows resistance badges with type and source |
| Immunities display | Shows immunity badges in success color |
| Vulnerabilities display | Shows vulnerability badges in error color |
| Conditional text | Appends condition when present |
| Mixed defenses | Handles character with multiple categories |
| Single category | Shows panel with only one populated category |

## Implementation Order

1. Add types to `character.ts`
2. Write tests for DefensesPanel (TDD)
3. Create DefensesPanel component
4. Integrate into character sheet page
5. Manual verification
