# Spell Slot Tracking & Preparation Toggle Design

**Issue:** #616
**Date:** 2025-12-14
**Status:** Approved

## Overview

Add interactive spell slot tracking and spell preparation toggling to the Character Sheet Spells Tab. Backend endpoints are ready (PR #170).

## Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Slot interaction | Click-to-cycle | Intuitive, clean UI |
| Preparation toggle | Click card body | Fast for common action |
| At-limit behavior | Grey out unprepared | Prevents errors proactively |
| State management | Optimistic updates | Snappy UX, revert on error |
| Store location | Extend characterPlayStateStore | Unified play session state |
| Initial spent state | Default to 0 | Backend issue for stats enrichment |

## API Endpoints (Backend Ready)

### Toggle Spell Preparation
```
PATCH /api/v1/characters/{id}/spells/{characterSpellId}
Body: { "is_prepared": true|false }
Response: CharacterSpell resource
```

### Update Spell Slot Usage
```
PATCH /api/v1/characters/{id}/spell-slots/{level}
Body: { "action": "use"|"restore"|"reset" }
   or { "spent": number }
   or { "action": "use", "slot_type": "pact_magic" }
Response: { level, total, spent, available, slot_type }
```

## Component Changes

### SpellSlots.vue → SpellSlotsManager.vue

**Props:**
```typescript
interface SpellSlotLevel {
  level: number
  total: number
  spent: number
}
spellSlots: SpellSlotLevel[]
characterId: number
editable: boolean
```

**Behavior:**
- Filled crystal = available (click to use)
- Empty crystal = spent (click to restore)
- Non-editable: display only

**Visual states:**
- Available: `text-spell-500`
- Spent: `text-gray-300 dark:text-gray-600` (hollow icon)
- Loading: pulse animation

### SpellCard.vue

**New props:**
```typescript
characterId: number
editable: boolean
atPrepLimit: boolean
```

**Click behavior:**
- Card body → toggle preparation
- Chevron icon → expand/collapse

**Visual states:**
- Prepared: solid border, full opacity
- Unprepared + can prepare: dimmed, cursor-pointer
- Unprepared + at limit: `opacity-40`, `cursor-not-allowed`
- Always prepared: amber icon, click shows toast
- Loading: pulse on indicator

## Store Extension (characterPlayStateStore)

**New state:**
```typescript
spellSlots: Map<number, { total: number, spent: number }>
preparedSpellIds: Set<number>
preparationLimit: number | null
```

**New actions:**
```typescript
useSpellSlot(level: number): Promise<void>
restoreSpellSlot(level: number): Promise<void>
resetSpellSlots(): Promise<void>
toggleSpellPreparation(characterSpellId: number, currentlyPrepared: boolean): Promise<void>
initializeSpellcasting(data: {...}): void
```

**Computed:**
```typescript
preparedSpellCount: number
atPreparationLimit: boolean
getSlotState(level: number): { total, spent, available }
```

## Nitro Routes

```
server/api/characters/[id]/spells/[spellId].patch.ts
server/api/characters/[id]/spell-slots/[level].patch.ts
```

Standard proxy pattern with error forwarding (422, 404).

## Testing Strategy

**Unit tests (TDD):**
- SpellSlotsManager: render states, click handlers, editable guard
- SpellCard: toggle vs expand, grey-out at limit, always-prepared toast
- Store: actions, computed, optimistic revert

**Integration tests (MSW):**
- Use slot → API → UI sync
- Prepare spell → API → count update
- Error handling flows

## Implementation Order

1. Nitro routes (2 files)
2. Store extension (state, actions, computed)
3. SpellSlotsManager tests + implementation
4. SpellCard tests + implementation
5. SpellsPanel prop threading
6. Character page store initialization
7. Integration tests
8. Browser verification

## Backend Follow-up

Create issue requesting `stats.spell_slots` enrichment:
- Current: `[2, 0, 0, ...]` (totals only)
- Requested: `[{ level: 1, total: 2, spent: 0, available: 2 }, ...]`

Until then, frontend initializes `spent: 0` and tracks via PATCH calls.

## Files to Create/Modify

**Create:**
- `server/api/characters/[id]/spells/[spellId].patch.ts`
- `server/api/characters/[id]/spell-slots/[level].patch.ts`
- `tests/components/character/sheet/SpellSlotsManager.test.ts`

**Modify:**
- `app/components/character/sheet/SpellSlots.vue` → rename to `SpellSlotsManager.vue`
- `app/components/character/sheet/SpellCard.vue`
- `app/components/character/sheet/SpellsPanel.vue`
- `app/stores/characterPlayState.ts`
- `app/pages/characters/[publicId]/index.vue`
- `tests/stores/characterPlayState.test.ts`
- `tests/components/character/sheet/SpellCard.test.ts`
