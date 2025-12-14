# Encounter Monsters Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Add monsters to the DM Screen initiative tracker with backend persistence.

**Architecture:** Party-scoped encounter monsters stored in backend, fetched alongside party stats. Combat state (initiative) stays in localStorage. Monster instances displayed in same table as characters with visual distinction.

**Tech Stack:** Vue 3, Nuxt 4, NuxtUI 4, Vitest, Nitro server routes, Laravel backend API

**Blocked by:** #610 (Backend API) - Phase 1 can proceed, Phase 2 requires backend

---

## Phase 1: Types & Nitro Routes (can start now)

### Task 1: Add Encounter Monster Types

**Files:**
- Modify: `app/types/dm-screen.ts`
- Create: `tests/types/encounter-monster.test.ts`

**Step 1: Add types to dm-screen.ts**

Add after `DmScreenPartyStats` interface:

```typescript
// Monster action for quick combat reference
export interface EncounterMonsterAction {
  name: string
  attack_bonus: number | null
  damage: string | null
  reach: string | null
  range: string | null
}

// Nested monster data from compendium
export interface EncounterMonsterData {
  name: string
  slug: string
  armor_class: number
  hit_points: {
    average: number
    formula: string
  }
  speed: {
    walk: number | null
    fly: number | null
    swim: number | null
    climb: number | null
  }
  challenge_rating: string
  actions: EncounterMonsterAction[]
}

// Monster instance in an encounter
export interface EncounterMonster {
  id: number
  monster_id: number
  label: string
  current_hp: number
  max_hp: number
  monster: EncounterMonsterData
}

// Combatant union type for initiative tracking
export type Combatant =
  | { type: 'character'; id: number; data: DmScreenCharacter }
  | { type: 'monster'; id: number; data: EncounterMonster }
```

**Step 2: Commit**

```bash
git add app/types/dm-screen.ts
git commit -m "feat(dm-screen): add encounter monster types"
```

---

### Task 2: Create Nitro Routes for Encounter Monsters

**Files:**
- Create: `server/api/parties/[id]/monsters/index.get.ts`
- Create: `server/api/parties/[id]/monsters/index.post.ts`
- Create: `server/api/parties/[id]/monsters/[monsterId].patch.ts`
- Create: `server/api/parties/[id]/monsters/[monsterId].delete.ts`
- Create: `server/api/parties/[id]/monsters/index.delete.ts`

**Step 1: Create GET route**

```typescript
// server/api/parties/[id]/monsters/index.get.ts
export default defineEventHandler(async (event) => {
  const config = useRuntimeConfig()
  const id = getRouterParam(event, 'id')

  try {
    const data = await $fetch(`${config.apiBaseServer}/parties/${id}/monsters`)
    return data
  } catch (error: unknown) {
    const err = error as { statusCode?: number; statusMessage?: string; data?: unknown }
    throw createError({
      statusCode: err.statusCode || 500,
      statusMessage: err.statusMessage || 'Failed to fetch encounter monsters',
      data: err.data
    })
  }
})
```

**Step 2: Create POST route**

```typescript
// server/api/parties/[id]/monsters/index.post.ts
export default defineEventHandler(async (event) => {
  const config = useRuntimeConfig()
  const id = getRouterParam(event, 'id')
  const body = await readBody(event)

  try {
    const data = await $fetch(`${config.apiBaseServer}/parties/${id}/monsters`, {
      method: 'POST',
      body
    })
    return data
  } catch (error: unknown) {
    const err = error as { statusCode?: number; statusMessage?: string; data?: unknown }
    throw createError({
      statusCode: err.statusCode || 500,
      statusMessage: err.statusMessage || 'Failed to add monster',
      data: err.data
    })
  }
})
```

**Step 3: Create PATCH route**

```typescript
// server/api/parties/[id]/monsters/[monsterId].patch.ts
export default defineEventHandler(async (event) => {
  const config = useRuntimeConfig()
  const id = getRouterParam(event, 'id')
  const monsterId = getRouterParam(event, 'monsterId')
  const body = await readBody(event)

  try {
    const data = await $fetch(`${config.apiBaseServer}/parties/${id}/monsters/${monsterId}`, {
      method: 'PATCH',
      body
    })
    return data
  } catch (error: unknown) {
    const err = error as { statusCode?: number; statusMessage?: string; data?: unknown }
    throw createError({
      statusCode: err.statusCode || 500,
      statusMessage: err.statusMessage || 'Failed to update monster',
      data: err.data
    })
  }
})
```

**Step 4: Create DELETE single route**

