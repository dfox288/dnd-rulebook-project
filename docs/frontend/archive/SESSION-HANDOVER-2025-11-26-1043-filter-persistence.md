# Handover: Filter Persistence with Pinia + IndexedDB

**Date:** 2025-11-26
**Status:** Design & Planning Complete - Ready for Implementation
**Context Used:** ~85%

---

## Summary

Designed and planned a filter persistence system that saves user filter selections across browser sessions using Pinia stores with IndexedDB storage.

---

## What Was Done

### 1. Backend Proposals Created

Moved two backend proposals to `../importer/docs/proposals/`:

| Document | Purpose |
|----------|---------|
| `BACKEND-CLASSES-DETAIL-OPTIMIZATION.md` | Reduce frontend inheritance logic (~200 lines) by adding `effective_data`, `hit_points`, `progression_table` to API |
| `BLOCKED-CLASSES-PROFICIENCY-FILTERS-2025-11-25.md` | Enable proficiency filters by denormalizing data (~4 hours backend work) |

### 2. Filter Persistence Design (Brainstorming Skill)

**File:** `docs/plans/2025-11-26-filter-persistence-design.md`

**Key Decisions:**
- **Per-tab state** with **cross-session persistence** (IndexedDB)
- **Pinia + pinia-plugin-persistedstate + idb-keyval** stack
- URL params override persisted state (for shareability)
- Last-used filters restored when no URL params present

### 3. Implementation Plan (Writing-Plans Skill)

**File:** `docs/plans/2025-11-26-filter-persistence-implementation.md`

**23 tasks** covering:
- Package installation & Nuxt config
- IndexedDB storage adapter
- 7 Pinia stores (one per entity type)
- URL sync composable
- Page refactoring (Feats → Backgrounds → Classes → Races → Spells → Items → Monsters)
- Integration tests
- Documentation updates

**TDD throughout** - Every task has explicit RED-GREEN-REFACTOR steps.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        User Actions                              │
│  1. Change filter    2. Clear filters    3. Navigate away/back  │
└──────────┬───────────────────────────────────────────┬──────────┘
           │                                           │
           ▼                                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Pinia Store                                 │
│  • useFeatFiltersStore, useSpellFiltersStore, etc.              │
│  • Reactive state (in-memory)                                   │
│  • Watched by components                                        │
└──────────┬───────────────────────────────────────────┬──────────┘
           │                                           │
           ▼                                           ▼
┌─────────────────────────┐              ┌─────────────────────────┐
│    URL Query Params     │              │       IndexedDB         │
│  • Shareable links      │              │  • Cross-session        │
│  • Browser history      │              │  • 7 keys (per entity)  │
└─────────────────────────┘              └─────────────────────────┘
```

---

## Files Created This Session

| File | Purpose |
|------|---------|
| `docs/plans/2025-11-26-filter-persistence-design.md` | Architecture design document |
| `docs/plans/2025-11-26-filter-persistence-implementation.md` | 23-task implementation plan |

---

## Commits Made

```
32cd13a docs: Add filter persistence design (Pinia + IndexedDB)
ef726c3 docs: Add filter persistence implementation plan
```

---

## Next Steps

### Immediate: Start Implementation

Run the implementation plan using the executing-plans skill:

```
/superpowers:execute-plan
```

Or manually start with **Task 1: Install Dependencies**:

```bash
docker compose exec nuxt npm install pinia @pinia/nuxt pinia-plugin-persistedstate idb-keyval
```

### Task Order (23 tasks)

1. Install dependencies
2. IndexedDB adapter (6 tests)
3. Pinia plugin config
4. Store type definitions
5. **Feats store** (12 tests) ← Start here for pattern
6. URL sync composable (6 tests)
7. **Refactor Feats page** ← Pilot migration
8-19. Repeat for other 6 entities
20. Update useSourceFilter
21. Integration tests
22. Documentation
23. Final verification

---

## Key Patterns to Follow

### Store Template

```typescript
// app/stores/featFilters.ts
export const useFeatFiltersStore = defineStore('featFilters', {
  state: (): FeatFiltersState => ({ ...DEFAULT_STATE }),

  getters: {
    hasActiveFilters: (state) => { /* ... */ },
    activeFilterCount: (state) => { /* ... */ },
    toUrlQuery: (state) => { /* ... */ }
  },

  actions: {
    clearAll() { /* preserves filtersOpen */ },
    setFromUrlQuery(query) { /* parses URL params */ }
  },

  persist: {
    key: 'dnd-filters-feats',
    storage: idbStorage,
    paths: [/* fields to persist */]
  }
})
```

### Page Integration

```typescript
// In page setup
const store = useFeatFiltersStore()
const { selectedSources, hasPrerequisites, ... } = storeToRefs(store)

onMounted(() => {
  if (hasUrlParams.value) {
    store.setFromUrlQuery(route.query)
  }
})

watch(() => store.toUrlQuery, (query) => {
  syncToUrl(query) // debounced 300ms
}, { deep: true })
```

---

## Estimated Effort

| Phase | Tasks | Time |
|-------|-------|------|
| Infrastructure | 1-6 | 2-3 hours |
| Store + Page migrations | 7-19 | 6-8 hours |
| Testing & cleanup | 20-23 | 1-2 hours |
| **Total** | 23 | **10-12 hours** |

---

## Reference Documents

- **Design:** `docs/plans/2025-11-26-filter-persistence-design.md`
- **Plan:** `docs/plans/2025-11-26-filter-persistence-implementation.md`
- **Current Status:** `docs/CURRENT_STATUS.md`
- **CLAUDE.md:** Project conventions and TDD mandate

---

## Filter Fields by Entity (Quick Reference)

| Entity | Fields | Complexity |
|--------|--------|------------|
| Feats | 4 filters | Simple (pilot) |
| Backgrounds | 3 filters | Simple |
| Classes | 2 filters | Simple |
| Races | 7 filters | Medium |
| Spells | 11 filters | Complex |
| Items | 15 filters | Complex |
| Monsters | 16 filters | Complex |

---

**Ready to implement!** Start a new session and use `/superpowers:execute-plan` or begin manually with Task 1.
