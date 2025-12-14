# Spells Tab Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Create a dedicated Spells page for spellcasting characters with expandable spell cards and crystal slot icons.

**Architecture:** New page at `/characters/[publicId]/spells` following Features page pattern. New SpellCard component for expandable spell display. Update SpellSlots component with crystal icons.

**Tech Stack:** Nuxt 4 | Vue 3 | NuxtUI 4 | Vitest | TypeScript

---

## Task 1: Update SpellSlots Component with Crystal Icons

**Files:**
- Modify: `app/components/character/sheet/SpellSlots.vue`
- Test: `tests/components/character/sheet/SpellSlots.test.ts`

**Step 1: Update icon in SpellSlots.vue**

Replace `i-heroicons-circle-stack-solid` with `i-game-icons-crystal-shine` and increase size from `w-5 h-5` to `w-7 h-7`.

```vue
<!-- In SpellSlots.vue, replace both icon instances -->

<!-- Standard slots (around line 81-87) -->
<UIcon
  v-for="i in count"
  :key="`slot-${level}-${i}`"
  name="i-game-icons-crystal-shine"
  class="w-7 h-7 text-spell-500 dark:text-spell-400"
  :data-testid="`slot-${level}`"
/>

<!-- Pact slots (around line 110-115) -->
<UIcon
  v-for="i in pactSlots!.count"
  :key="`pact-slot-${i}`"
  name="i-game-icons-crystal-shine"
  class="w-7 h-7 text-spell-500 dark:text-spell-400"
  data-testid="pact-slot"
/>
```

**Step 2: Run existing tests to verify no regressions**

```bash
docker compose exec nuxt npm run test -- tests/components/character/sheet/SpellSlots.test.ts
```