```typescript
// server/api/parties/[id]/monsters/[monsterId].delete.ts
export default defineEventHandler(async (event) => {
  const config = useRuntimeConfig()
  const id = getRouterParam(event, 'id')
  const monsterId = getRouterParam(event, 'monsterId')

  try {
    await $fetch(`${config.apiBaseServer}/parties/${id}/monsters/${monsterId}`, {
      method: 'DELETE'
    })
    return { success: true }
  } catch (error: unknown) {
    const err = error as { statusCode?: number; statusMessage?: string; data?: unknown }
    throw createError({
      statusCode: err.statusCode || 500,
      statusMessage: err.statusMessage || 'Failed to remove monster',
      data: err.data
    })
  }
})
```

**Step 5: Create DELETE all route**

```typescript
// server/api/parties/[id]/monsters/index.delete.ts
export default defineEventHandler(async (event) => {
  const config = useRuntimeConfig()
  const id = getRouterParam(event, 'id')

  try {
    await $fetch(`${config.apiBaseServer}/parties/${id}/monsters`, {
      method: 'DELETE'
    })
    return { success: true }
  } catch (error: unknown) {
    const err = error as { statusCode?: number; statusMessage?: string; data?: unknown }
    throw createError({
      statusCode: err.statusCode || 500,
      statusMessage: err.statusMessage || 'Failed to clear encounter',
      data: err.data
    })
  }
})
```

**Step 6: Commit**

```bash
git add server/api/parties/\[id\]/monsters/
git commit -m "feat(dm-screen): add Nitro routes for encounter monsters API"
```

---

### Task 3: Create useEncounterMonsters Composable

**Files:**
- Create: `app/composables/useEncounterMonsters.ts`
- Create: `tests/composables/useEncounterMonsters.test.ts`

**Step 1: Write the test file**

```typescript
// tests/composables/useEncounterMonsters.test.ts
import { describe, it, expect, vi, beforeEach } from 'vitest'
import { mockNuxtImport } from '@nuxt/test-utils/runtime'

// Mock apiFetch
const mockApiFetch = vi.fn()
mockNuxtImport('useApi', () => () => ({ apiFetch: mockApiFetch }))

describe('useEncounterMonsters', () => {
  beforeEach(() => {
    mockApiFetch.mockClear()
  })

  describe('fetchMonsters', () => {
    it('fetches monsters for a party', async () => {
      const mockMonsters = [
        { id: 1, monster_id: 42, label: 'Goblin 1', current_hp: 7, max_hp: 7 }
      ]
      mockApiFetch.mockResolvedValue({ data: mockMonsters })

      const { useEncounterMonsters } = await import('~/composables/useEncounterMonsters')
      const { monsters, fetchMonsters } = useEncounterMonsters('party-1')

      await fetchMonsters()

      expect(mockApiFetch).toHaveBeenCalledWith('/parties/party-1/monsters')
      expect(monsters.value).toEqual(mockMonsters)
    })
  })

  describe('addMonster', () => {
    it('posts monster with quantity and refreshes list', async () => {
      const newMonsters = [
        { id: 1, label: 'Goblin 1' },
        { id: 2, label: 'Goblin 2' }
      ]
      mockApiFetch.mockResolvedValueOnce({ data: newMonsters })
      mockApiFetch.mockResolvedValueOnce({ data: newMonsters })

      const { useEncounterMonsters } = await import('~/composables/useEncounterMonsters')
      const { addMonster } = useEncounterMonsters('party-1')

      await addMonster(42, 2)

      expect(mockApiFetch).toHaveBeenCalledWith('/parties/party-1/monsters', {
        method: 'POST',
        body: { monster_id: 42, quantity: 2 }
      })
    })
  })

  describe('updateMonsterHp', () => {
    it('patches monster HP', async () => {
      mockApiFetch.mockResolvedValue({ data: { id: 1, current_hp: 5 } })

      const { useEncounterMonsters } = await import('~/composables/useEncounterMonsters')
      const { updateMonsterHp } = useEncounterMonsters('party-1')

      await updateMonsterHp(1, 5)

      expect(mockApiFetch).toHaveBeenCalledWith('/parties/party-1/monsters/1', {
        method: 'PATCH',
        body: { current_hp: 5 }
      })
    })
  })

  describe('removeMonster', () => {
    it('deletes a monster instance', async () => {
      mockApiFetch.mockResolvedValue({ success: true })

      const { useEncounterMonsters } = await import('~/composables/useEncounterMonsters')
      const { removeMonster } = useEncounterMonsters('party-1')

      await removeMonster(1)

      expect(mockApiFetch).toHaveBeenCalledWith('/parties/party-1/monsters/1', {
        method: 'DELETE'
      })
    })
  })
})
```

**Step 2: Run test to verify it fails**

```bash
docker compose exec nuxt npm run test -- tests/composables/useEncounterMonsters.test.ts
```

Expected: FAIL - module not found

**Step 3: Write the composable**

