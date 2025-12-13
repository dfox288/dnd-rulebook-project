# Character Sheet Component Refactor Design

**Issue:** #584
**Created:** 2025-12-13
**Status:** Approved

## Overview

Refactor character sheet components to be self-contained and reusable by:
1. Creating a Pinia store for play mode state (HP, death saves, currency)
2. Creating manager components that wrap display components
3. Moving ~300 lines of business logic out of the main page

## Goals

- Components become truly independent and reusable (e.g., for DM Screen)
- Single source of truth for play state via Pinia store
- Clean separation: display components vs manager components
- Maintain TDD discipline throughout

## Architecture

### Component Structure

```
app/components/character/sheet/
├── CurrencyManager.vue      # NEW - wraps StatCurrency + CurrencyEditModal
├── HitPointsManager.vue     # NEW - wraps StatHitPoints + HpEditModal + TempHpModal
├── DeathSavesManager.vue    # NEW - wraps DeathSaves
├── StatCurrency.vue         # KEEP - pure display
├── StatHitPoints.vue        # KEEP - pure display
├── DeathSaves.vue           # KEEP - pure display
├── CurrencyEditModal.vue    # KEEP - form/validation only
├── HpEditModal.vue          # KEEP - form only
├── TempHpModal.vue          # KEEP - form only
└── ... (other components unchanged)
```

### Pinia Store: `characterPlayState`

```typescript
// stores/characterPlayState.ts
export const useCharacterPlayStateStore = defineStore('characterPlayState', () => {
  // Identity
  const characterId = ref<number | null>(null)
  const isDead = ref(false)

  // Play state
  const hitPoints = reactive({ current: 0, max: 0, temporary: 0 })
  const deathSaves = reactive({ successes: 0, failures: 0 })
  const currency = reactive({ pp: 0, gp: 0, ep: 0, sp: 0, cp: 0 })

  // Loading flags (prevent race conditions)
  const isUpdatingHp = ref(false)
  const isUpdatingCurrency = ref(false)
  const isUpdatingDeathSaves = ref(false)

  // Initialize from server data
  function initialize(data: CharacterPlayData) { ... }

  // Actions (API calls + state updates)
  async function updateHp(delta: number) { ... }
  async function setTempHp(value: number) { ... }
  async function clearTempHp() { ... }
  async function updateCurrency(payload: CurrencyDelta) { ... }
  async function updateDeathSaves(field: 'successes' | 'failures', value: number) { ... }

  // Reset on page leave
  function $reset() { ... }

  return {
    characterId, isDead, hitPoints, deathSaves, currency,
    isUpdatingHp, isUpdatingCurrency, isUpdatingDeathSaves,
    initialize, updateHp, setTempHp, clearTempHp, updateCurrency, updateDeathSaves
  }
})
```

### Manager Component Pattern

Each manager:
- Reads state from Pinia store
- Calls store actions for updates
- Handles UI concerns (modal state, error display, toasts)
- Wraps a pure display component

```vue
<!-- CurrencyManager.vue -->
<script setup lang="ts">
const props = defineProps<{ editable?: boolean }>()

const store = useCharacterPlayStateStore()
const toast = useToast()

const modalOpen = ref(false)
const error = ref<string | null>(null)

async function handleUpdate(payload: CurrencyDelta) {
  error.value = null
  try {
    await store.updateCurrency(payload)
    modalOpen.value = false
    toast.add({ title: 'Currency updated', color: 'success' })
  } catch (err: any) {
    if (err.statusCode === 422) {
      error.value = err.data?.message || 'Insufficient funds'
    } else {
      toast.add({ title: 'Failed to update currency', color: 'error' })
    }
  }
}
</script>

<template>
  <StatCurrency
    :currency="store.currency"
    :editable="editable"
    @click="modalOpen = true"
  />
  <CurrencyEditModal
    v-model:open="modalOpen"
    :currency="store.currency"
    :loading="store.isUpdatingCurrency"
    :error="error"
    @apply="handleUpdate"
    @clear-error="error = null"
  />
</template>
```

### Page Integration

```vue
<!-- pages/characters/[publicId]/index.vue -->
<script setup>
const store = useCharacterPlayStateStore()

// Initialize store when data loads
watch([character, stats], ([char, s]) => {
  if (char && s) {
    store.initialize({
      characterId: char.id,
      isDead: char.is_dead,
      hitPoints: s.hit_points,
      deathSaves: {
        successes: char.death_save_successes,
        failures: char.death_save_failures
      },
      currency: char.currency
    })
  }
}, { immediate: true })

// Cleanup on leave
onUnmounted(() => store.$reset())
</script>

<template>
  <CurrencyManager :editable="canEdit" />
  <HitPointsManager :editable="canEdit" />
  <DeathSavesManager :editable="canEdit" />
</template>
```

## Death Saves Coordination

HP and death saves are coupled by D&D rules:
- Healing from 0 HP resets death saves
- HP endpoint response includes updated death saves

Solution: Store handles the coupling internally. When `updateHp()` receives a response with death save data, it updates both `hitPoints` and `deathSaves` atomically. Components react automatically via Pinia reactivity.

DeathSavesManager derives its own editability:
```typescript
const canEdit = computed(() => {
  return props.editable && store.hitPoints.current === 0 && !store.isDead
})
```

## File Changes

### New Files
```
app/stores/characterPlayState.ts              # ~150 lines
app/components/character/sheet/
├── CurrencyManager.vue                       # ~80 lines
├── HitPointsManager.vue                      # ~100 lines
├── DeathSavesManager.vue                     # ~60 lines
tests/stores/characterPlayState.test.ts
tests/components/character/sheet/
├── CurrencyManager.test.ts
├── HitPointsManager.test.ts
├── DeathSavesManager.test.ts
tests/helpers/playStateTestSetup.ts
```

### Modified Files
```
app/pages/characters/[publicId]/index.vue     # Remove ~300 lines, add store init
```

### Unchanged Files
```
app/components/character/sheet/StatCurrency.vue
app/components/character/sheet/StatHitPoints.vue
app/components/character/sheet/DeathSaves.vue
app/components/character/sheet/CurrencyEditModal.vue
app/components/character/sheet/HpEditModal.vue
app/components/character/sheet/TempHpModal.vue
```

## Testing Strategy

### TDD Order
1. Store tests first (core logic)
2. Manager tests second (UI integration)
3. Page integration test (everything wired up)

### Store Tests
- Initialize with data
- Each action updates state correctly
- API error handling (rollback on failure)
- Death saves sync when HP response includes them

### Manager Tests
- Renders display component with store data
- Opens modal on click (when editable)
- Calls store action on submit
- Shows toast on success/error
- Shows error in modal for 422

### Test Helper
```typescript
// tests/helpers/playStateTestSetup.ts
export function setupPlayStateStore(overrides?: Partial<CharacterPlayData>) {
  setActivePinia(createPinia())
  const store = useCharacterPlayStateStore()
  store.initialize({
    characterId: 1,
    isDead: false,
    hitPoints: { current: 25, max: 30, temporary: 0 },
    deathSaves: { successes: 0, failures: 0 },
    currency: { pp: 0, gp: 10, ep: 0, sp: 5, cp: 0 },
    ...overrides
  })
  return store
}
```

## Outcome

- Page drops from ~860 lines to ~560 lines
- Logic properly encapsulated and reusable
- Ready for DM Screen reuse
- Foundation for future Pinia centralization
