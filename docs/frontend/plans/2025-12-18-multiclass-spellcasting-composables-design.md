# Multiclass Spellcasting Composables Design

**Issue:** [#780 - refactor(spells): Extract multiclass spellcasting logic to composables](https://github.com/dfox288/ledger-of-heroes/issues/780)
**Created:** 2025-12-18
**Status:** Approved

## Problem

The `spells.vue` page is over 1000 lines and handles complex multiclass spellcasting logic that should be extracted into reusable composables.

## Solution

Extract logic into two composables:

1. **`useSpellcastingTabs`** - Multiclass class detection and tab generation
2. **`useSpellPreparation`** - Preparation mode state and limit tracking

---

## Composable 1: useSpellcastingTabs

**File:** `app/composables/useSpellcastingTabs.ts`

**Purpose:** Extract multiclass spellcasting class detection and tab generation. Answers "what spellcasting classes does this character have?"

### Interface

```typescript
import type { CharacterStats, ClassSpellcastingInfo } from '~/types/character'

export interface SpellcastingClass {
  slug: string
  slotName: string  // Safe for UTabs slot names (e.g., "phb-wizard")
  name: string
  color: string
  info: ClassSpellcastingInfo
}

interface TabItem {
  label: string
  slot: string
  value: string
}

export function useSpellcastingTabs(stats: Ref<CharacterStats | null>) {
  return {
    spellcastingClasses: ComputedRef<SpellcastingClass[]>
    isMulticlassSpellcaster: ComputedRef<boolean>
    primarySpellcasting: ComputedRef<SpellcastingClass | null>
    tabItems: ComputedRef<TabItem[]>
  }
}
```

### Logic Extracted (from spells.vue)

- Lines 353-359: `SpellcastingClass` interface
- Lines 361-372: `spellcastingClasses` computed
- Line 377: `isMulticlassSpellcaster` computed
- Line 382: `primarySpellcasting` computed
- Lines 391-400: `tabItems` computed

### Dependencies

- `~/utils/classColors` - `getClassColor`, `getClassName`

---

## Composable 2: useSpellPreparation

**File:** `app/composables/useSpellPreparation.ts`

**Purpose:** Extract preparation mode state and limit tracking logic. Answers "how does spell preparation work for this character?"

### Interface

```typescript
import type { CharacterStats, CharacterSpell, SpellSlotsResponse, PreparationMethod, Character } from '~/types/character'
import type { SpellcastingClass } from './useSpellcastingTabs'

interface UseSpellPreparationOptions {
  stats: Ref<CharacterStats | null>
  spellSlots: Ref<SpellSlotsResponse | null>
  spellcastingClasses: ComputedRef<SpellcastingClass[]>
  validSpells: ComputedRef<CharacterSpell[]>
  playStateStore: ReturnType<typeof usePlayStateStore>
  character: Ref<Character | null>
}

export function useSpellPreparation(options: UseSpellPreparationOptions) {
  return {
    // Preparation method info
    preparationMethod: ComputedRef<PreparationMethod>
    isSpellbookCaster: ComputedRef<boolean>
    isPreparedCaster: ComputedRef<boolean>
    showPreparationUI: ComputedRef<boolean>

    // Prepare mode state
    prepareSpellsMode: Ref<string | null>
    isPrepareSpellsModeFor: (classSlug: string) => boolean
    enterPrepareSpellsMode: (classSlug: string) => void
    exitPrepareSpellsMode: () => void

    // Spell level limits
    maxCastableLevel: ComputedRef<number>
    getClassMaxSpellLevel: (classSlug: string) => number

    // Preparation limits & counts
    hasPerClassLimits: ComputedRef<boolean>
    getClassPreparationLimit: (classSlug: string) => { limit: number; prepared: number } | null
    getReactivePreparedCount: (classSlug: string) => number
    isAtClassPreparationLimit: (classSlug: string) => boolean
    reactiveTotalPreparedCount: ComputedRef<number>

    // Per-class method lookup
    getClassPreparationMethod: (classSlug: string) => PreparationMethod

    // Cross-class tracking
    getOtherClassPrepared: (spellSlug: string, currentClassSlug: string | null) => string | null

    // Spellbook-specific (wizard)
    spellbookClass: ComputedRef<SpellcastingClass | null>
    spellbookPreparationLimit: ComputedRef<number>
    spellbookPreparedCount: ComputedRef<number>
  }
}
```

### Logic Extracted (from spells.vue)

- Lines 211-218: `preparationMethod`, `isSpellbookCaster`
- Lines 225-256: Spellbook class detection and limits
- Lines 263-272: `showPreparationUI`, `isPreparedCaster`
- Lines 280-301: Prepare mode state and functions
- Lines 307-334: Level limit calculations
- Lines 340-343: `getClassPreparationMethod`
- Lines 404-439: Per-class limit functions
- Lines 460-489: Cross-class preparation tracking
- Lines 496-500: Reactive total prepared count

### Dependencies

- `useSpellcastingTabs` output (spellcastingClasses)
- `usePlayStateStore` (for preparedSpellIds)

---

## What Stays in spells.vue

- Spell fetching (`useAsyncData` for spells and slots)
- Spell grouping logic (`validSpells`, `cantrips`, `spellsByLevel`, etc.)
- Per-class spell filtering functions (`getSpellsForClass`, etc.)
- Store initialization watchers
- Template and UI logic

---

## Testing Strategy

### tests/composables/useSpellcastingTabs.test.ts (~15 tests)

| Test | Description |
|------|-------------|
| Single-class detection | Returns 1 class, `isMulticlassSpellcaster` false |
| Multiclass detection | Cleric/Wizard returns 2 classes |
| Non-spellcaster | Returns empty array |
| Tab items structure | Includes "All Prepared Spells" first |
| Primary spellcasting | Returns first class |
| Slot name generation | Converts "phb:wizard" to "phb-wizard" |

### tests/composables/useSpellPreparation.test.ts (~25 tests)

| Test | Description |
|------|-------------|
| Known caster | `showPreparationUI` false for Sorcerer |
| Prepared caster | `isPreparedCaster` true for Cleric |
| Spellbook caster | `isSpellbookCaster` true for Wizard |
| Preparation limits | Per-class limits respected |
| Reactive counts | Updates from store's preparedSpellIds |
| Cross-class detection | Identifies spells prepared by other class |
| Mode state | Enter/exit/check functions work |
| Max spell level | Calculates correctly per class level |

---

## Expected Outcome

- **Current:** ~1040 lines in spells.vue
- **After:** ~700-750 lines in spells.vue (~300 lines extracted)
- **New files:** 2 composables (~260 lines total) + 2 test files (~300 lines total)
- **Reusability:** Composables can be used by other views (battle page, character sheet)

---

## Implementation Order

1. Create `useSpellcastingTabs` with tests (TDD)
2. Create `useSpellPreparation` with tests (TDD)
3. Refactor `spells.vue` to use new composables
4. Run full test suite to verify no regressions