```typescript
// app/composables/useEncounterMonsters.ts
import type { EncounterMonster } from '~/types/dm-screen'

export function useEncounterMonsters(partyId: string) {
  const { apiFetch } = useApi()

  const monsters = ref<EncounterMonster[]>([])
  const loading = ref(false)
  const error = ref<Error | null>(null)

  /**
   * Fetch all monsters in the party's encounter
   */
  async function fetchMonsters(): Promise<void> {
    loading.value = true
    error.value = null
    try {
      const response = await apiFetch<{ data: EncounterMonster[] }>(
        `/parties/${partyId}/monsters`
      )
      monsters.value = response.data
    } catch (e) {
      error.value = e as Error
    } finally {
      loading.value = false
    }
  }

  /**
   * Add monster(s) to the encounter
   */
  async function addMonster(monsterId: number, quantity: number = 1): Promise<EncounterMonster[]> {
    const response = await apiFetch<{ data: EncounterMonster[] }>(
      `/parties/${partyId}/monsters`,
      {
        method: 'POST',
        body: { monster_id: monsterId, quantity }
      }
    )
    await fetchMonsters() // Refresh list
    return response.data
  }

  /**
   * Update monster HP (debounced in component)
   */
  async function updateMonsterHp(instanceId: number, currentHp: number): Promise<void> {
    await apiFetch(`/parties/${partyId}/monsters/${instanceId}`, {
      method: 'PATCH',
      body: { current_hp: currentHp }
    })
    // Optimistic update already done in component
  }

  /**
   * Update monster label
   */
  async function updateMonsterLabel(instanceId: number, label: string): Promise<void> {
    await apiFetch(`/parties/${partyId}/monsters/${instanceId}`, {
      method: 'PATCH',
      body: { label }
    })
    const monster = monsters.value.find(m => m.id === instanceId)
    if (monster) monster.label = label
  }

  /**
   * Remove a monster instance
   */
  async function removeMonster(instanceId: number): Promise<void> {
    await apiFetch(`/parties/${partyId}/monsters/${instanceId}`, {
      method: 'DELETE'
    })
    monsters.value = monsters.value.filter(m => m.id !== instanceId)
  }

  /**
   * Clear all monsters (end encounter)
   */
  async function clearEncounter(): Promise<void> {
    await apiFetch(`/parties/${partyId}/monsters`, {
      method: 'DELETE'
    })
    monsters.value = []
  }

  return {
    monsters,
    loading,
    error,
    fetchMonsters,
    addMonster,
    updateMonsterHp,
    updateMonsterLabel,
    removeMonster,
    clearEncounter
  }
}
```

**Step 4: Run tests**

```bash
docker compose exec nuxt npm run test -- tests/composables/useEncounterMonsters.test.ts
```

Expected: PASS

**Step 5: Commit**

```bash
git add app/composables/useEncounterMonsters.ts tests/composables/useEncounterMonsters.test.ts
git commit -m "feat(dm-screen): add useEncounterMonsters composable"
```

---

## Phase 2: UI Components (requires backend for full testing)

### Task 4: Create AddMonsterModal Component

**Files:**
- Create: `app/components/dm-screen/AddMonsterModal.vue`
- Create: `tests/components/dm-screen/AddMonsterModal.test.ts`

**Step 1: Write tests**

```typescript
// tests/components/dm-screen/AddMonsterModal.test.ts
import { describe, it, expect, vi } from 'vitest'
import { mountSuspended, mockNuxtImport } from '@nuxt/test-utils/runtime'
import AddMonsterModal from '~/components/dm-screen/AddMonsterModal.vue'

const mockApiFetch = vi.fn()
mockNuxtImport('useApi', () => () => ({ apiFetch: mockApiFetch }))

const mockMonsters = [
  { id: 1, name: 'Goblin', slug: 'mm:goblin', challenge_rating: '1/4', hit_points: { average: 7 }, armor_class: 15 },
  { id: 2, name: 'Bugbear', slug: 'mm:bugbear', challenge_rating: '1', hit_points: { average: 27 }, armor_class: 16 }
]

describe('AddMonsterModal', () => {
  beforeEach(() => {
    mockApiFetch.mockResolvedValue({ data: mockMonsters })
  })

  it('renders search input when open', async () => {
    const wrapper = await mountSuspended(AddMonsterModal, {
      props: { open: true }
    })
    expect(wrapper.find('input[placeholder*="Search"]').exists()).toBe(true)
  })

  it('displays monster results', async () => {
    const wrapper = await mountSuspended(AddMonsterModal, {
      props: { open: true }
    })
    await wrapper.find('input').setValue('goblin')
    await flushPromises()
    expect(wrapper.text()).toContain('Goblin')
  })

  it('emits add event with monster id and quantity', async () => {
    const wrapper = await mountSuspended(AddMonsterModal, {
      props: { open: true }
    })
    // Select monster and set quantity, then add
    // ... test interaction
    expect(wrapper.emitted('add')).toBeTruthy()
  })

  it('closes on cancel', async () => {
    const wrapper = await mountSuspended(AddMonsterModal, {
      props: { open: true }
    })
    await wrapper.find('[data-testid="cancel-btn"]').trigger('click')
    expect(wrapper.emitted('update:open')?.[0]).toEqual([false])
  })
})
```

