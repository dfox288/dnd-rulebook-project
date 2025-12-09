# Test Suite Consolidation Plan

**Date:** 2025-12-07
**Status:** Proposed
**Scope:** Reduce test duplication, consolidate patterns, delete redundant tests
**Estimated Impact:** ~30% reduction in test code (19,000 → ~13,000 lines)

---

## Executive Summary

An audit of the frontend test suite (203 files, ~19,000 lines) revealed significant duplication across filter stores, page filter tests, entity cards, and accordion components. This plan outlines a phased approach to consolidate tests using parameterized patterns and shared helpers, resulting in easier maintenance without sacrificing coverage.

---

## Phase 1: Filter Store Consolidation (HIGH PRIORITY)

### Problem

Seven filter store test files (`spellFilters.test.ts`, `itemFilters.test.ts`, etc.) are structurally identical. Each file:
1. Calls `usePiniaSetup()`
2. Calls `testHasActiveFilters()` with store-specific config
3. Calls `testActiveFilterCount()` with store-specific config
4. Calls `testClearAllAction()` with store-specific config
5. Calls URL sync tests with store-specific config

The only difference is the configuration data passed to each helper.

### Files Affected

```
tests/stores/
├── spellFilters.test.ts     (249 lines) → DELETE
├── itemFilters.test.ts      (398 lines) → DELETE
├── monsterFilters.test.ts   (352 lines) → DELETE
├── classFilters.test.ts     (~250 lines) → DELETE
├── raceFilters.test.ts      (~250 lines) → DELETE
├── backgroundFilters.test.ts (~250 lines) → DELETE
├── featFilters.test.ts      (~200 lines) → DELETE
└── filterStores.test.ts     (NEW ~400 lines) → CREATE
```

### Solution

Create a single parameterized test file that loops through all store configurations:

```typescript
// tests/stores/filterStores.test.ts
import { useSpellFiltersStore } from '~/stores/spellFilters'
import { useItemFiltersStore } from '~/stores/itemFilters'
// ... other stores

const FILTER_STORE_CONFIGS = [
  {
    name: 'Spell',
    store: useSpellFiltersStore,
    fields: [
      { field: 'searchQuery', value: 'fireball' },
      { field: 'selectedLevels', value: ['3'] },
      { field: 'selectedSchool', value: 5 },
      // ...
    ],
    filterCountTests: [...],
    clearAllConfig: {...},
    urlQueryTests: [...],
  },
  {
    name: 'Item',
    store: useItemFiltersStore,
    // ...
  },
  // ... 5 more stores
]

FILTER_STORE_CONFIGS.forEach(({ name, store, fields, filterCountTests, clearAllConfig, urlQueryTests }) => {
  describe(`use${name}FiltersStore`, () => {
    usePiniaSetup()
    testHasActiveFilters(store, fields)
    testActiveFilterCount(store, filterCountTests)
    testClearAllAction(store, clearAllConfig)
    testSetFromUrlQuery(store, urlQueryTests)
    testToUrlQuery(store, urlQueryTests)
  })
})
```

### Implementation Steps

1. Create `tests/stores/configs/` directory for store-specific configurations
2. Extract configuration from each existing test file into config objects
3. Create `tests/stores/filterStores.test.ts` with parameterized loop
4. Run tests to verify identical coverage
5. Delete 7 individual store test files
6. Update `package.json` test scripts if needed

### Estimated Savings

- **Before:** 7 files, ~1,950 lines
- **After:** 1 file + configs, ~400 lines
- **Savings:** ~1,550 lines (79% reduction)

---

## Phase 2: Page Filter Test Consolidation (HIGH PRIORITY)

### Problem

Page-level filter tests are split across multiple files per entity with overlapping patterns:

| Entity | Files | Lines | Issue |
|--------|-------|-------|-------|
| Backgrounds | 3 | ~450 | filters.test.ts + new-filters.test.ts + source-filter.test.ts |
| Monsters | 6 | ~800 | Granular split by filter type |
| Items | 4 | ~600 | Main + advanced + weapon-armor + api |
| Classes | 2 | ~300 | filters + filters-expanded |
| Spells | 2 | ~300 | level-filter + tag-filter |
| Races | 2 | ~250 | filters + parent-filter |
| Feats | 2 | ~250 | filter-api + filters |

### Files to Consolidate

```
tests/pages/backgrounds/
├── filters.test.ts        ─┐
├── new-filters.test.ts    ─┼→ filters.test.ts (merged)
└── source-filter.test.ts  ─┘

tests/pages/monsters/
├── filters.cr.test.ts           ─┐
├── filters.multiselect.test.ts  ─┼→ filters.interactions.test.ts
├── filters.range-boolean.test.ts─┘
├── filters.layout.test.ts       ─┐
├── filter-chips.test.ts         ─┼→ filters.presentation.test.ts
└── api-filters.test.ts          ─→ filters.api.test.ts (keep separate)

tests/pages/items/
├── filters.test.ts              ─┐
├── filters-advanced.test.ts     ─┼→ filters.test.ts (merged)
├── filters-weapon-armor.test.ts ─┘
└── api-filters.test.ts          ─→ (keep separate)
```

