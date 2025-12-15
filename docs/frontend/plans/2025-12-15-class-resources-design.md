# Class Resources Design

**Issue:** #632 - Character sheet: Add class resource counters (Rage, Ki, etc.)
**Date:** 2025-12-15
**Status:** Approved

## Overview

Add class-specific resource counters to the character sheet that can be tracked during play. Resources like Rage, Ki Points, Bardic Inspiration, etc. are displayed with current/max values and can be spent during play mode.

## Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Placement | Left sidebar, below Hit Dice | Logical grouping with consumable resources |
| Display mode | Hybrid: icons (max ≤ 6), numeric for larger | Icons work for small counts, numeric needed for Ki/Sorcery (up to 20) |
| Interaction | Click to spend, rest to recover | Matches Hit Dice pattern users already know |
| Component | Separate ClassResources + Manager | Clean separation from Hit Dice logic |
| Depleted state | Grayscale icons | Consistent with Hit Dice styling |
| Grouping | Flat list | Simpler UI, most characters single-class |
| State management | Optimistic updates | Matches spell slots pattern, snappy UX |
| Rest reset | Via existing rest endpoints | Counters reset server-side automatically |

## API Data Shape

From `GET /api/v1/characters/{id}`:

```json
{
  "counters": [
    {
      "id": 7,
      "slug": "phb:bard:bardic-inspiration",
      "name": "Bardic Inspiration",
      "current": 3,
      "max": 5,
      "reset_on": "long_rest",
      "source": "Bard",
      "source_type": "class",
      "unlimited": false
    }
  ]
}
```

## Component Structure

```
app/components/character/sheet/
├── ClassResources.vue         # Display component (presentational)
├── ClassResourcesManager.vue  # Manager with API logic
└── ClassResourceCounter.vue   # Individual counter (icons or numeric)

server/api/characters/[id]/
└── counters/[counterId].patch.ts  # API proxy for use/restore
```

## Visual Design

### Icon Mode (max ≤ 6)

```
┌─────────────────────────────────┐
│ Bardic Inspiration    ⟳ Long   │  ← Name + reset indicator
│ ● ● ● ○ ○                      │  ← Icons (3/5 available)
└─────────────────────────────────┘
```

### Numeric Mode (max > 6)

```
┌─────────────────────────────────┐
│ Ki Points             ⟳ Short  │
│ [-]  12 / 15  [+]              │  ← Numeric with buttons
└─────────────────────────────────┘
```

### States

- **Filled icons:** Primary color, clickable in play mode
- **Empty icons:** Gray, 50% opacity
- **Reset badge:** Small `⟳ Short` or `⟳ Long` indicator
- **Disabled:** When not in play mode or character is dead

## API Integration

### New Nitro Route

```typescript
// server/api/characters/[id]/counters/[counterId].patch.ts
// Body: { action: "use" | "restore" }
// Returns: Updated counter object
```

### Optimistic Updates

```typescript
async function spendCounter(counterId: number) {
  // Optimistic: decrement locally
  const counter = counters.value.find(c => c.id === counterId)
  if (counter && counter.current > 0) {
    counter.current--
  }

  try {
    await apiFetch(`/characters/${characterId}/counters/${counterId}`, {
      method: 'PATCH',
      body: { action: 'use' }
    })
  } catch (error) {
    // Rollback on error
    counter.current++
    toast.add({ title: 'Failed to update', color: 'error' })
  }
}
```

### Rest Integration

- Existing `short-rest.post` and `long-rest.post` endpoints reset counters server-side
- After rest, `refreshForShortRest()` / `refreshForLongRest()` refresh character data
- Counters come back with reset values automatically

## Props Interface

```typescript
interface Counter {
  id: number
  slug: string
  name: string
  current: number
  max: number
  reset_on: 'short_rest' | 'long_rest' | null
  source: string
  source_type: string
  unlimited: boolean
}

// ClassResources.vue props
interface Props {
  counters: Counter[]
  editable?: boolean
  disabled?: boolean
  isDead?: boolean
}
```

## Test Cases

### ClassResources.vue
- Renders nothing when counters array is empty
- Displays counter name and current/max values
- Shows icon mode for max ≤ 6
- Shows numeric mode for max > 6
- Displays reset indicator badge (Short/Long/none)
- Applies grayscale to empty icons
- Emits `spend` event when icon clicked (editable mode)
- Does not emit when not editable
- Does not emit when counter is at 0

### ClassResourcesManager.vue
- Calls API on spend action
- Optimistically decrements counter
- Rolls back on API error
- Shows toast on error
- Disables interactions when `isDead`

## Files to Create

1. `app/components/character/sheet/ClassResources.vue`
2. `app/components/character/sheet/ClassResourcesManager.vue`
3. `app/components/character/sheet/ClassResourceCounter.vue`
4. `server/api/characters/[id]/counters/[counterId].patch.ts`
5. `tests/components/character/sheet/ClassResources.test.ts`
6. `tests/components/character/sheet/ClassResourcesManager.test.ts`
7. `tests/components/character/sheet/ClassResourceCounter.test.ts`

## Dependencies

- Backend endpoint `PATCH /characters/{id}/counters/{counterId}` (from #647 - verify exists)

## Page Integration

Add to character sheet page (`app/pages/characters/[publicId]/index.vue`):

```vue
<!-- Left Sidebar, after HitDiceManager -->
<CharacterSheetClassResourcesManager
  v-if="character.counters?.length"
  :counters="character.counters"
  :character-id="character.id"
  :editable="canEdit"
  :is-dead="character.is_dead"
/>
```