**Step 2: Run tests to verify failure**

```bash
docker compose exec nuxt npm run test -- tests/components/dm-screen/AddMonsterModal.test.ts
```

**Step 3: Create the component**

```vue
<!-- app/components/dm-screen/AddMonsterModal.vue -->
<script setup lang="ts">
import type { Monster } from '~/types'
import { useDebounceFn } from '@vueuse/core'

const props = defineProps<{
  open: boolean
  loading?: boolean
}>()

const emit = defineEmits<{
  'update:open': [value: boolean]
  'add': [monsterId: number, quantity: number]
}>()

const { apiFetch } = useApi()

const searchQuery = ref('')
const searchResults = ref<Monster[]>([])
const searching = ref(false)
const selectedMonster = ref<Monster | null>(null)
const quantity = ref(1)

// Debounced search
const debouncedSearch = useDebounceFn(async (query: string) => {
  if (!query.trim()) {
    searchResults.value = []
    return
  }
  searching.value = true
  try {
    const response = await apiFetch<{ data: Monster[] }>(
      `/monsters?q=${encodeURIComponent(query)}&per_page=10`
    )
    searchResults.value = response.data
  } catch {
    searchResults.value = []
  } finally {
    searching.value = false
  }
}, 300)

watch(searchQuery, (query) => {
  debouncedSearch(query)
})

function selectMonster(monster: Monster) {
  selectedMonster.value = monster
}

function handleAdd() {
  if (!selectedMonster.value) return
  emit('add', selectedMonster.value.id, quantity.value)
  handleClose()
}

function handleClose() {
  emit('update:open', false)
}

// Reset state when modal opens
watch(() => props.open, (isOpen) => {
  if (isOpen) {
    searchQuery.value = ''
    searchResults.value = []
    selectedMonster.value = null
    quantity.value = 1
  }
})
</script>

<template>
  <UModal
    :open="open"
    @update:open="emit('update:open', $event)"
  >
    <template #header>
      <h3 class="text-lg font-semibold">
        Add Monster to Encounter
      </h3>
    </template>

    <template #body>
      <div class="space-y-4">
        <!-- Search Input -->
        <UInput
          v-model="searchQuery"
          placeholder="Search monsters..."
          icon="i-heroicons-magnifying-glass"
          :loading="searching"
        />

        <!-- Selected Monster -->
        <div
          v-if="selectedMonster"
          class="p-3 rounded-lg border-2 border-primary-500 bg-primary-50 dark:bg-primary-900/20"
        >
          <div class="flex justify-between items-start">
            <div>
              <div class="font-medium">{{ selectedMonster.name }}</div>
              <div class="text-sm text-neutral-500">
                CR {{ selectedMonster.challenge_rating }} ·
                HP {{ selectedMonster.hit_points?.average }} ·
                AC {{ selectedMonster.armor_class }}
              </div>
            </div>
            <UButton
              icon="i-heroicons-x-mark"
              variant="ghost"
              size="xs"
              @click="selectedMonster = null"
            />
          </div>

          <!-- Quantity -->
          <div class="mt-3 flex items-center gap-3">
            <span class="text-sm text-neutral-600 dark:text-neutral-400">Quantity:</span>
            <UButtonGroup>
              <UButton
                icon="i-heroicons-minus"
                size="xs"
                variant="soft"
                :disabled="quantity <= 1"
                @click="quantity = Math.max(1, quantity - 1)"
              />
              <UInput
                v-model.number="quantity"
                type="number"
                class="w-16 text-center"
                :min="1"
                :max="20"
              />
              <UButton
                icon="i-heroicons-plus"
                size="xs"
                variant="soft"
                :disabled="quantity >= 20"
                @click="quantity = Math.min(20, quantity + 1)"
              />
            </UButtonGroup>
          </div>
        </div>

        <!-- Search Results -->
        <div
          v-else-if="searchResults.length > 0"
          class="space-y-2 max-h-64 overflow-y-auto"
        >
          <div
            v-for="monster in searchResults"
            :key="monster.id"
            class="p-3 rounded-lg border border-neutral-200 dark:border-neutral-700 cursor-pointer hover:border-primary-500 transition-colors"
            @click="selectMonster(monster)"
          >
            <div class="font-medium">{{ monster.name }}</div>
            <div class="text-sm text-neutral-500">
              CR {{ monster.challenge_rating }} ·
              HP {{ monster.hit_points?.average }} ·
              AC {{ monster.armor_class }}
            </div>
          </div>
        </div>

        <!-- Empty State -->
        <div
          v-else-if="searchQuery && !searching"
          class="text-center py-8 text-neutral-500"
        >
          <p>No monsters found</p>
        </div>

        <div
          v-else
          class="text-center py-8 text-neutral-500"
        >
          <UIcon
            name="i-heroicons-magnifying-glass"
            class="w-8 h-8 mx-auto mb-2 opacity-50"
          />
          <p>Search for a monster to add</p>
        </div>
      </div>
    </template>

    <template #footer>
      <div class="flex justify-end gap-3">
        <UButton
          data-testid="cancel-btn"
          color="neutral"
          variant="ghost"
          @click="handleClose"
        >
          Cancel
        </UButton>
        <UButton
          color="primary"
          :disabled="!selectedMonster"
          :loading="loading"
          @click="handleAdd"
        >
          Add {{ quantity > 1 ? `${quantity} Monsters` : 'Monster' }}
        </UButton>
      </div>
    </template>
  </UModal>
</template>
```