### Solution

1. **Create page filter test helpers** (similar to filterStoreBehavior):

```typescript
// tests/helpers/pageFilterBehavior.ts
export function testFilterMultiSelect(config: {
  page: Component
  filterName: string
  filterSelector: string
  options: string[]
  store: () => FilterStore
}) {
  describe(`${config.filterName} multi-select`, () => {
    it('renders options', async () => {...})
    it('allows selecting multiple values', async () => {...})
    it('syncs with store', async () => {...})
  })
}

export function testFilterChipDisplay(config: {...}) {...}
export function testFilterRangeSlider(config: {...}) {...}
export function testFilterBooleanToggle(config: {...}) {...}
```

2. **Merge related test files** using describe blocks for organization

### Implementation Steps

1. Create `tests/helpers/pageFilterBehavior.ts` with common patterns
2. Consolidate backgrounds: Merge 3 files → 1
3. Consolidate monsters: Merge 6 files → 3
4. Consolidate items: Merge 4 files → 2
5. Consolidate remaining entities (classes, spells, races, feats)
6. Update any CI/test script references

### Estimated Savings

- **Before:** 21 files, ~2,950 lines
- **After:** ~12 files, ~1,800 lines
- **Savings:** ~1,150 lines (39% reduction)

---

## Phase 3: Accordion Component Test Consolidation (MEDIUM PRIORITY)

### Problem

17 accordion component tests follow identical patterns for list rendering:

```
tests/components/ui/accordion/
├── UiAccordionAbilitiesList.test.ts   (84 lines)
├── UiAccordionBulletList.test.ts      (212 lines)
├── UiAccordionDamageList.test.ts      (157 lines)
├── UiAccordionEquipmentList.test.ts   (468 lines) ← HAS UNIQUE LOGIC
├── UiAccordionFeaturesList.test.ts    (150 lines)
├── UiAccordionLanguagesList.test.ts   (100 lines)
├── UiAccordionPropertiesList.test.ts  (134 lines)
├── UiAccordionProficienciesList.test.ts (186 lines)
├── UiAccordionResourcesList.test.ts   (104 lines)
├── UiAccordionSensesList.test.ts      (99 lines)
├── UiAccordionSkillsList.test.ts      (81 lines)
├── UiAccordionSpeedsList.test.ts      (84 lines)
├── UiAccordionSpellsList.test.ts      (123 lines)
├── UiAccordionStartingEquipment.test.ts (118 lines)
├── UiAccordionTraitsList.test.ts      (171 lines)
└── ... (2 more)
```

Most test the same behaviors:
- Renders items from array prop
- Shows item names
- Handles empty arrays
- Applies correct styling

### Solution

Create accordion list test helper and consolidate generic list tests:

```typescript
// tests/helpers/accordionListBehavior.ts
export function testAccordionListRendering(config: {
  component: Component
  propName: string
  mockItems: any[]
  expectedTexts: string[]
}) {
  describe('list rendering', () => {
    it('renders all items', async () => {...})
    it('handles empty array', async () => {...})
    it('displays item names', async () => {...})
  })
}
```

### Files to Keep Separate

- `UiAccordionEquipmentList.test.ts` - Has unique equipment-specific logic (468 lines)
- Any accordion with specialized formatting tests

### Files to Merge

Create 2-3 consolidated files:
- `accordionSimpleLists.test.ts` - For basic name/value lists
- `accordionFormattedLists.test.ts` - For lists with special formatting

### Estimated Savings

- **Before:** 17 files, ~2,500 lines
- **After:** ~5 files, ~1,200 lines
- **Savings:** ~1,300 lines (52% reduction)

---

## Phase 4: Entity Card Test Standardization (MEDIUM PRIORITY)

### Problem

7 entity card tests repeat identical patterns:

```typescript
// SpellCard.test.ts
it('renders spell name', async () => {
  const wrapper = await mountSuspended(SpellCard, { props: { spell: mockSpell } })
  expect(wrapper.text()).toContain('Fireball')
})

// ItemCard.test.ts - IDENTICAL PATTERN
it('renders item name', async () => {
  const wrapper = await mountSuspended(ItemCard, { props: { item: mockItem } })
  expect(wrapper.text()).toContain('Longsword')
})
```

### Files Affected

```
tests/components/
├── spell/SpellCard.test.ts
├── item/ItemCard.test.ts
├── monster/MonsterCard.test.ts
├── class/ClassCard.test.ts
├── race/RaceCard.test.ts
├── background/BackgroundCard.test.ts
└── feat/FeatCard.test.ts
```

### Solution

Enhance existing `cardBehavior.ts` helper:

```typescript
// tests/helpers/cardBehavior.ts
export function testEntityCardBasics(config: {
  component: Component
  propName: string
  mockFactory: () => any
  expectedName: string
  linkPath: string
}) {
  it(`renders ${config.propName} name`, async () => {...})
  it(`links to detail page at ${config.linkPath}`, async () => {...})
  it('handles missing optional fields gracefully', async () => {...})
}
```

### Implementation Steps