Expected: All tests PASS (icon change doesn't affect test assertions)

**Step 3: Commit**

```bash
git add app/components/character/sheet/SpellSlots.vue
git commit -m "style: update spell slot icons to crystal design"
```

---

## Task 2: Create SpellCard Component - Test First

**Files:**
- Create: `tests/components/character/sheet/SpellCard.test.ts`
- Create: `app/components/character/sheet/SpellCard.vue`

**Step 1: Write failing tests for SpellCard**

```typescript
// tests/components/character/sheet/SpellCard.test.ts
import { describe, it, expect } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import SpellCard from '~/components/character/sheet/SpellCard.vue'
import type { CharacterSpell } from '~/types/character'

// Mock spell data
const mockCantrip: CharacterSpell = {
  id: 1,
  spell: {
    id: 101,
    name: 'Fire Bolt',
    slug: 'phb:fire-bolt',
    level: 0,
    school: 'Evocation',
    casting_time: '1 action',
    range: '120 feet',
    components: 'V, S',
    duration: 'Instantaneous',
    concentration: false,
    ritual: false
  },
  spell_slug: 'phb:fire-bolt',
  is_dangling: false,
  preparation_status: 'known',
  source: 'class',
  level_acquired: 1,
  is_prepared: false,
  is_always_prepared: false
}

const mockLeveledSpell: CharacterSpell = {
  id: 2,
  spell: {
    id: 102,
    name: 'Fireball',
    slug: 'phb:fireball',
    level: 3,
    school: 'Evocation',
    casting_time: '1 action',
    range: '150 feet',
    components: 'V, S, M',
    duration: 'Instantaneous',
    concentration: false,
    ritual: false
  },
  spell_slug: 'phb:fireball',
  is_dangling: false,
  preparation_status: 'prepared',
  source: 'class',
  level_acquired: 5,
  is_prepared: true,
  is_always_prepared: false
}

const mockConcentrationSpell: CharacterSpell = {
  id: 3,
  spell: {
    id: 103,
    name: 'Hold Person',
    slug: 'phb:hold-person',
    level: 2,
    school: 'Enchantment',
    casting_time: '1 action',
    range: '60 feet',
    components: 'V, S, M',
    duration: 'Concentration, up to 1 minute',
    concentration: true,
    ritual: false
  },
  spell_slug: 'phb:hold-person',
  is_dangling: false,
  preparation_status: 'prepared',
  source: 'class',
  level_acquired: 3,
  is_prepared: true,
  is_always_prepared: false
}

const mockRitualSpell: CharacterSpell = {
  id: 4,
  spell: {
    id: 104,
    name: 'Detect Magic',
    slug: 'phb:detect-magic',
    level: 1,
    school: 'Divination',
    casting_time: '1 action',
    range: 'Self',
    components: 'V, S',
    duration: 'Concentration, up to 10 minutes',
    concentration: true,
    ritual: true
  },
  spell_slug: 'phb:detect-magic',
  is_dangling: false,
  preparation_status: 'prepared',
  source: 'class',
  level_acquired: 1,
  is_prepared: true,
  is_always_prepared: false
}

const mockAlwaysPreparedSpell: CharacterSpell = {
  id: 5,
  spell: {
    id: 105,
    name: 'Cure Wounds',
    slug: 'phb:cure-wounds',
    level: 1,
    school: 'Evocation',
    casting_time: '1 action',
    range: 'Touch',
    components: 'V, S',
    duration: 'Instantaneous',
    concentration: false,
    ritual: false
  },
  spell_slug: 'phb:cure-wounds',
  is_dangling: false,
  preparation_status: 'always_prepared',
  source: 'class',
  level_acquired: 1,
  is_prepared: true,
  is_always_prepared: true
}

describe('SpellCard', () => {
  describe('collapsed state', () => {
    it('displays spell name', async () => {
      const wrapper = await mountSuspended(SpellCard, {
        props: { spell: mockLeveledSpell }
      })
      expect(wrapper.text()).toContain('Fireball')
    })

    it('displays spell level for leveled spells', async () => {
      const wrapper = await mountSuspended(SpellCard, {
        props: { spell: mockLeveledSpell }
      })
      expect(wrapper.text()).toContain('3rd')
    })

    it('displays "Cantrip" for level 0 spells', async () => {
      const wrapper = await mountSuspended(SpellCard, {
        props: { spell: mockCantrip }
      })
      expect(wrapper.text()).toContain('Cantrip')
    })

    it('shows Concentration badge when applicable', async () => {
      const wrapper = await mountSuspended(SpellCard, {
        props: { spell: mockConcentrationSpell }
      })
      expect(wrapper.text()).toContain('Concentration')
    })

    it('shows Ritual badge when applicable', async () => {
      const wrapper = await mountSuspended(SpellCard, {
        props: { spell: mockRitualSpell }
      })
      expect(wrapper.text()).toContain('Ritual')
    })

    it('shows prepared indicator for prepared spells', async () => {
      const wrapper = await mountSuspended(SpellCard, {
        props: { spell: mockLeveledSpell }
      })
      const preparedIcon = wrapper.find('[data-testid="prepared-icon"]')
      expect(preparedIcon.exists()).toBe(true)
    })

    it('shows always-prepared indicator', async () => {
      const wrapper = await mountSuspended(SpellCard, {
        props: { spell: mockAlwaysPreparedSpell }
      })
      expect(wrapper.text()).toContain('Always')
    })

    it('dims unprepared spells', async () => {
      const unpreparedSpell = { ...mockLeveledSpell, is_prepared: false, preparation_status: 'known' }
      const wrapper = await mountSuspended(SpellCard, {
        props: { spell: unpreparedSpell }
      })
      const card = wrapper.find('[data-testid="spell-card"]')
      expect(card.classes().join(' ')).toMatch(/opacity/)
    })
  })

  describe('expanded state', () => {
    it('starts collapsed by default', async () => {
      const wrapper = await mountSuspended(SpellCard, {
        props: { spell: mockLeveledSpell }
      })
      expect(wrapper.text()).not.toContain('Casting Time')
    })

    it('expands when clicked', async () => {
      const wrapper = await mountSuspended(SpellCard, {
        props: { spell: mockLeveledSpell }
      })

      await wrapper.find('[data-testid="spell-card"]').trigger('click')

      expect(wrapper.text()).toContain('Casting Time')
      expect(wrapper.text()).toContain('1 action')
    })

    it('shows range when expanded', async () => {
      const wrapper = await mountSuspended(SpellCard, {
        props: { spell: mockLeveledSpell }
      })

      await wrapper.find('[data-testid="spell-card"]').trigger('click')

      expect(wrapper.text()).toContain('Range')
      expect(wrapper.text()).toContain('150 feet')
    })

    it('shows components when expanded', async () => {
      const wrapper = await mountSuspended(SpellCard, {
        props: { spell: mockLeveledSpell }
      })

      await wrapper.find('[data-testid="spell-card"]').trigger('click')

      expect(wrapper.text()).toContain('Components')
      expect(wrapper.text()).toContain('V, S, M')
    })

    it('shows duration when expanded', async () => {
      const wrapper = await mountSuspended(SpellCard, {
        props: { spell: mockLeveledSpell }
      })

      await wrapper.find('[data-testid="spell-card"]').trigger('click')

      expect(wrapper.text()).toContain('Duration')
      expect(wrapper.text()).toContain('Instantaneous')
    })

    it('collapses when clicked again', async () => {
      const wrapper = await mountSuspended(SpellCard, {
        props: { spell: mockLeveledSpell }
      })

      const card = wrapper.find('[data-testid="spell-card"]')
      await card.trigger('click') // Expand
      await card.trigger('click') // Collapse

      expect(wrapper.text()).not.toContain('Casting Time')
    })
  })

  describe('school display', () => {
    it('shows spell school', async () => {
      const wrapper = await mountSuspended(SpellCard, {
        props: { spell: mockLeveledSpell }
      })
      expect(wrapper.text()).toContain('Evocation')
    })
  })
})
```

**Step 2: Run tests to verify they fail**

```bash
docker compose exec nuxt npm run test -- tests/components/character/sheet/SpellCard.test.ts
```

Expected: FAIL - component doesn't exist

**Step 3: Create minimal SpellCard component**

```vue
<!-- app/components/character/sheet/SpellCard.vue -->
<script setup lang="ts">
/**
 * Expandable Spell Card
 *
 * Displays a character's spell with collapsed/expanded states.
 * Collapsed: name, level, school, badges (concentration, ritual), prepared status
 * Expanded: adds casting time, range, components, duration
 *
 * @see Issue #556 - Spells Tab
 */

import type { CharacterSpell } from '~/types/character'

const props = defineProps<{
  spell: CharacterSpell
}>()

const isExpanded = ref(false)

/**
 * Toggle expanded state
 */
function toggle() {
  isExpanded.value = !isExpanded.value
}

/**
 * Format spell level as ordinal or "Cantrip"
 */
function formatLevel(level: number): string {
  if (level === 0) return 'Cantrip'
  const suffixes = ['th', 'st', 'nd', 'rd']
  const v = level % 100
  const suffix = (v - 20 >= 0 && v - 20 < 10 && suffixes[v - 20]) || suffixes[v] || 'th'
  return `${level}${suffix}`
}

/**
 * Check if spell is prepared (includes always prepared)
 */
const isPrepared = computed(() => props.spell.is_prepared || props.spell.is_always_prepared)

/**
 * Check if spell is always prepared (domain spells, etc.)
 */
const isAlwaysPrepared = computed(() => props.spell.is_always_prepared)

/**
 * Get spell data (handles null spell for dangling references)
 */
const spellData = computed(() => props.spell.spell)
</script>

<template>
  <div
    v-if="spellData"
    data-testid="spell-card"
    :class="[
      'p-3 rounded-lg border cursor-pointer transition-all',
      'bg-white dark:bg-gray-800',
      isPrepared
        ? 'border-spell-300 dark:border-spell-700'
        : 'border-gray-200 dark:border-gray-700 opacity-60',
      'hover:shadow-md'
    ]"
    @click="toggle"
  >
    <!-- Collapsed Header -->
    <div class="flex items-center justify-between gap-2">
      <div class="flex items-center gap-2 min-w-0">
        <!-- Prepared indicator -->
        <UIcon
          v-if="isPrepared"
          data-testid="prepared-icon"
          name="i-heroicons-check-circle-solid"
          :class="[
            'w-5 h-5 flex-shrink-0',
            isAlwaysPrepared
              ? 'text-amber-500 dark:text-amber-400'
              : 'text-spell-500 dark:text-spell-400'
          ]"
        />
        <UIcon
          v-else
          name="i-heroicons-circle"
          class="w-5 h-5 flex-shrink-0 text-gray-300 dark:text-gray-600"
        />

        <!-- Spell name -->
        <span class="font-medium truncate">{{ spellData.name }}</span>
      </div>

      <!-- Right side: badges and expand icon -->
      <div class="flex items-center gap-2 flex-shrink-0">
        <!-- Always prepared badge -->
        <UBadge
          v-if="isAlwaysPrepared"
          color="warning"
          variant="subtle"
          size="xs"
        >
          Always
        </UBadge>

        <!-- Concentration badge -->
        <UBadge
          v-if="spellData.concentration"
          color="spell"
          variant="subtle"
          size="xs"
        >
          Concentration
        </UBadge>

        <!-- Ritual badge -->
        <UBadge
          v-if="spellData.ritual"
          color="neutral"
          variant="subtle"
          size="xs"
        >
          Ritual
        </UBadge>

        <!-- Level/School -->
        <span class="text-sm text-gray-500 dark:text-gray-400">
          {{ formatLevel(spellData.level) }} {{ spellData.school }}
        </span>

        <!-- Expand/collapse icon -->
        <UIcon
          :name="isExpanded ? 'i-heroicons-chevron-up' : 'i-heroicons-chevron-down'"
          class="w-5 h-5 text-gray-400"
        />
      </div>
    </div>

    <!-- Expanded Details -->
    <div
      v-if="isExpanded"
      class="mt-3 pt-3 border-t border-gray-200 dark:border-gray-700 space-y-2 text-sm"
    >
      <div class="grid grid-cols-2 gap-2">
        <div>
          <span class="text-gray-500 dark:text-gray-400">Casting Time</span>
          <p class="font-medium">{{ spellData.casting_time }}</p>
        </div>
        <div>
          <span class="text-gray-500 dark:text-gray-400">Range</span>
          <p class="font-medium">{{ spellData.range }}</p>
        </div>
        <div>
          <span class="text-gray-500 dark:text-gray-400">Components</span>
          <p class="font-medium">{{ spellData.components }}</p>
        </div>
        <div>
          <span class="text-gray-500 dark:text-gray-400">Duration</span>
          <p class="font-medium">{{ spellData.duration }}</p>
        </div>
      </div>
    </div>
  </div>
</template>
```

**Step 4: Run tests to verify they pass**

```bash
docker compose exec nuxt npm run test -- tests/components/character/sheet/SpellCard.test.ts
```

Expected: All tests PASS

**Step 5: Commit**

```bash
git add tests/components/character/sheet/SpellCard.test.ts app/components/character/sheet/SpellCard.vue
git commit -m "feat: add expandable SpellCard component"
```

---

## Task 3: Create Spells Page - Test First

**Files:**
- Create: `tests/pages/characters/spells.test.ts`
- Create: `app/pages/characters/[publicId]/spells.vue`
- Create: `server/api/characters/[id]/spell-slots.get.ts`

**Step 1: Create Nitro route for spell-slots endpoint**

```typescript
// server/api/characters/[id]/spell-slots.get.ts
export default defineEventHandler(async (event) => {
  const config = useRuntimeConfig()
  const id = getRouterParam(event, 'id')

  const data = await $fetch(`${config.apiBaseServer}/characters/${id}/spell-slots`)
  return data
})
```

**Step 2: Write page tests**

```typescript
// tests/pages/characters/spells.test.ts
import { describe, it, expect, beforeEach, vi } from 'vitest'
import { mountSuspended, mockNuxtImport } from '@nuxt/test-utils/runtime'
import { setActivePinia, createPinia } from 'pinia'
import SpellsPage from '~/pages/characters/[publicId]/spells.vue'
import type { CharacterSpell } from '~/types/character'

// Mock route
mockNuxtImport('useRoute', () => () => ({
  params: { publicId: 'test-wizard-abc1' }
}))

// Mock character data
const mockCharacter = {
  id: 1,
  public_id: 'test-wizard-abc1',
  name: 'Gandalf',
  is_dead: false,
  death_save_successes: 0,
  death_save_failures: 0,
  currency: { pp: 0, gp: 100, ep: 0, sp: 50, cp: 20 }
}

// Mock stats with spellcasting
const mockStats = {
  spellcasting: {
    ability: 'INT',
    ability_modifier: 4,
    spell_save_dc: 15,
    spell_attack_bonus: 7
  },
  spell_slots: { 1: 4, 2: 3, 3: 2 },
  prepared_spell_count: 8,
  preparation_limit: 12,
  hit_points: { current: 30, max: 30, temporary: 0 }
}

// Mock spells
const mockSpells: CharacterSpell[] = [
  {
    id: 1,
    spell: {
      id: 101,
      name: 'Fire Bolt',
      slug: 'phb:fire-bolt',
      level: 0,
      school: 'Evocation',
      casting_time: '1 action',
      range: '120 feet',
      components: 'V, S',
      duration: 'Instantaneous',
      concentration: false,
      ritual: false
    },
    spell_slug: 'phb:fire-bolt',
    is_dangling: false,
    preparation_status: 'known',
    source: 'class',
    level_acquired: 1,
    is_prepared: false,
    is_always_prepared: false
  },
  {
    id: 2,
    spell: {
      id: 102,
      name: 'Magic Missile',
      slug: 'phb:magic-missile',
      level: 1,
      school: 'Evocation',
      casting_time: '1 action',
      range: '120 feet',
      components: 'V, S',
      duration: 'Instantaneous',
      concentration: false,
      ritual: false
    },
    spell_slug: 'phb:magic-missile',
    is_dangling: false,
    preparation_status: 'prepared',
    source: 'class',
    level_acquired: 1,
    is_prepared: true,
    is_always_prepared: false
  },
  {
    id: 3,
    spell: {
      id: 103,
      name: 'Fireball',
      slug: 'phb:fireball',
      level: 3,
      school: 'Evocation',
      casting_time: '1 action',
      range: '150 feet',
      components: 'V, S, M',
      duration: 'Instantaneous',
      concentration: false,
      ritual: false
    },
    spell_slug: 'phb:fireball',
    is_dangling: false,
    preparation_status: 'prepared',
    source: 'class',
    level_acquired: 5,
    is_prepared: true,
    is_always_prepared: false
  }
]

// Mock spell slots
const mockSpellSlots = {
  slots: {
    1: { total: 4, spent: 1, available: 3 },
    2: { total: 3, spent: 0, available: 3 },
    3: { total: 2, spent: 0, available: 2 }
  },
  pact_magic: null,
  preparation_limit: 12,
  prepared_count: 8
}

// Mock API
const apiFetchMock = vi.fn()
mockNuxtImport('useApi', () => () => ({
  apiFetch: apiFetchMock
}))

describe('SpellsPage', () => {
  beforeEach(() => {
    setActivePinia(createPinia())
    apiFetchMock.mockReset()

    // Setup default mocks
    apiFetchMock.mockImplementation((url: string) => {
      if (url.includes('/spells')) {
        return Promise.resolve({ data: mockSpells })
      }
      if (url.includes('/spell-slots')) {
        return Promise.resolve({ data: mockSpellSlots })
      }
      if (url.includes('/stats')) {
        return Promise.resolve({ data: mockStats })
      }
      return Promise.resolve({ data: mockCharacter })
    })
  })

  it('displays character name in page title', async () => {
    const wrapper = await mountSuspended(SpellsPage)
    // SEO meta is set, we can check component renders
    expect(wrapper.text()).toContain('Gandalf')
  })

  it('displays spellcasting stats', async () => {
    const wrapper = await mountSuspended(SpellsPage)

    expect(wrapper.text()).toContain('Spell DC')
    expect(wrapper.text()).toContain('15')
    expect(wrapper.text()).toContain('Attack')
    expect(wrapper.text()).toContain('+7')
  })

  it('displays preparation count', async () => {
    const wrapper = await mountSuspended(SpellsPage)

    expect(wrapper.text()).toContain('Prepared')
    expect(wrapper.text()).toContain('8')
    expect(wrapper.text()).toContain('12')
  })

  it('displays spell slots', async () => {
    const wrapper = await mountSuspended(SpellsPage)

    expect(wrapper.text()).toContain('1st')
    expect(wrapper.text()).toContain('2nd')
    expect(wrapper.text()).toContain('3rd')
  })

  it('separates cantrips from leveled spells', async () => {
    const wrapper = await mountSuspended(SpellsPage)

    expect(wrapper.text()).toContain('Cantrips')
    expect(wrapper.text()).toContain('Fire Bolt')
  })

  it('groups leveled spells by level', async () => {
    const wrapper = await mountSuspended(SpellsPage)

    expect(wrapper.text()).toContain('1st Level')
    expect(wrapper.text()).toContain('Magic Missile')
    expect(wrapper.text()).toContain('3rd Level')
    expect(wrapper.text()).toContain('Fireball')
  })

  it('shows empty state for non-spellcaster', async () => {
    apiFetchMock.mockImplementation((url: string) => {
      if (url.includes('/stats')) {
        return Promise.resolve({ data: { ...mockStats, spellcasting: null } })
      }
      if (url.includes('/spells')) {
        return Promise.resolve({ data: [] })
      }
      return Promise.resolve({ data: mockCharacter })
    })

    const wrapper = await mountSuspended(SpellsPage)

    expect(wrapper.text()).toContain('cannot cast spells')
  })
})
```

**Step 3: Run tests to verify they fail**

```bash
docker compose exec nuxt npm run test -- tests/pages/characters/spells.test.ts
```

Expected: FAIL - page doesn't exist

**Step 4: Create the Spells page**

```vue
<!-- app/pages/characters/[publicId]/spells.vue -->
<script setup lang="ts">
/**
 * Spells Page
 *
 * Dedicated page for character spell management.
 * Shows spellcasting stats, spell slots, and all known spells
 * grouped by level with expandable cards.
 *
 * Uses CharacterPageHeader for unified header with play mode, inspiration, etc.
 *
 * @see Issue #556 - Spells Tab implementation
 */

import type { Character, CharacterSpell } from '~/types/character'
import { useCharacterPlayStateStore } from '~/stores/characterPlayState'

const route = useRoute()
const publicId = computed(() => route.params.publicId as string)
const { apiFetch } = useApi()

// Play State Store
const playStateStore = useCharacterPlayStateStore()

// Fetch character data (needed for PageHeader)
const { data: characterData, pending: characterPending, refresh: refreshCharacter } = await useAsyncData(
  `spells-character-${publicId.value}`,
  () => apiFetch<{ data: Character }>(`/characters/${publicId.value}`)
)

// Fetch spells data
const { data: spellsData, pending: spellsPending } = await useAsyncData(
  `spells-data-${publicId.value}`,
  () => apiFetch<{ data: CharacterSpell[] }>(`/characters/${publicId.value}/spells`)
)

// Fetch stats for spellcasting info
interface StatsResponse {
  spellcasting?: {
    ability: string
    ability_modifier: number
    spell_save_dc: number
    spell_attack_bonus: number
  } | null
  spell_slots?: Record<string, number>
  prepared_spell_count?: number
  preparation_limit?: number | null
  hit_points?: { current: number | null, max: number | null, temporary?: number | null }
}
const { data: statsData, pending: statsPending } = await useAsyncData(
  `spells-stats-${publicId.value}`,
  () => apiFetch<{ data: StatsResponse }>(`/characters/${publicId.value}/stats`)
)

// Fetch spell slots for detailed tracking
interface SpellSlotsResponse {
  slots: Record<string, { total: number, spent: number, available: number }>
  pact_magic: { level: number, count: number } | null
  preparation_limit: number | null
  prepared_count: number
}
const { data: slotsData, pending: slotsPending } = await useAsyncData(
  `spells-slots-${publicId.value}`,
  () => apiFetch<{ data: SpellSlotsResponse }>(`/characters/${publicId.value}/spell-slots`)
)

// Track initial load vs refresh
const hasLoadedOnce = ref(false)
const loading = computed(() => {
  if (hasLoadedOnce.value) return false
  return characterPending.value || spellsPending.value || statsPending.value || slotsPending.value
})

watch(
  () => !characterPending.value && !spellsPending.value && !statsPending.value && !slotsPending.value,
  (allLoaded) => {
    if (allLoaded && !hasLoadedOnce.value) {
      hasLoadedOnce.value = true
    }
  },
  { immediate: true }
)

const character = computed(() => characterData.value?.data ?? null)
const stats = computed(() => statsData.value?.data ?? null)
const spellSlots = computed(() => slotsData.value?.data ?? null)
const isSpellcaster = computed(() => !!stats.value?.spellcasting)

// Filter and group spells
const validSpells = computed(() =>
  (spellsData.value?.data ?? []).filter(s => s.spell !== null)
)
const cantrips = computed(() =>
  validSpells.value.filter(s => s.spell!.level === 0)
)
const leveledSpells = computed(() =>
  validSpells.value.filter(s => s.spell!.level > 0)
)

// Group leveled spells by level
const spellsByLevel = computed(() => {
  const grouped: Record<number, CharacterSpell[]> = {}
  for (const spell of leveledSpells.value) {
    const level = spell.spell!.level
    if (!grouped[level]) grouped[level] = []
    grouped[level].push(spell)
  }
  // Sort by spell name within each level
  for (const level in grouped) {
    grouped[level].sort((a, b) => a.spell!.name.localeCompare(b.spell!.name))
  }
  return grouped
})

// Get sorted level keys
const sortedLevels = computed(() =>
  Object.keys(spellsByLevel.value).map(Number).sort((a, b) => a - b)
)

/**
 * Format spell level as ordinal
 */
function formatLevelOrdinal(level: number): string {
  const suffixes = ['th', 'st', 'nd', 'rd']
  const v = level % 100
  const suffix = (v - 20 >= 0 && v - 20 < 10 && suffixes[v - 20]) || suffixes[v] || 'th'
  return `${level}${suffix}`
}

/**
 * Format modifier with sign
 */
function formatModifier(value: number): string {
  return value >= 0 ? `+${value}` : `${value}`
}

// Initialize play state store when character and stats load
watch([character, statsData], ([char, s]) => {
  if (char && s?.data) {
    playStateStore.initialize({
      characterId: char.id,
      isDead: char.is_dead ?? false,
      hitPoints: {
        current: s.data.hit_points?.current ?? null,
        max: s.data.hit_points?.max ?? null,
        temporary: s.data.hit_points?.temporary ?? null
      },
      deathSaves: {
        successes: char.death_save_successes ?? 0,
        failures: char.death_save_failures ?? 0
      },
      currency: char.currency ?? { pp: 0, gp: 0, ep: 0, sp: 0, cp: 0 }
    })
  }
}, { immediate: true })

useSeoMeta({
  title: () => character.value ? `${character.value.name} - Spells` : 'Spells'
})
</script>

<template>
  <div class="container mx-auto px-4 py-8 max-w-5xl">
    <!-- Loading State -->
    <div
      v-if="loading"
      data-testid="loading-skeleton"
      class="space-y-4"
    >
      <USkeleton class="h-32 w-full" />
      <USkeleton class="h-24 w-full" />
      <USkeleton class="h-64 w-full" />
    </div>

    <!-- Main Content -->
    <template v-else-if="character">
      <!-- Unified Page Header -->
      <CharacterPageHeader
        :character="character"
        :is-spellcaster="isSpellcaster"
        :back-to="`/characters/${publicId}`"
        back-label="Back to Character"
        @updated="refreshCharacter"
      />

      <!-- Non-spellcaster message -->
      <div
        v-if="!isSpellcaster"
        class="mt-6 text-center py-12 text-gray-500 dark:text-gray-400"
      >
        <UIcon
          name="i-heroicons-x-circle"
          class="w-12 h-12 mx-auto mb-4"
        />
        <p class="text-lg">This character cannot cast spells.</p>
        <NuxtLink
          :to="`/characters/${publicId}`"
          class="text-primary-500 hover:underline mt-2 inline-block"
        >
          Return to character sheet
        </NuxtLink>
      </div>

      <!-- Spellcaster Content -->
      <template v-else>
        <!-- Spellcasting Stats Bar -->
        <div class="mt-6 p-4 bg-gray-50 dark:bg-gray-800 rounded-lg">
          <div class="flex flex-wrap gap-6 items-center justify-between">
            <!-- Stats -->
            <div class="flex gap-4 flex-wrap">
              <div class="text-center">
                <div class="text-xs text-gray-500 uppercase">Spell DC</div>
                <div class="text-2xl font-bold">{{ stats?.spellcasting?.spell_save_dc }}</div>
              </div>
              <div class="text-center">
                <div class="text-xs text-gray-500 uppercase">Attack</div>
                <div class="text-2xl font-bold">{{ formatModifier(stats?.spellcasting?.spell_attack_bonus ?? 0) }}</div>
              </div>
              <div class="text-center">
                <div class="text-xs text-gray-500 uppercase">Ability</div>
                <div class="text-2xl font-bold">{{ stats?.spellcasting?.ability }}</div>
              </div>
            </div>

            <!-- Preparation Counter -->
            <div
              v-if="spellSlots?.preparation_limit !== null"
              class="text-center"
            >
              <div class="text-xs text-gray-500 uppercase">Prepared</div>
              <div class="text-lg font-medium">
                {{ spellSlots?.prepared_count ?? 0 }} / {{ spellSlots?.preparation_limit }}
              </div>
            </div>
          </div>
        </div>

        <!-- Spell Slots -->
        <div
          v-if="spellSlots?.slots && Object.keys(spellSlots.slots).length > 0"
          class="mt-6"
        >
          <CharacterSheetSpellSlots
            :spell-slots="Object.fromEntries(
              Object.entries(spellSlots.slots).map(([k, v]) => [k, v.total])
            )"
            :pact-slots="spellSlots.pact_magic ? { count: spellSlots.pact_magic.count, level: spellSlots.pact_magic.level } : null"
          />
        </div>

        <!-- Cantrips Section -->
        <div
          v-if="cantrips.length > 0"
          class="mt-8"
        >
          <h3 class="text-lg font-semibold text-gray-700 dark:text-gray-300 mb-3">
            Cantrips
          </h3>
          <div class="space-y-2">
            <CharacterSheetSpellCard
              v-for="spell in cantrips"
              :key="spell.id"
              :spell="spell"
            />
          </div>
        </div>

        <!-- Leveled Spells by Level -->
        <div
          v-for="level in sortedLevels"
          :key="level"
          class="mt-8"
        >
          <h3 class="text-lg font-semibold text-gray-700 dark:text-gray-300 mb-3">
            {{ formatLevelOrdinal(level) }} Level
          </h3>
          <div class="space-y-2">
            <CharacterSheetSpellCard
              v-for="spell in spellsByLevel[level]"
              :key="spell.id"
              :spell="spell"
            />
          </div>
        </div>

        <!-- Empty State -->
        <div
          v-if="validSpells.length === 0"
          class="mt-8 text-center py-12 text-gray-500 dark:text-gray-400"
        >
          <UIcon
            name="i-heroicons-sparkles"
            class="w-12 h-12 mx-auto mb-4"
          />
          <p class="text-lg">No spells known yet.</p>
          <p class="text-sm mt-2">Learn spells through the character builder.</p>
        </div>
      </template>
    </template>
  </div>
</template>
```

**Step 5: Run tests to verify they pass**

```bash
docker compose exec nuxt npm run test -- tests/pages/characters/spells.test.ts
```

Expected: All tests PASS

**Step 6: Commit**

```bash
git add server/api/characters/[id]/spell-slots.get.ts tests/pages/characters/spells.test.ts app/pages/characters/[publicId]/spells.vue
git commit -m "feat: add Spells page with expandable cards"
```

---

## Task 4: Add Navigation Link to Spells Page

**Files:**
- Modify: `app/pages/characters/[publicId]/index.vue`

**Step 1: Add Spells link to character sheet navigation**

Find the navigation/tabs section in the character sheet and add a link to the spells page. The link should only show for spellcasters (using `isSpellcaster` computed).

Look for where Features and Notes links are added, and add Spells similarly:

```vue
<!-- Add to navigation section, near other sub-page links -->
<NuxtLink
  v-if="isSpellcaster"
  :to="`/characters/${publicId}/spells`"
  class="..."
>
  <UIcon name="i-heroicons-sparkles" class="w-5 h-5" />
  Spells
</NuxtLink>
```

**Step 2: Verify in browser**

1. Open a spellcaster character
2. Verify "Spells" link appears in navigation
3. Click link and verify spells page loads
4. Open a non-spellcaster character
5. Verify "Spells" link does not appear

**Step 3: Commit**

```bash
git add app/pages/characters/[publicId]/index.vue
git commit -m "feat: add Spells navigation link for spellcasters"
```

---

## Task 5: Run Full Test Suite and Create PR

**Step 1: Run character test suite**

```bash
docker compose exec nuxt npm run test:character
```

Expected: All tests PASS

**Step 2: Run typecheck**

```bash
docker compose exec nuxt npm run typecheck
```

Expected: No errors

**Step 3: Run lint**

```bash
docker compose exec nuxt npm run lint:fix
```

Expected: No errors (auto-fixed if any)

**Step 4: Manual browser testing**

1. Navigate to a spellcaster character
2. Click "Spells" in navigation
3. Verify:
   - Spellcasting stats display correctly (DC, Attack, Ability)
   - Preparation count shows (if applicable)
   - Spell slots display with crystal icons
   - Cantrips section shows cantrips
   - Leveled spells grouped by level
   - Click spell card to expand
   - Expanded view shows casting time, range, components, duration
   - Click again to collapse
   - Concentration badge shows for concentration spells
   - Ritual badge shows for ritual spells
   - Prepared/unprepared visual distinction works
4. Test dark mode

**Step 5: Create feature branch and PR**

```bash
git checkout -b feature/issue-556-spells-tab
git push -u origin feature/issue-556-spells-tab

gh pr create --title "feat: Character Sheet Spells Tab (#556)" --body "$(cat <<'EOF'
## Summary
- Add dedicated Spells page for spellcasting characters
- New expandable SpellCard component with collapsed/expanded states
- Update SpellSlots with crystal icons (w-7 size)
- Display spellcasting stats, preparation count, spell slots
- Group spells by level with cantrips separated

## Changes
- `app/pages/characters/[publicId]/spells.vue` - New spells page
- `app/components/character/sheet/SpellCard.vue` - Expandable card
- `app/components/character/sheet/SpellSlots.vue` - Crystal icons
- `server/api/characters/[id]/spell-slots.get.ts` - Nitro route

## Test plan
- [x] SpellCard tests pass
- [x] SpellSlots tests pass
- [x] Spells page tests pass
- [x] Character test suite passes
- [x] Manual testing: expand/collapse, badges, prepared states

Closes #556
EOF
)"
```

---

## Summary

| Task | Component | Tests |
|------|-----------|-------|
| 1 | SpellSlots crystal icons | Existing tests |
| 2 | SpellCard component | New test file |
| 3 | Spells page | New test file |
| 4 | Navigation link | Manual verification |
| 5 | PR | Full suite |

**Total new files:** 4
**Total test files:** 2 new
**Estimated commits:** 5