**Step 4: Run tests**

```bash
docker compose exec nuxt npm run test -- tests/components/dm-screen/AddMonsterModal.test.ts
```

**Step 5: Commit**

```bash
git add app/components/dm-screen/AddMonsterModal.vue tests/components/dm-screen/AddMonsterModal.test.ts
git commit -m "feat(dm-screen): add AddMonsterModal component"
```

---

### Task 5: Create MonsterTableRow Component

**Files:**
- Create: `app/components/dm-screen/MonsterTableRow.vue`
- Create: `tests/components/dm-screen/MonsterTableRow.test.ts`

**Step 1: Write tests**

```typescript
// tests/components/dm-screen/MonsterTableRow.test.ts
import { describe, it, expect } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import MonsterTableRow from '~/components/dm-screen/MonsterTableRow.vue'

const mockMonster = {
  id: 1,
  monster_id: 42,
  label: 'Goblin 1',
  current_hp: 5,
  max_hp: 7,
  monster: {
    name: 'Goblin',
    slug: 'mm:goblin',
    armor_class: 15,
    hit_points: { average: 7, formula: '2d6' },
    speed: { walk: 30, fly: null, swim: null, climb: null },
    challenge_rating: '1/4',
    actions: [
      { name: 'Scimitar', attack_bonus: 4, damage: '1d6+2 slashing', reach: '5 ft.', range: null }
    ]
  }
}

const mockCombatState = {
  initiatives: {},
  currentTurnId: null,
  round: 1,
  inCombat: false
}

describe('MonsterTableRow', () => {
  it('displays monster label', async () => {
    const wrapper = await mountSuspended(MonsterTableRow, {
      props: { monster: mockMonster, combatState: mockCombatState }
    })
    expect(wrapper.text()).toContain('Goblin 1')
  })

  it('displays HP bar', async () => {
    const wrapper = await mountSuspended(MonsterTableRow, {
      props: { monster: mockMonster, combatState: mockCombatState }
    })
    expect(wrapper.find('[data-testid="hp-bar"]').exists()).toBe(true)
  })

  it('displays AC', async () => {
    const wrapper = await mountSuspended(MonsterTableRow, {
      props: { monster: mockMonster, combatState: mockCombatState }
    })
    expect(wrapper.text()).toContain('15')
  })

  it('has monster visual styling (not character)', async () => {
    const wrapper = await mountSuspended(MonsterTableRow, {
      props: { monster: mockMonster, combatState: mockCombatState }
    })
    expect(wrapper.find('[data-testid="monster-row"]').classes()).toContain('border-l-red-500')
  })

  it('emits update:hp when HP changed', async () => {
    const wrapper = await mountSuspended(MonsterTableRow, {
      props: { monster: mockMonster, combatState: mockCombatState }
    })
    // Trigger HP edit
    // ...
    expect(wrapper.emitted('update:hp')).toBeTruthy()
  })

  it('emits remove when trash clicked', async () => {
    const wrapper = await mountSuspended(MonsterTableRow, {
      props: { monster: mockMonster, combatState: mockCombatState }
    })
    await wrapper.find('[data-testid="remove-btn"]').trigger('click')
    expect(wrapper.emitted('remove')).toBeTruthy()
  })
})
```

**Step 2: Run tests to verify failure**

```bash
docker compose exec nuxt npm run test -- tests/components/dm-screen/MonsterTableRow.test.ts
```

**Step 3: Create component**