1. Enhance `cardBehavior.ts` with entity card basics
2. Apply helper to all 7 entity card tests
3. Remove duplicated manual tests
4. Keep entity-specific tests (badge visibility, unique displays)

### Estimated Savings

- **Before:** ~1,400 lines across 7 files
- **After:** ~900 lines
- **Savings:** ~500 lines (36% reduction)

---

## Phase 5: Apply Unused Helpers (MEDIUM PRIORITY)

### Problem

Two test helpers were created but never imported:

```
tests/helpers/
├── descriptionBehavior.ts   (54 lines) - NOT USED
└── sourceBehavior.ts        (44 lines) - NOT USED
```

### Files That Should Use These

**descriptionBehavior.ts** - Test description truncation:
- SpellCard.test.ts
- ItemCard.test.ts
- MonsterCard.test.ts
- FeatCard.test.ts
- BackgroundCard.test.ts

**sourceBehavior.ts** - Test source footer rendering:
- All entity card tests
- All detail page tests

### Implementation Steps

1. Audit card tests for inline description tests
2. Replace with `testDescriptionTruncation()` helper
3. Audit card tests for inline source tests
4. Replace with `testSourceFooter()` helper

### Estimated Savings

- **Lines removed:** ~300 (inline tests)
- **Lines added:** ~50 (helper calls)
- **Net savings:** ~250 lines

---

## Phase 6: Cleanup Low-Value Tests (LOW PRIORITY)

### Tests to Remove

1. **Import-only tests** - Tests that only verify a module is importable:
   ```typescript
   it('composable exists and is importable', async () => {
     const { useEntityDetail } = await import('~/composables/useEntityDetail')
     expect(useEntityDetail).toBeDefined()
   })
   ```
   If the import fails, the test file itself won't load.

2. **Pure JS operator tests** - Tests that verify JavaScript works:
   ```typescript
   it('detects melee weapon (item_type.code === "M")', () => {
     const isWeapon = mockItem.item_type?.code === 'M'
     expect(isWeapon).toBe(true)
   })
   ```

3. **Obvious property assignment tests**:
   ```typescript
   it('updates selectedSources when set', () => {
     store.selectedSources = ['PHB']
     expect(store.selectedSources).toEqual(['PHB'])
   })
   ```

### Files Affected

- `tests/composables/useEntityDetail.test.ts`
- `tests/composables/useItemDetail.test.ts`
- Various store tests with setter verification

### Estimated Savings

- ~200 lines across various files

---

## Phase 7: Mock Factory Standardization (LOW PRIORITY)

### Problem

Many tests define inline mock data instead of using `mockFactories.ts`:

```typescript
// RaceCard.test.ts - 32 lines of inline mock
const mockRace: Race = {
  id: 1,
  name: 'Elf',
  slug: 'elf',
  // ... 26 more lines
}

// Should use:
import { createMockRace } from '~/tests/helpers/mockFactories'
const mockRace = createMockRace({ name: 'Elf' })
```

### Files to Update

- `RaceCard.test.ts`
- `RacePickerCard.test.ts`
- `ClassPickerCard.test.ts`
- `BackgroundPickerCard.test.ts`
- `SubclassPickerCard.test.ts`
- `SubracePickerCard.test.ts`

### Implementation Steps

1. Audit for missing factory functions
2. Add any missing factories to `mockFactories.ts`
3. Replace inline mocks with factory calls + overrides
4. Verify tests still pass

### Estimated Savings

- ~200 lines of inline mock definitions

---

## Implementation Timeline

| Phase | Priority | Files Changed | Est. Savings | Complexity |
|-------|----------|---------------|--------------|------------|
| 1. Filter Stores | HIGH | 8 | ~1,550 lines | Low |
| 2. Page Filters | HIGH | 21 → 12 | ~1,150 lines | Medium |
| 3. Accordion Tests | MEDIUM | 17 → 5 | ~1,300 lines | Medium |
| 4. Entity Cards | MEDIUM | 7 | ~500 lines | Low |
| 5. Apply Helpers | MEDIUM | 10+ | ~250 lines | Low |
| 6. Cleanup | LOW | Various | ~200 lines | Low |
| 7. Mock Factories | LOW | 6+ | ~200 lines | Low |

**Total Estimated Savings: ~5,150 lines (27% reduction)**

---

## Success Criteria

1. All existing test coverage maintained (no reduction in assertions)
2. Test execution time equal or faster
3. Easier to add new entity types (just add config, not new file)
4. Reduced maintenance burden when patterns change
5. All tests pass after each phase

---

## Risks and Mitigations

| Risk | Mitigation |
|------|------------|
| Accidentally reduce coverage | Run coverage report before/after each phase |
| Break CI pipeline | Incremental commits, run full suite after each file change |
| Helper complexity | Keep helpers focused and well-documented |
| Lost test discoverability | Use clear describe blocks and naming in consolidated files |

---

## Notes

- Each phase can be implemented independently
- Phases 1-2 provide the highest ROI and should be prioritized
- Phases 6-7 are optional cleanup that can be done opportunistically
- All changes should follow TDD - verify tests pass before and after consolidation
