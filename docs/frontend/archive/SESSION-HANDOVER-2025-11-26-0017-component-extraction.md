# Handover: Component Extraction & Code Deduplication

**Date:** 2025-11-26
**Session Focus:** Extract reusable components from 7 entity list pages to eliminate code duplication
**Status:** ✅ **COMPLETE**

---

## Executive Summary

Successfully extracted 5 reusable components/composables from 7 entity list pages, achieving **~24% total code reduction** (4,241 → 3,226 lines). All pages now use consistent patterns for search, sorting, source filtering, filter chips, and list states.

---

## What Was Created

### 1. `useSortValue()` Composable
**Location:** `app/composables/useSortValue.ts`
**Purpose:** Manages sort value as combined "field:direction" string for dropdown binding

```typescript
const sortValue = useSortValue(sortBy, sortDirection)
// Returns computed that syncs "name:asc" ↔ sortBy="name", sortDirection="asc"
```

**Before:** Each page had 8-line computed property
**After:** Single line composable call

---

### 2. `useSourceFilter()` Composable
**Location:** `app/composables/useSourceFilter.ts`
**Purpose:** Encapsulates source filter state, options fetching, and helper functions

```typescript
const { selectedSources, sourceOptions, getSourceName, clearSources } = useSourceFilter()
// Optional: transform option for filtering sources
```

**Before:** 15-20 lines of ref, computed, and helper function per page
**After:** Single composable call with all functionality

---

### 3. `UiEntitySearchRow` Component
**Location:** `app/components/ui/entity/UiEntitySearchRow.vue`
**Purpose:** Unified search input + source filter + sort dropdown row

```vue
<UiEntitySearchRow
  v-model:search="searchQuery"
  v-model:sources="selectedSources"
  v-model:sort="sortValue"
  placeholder="Search spells..."
  :source-options="sourceOptions"
  :sort-options="sortOptions"
  color="spell"
/>
```

**Before:** 35-40 lines of template per page
**After:** Single component with props

---

### 4. `UiListStates` Component
**Location:** `app/components/ui/list/UiListStates.vue`
**Purpose:** Handles loading, error, empty, and results states with pagination

```vue
<UiListStates
  :loading="loading"
  :error="error"
  :empty="items.length === 0"
  :meta="meta"
  :total="totalResults"
  entity-name="spell"
  entity-name-plural="Spells"
  :has-filters="hasActiveFilters"
  :current-page="currentPage"
  :per-page="perPage"
  @retry="refresh"
  @clear-filters="clearFilters"
  @update:current-page="currentPage = $event"
>
  <template #grid>
    <SpellCard v-for="spell in spells" :key="spell.id" :spell="spell" />
  </template>
</UiListStates>
```

**Before:** 40-50 lines of v-if/v-else-if/v-else template per page
**After:** Single component with slot for grid content

---

### 5. `UiFilterChips` + `UiFilterChip` Components
**Location:** `app/components/ui/filter/UiFilterChips.vue`, `UiFilterChip.vue`
**Purpose:** Container and individual chip components for active filter display

```vue
<UiFilterChips
  :visible="hasActiveFilters"
  :search-query="searchQuery"
  :active-count="activeFilterCount"
  @clear-search="searchQuery = ''"
  @clear-all="clearFilters"
>
  <template #sources>
    <UiFilterChip v-for="source in selectedSources" :key="source" color="neutral" @remove="...">
      {{ getSourceName(source) }}
    </UiFilterChip>
  </template>

  <!-- Entity-specific chips in default slot -->
  <UiFilterChip v-if="selectedLevel" color="spell" @remove="selectedLevel = null">
    Level: {{ selectedLevel }}
  </UiFilterChip>

  <template #toggles>
    <!-- Boolean toggle chips -->
  </template>
</UiFilterChips>
```

**Before:** 50-200 lines of UButton-based chips per page
**After:** Slot-based composition with consistent styling

---

## Page-by-Page Results

| Page | Before | After | Reduction | Notes |
|------|--------|-------|-----------|-------|
| Backgrounds | 375 | 263 | **30%** | Simplest page, most benefit |
| Feats | 421 | 289 | **31%** | Few custom filters |
| Classes | 420 | 304 | **28%** | Medium complexity |
| Races | 535 | 375 | **30%** | Custom speed filter retained |
| Spells | 723 | 494 | **32%** | Most filters, still biggest savings |
| Items | 873 | 734 | **16%** | Many unique filters |
| Monsters | 894 | 767 | **14%** | Most unique filters |
| **Total** | **4,241** | **3,226** | **~24%** | **1,015 lines eliminated** |