```vue
<!-- app/components/dm-screen/MonsterTableRow.vue -->
<script setup lang="ts">
import type { EncounterMonster } from '~/types/dm-screen'

interface CombatState {
  initiatives: Record<string, number>
  currentTurnId: string | null
  round: number
  inCombat: boolean
}

interface Props {
  monster: EncounterMonster
  combatState: CombatState
  expanded?: boolean
  initiative?: number | null
}

const props = withDefaults(defineProps<Props>(), {
  expanded: false,
  initiative: null
})

const emit = defineEmits<{
  toggle: []
  'update:hp': [value: number]
  'update:label': [value: string]
  'update:initiative': [value: number]
  remove: []
}>()

// Initiative editing (same pattern as CombatTableRow)
const isEditingInit = ref(false)
const editValue = ref('')
const initInput = ref<HTMLInputElement | null>(null)

// HP editing
const isEditingHp = ref(false)
const hpValue = ref('')
const hpInput = ref<HTMLInputElement | null>(null)

// Label editing
const isEditingLabel = ref(false)
const labelValue = ref('')
const labelInput = ref<HTMLInputElement | null>(null)

const monsterKey = computed(() => `monster_${props.monster.id}`)
const isCurrentTurn = computed(() =>
  props.combatState.inCombat && props.combatState.currentTurnId === monsterKey.value
)

const initiativeDisplay = computed(() => {
  if (props.initiative !== null) return props.initiative.toString()
  return '—'
})

// HP percentage for bar
const hpPercent = computed(() => {
  if (props.monster.max_hp === 0) return 0
  return Math.max(0, Math.min(100, (props.monster.current_hp / props.monster.max_hp) * 100))
})

const hpColor = computed(() => {
  if (hpPercent.value > 50) return 'bg-green-500'
  if (hpPercent.value > 25) return 'bg-yellow-500'
  return 'bg-red-500'
})

function startEditInit(event: Event) {
  event.stopPropagation()
  isEditingInit.value = true
  editValue.value = props.initiative?.toString() ?? ''
  nextTick(() => {
    initInput.value?.focus()
    initInput.value?.select()
  })
}

function saveInit() {
  const value = parseInt(editValue.value, 10)
  if (!isNaN(value) && value >= -10 && value <= 50) {
    emit('update:initiative', value)
  }
  isEditingInit.value = false
}

function startEditHp(event: Event) {
  event.stopPropagation()
  isEditingHp.value = true
  hpValue.value = props.monster.current_hp.toString()
  nextTick(() => {
    hpInput.value?.focus()
    hpInput.value?.select()
  })
}

function saveHp() {
  const value = parseInt(hpValue.value, 10)
  if (!isNaN(value) && value >= 0 && value <= props.monster.max_hp * 2) {
    emit('update:hp', value)
  }
  isEditingHp.value = false
}

function handleRemove(event: Event) {
  event.stopPropagation()
  emit('remove')
}
</script>

<template>
  <tr
    data-testid="monster-row"
    class="cursor-pointer transition-colors border-l-4 border-l-red-500"
    :class="{
      'bg-neutral-50 dark:bg-neutral-800': expanded && !isCurrentTurn,
      'bg-red-50 dark:bg-red-950 border-l-red-600': isCurrentTurn,
      'hover:bg-neutral-50 dark:hover:bg-neutral-800': !isCurrentTurn
    }"
    @click="emit('toggle')"
  >
    <!-- Label + CR -->
    <td class="py-3 px-4">
      <div class="flex items-center gap-2">
        <span
          v-if="isCurrentTurn"
          class="text-red-500 font-bold"
        >▶</span>
        <div>
          <div class="font-medium text-neutral-900 dark:text-white flex items-center gap-2">
            {{ monster.label }}
            <UBadge color="red" variant="subtle" size="sm">
              CR {{ monster.monster.challenge_rating }}
            </UBadge>
          </div>
          <div class="text-xs text-neutral-500">
            {{ monster.monster.name }}
          </div>
        </div>
      </div>
    </td>

    <!-- HP Bar (editable) -->
    <td
      class="py-3 px-4 min-w-[180px]"
      @click.stop
    >
      <div
        v-if="isEditingHp"
        class="flex items-center gap-2"
      >
        <input
          ref="hpInput"
          v-model="hpValue"
          type="number"
          class="w-16 px-2 py-1 text-center font-mono text-sm rounded border"
          @blur="saveHp"
          @keydown.enter="saveHp"
          @keydown.escape="isEditingHp = false"
        >
        <span class="text-neutral-500">/ {{ monster.max_hp }}</span>
      </div>
      <button
        v-else
        data-testid="hp-bar"
        class="w-full text-left"
        @click="startEditHp"
      >
        <div class="flex items-center gap-2">
          <div class="flex-1 h-4 bg-neutral-200 dark:bg-neutral-700 rounded overflow-hidden">
            <div
              class="h-full transition-all"
              :class="hpColor"
              :style="{ width: `${hpPercent}%` }"
            />
          </div>
          <span class="text-sm font-mono w-16 text-right">
            {{ monster.current_hp }}/{{ monster.max_hp }}
          </span>
        </div>
      </button>
    </td>

    <!-- AC -->
    <td class="py-3 px-4 text-center">
      <UBadge color="red" variant="subtle" size="lg" class="font-mono font-bold">
        {{ monster.monster.armor_class }}
      </UBadge>
    </td>

    <!-- Initiative -->
    <td
      class="py-3 px-4 text-center"
      @click.stop
    >
      <input
        v-if="isEditingInit"
        ref="initInput"
        v-model="editValue"
        type="number"
        class="w-16 px-2 py-1 text-center font-mono text-sm rounded border"
        @blur="saveInit"
        @keydown.enter="saveInit"
        @keydown.escape="isEditingInit = false"
      >
      <button
        v-else
        class="px-2 py-1 rounded hover:bg-neutral-100 dark:hover:bg-neutral-700"
        :aria-label="`Edit initiative for ${monster.label}`"
        @click="startEditInit"
      >
        <span class="font-mono font-medium">{{ initiativeDisplay }}</span>
      </button>
    </td>

    <!-- Actions (condensed) -->
    <td class="py-3 px-4 text-sm text-neutral-600 dark:text-neutral-400">
      <div
        v-for="action in monster.monster.actions.slice(0, 2)"
        :key="action.name"
        class="truncate"
      >
        {{ action.name }}: +{{ action.attack_bonus }} ({{ action.damage }})
      </div>
    </td>

    <!-- Remove -->
    <td
      class="py-3 px-4 text-center"
      @click.stop
    >
      <UButton
        data-testid="remove-btn"
        icon="i-heroicons-trash"
        color="red"
        variant="ghost"
        size="xs"
        @click="handleRemove"
      />
    </td>
  </tr>
</template>
```

**Step 4: Run tests**

```bash
docker compose exec nuxt npm run test -- tests/components/dm-screen/MonsterTableRow.test.ts
```

**Step 5: Commit**

```bash
git add app/components/dm-screen/MonsterTableRow.vue tests/components/dm-screen/MonsterTableRow.test.ts
git commit -m "feat(dm-screen): add MonsterTableRow component"
```

---

### Task 6: Integrate Monsters into CombatTable

**Files:**
- Modify: `app/components/dm-screen/CombatTable.vue`
- Modify: `tests/components/dm-screen/CombatTable.test.ts`

**Step 1: Update CombatTable props**

Add to Props interface:

```typescript
interface Props {
  characters: DmScreenCharacter[]
  combatState: CombatState
  monsters?: EncounterMonster[]  // NEW
}
```

**Step 2: Update sorting to include monsters**

Replace `sortedCharacters` with `sortedCombatants`:

```typescript
const sortedCombatants = computed(() => {
  const combatants: Array<{ type: 'character' | 'monster'; key: string; data: DmScreenCharacter | EncounterMonster; init: number | null }> = []

  // Add characters
  for (const character of props.characters) {
    const key = `char_${character.id}`
    combatants.push({
      type: 'character',
      key,
      data: character,
      init: props.combatState.initiatives[key] ?? null
    })
  }

  // Add monsters
  for (const monster of props.monsters ?? []) {
    const key = `monster_${monster.id}`
    combatants.push({
      type: 'monster',
      key,
      data: monster,
      init: props.combatState.initiatives[key] ?? null
    })
  }

  // Sort: those with initiative first (descending), then without
  const withInit = combatants.filter(c => c.init !== null)
  const withoutInit = combatants.filter(c => c.init === null)

  withInit.sort((a, b) => {
    if (b.init !== a.init) return (b.init ?? 0) - (a.init ?? 0)
    // Tiebreaker by DEX modifier
    const modA = a.type === 'character'
      ? (a.data as DmScreenCharacter).combat.initiative_modifier
      : 0
    const modB = b.type === 'character'
      ? (b.data as DmScreenCharacter).combat.initiative_modifier
      : 0
    return modB - modA
  })

  return [...withInit, ...withoutInit]
})
```

**Step 3: Add "Add Monster" button to toolbar**

```vue
<UButton
  data-testid="add-monster-btn"
  icon="i-heroicons-plus"
  size="sm"
  variant="soft"
  color="red"
  @click="emit('addMonster')"
>
  Add Monster
</UButton>
```

**Step 4: Update template to render both character and monster rows**