---

## Why Different Reduction Rates?

### High Reduction (28-32%): Backgrounds, Feats, Classes, Races, Spells
- Simpler filter sets
- Standard source/search/sort patterns were dominant
- Filter chips followed common patterns

### Lower Reduction (14-16%): Items, Monsters
- **Many unique filters** (Items: 17 filters, Monsters: 14 filters)
- Custom filter logic (cost ranges, AC ranges, movement types)
- Entity-specific chip display (CR with fractions, armor types)
- These unique patterns can't be abstracted without creating overly specific components

The composables eliminate the **common duplication** while preserving **necessary entity-specific logic**.

---

## Architecture Decisions

### Slot-Based Composition
`UiFilterChips` uses 3 slots:
- `#sources` - Source chips (always neutral color)
- `default` - Entity-specific chips (entity color)
- `#toggles` - Boolean toggle chips (primary color)

This allows consistent ordering while supporting unique chip content.

### V-Model Patterns
`UiEntitySearchRow` uses multiple v-model bindings:
```vue
v-model:search="searchQuery"
v-model:sources="selectedSources"
v-model:sort="sortValue"
```

This keeps two-way binding while encapsulating the UI.

### Composable vs Component
- **Composables** (`useSortValue`, `useSourceFilter`): Pure logic, no template
- **Components** (`UiEntitySearchRow`, `UiListStates`, `UiFilterChips`): Template + styling

This separation allows mixing approaches based on what's being reused.

---

## Files Changed

### New Files Created
- `app/composables/useSortValue.ts`
- `app/composables/useSourceFilter.ts`
- `app/components/ui/entity/UiEntitySearchRow.vue`
- `app/components/ui/list/UiListStates.vue`
- `app/components/ui/filter/UiFilterChips.vue`
- `app/components/ui/filter/UiFilterChip.vue`

### Files Modified
- `app/pages/backgrounds/index.vue`
- `app/pages/feats/index.vue`
- `app/pages/classes/index.vue`
- `app/pages/races/index.vue`
- `app/pages/spells/index.vue`
- `app/pages/items/index.vue`
- `app/pages/monsters/index.vue`

---

## Git Commits

1. `d955c95` - refactor: Major filter UX improvements across all entity pages
2. `0e4b89e` - refactor: Standardize filter sections across all entity pages
3. `[previous]` - refactor: Apply reusable components to Spells page
4. `97623c2` - refactor: Apply reusable components to Items and Monsters pages

---

## Verification

All 7 entity pages verified working:
```
spells: 200
items: 200
races: 200
classes: 200
backgrounds: 200
feats: 200
monsters: 200
```

---

## Future Optimization Opportunities

1. **Filter definition objects** - Could create standard filter config objects for common patterns
2. **Generic filter chip renderer** - Could auto-generate chips from filter definitions
3. **Filter persistence composable** - URL query param sync could be extracted
4. **Test coverage** - Add tests for new composables and components

These are diminishing returns - the 24% reduction achieved covers the major duplication.

---

## For Next Agent

### Quick Start
The pattern is now established. To add a new entity list page:

1. Use `useSortValue()` for sort dropdown
2. Use `useSourceFilter()` for source filter
3. Use `<UiEntitySearchRow>` for search row
4. Use `<UiFilterChips>` + `<UiFilterChip>` for active filters
5. Use `<UiListStates>` for loading/error/empty/results

### Example Minimal Page
```vue
<script setup lang="ts">
const sortBy = ref('name')
const sortDirection = ref<'asc' | 'desc'>('asc')
const sortValue = useSortValue(sortBy, sortDirection)
const { selectedSources, sourceOptions, getSourceName, clearSources } = useSourceFilter()

const { searchQuery, data, loading, error, ... } = useEntityList({ ... })
</script>

<template>
  <UiEntitySearchRow v-model:search="searchQuery" v-model:sources="selectedSources" v-model:sort="sortValue" ... />
  <UiFilterChips :visible="hasActiveFilters" ...>
    <template #sources>...</template>
  </UiFilterChips>
  <UiListStates :loading="loading" :error="error" ...>
    <template #grid>...</template>
  </UiListStates>
</template>
```

---

**End of Handover**