```vue
<tbody
  v-for="combatant in sortedCombatants"
  :key="combatant.key"
>
  <!-- Character Row -->
  <DmScreenCombatTableRow
    v-if="combatant.type === 'character'"
    :character="combatant.data as DmScreenCharacter"
    :expanded="expandedKey === combatant.key"
    :is-current-turn="isCurrentTurn(combatant.key)"
    :initiative="combatant.init"
    @toggle="toggleExpand(combatant.key)"
    @update:initiative="(value) => emit('setInitiative', combatant.key, value)"
  />
  <!-- Monster Row -->
  <DmScreenMonsterTableRow
    v-else
    :monster="combatant.data as EncounterMonster"
    :combat-state="combatState"
    :expanded="expandedKey === combatant.key"
    :initiative="combatant.init"
    @toggle="toggleExpand(combatant.key)"
    @update:initiative="(value) => emit('setInitiative', combatant.key, value)"
    @update:hp="(value) => emit('updateMonsterHp', (combatant.data as EncounterMonster).id, value)"
    @remove="emit('removeMonster', (combatant.data as EncounterMonster).id)"
  />
  <!-- Expanded Detail -->
  <tr v-if="expandedKey === combatant.key">
    <td colspan="7">
      <DmScreenCharacterDetail
        v-if="combatant.type === 'character'"
        :character="combatant.data as DmScreenCharacter"
      />
      <DmScreenMonsterDetail
        v-else
        :monster="combatant.data as EncounterMonster"
      />
    </td>
  </tr>
</tbody>
```

**Step 5: Update tests**

Add test for monster rendering and mixed sorting.

**Step 6: Commit**

```bash
git add app/components/dm-screen/CombatTable.vue tests/components/dm-screen/CombatTable.test.ts
git commit -m "feat(dm-screen): integrate monsters into CombatTable"
```

---

### Task 7: Update dm-screen.vue Page

**Files:**
- Modify: `app/pages/parties/[id]/dm-screen.vue`

**Step 1: Add monster composable and state**

```typescript
// Add imports and composable
const encounterMonsters = useEncounterMonsters(partyId.value)

// Fetch monsters on mount
onMounted(async () => {
  await encounterMonsters.fetchMonsters()
})

// Modal state
const showAddMonsterModal = ref(false)
const addingMonster = ref(false)

// Handlers
async function handleAddMonster(monsterId: number, quantity: number) {
  addingMonster.value = true
  try {
    await encounterMonsters.addMonster(monsterId, quantity)
  } finally {
    addingMonster.value = false
    showAddMonsterModal.value = false
  }
}

async function handleUpdateMonsterHp(instanceId: number, hp: number) {
  // Optimistic update
  const monster = encounterMonsters.monsters.value.find(m => m.id === instanceId)
  if (monster) monster.current_hp = hp
  // Debounced sync
  await encounterMonsters.updateMonsterHp(instanceId, hp)
}

async function handleRemoveMonster(instanceId: number) {
  await encounterMonsters.removeMonster(instanceId)
}
```

**Step 2: Update template**

Pass monsters to CombatTable and add modal:

```vue
<DmScreenCombatTable
  :characters="stats.characters"
  :monsters="encounterMonsters.monsters.value"
  :combat-state="combatState"
  @add-monster="showAddMonsterModal = true"
  @update-monster-hp="handleUpdateMonsterHp"
  @remove-monster="handleRemoveMonster"
  ...other handlers
/>

<DmScreenAddMonsterModal
  v-model:open="showAddMonsterModal"
  :loading="addingMonster"
  @add="handleAddMonster"
/>
```

**Step 3: Commit**

```bash
git add app/pages/parties/\[id\]/dm-screen.vue
git commit -m "feat(dm-screen): wire up encounter monsters in page"
```

---

### Task 8: Update Combat Composable for String Keys

**Files:**
- Modify: `app/composables/useDmScreenCombat.ts`
- Modify: `tests/composables/useDmScreenCombat.test.ts`

**Step 1: Change initiative key type from number to string**

```typescript
interface CombatState {
  initiatives: Record<string, number>  // Changed from Record<number, number>
  currentTurnId: string | null         // Changed from number | null
  ...
}

function setInitiative(key: string, value: number): void {
  state.value.initiatives[key] = value
}

function getInitiative(key: string): number | null {
  return state.value.initiatives[key] ?? null
}
```

**Step 2: Update all functions to use string keys**

**Step 3: Update tests**

**Step 4: Commit**

```bash
git add app/composables/useDmScreenCombat.ts tests/composables/useDmScreenCombat.test.ts
git commit -m "refactor(dm-screen): change initiative keys from number to string"
```

---

## Phase 3: Polish

### Task 9: Create MonsterDetail Component

For expanded monster view with full stat block.

### Task 10: Add Clear Encounter Button

With confirmation modal.

### Task 11: Add HP Adjustment Buttons

+/- buttons for quick HP changes without typing.

---

## Summary

| Phase | Tasks | Blocked By |
|-------|-------|------------|
| Phase 1 | Types, Nitro routes, composable | Nothing |
| Phase 2 | UI components, page integration | Backend #610 |
| Phase 3 | Polish, stat block, HP buttons | Phase 2 |

**Estimated commits:** 11
**Key files created:** 8
**Key files modified:** 5
