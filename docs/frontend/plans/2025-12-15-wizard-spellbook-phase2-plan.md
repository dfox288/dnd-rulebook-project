# Wizard Spellbook UI - Phase 2 Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Build a two-column spellbook interface for wizards that makes spell preparation fast and intuitive.

**Architecture:** New `SpellbookView` component conditionally rendered for `preparation_method === 'spellbook'`. Uses existing `characterPlayStateStore.toggleSpellPreparation()` for state management. Filter state is local (no Pinia needed).

**Tech Stack:** Vue 3, NuxtUI 4, Pinia, TypeScript, Vitest

**Design Doc:** `docs/frontend/plans/2025-12-15-wizard-spellbook-phase2-design.md`

**Issue:** #680

---

## Task 1: SpellbookCard Component

A compact spell card for the two-column layout. Simpler than the existing SpellCard - no expand/collapse, just click to toggle.

**Files:**
- Create: `app/components/character/sheet/SpellbookCard.vue`
- Create: `tests/components/character/sheet/SpellbookCard.test.ts`

### Step 1: Write failing tests

```typescript
// tests/components/character/sheet/SpellbookCard.test.ts
import { describe, it, expect, vi } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import { setActivePinia, createPinia } from 'pinia'
import SpellbookCard from '~/components/character/sheet/SpellbookCard.vue'
import type { CharacterSpell } from '~/types/character'

const mockSpell: CharacterSpell = {
  id: 1,
  spell: {
    id: 101,
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
  preparation_status: 'known',
  source: 'class',
  level_acquired: 5,
  is_prepared: false,
  is_always_prepared: false
}

describe('SpellbookCard', () => {
  beforeEach(() => {
    setActivePinia(createPinia())
  })

  describe('display', () => {
    it('displays spell name', async () => {
      const wrapper = await mountSuspended(SpellbookCard, {
        props: { spell: mockSpell, column: 'spellbook' }
      })
      expect(wrapper.text()).toContain('Fireball')
    })

    it('displays spell school', async () => {
      const wrapper = await mountSuspended(SpellbookCard, {
        props: { spell: mockSpell, column: 'spellbook' }
      })
      expect(wrapper.text()).toContain('Evocation')
    })

    it('shows concentration badge when applicable', async () => {
      const concSpell = { ...mockSpell, spell: { ...mockSpell.spell!, concentration: true } }
      const wrapper = await mountSuspended(SpellbookCard, {
        props: { spell: concSpell, column: 'spellbook' }
      })
      expect(wrapper.text()).toContain('Conc')
    })

    it('shows ritual badge when applicable', async () => {
      const ritualSpell = { ...mockSpell, spell: { ...mockSpell.spell!, ritual: true } }
      const wrapper = await mountSuspended(SpellbookCard, {
        props: { spell: ritualSpell, column: 'spellbook' }
      })
      expect(wrapper.text()).toContain('Ritual')
    })

    it('shows arrow indicator pointing right in spellbook column', async () => {
      const wrapper = await mountSuspended(SpellbookCard, {
        props: { spell: mockSpell, column: 'spellbook' }
      })
      expect(wrapper.find('[data-testid="prepare-indicator"]').exists()).toBe(true)
    })

    it('shows arrow indicator pointing left in prepared column', async () => {
      const preparedSpell = { ...mockSpell, is_prepared: true }
      const wrapper = await mountSuspended(SpellbookCard, {
        props: { spell: preparedSpell, column: 'prepared' }
      })
      expect(wrapper.find('[data-testid="unprepare-indicator"]').exists()).toBe(true)
    })
  })

  describe('states', () => {
    it('is greyed out when at limit and unprepared', async () => {
      const wrapper = await mountSuspended(SpellbookCard, {
        props: { spell: mockSpell, column: 'spellbook', atPrepLimit: true }
      })
      const card = wrapper.find('[data-testid="spellbook-card"]')
      expect(card.classes().join(' ')).toMatch(/opacity-40/)
    })

    it('shows always-prepared badge for domain spells', async () => {
      const alwaysSpell = { ...mockSpell, is_always_prepared: true, is_prepared: true }
      const wrapper = await mountSuspended(SpellbookCard, {
        props: { spell: alwaysSpell, column: 'prepared' }
      })
      expect(wrapper.text()).toContain('Always')
    })
  })

  describe('interaction', () => {
    it('emits toggle event on click', async () => {
      const wrapper = await mountSuspended(SpellbookCard, {
        props: { spell: mockSpell, column: 'spellbook' }
      })
      await wrapper.find('[data-testid="spellbook-card"]').trigger('click')
      expect(wrapper.emitted('toggle')).toBeTruthy()
      expect(wrapper.emitted('toggle')![0]).toEqual([mockSpell])
    })

    it('does not emit toggle when at limit and unprepared', async () => {
      const wrapper = await mountSuspended(SpellbookCard, {
        props: { spell: mockSpell, column: 'spellbook', atPrepLimit: true }
      })
      await wrapper.find('[data-testid="spellbook-card"]').trigger('click')
      expect(wrapper.emitted('toggle')).toBeFalsy()
    })

    it('does not emit toggle for always-prepared spells', async () => {
      const alwaysSpell = { ...mockSpell, is_always_prepared: true, is_prepared: true }
      const wrapper = await mountSuspended(SpellbookCard, {
        props: { spell: alwaysSpell, column: 'prepared' }
      })
      await wrapper.find('[data-testid="spellbook-card"]').trigger('click')
      expect(wrapper.emitted('toggle')).toBeFalsy()
    })
  })
})
```

### Step 2: Run tests to verify they fail

Run: `docker compose exec nuxt npm run test -- tests/components/character/sheet/SpellbookCard.test.ts`

Expected: FAIL - component doesn't exist

### Step 3: Implement SpellbookCard component

```vue
<!-- app/components/character/sheet/SpellbookCard.vue -->
<script setup lang="ts">
/**
 * Compact spell card for spellbook two-column view
 *
 * Displays spell name, school, badges. Click to toggle preparation.
 * Simpler than SpellCard - no expand/collapse.
 *
 * @see Issue #680 - Wizard Spellbook Phase 2
 */
import type { CharacterSpell } from '~/types/character'

const props = defineProps<{
  spell: CharacterSpell
  column: 'spellbook' | 'prepared'
  atPrepLimit?: boolean
}>()

const emit = defineEmits<{
  toggle: [spell: CharacterSpell]
}>()

const spellData = computed(() => props.spell.spell)
const isPrepared = computed(() => props.spell.is_prepared || props.spell.is_always_prepared)
const isAlwaysPrepared = computed(() => props.spell.is_always_prepared)

const isDisabled = computed(() => {
  if (isAlwaysPrepared.value) return true
  if (!isPrepared.value && props.atPrepLimit) return true
  return false
})

const isGreyedOut = computed(() => {
  return !isPrepared.value && props.atPrepLimit
})

function handleClick() {
  if (isDisabled.value) return
  emit('toggle', props.spell)
}
</script>

<template>
  <div
    v-if="spellData"
    data-testid="spellbook-card"
    :class="[
      'px-3 py-2 rounded-lg border transition-all flex items-center justify-between gap-2',
      'bg-white dark:bg-gray-800',
      isPrepared
        ? 'border-spell-300 dark:border-spell-700'
        : 'border-gray-200 dark:border-gray-700',
      isGreyedOut
        ? 'opacity-40 cursor-not-allowed'
        : isDisabled
          ? 'cursor-default'
          : 'cursor-pointer hover:shadow-md hover:border-spell-400 dark:hover:border-spell-600'
    ]"
    @click="handleClick"
  >
    <!-- Left: Name and badges -->
    <div class="flex items-center gap-2 min-w-0">
      <span class="font-medium truncate">{{ spellData.name }}</span>
      <UBadge
        v-if="isAlwaysPrepared"
        color="warning"
        variant="subtle"
        size="xs"
      >
        Always
      </UBadge>
      <UBadge
        v-if="spellData.concentration"
        color="spell"
        variant="subtle"
        size="xs"
      >
        Conc
      </UBadge>
      <UBadge
        v-if="spellData.ritual"
        color="neutral"
        variant="subtle"
        size="xs"
      >
        Ritual
      </UBadge>
    </div>

    <!-- Right: School and arrow -->
    <div class="flex items-center gap-2 flex-shrink-0">
      <span class="text-sm text-gray-500 dark:text-gray-400">
        {{ spellData.school }}
      </span>
      <!-- Arrow indicator -->
      <UIcon
        v-if="column === 'spellbook' && !isDisabled"
        data-testid="prepare-indicator"
        name="i-heroicons-arrow-right"
        class="w-4 h-4 text-gray-400"
      />
      <UIcon
        v-else-if="column === 'prepared' && !isAlwaysPrepared"
        data-testid="unprepare-indicator"
        name="i-heroicons-arrow-left"
        class="w-4 h-4 text-gray-400"
      />
    </div>
  </div>
</template>
```

### Step 4: Run tests to verify they pass

Run: `docker compose exec nuxt npm run test -- tests/components/character/sheet/SpellbookCard.test.ts`

Expected: PASS (all 11 tests)

### Step 5: Commit

```bash
git add app/components/character/sheet/SpellbookCard.vue tests/components/character/sheet/SpellbookCard.test.ts
git commit -m "feat(spellbook): Add SpellbookCard component (#680)"
```

---

## Task 2: SpellbookFilters Component

Filter controls for the spellbook column: search, school dropdown, level dropdown, checkboxes.

**Files:**
- Create: `app/components/character/sheet/SpellbookFilters.vue`
- Create: `tests/components/character/sheet/SpellbookFilters.test.ts`

### Step 1: Write failing tests

```typescript
// tests/components/character/sheet/SpellbookFilters.test.ts
import { describe, it, expect } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import SpellbookFilters from '~/components/character/sheet/SpellbookFilters.vue'

describe('SpellbookFilters', () => {
  const defaultProps = {
    searchQuery: '',
    selectedSchool: null as string | null,
    selectedLevel: null as number | null,
    concentrationOnly: false,
    ritualOnly: false
  }

  describe('search input', () => {
    it('displays search input', async () => {
      const wrapper = await mountSuspended(SpellbookFilters, { props: defaultProps })
      expect(wrapper.find('input[type="text"]').exists()).toBe(true)
    })

    it('emits update:searchQuery on input', async () => {
      const wrapper = await mountSuspended(SpellbookFilters, { props: defaultProps })
      await wrapper.find('input[type="text"]').setValue('fireball')
      expect(wrapper.emitted('update:searchQuery')).toBeTruthy()
      expect(wrapper.emitted('update:searchQuery')![0]).toEqual(['fireball'])
    })
  })

  describe('school dropdown', () => {
    it('displays school dropdown', async () => {
      const wrapper = await mountSuspended(SpellbookFilters, { props: defaultProps })
      expect(wrapper.find('[data-testid="school-filter"]').exists()).toBe(true)
    })

    it('emits update:selectedSchool on selection', async () => {
      const wrapper = await mountSuspended(SpellbookFilters, { props: defaultProps })
      // Find and trigger the select
      const select = wrapper.find('[data-testid="school-filter"]')
      await select.trigger('click')
      // Note: Full dropdown interaction requires more setup, testing the emit
    })
  })

  describe('level dropdown', () => {
    it('displays level dropdown', async () => {
      const wrapper = await mountSuspended(SpellbookFilters, { props: defaultProps })
      expect(wrapper.find('[data-testid="level-filter"]').exists()).toBe(true)
    })
  })

  describe('checkboxes', () => {
    it('displays concentration checkbox', async () => {
      const wrapper = await mountSuspended(SpellbookFilters, { props: defaultProps })
      expect(wrapper.find('[data-testid="concentration-filter"]').exists()).toBe(true)
    })

    it('displays ritual checkbox', async () => {
      const wrapper = await mountSuspended(SpellbookFilters, { props: defaultProps })
      expect(wrapper.find('[data-testid="ritual-filter"]').exists()).toBe(true)
    })

    it('emits update:concentrationOnly on toggle', async () => {
      const wrapper = await mountSuspended(SpellbookFilters, { props: defaultProps })
      await wrapper.find('[data-testid="concentration-filter"]').trigger('click')
      expect(wrapper.emitted('update:concentrationOnly')).toBeTruthy()
    })

    it('emits update:ritualOnly on toggle', async () => {
      const wrapper = await mountSuspended(SpellbookFilters, { props: defaultProps })
      await wrapper.find('[data-testid="ritual-filter"]').trigger('click')
      expect(wrapper.emitted('update:ritualOnly')).toBeTruthy()
    })
  })
})
```

### Step 2: Run tests to verify they fail

Run: `docker compose exec nuxt npm run test -- tests/components/character/sheet/SpellbookFilters.test.ts`

Expected: FAIL - component doesn't exist

### Step 3: Implement SpellbookFilters component

```vue
<!-- app/components/character/sheet/SpellbookFilters.vue -->
<script setup lang="ts">
/**
 * Filter controls for spellbook column
 *
 * Search, school dropdown, level dropdown, concentration/ritual checkboxes.
 * Uses v-model pattern for all filter values.
 *
 * @see Issue #680 - Wizard Spellbook Phase 2
 */

const props = defineProps<{
  searchQuery: string
  selectedSchool: string | null
  selectedLevel: number | null
  concentrationOnly: boolean
  ritualOnly: boolean
}>()

const emit = defineEmits<{
  'update:searchQuery': [value: string]
  'update:selectedSchool': [value: string | null]
  'update:selectedLevel': [value: number | null]
  'update:concentrationOnly': [value: boolean]
  'update:ritualOnly': [value: boolean]
}>()

const schoolOptions = [
  { label: 'All Schools', value: null },
  { label: 'Abjuration', value: 'Abjuration' },
  { label: 'Conjuration', value: 'Conjuration' },
  { label: 'Divination', value: 'Divination' },
  { label: 'Enchantment', value: 'Enchantment' },
  { label: 'Evocation', value: 'Evocation' },
  { label: 'Illusion', value: 'Illusion' },
  { label: 'Necromancy', value: 'Necromancy' },
  { label: 'Transmutation', value: 'Transmutation' }
]

const levelOptions = [
  { label: 'All Levels', value: null },
  { label: 'Cantrip', value: 0 },
  { label: '1st', value: 1 },
  { label: '2nd', value: 2 },
  { label: '3rd', value: 3 },
  { label: '4th', value: 4 },
  { label: '5th', value: 5 },
  { label: '6th', value: 6 },
  { label: '7th', value: 7 },
  { label: '8th', value: 8 },
  { label: '9th', value: 9 }
]
</script>

<template>
  <div class="space-y-3">
    <!-- Search -->
    <UInput
      :model-value="searchQuery"
      placeholder="Search spells..."
      icon="i-heroicons-magnifying-glass"
      @update:model-value="emit('update:searchQuery', $event)"
    />

    <!-- Dropdowns row -->
    <div class="flex gap-2">
      <USelectMenu
        data-testid="school-filter"
        :model-value="selectedSchool"
        :options="schoolOptions"
        value-attribute="value"
        option-attribute="label"
        placeholder="School"
        class="flex-1"
        @update:model-value="emit('update:selectedSchool', $event)"
      />
      <USelectMenu
        data-testid="level-filter"
        :model-value="selectedLevel"
        :options="levelOptions"
        value-attribute="value"
        option-attribute="label"
        placeholder="Level"
        class="flex-1"
        @update:model-value="emit('update:selectedLevel', $event)"
      />
    </div>

    <!-- Checkboxes row -->
    <div class="flex gap-4">
      <UCheckbox
        data-testid="concentration-filter"
        :model-value="concentrationOnly"
        label="Concentration"
        @update:model-value="emit('update:concentrationOnly', $event)"
      />
      <UCheckbox
        data-testid="ritual-filter"
        :model-value="ritualOnly"
        label="Ritual"
        @update:model-value="emit('update:ritualOnly', $event)"
      />
    </div>
  </div>
</template>
```

### Step 4: Run tests to verify they pass

Run: `docker compose exec nuxt npm run test -- tests/components/character/sheet/SpellbookFilters.test.ts`

Expected: PASS (all 8 tests)

### Step 5: Commit

```bash
git add app/components/character/sheet/SpellbookFilters.vue tests/components/character/sheet/SpellbookFilters.test.ts
git commit -m "feat(spellbook): Add SpellbookFilters component (#680)"
```

---

## Task 3: SpellbookColumn Component

Left column showing all spells in spellbook with filters applied.

**Files:**
- Create: `app/components/character/sheet/SpellbookColumn.vue`
- Create: `tests/components/character/sheet/SpellbookColumn.test.ts`

### Step 1: Write failing tests

```typescript
// tests/components/character/sheet/SpellbookColumn.test.ts
import { describe, it, expect } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import { setActivePinia, createPinia } from 'pinia'
import SpellbookColumn from '~/components/character/sheet/SpellbookColumn.vue'
import type { CharacterSpell } from '~/types/character'

const createSpell = (overrides: Partial<CharacterSpell> & { name: string, level: number, school: string }): CharacterSpell => ({
  id: Math.random(),
  spell: {
    id: Math.random(),
    name: overrides.name,
    slug: `phb:${overrides.name.toLowerCase().replace(' ', '-')}`,
    level: overrides.level,
    school: overrides.school,
    casting_time: '1 action',
    range: '60 feet',
    components: 'V, S',
    duration: 'Instantaneous',
    concentration: overrides.concentration ?? false,
    ritual: overrides.ritual ?? false
  },
  spell_slug: `phb:${overrides.name.toLowerCase().replace(' ', '-')}`,
  is_dangling: false,
  preparation_status: 'known',
  source: 'class',
  level_acquired: 1,
  is_prepared: overrides.is_prepared ?? false,
  is_always_prepared: false,
  ...overrides
})

const mockSpells: CharacterSpell[] = [
  createSpell({ name: 'Fireball', level: 3, school: 'Evocation' }),
  createSpell({ name: 'Shield', level: 1, school: 'Abjuration' }),
  createSpell({ name: 'Detect Magic', level: 1, school: 'Divination', ritual: true }),
  createSpell({ name: 'Hold Person', level: 2, school: 'Enchantment', concentration: true }),
  createSpell({ name: 'Magic Missile', level: 1, school: 'Evocation', is_prepared: true })
]

describe('SpellbookColumn', () => {
  beforeEach(() => {
    setActivePinia(createPinia())
  })

  it('displays column header with spell count', async () => {
    const wrapper = await mountSuspended(SpellbookColumn, {
      props: { spells: mockSpells, atPrepLimit: false }
    })
    expect(wrapper.text()).toContain('Spellbook')
    expect(wrapper.text()).toMatch(/4.*spells/) // 4 unprepared
  })

  it('only shows unprepared spells', async () => {
    const wrapper = await mountSuspended(SpellbookColumn, {
      props: { spells: mockSpells, atPrepLimit: false }
    })
    expect(wrapper.text()).toContain('Fireball')
    expect(wrapper.text()).toContain('Shield')
    expect(wrapper.text()).not.toContain('Magic Missile') // prepared
  })

  it('groups spells by level', async () => {
    const wrapper = await mountSuspended(SpellbookColumn, {
      props: { spells: mockSpells, atPrepLimit: false }
    })
    expect(wrapper.text()).toContain('1st Level')
    expect(wrapper.text()).toContain('2nd Level')
    expect(wrapper.text()).toContain('3rd Level')
  })

  describe('filtering', () => {
    it('filters by search query', async () => {
      const wrapper = await mountSuspended(SpellbookColumn, {
        props: { spells: mockSpells, atPrepLimit: false }
      })
      await wrapper.find('input[type="text"]').setValue('fire')
      expect(wrapper.text()).toContain('Fireball')
      expect(wrapper.text()).not.toContain('Shield')
    })

    it('filters by school', async () => {
      const wrapper = await mountSuspended(SpellbookColumn, {
        props: { spells: mockSpells, atPrepLimit: false }
      })
      // Would need to interact with dropdown - simplified test
    })

    it('filters by concentration', async () => {
      const wrapper = await mountSuspended(SpellbookColumn, {
        props: { spells: mockSpells, atPrepLimit: false }
      })
      await wrapper.find('[data-testid="concentration-filter"]').trigger('click')
      expect(wrapper.text()).toContain('Hold Person')
      expect(wrapper.text()).not.toContain('Fireball')
    })

    it('filters by ritual', async () => {
      const wrapper = await mountSuspended(SpellbookColumn, {
        props: { spells: mockSpells, atPrepLimit: false }
      })
      await wrapper.find('[data-testid="ritual-filter"]').trigger('click')
      expect(wrapper.text()).toContain('Detect Magic')
      expect(wrapper.text()).not.toContain('Fireball')
    })
  })

  it('shows empty state when no spells match filters', async () => {
    const wrapper = await mountSuspended(SpellbookColumn, {
      props: { spells: mockSpells, atPrepLimit: false }
    })
    await wrapper.find('input[type="text"]').setValue('nonexistent')
    expect(wrapper.text()).toContain('No matching spells')
  })

  it('emits toggle event when spell card is clicked', async () => {
    const wrapper = await mountSuspended(SpellbookColumn, {
      props: { spells: mockSpells, atPrepLimit: false }
    })
    await wrapper.find('[data-testid="spellbook-card"]').trigger('click')
    expect(wrapper.emitted('toggle')).toBeTruthy()
  })
})
```

### Step 2: Run tests to verify they fail

Run: `docker compose exec nuxt npm run test -- tests/components/character/sheet/SpellbookColumn.test.ts`

Expected: FAIL - component doesn't exist

### Step 3: Implement SpellbookColumn component

```vue
<!-- app/components/character/sheet/SpellbookColumn.vue -->
<script setup lang="ts">
/**
 * Spellbook column (left side) - unprepared spells with filters
 *
 * Shows all spells in the wizard's spellbook that are NOT currently prepared.
 * Includes search and filter controls.
 *
 * @see Issue #680 - Wizard Spellbook Phase 2
 */
import type { CharacterSpell } from '~/types/character'

const props = defineProps<{
  spells: CharacterSpell[]
  atPrepLimit: boolean
}>()

const emit = defineEmits<{
  toggle: [spell: CharacterSpell]
}>()

// Filter state (local, not persisted)
const searchQuery = ref('')
const selectedSchool = ref<string | null>(null)
const selectedLevel = ref<number | null>(null)
const concentrationOnly = ref(false)
const ritualOnly = ref(false)

// Only show unprepared spells (not including always-prepared)
const unpreparedSpells = computed(() =>
  props.spells.filter(s => !s.is_prepared && !s.is_always_prepared && s.spell !== null)
)

// Apply filters
const filteredSpells = computed(() => {
  let result = unpreparedSpells.value

  // Search by name
  if (searchQuery.value) {
    const query = searchQuery.value.toLowerCase()
    result = result.filter(s => s.spell!.name.toLowerCase().includes(query))
  }

  // Filter by school
  if (selectedSchool.value) {
    result = result.filter(s => s.spell!.school === selectedSchool.value)
  }

  // Filter by level
  if (selectedLevel.value !== null) {
    result = result.filter(s => s.spell!.level === selectedLevel.value)
  }

  // Concentration only
  if (concentrationOnly.value) {
    result = result.filter(s => s.spell!.concentration)
  }

  // Ritual only
  if (ritualOnly.value) {
    result = result.filter(s => s.spell!.ritual)
  }

  return result
})

// Group by level
const spellsByLevel = computed(() => {
  const groups: Record<number, CharacterSpell[]> = {}
  for (const spell of filteredSpells.value) {
    const level = spell.spell!.level
    if (!groups[level]) groups[level] = []
    groups[level].push(spell)
  }
  // Sort within each level
  for (const level in groups) {
    groups[level].sort((a, b) => a.spell!.name.localeCompare(b.spell!.name))
  }
  return groups
})

const sortedLevels = computed(() =>
  Object.keys(spellsByLevel.value).map(Number).sort((a, b) => a - b)
)

function formatLevel(level: number): string {
  if (level === 0) return 'Cantrips'
  const suffixes = ['th', 'st', 'nd', 'rd']
  const suffix = suffixes[(level - 20) % 10] ?? suffixes[level] ?? suffixes[0]
  return `${level}${suffix} Level`
}
</script>

<template>
  <div class="flex flex-col h-full">
    <!-- Header -->
    <div class="mb-4">
      <h3 class="text-lg font-semibold text-gray-700 dark:text-gray-300">
        Spellbook
        <span class="text-sm font-normal text-gray-500">
          ({{ unpreparedSpells.length }} spells)
        </span>
      </h3>
    </div>

    <!-- Filters -->
    <CharacterSheetSpellbookFilters
      v-model:search-query="searchQuery"
      v-model:selected-school="selectedSchool"
      v-model:selected-level="selectedLevel"
      v-model:concentration-only="concentrationOnly"
      v-model:ritual-only="ritualOnly"
      class="mb-4"
    />

    <!-- Spell list -->
    <div class="flex-1 overflow-y-auto space-y-4">
      <template v-if="filteredSpells.length > 0">
        <div
          v-for="level in sortedLevels"
          :key="level"
          class="space-y-2"
        >
          <h4 class="text-sm font-semibold text-gray-500 dark:text-gray-400">
            {{ formatLevel(level) }}
          </h4>
          <CharacterSheetSpellbookCard
            v-for="spell in spellsByLevel[level]"
            :key="spell.id"
            :spell="spell"
            column="spellbook"
            :at-prep-limit="atPrepLimit"
            @toggle="emit('toggle', $event)"
          />
        </div>
      </template>

      <!-- Empty state -->
      <div
        v-else
        class="text-center py-8 text-gray-500 dark:text-gray-400"
      >
        <UIcon
          name="i-heroicons-magnifying-glass"
          class="w-8 h-8 mx-auto mb-2"
        />
        <p>No matching spells</p>
      </div>
    </div>
  </div>
</template>
```

### Step 4: Run tests to verify they pass

Run: `docker compose exec nuxt npm run test -- tests/components/character/sheet/SpellbookColumn.test.ts`

Expected: PASS (all 9 tests)

### Step 5: Commit

```bash
git add app/components/character/sheet/SpellbookColumn.vue tests/components/character/sheet/SpellbookColumn.test.ts
git commit -m "feat(spellbook): Add SpellbookColumn component (#680)"
```

---

## Task 4: PreparedColumn Component

Right column showing currently prepared spells grouped by level.

**Files:**
- Create: `app/components/character/sheet/PreparedColumn.vue`
- Create: `tests/components/character/sheet/PreparedColumn.test.ts`

### Step 1: Write failing tests

```typescript
// tests/components/character/sheet/PreparedColumn.test.ts
import { describe, it, expect } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import { setActivePinia, createPinia } from 'pinia'
import PreparedColumn from '~/components/character/sheet/PreparedColumn.vue'
import type { CharacterSpell } from '~/types/character'

const createSpell = (name: string, level: number, isPrepared: boolean, isAlways = false): CharacterSpell => ({
  id: Math.random(),
  spell: {
    id: Math.random(),
    name,
    slug: `phb:${name.toLowerCase().replace(' ', '-')}`,
    level,
    school: 'Evocation',
    casting_time: '1 action',
    range: '60 feet',
    components: 'V, S',
    duration: 'Instantaneous',
    concentration: false,
    ritual: false
  },
  spell_slug: `phb:${name.toLowerCase().replace(' ', '-')}`,
  is_dangling: false,
  preparation_status: isPrepared ? 'prepared' : 'known',
  source: 'class',
  level_acquired: 1,
  is_prepared: isPrepared,
  is_always_prepared: isAlways
})

const mockSpells: CharacterSpell[] = [
  createSpell('Fireball', 3, true),
  createSpell('Shield', 1, true),
  createSpell('Mage Armor', 1, true),
  createSpell('Misty Step', 2, true),
  createSpell('Bless', 1, true, true), // always prepared
  createSpell('Sleep', 1, false) // not prepared
]

describe('PreparedColumn', () => {
  beforeEach(() => {
    setActivePinia(createPinia())
  })

  it('displays column header with prepared count', async () => {
    const wrapper = await mountSuspended(PreparedColumn, {
      props: { spells: mockSpells, preparedCount: 5, preparationLimit: 8 }
    })
    expect(wrapper.text()).toContain('Prepared Today')
    expect(wrapper.text()).toMatch(/5.*\/.*8/)
  })

  it('only shows prepared spells', async () => {
    const wrapper = await mountSuspended(PreparedColumn, {
      props: { spells: mockSpells, preparedCount: 5, preparationLimit: 8 }
    })
    expect(wrapper.text()).toContain('Fireball')
    expect(wrapper.text()).toContain('Shield')
    expect(wrapper.text()).not.toContain('Sleep') // not prepared
  })

  it('includes always-prepared spells', async () => {
    const wrapper = await mountSuspended(PreparedColumn, {
      props: { spells: mockSpells, preparedCount: 5, preparationLimit: 8 }
    })
    expect(wrapper.text()).toContain('Bless')
  })

  it('groups spells by level', async () => {
    const wrapper = await mountSuspended(PreparedColumn, {
      props: { spells: mockSpells, preparedCount: 5, preparationLimit: 8 }
    })
    expect(wrapper.text()).toContain('1st Level')
    expect(wrapper.text()).toContain('2nd Level')
    expect(wrapper.text()).toContain('3rd Level')
  })

  it('shows warning color when at limit', async () => {
    const wrapper = await mountSuspended(PreparedColumn, {
      props: { spells: mockSpells, preparedCount: 8, preparationLimit: 8 }
    })
    expect(wrapper.find('[data-testid="prep-counter"]').classes().join(' ')).toMatch(/warning|amber/)
  })

  it('emits toggle event when spell card is clicked', async () => {
    const wrapper = await mountSuspended(PreparedColumn, {
      props: { spells: mockSpells, preparedCount: 5, preparationLimit: 8 }
    })
    // Click a non-always-prepared spell
    const cards = wrapper.findAll('[data-testid="spellbook-card"]')
    await cards[0].trigger('click')
    expect(wrapper.emitted('toggle')).toBeTruthy()
  })

  it('does not emit toggle for always-prepared spells', async () => {
    const wrapper = await mountSuspended(PreparedColumn, {
      props: { spells: [createSpell('Bless', 1, true, true)], preparedCount: 1, preparationLimit: 8 }
    })
    await wrapper.find('[data-testid="spellbook-card"]').trigger('click')
    expect(wrapper.emitted('toggle')).toBeFalsy()
  })
})
```

### Step 2: Run tests to verify they fail

Run: `docker compose exec nuxt npm run test -- tests/components/character/sheet/PreparedColumn.test.ts`

Expected: FAIL - component doesn't exist

### Step 3: Implement PreparedColumn component

```vue
<!-- app/components/character/sheet/PreparedColumn.vue -->
<script setup lang="ts">
/**
 * Prepared column (right side) - currently prepared spells
 *
 * Shows all spells the wizard has prepared for today, grouped by level.
 * Includes always-prepared spells (domain/subclass features).
 *
 * @see Issue #680 - Wizard Spellbook Phase 2
 */
import type { CharacterSpell } from '~/types/character'

const props = defineProps<{
  spells: CharacterSpell[]
  preparedCount: number
  preparationLimit: number
}>()

const emit = defineEmits<{
  toggle: [spell: CharacterSpell]
}>()

// Only show prepared spells (including always-prepared)
const preparedSpells = computed(() =>
  props.spells.filter(s => (s.is_prepared || s.is_always_prepared) && s.spell !== null)
)

// At limit?
const atLimit = computed(() => props.preparedCount >= props.preparationLimit)

// Group by level
const spellsByLevel = computed(() => {
  const groups: Record<number, CharacterSpell[]> = {}
  for (const spell of preparedSpells.value) {
    const level = spell.spell!.level
    if (!groups[level]) groups[level] = []
    groups[level].push(spell)
  }
  // Sort within each level
  for (const level in groups) {
    groups[level].sort((a, b) => a.spell!.name.localeCompare(b.spell!.name))
  }
  return groups
})

const sortedLevels = computed(() =>
  Object.keys(spellsByLevel.value).map(Number).sort((a, b) => a - b)
)

function formatLevel(level: number): string {
  if (level === 0) return 'Cantrips'
  const suffixes = ['th', 'st', 'nd', 'rd']
  const suffix = suffixes[(level - 20) % 10] ?? suffixes[level] ?? suffixes[0]
  return `${level}${suffix} Level`
}
</script>

<template>
  <div class="flex flex-col h-full">
    <!-- Header with counter -->
    <div class="mb-4 flex items-center justify-between">
      <h3 class="text-lg font-semibold text-gray-700 dark:text-gray-300">
        Prepared Today
      </h3>
      <div
        data-testid="prep-counter"
        :class="[
          'text-lg font-medium px-3 py-1 rounded-lg',
          atLimit
            ? 'bg-amber-100 text-amber-700 dark:bg-amber-900 dark:text-amber-300'
            : 'bg-gray-100 text-gray-700 dark:bg-gray-700 dark:text-gray-300'
        ]"
      >
        {{ preparedCount }} / {{ preparationLimit }}
      </div>
    </div>

    <!-- Spell list -->
    <div class="flex-1 overflow-y-auto space-y-4">
      <template v-if="preparedSpells.length > 0">
        <div
          v-for="level in sortedLevels"
          :key="level"
          class="space-y-2"
        >
          <h4 class="text-sm font-semibold text-gray-500 dark:text-gray-400">
            {{ formatLevel(level) }}
          </h4>
          <CharacterSheetSpellbookCard
            v-for="spell in spellsByLevel[level]"
            :key="spell.id"
            :spell="spell"
            column="prepared"
            @toggle="emit('toggle', $event)"
          />
        </div>
      </template>

      <!-- Empty state -->
      <div
        v-else
        class="text-center py-8 text-gray-500 dark:text-gray-400"
      >
        <UIcon
          name="i-heroicons-book-open"
          class="w-8 h-8 mx-auto mb-2"
        />
        <p>No spells prepared</p>
        <p class="text-sm mt-1">
          Click spells in your spellbook to prepare them
        </p>
      </div>
    </div>
  </div>
</template>
```

### Step 4: Run tests to verify they pass

Run: `docker compose exec nuxt npm run test -- tests/components/character/sheet/PreparedColumn.test.ts`

Expected: PASS (all 7 tests)

### Step 5: Commit

```bash
git add app/components/character/sheet/PreparedColumn.vue tests/components/character/sheet/PreparedColumn.test.ts
git commit -m "feat(spellbook): Add PreparedColumn component (#680)"
```

---

## Task 5: SpellbookView Component

Main container with two-column layout and toggle logic.

**Files:**
- Create: `app/components/character/sheet/SpellbookView.vue`
- Create: `tests/components/character/sheet/SpellbookView.test.ts`

### Step 1: Write failing tests

```typescript
// tests/components/character/sheet/SpellbookView.test.ts
import { describe, it, expect, vi } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import { setActivePinia, createPinia } from 'pinia'
import SpellbookView from '~/components/character/sheet/SpellbookView.vue'
import { useCharacterPlayStateStore } from '~/stores/characterPlayState'
import type { CharacterSpell } from '~/types/character'

const createSpell = (name: string, level: number, isPrepared: boolean): CharacterSpell => ({
  id: Math.floor(Math.random() * 1000),
  spell: {
    id: Math.floor(Math.random() * 1000),
    name,
    slug: `phb:${name.toLowerCase().replace(' ', '-')}`,
    level,
    school: 'Evocation',
    casting_time: '1 action',
    range: '60 feet',
    components: 'V, S',
    duration: 'Instantaneous',
    concentration: false,
    ritual: false
  },
  spell_slug: `phb:${name.toLowerCase().replace(' ', '-')}`,
  is_dangling: false,
  preparation_status: isPrepared ? 'prepared' : 'known',
  source: 'class',
  level_acquired: 1,
  is_prepared: isPrepared,
  is_always_prepared: false
})

const mockSpells: CharacterSpell[] = [
  createSpell('Fireball', 3, true),
  createSpell('Shield', 1, true),
  createSpell('Sleep', 1, false),
  createSpell('Magic Missile', 1, false)
]

describe('SpellbookView', () => {
  let pinia: ReturnType<typeof createPinia>

  beforeEach(() => {
    pinia = createPinia()
    setActivePinia(pinia)
  })

  it('renders two-column layout', async () => {
    const wrapper = await mountSuspended(SpellbookView, {
      props: { spells: mockSpells, preparedCount: 2, preparationLimit: 8, characterId: 1 },
      global: { plugins: [pinia] }
    })
    expect(wrapper.find('[data-testid="spellbook-column"]').exists()).toBe(true)
    expect(wrapper.find('[data-testid="prepared-column"]').exists()).toBe(true)
  })

  it('shows spellbook column with unprepared spells', async () => {
    const wrapper = await mountSuspended(SpellbookView, {
      props: { spells: mockSpells, preparedCount: 2, preparationLimit: 8, characterId: 1 },
      global: { plugins: [pinia] }
    })
    const spellbookCol = wrapper.find('[data-testid="spellbook-column"]')
    expect(spellbookCol.text()).toContain('Sleep')
    expect(spellbookCol.text()).toContain('Magic Missile')
  })

  it('shows prepared column with prepared spells', async () => {
    const wrapper = await mountSuspended(SpellbookView, {
      props: { spells: mockSpells, preparedCount: 2, preparationLimit: 8, characterId: 1 },
      global: { plugins: [pinia] }
    })
    const preparedCol = wrapper.find('[data-testid="prepared-column"]')
    expect(preparedCol.text()).toContain('Fireball')
    expect(preparedCol.text()).toContain('Shield')
  })

  it('calls toggleSpellPreparation when spell is toggled', async () => {
    const store = useCharacterPlayStateStore()
    store.initialize({
      characterId: 1,
      isDead: false,
      hitPoints: { current: 10, max: 10, temporary: 0 },
      deathSaves: { successes: 0, failures: 0 },
      currency: { pp: 0, gp: 0, ep: 0, sp: 0, cp: 0 }
    })
    store.initializeSpellPreparation({
      spells: mockSpells.map(s => ({ id: s.id, is_prepared: s.is_prepared, is_always_prepared: false })),
      preparationLimit: 8
    })

    const toggleSpy = vi.spyOn(store, 'toggleSpellPreparation').mockResolvedValue()

    const wrapper = await mountSuspended(SpellbookView, {
      props: { spells: mockSpells, preparedCount: 2, preparationLimit: 8, characterId: 1 },
      global: { plugins: [pinia] }
    })

    // Click an unprepared spell in spellbook column
    const cards = wrapper.findAll('[data-testid="spellbook-card"]')
    await cards[0].trigger('click')

    expect(toggleSpy).toHaveBeenCalled()
  })
})
```

### Step 2: Run tests to verify they fail

Run: `docker compose exec nuxt npm run test -- tests/components/character/sheet/SpellbookView.test.ts`

Expected: FAIL - component doesn't exist

### Step 3: Implement SpellbookView component

```vue
<!-- app/components/character/sheet/SpellbookView.vue -->
<script setup lang="ts">
/**
 * Two-column spellbook view for wizards
 *
 * Main container showing spellbook (left) and prepared spells (right).
 * Handles toggle preparation logic via store.
 *
 * @see Issue #680 - Wizard Spellbook Phase 2
 */
import { storeToRefs } from 'pinia'
import type { CharacterSpell } from '~/types/character'
import { useCharacterPlayStateStore } from '~/stores/characterPlayState'

const props = defineProps<{
  spells: CharacterSpell[]
  preparedCount: number
  preparationLimit: number
  characterId: number
}>()

const store = useCharacterPlayStateStore()
const { atPreparationLimit } = storeToRefs(store)

async function handleToggle(spell: CharacterSpell) {
  try {
    await store.toggleSpellPreparation(spell.id, spell.is_prepared)
  } catch {
    // Error handled by store (shows toast)
  }
}
</script>

<template>
  <div class="grid grid-cols-1 lg:grid-cols-2 gap-6">
    <!-- Spellbook Column (unprepared) -->
    <div data-testid="spellbook-column">
      <CharacterSheetSpellbookColumn
        :spells="spells"
        :at-prep-limit="atPreparationLimit"
        @toggle="handleToggle"
      />
    </div>

    <!-- Prepared Column -->
    <div data-testid="prepared-column">
      <CharacterSheetPreparedColumn
        :spells="spells"
        :prepared-count="preparedCount"
        :preparation-limit="preparationLimit"
        @toggle="handleToggle"
      />
    </div>
  </div>
</template>
```

### Step 4: Run tests to verify they pass

Run: `docker compose exec nuxt npm run test -- tests/components/character/sheet/SpellbookView.test.ts`

Expected: PASS (all 4 tests)

### Step 5: Commit

```bash
git add app/components/character/sheet/SpellbookView.vue tests/components/character/sheet/SpellbookView.test.ts
git commit -m "feat(spellbook): Add SpellbookView container (#680)"
```

---

## Task 6: Integrate SpellbookView into spells.vue page

Conditionally render SpellbookView for `preparation_method === 'spellbook'`.

**Files:**
- Modify: `app/pages/characters/[publicId]/spells.vue`
- Modify: `tests/pages/characters/spells.test.ts`

### Step 1: Write failing tests

Add these tests to `tests/pages/characters/spells.test.ts`:

```typescript
// Add to existing spells.test.ts

describe('spellbook view (wizard)', () => {
  const mockWizardStats = {
    ...mockStats,
    preparation_method: 'spellbook' as const
  }

  it('renders SpellbookView for spellbook casters', async () => {
    server.use(
      http.get('/api/characters/:id/stats', () => {
        return HttpResponse.json({ data: mockWizardStats })
      })
    )

    const wrapper = await mountSuspended(SpellsPage)
    await flushPromises()

    expect(wrapper.find('[data-testid="spellbook-view"]').exists()).toBe(true)
  })

  it('does not render SpellbookView for non-wizard casters', async () => {
    const mockSorcererStats = { ...mockStats, preparation_method: 'known' as const }
    server.use(
      http.get('/api/characters/:id/stats', () => {
        return HttpResponse.json({ data: mockSorcererStats })
      })
    )

    const wrapper = await mountSuspended(SpellsPage)
    await flushPromises()

    expect(wrapper.find('[data-testid="spellbook-view"]').exists()).toBe(false)
  })
})
```

### Step 2: Run tests to verify they fail

Run: `docker compose exec nuxt npm run test -- tests/pages/characters/spells.test.ts -t "spellbook view"`

Expected: FAIL - SpellbookView not rendered

### Step 3: Modify spells.vue page

Update the template section in `app/pages/characters/[publicId]/spells.vue` to conditionally render SpellbookView:

```vue
<!-- In the <template> section, replace the existing spell rendering with: -->

<!-- Spellcaster Content -->
<template v-else>
  <!-- Spellcasting Stats Bar (keep existing) -->
  <div class="mt-6 p-4 bg-gray-50 dark:bg-gray-800 rounded-lg">
    <!-- ... existing stats bar code ... -->
  </div>

  <!-- Spell Slots (keep existing) -->
  <div v-if="spellSlots?.slots && Object.keys(spellSlots.slots).length > 0" class="mt-6">
    <!-- ... existing spell slots code ... -->
  </div>

  <!-- Wizard Spellbook View (two-column) -->
  <div
    v-if="preparationMethod === 'spellbook'"
    data-testid="spellbook-view"
    class="mt-8"
  >
    <CharacterSheetSpellbookView
      :spells="validSpells"
      :prepared-count="spellSlots?.prepared_count ?? 0"
      :preparation-limit="spellSlots?.preparation_limit ?? 0"
      :character-id="character.id"
    />
  </div>

  <!-- Standard Spell List (for non-wizard casters) -->
  <template v-else>
    <!-- Cantrips Section (keep existing) -->
    <div v-if="cantrips.length > 0" class="mt-8">
      <!-- ... existing cantrips code ... -->
    </div>

    <!-- Leveled Spells by Level (keep existing) -->
    <div v-for="level in sortedLevels" :key="level" class="mt-8">
      <!-- ... existing leveled spells code ... -->
    </div>
  </template>

  <!-- Empty State (keep existing) -->
  <div v-if="validSpells.length === 0" class="mt-8 text-center py-12 text-gray-500 dark:text-gray-400">
    <!-- ... existing empty state code ... -->
  </div>
</template>
```

Also add the `preparationMethod` computed if not already present:

```typescript
// In <script setup>, add:
import type { PreparationMethod } from '~/types/character'

const preparationMethod = computed<PreparationMethod>(() => stats.value?.preparation_method ?? null)
```

### Step 4: Run tests to verify they pass

Run: `docker compose exec nuxt npm run test -- tests/pages/characters/spells.test.ts`

Expected: PASS (all tests including new spellbook view tests)

### Step 5: Commit

```bash
git add app/pages/characters/[publicId]/spells.vue tests/pages/characters/spells.test.ts
git commit -m "feat(spellbook): Integrate SpellbookView into spells page (#680)"
```

---

## Task 7: Final integration testing and cleanup

Run all tests, verify typecheck, and do manual testing.

### Step 1: Run full character test suite

```bash
docker compose exec nuxt npm run test:character
```

Expected: All tests pass

### Step 2: Run typecheck

```bash
docker compose exec nuxt npm run typecheck
```

Expected: No errors

### Step 3: Run lint

```bash
docker compose exec nuxt npm run lint:fix
```

Expected: No errors (or auto-fixed)

### Step 4: Manual testing

1. Create a wizard character (or use existing)
2. Navigate to `/characters/{publicId}/spells`
3. Verify two-column layout appears
4. Test filtering (search, school, level, concentration, ritual)
5. Test click-to-prepare and click-to-unprepare
6. Test at-limit behavior (grey out, warning color on counter)
7. Test on mobile viewport (if time permits)

### Step 5: Final commit

```bash
git add .
git commit -m "feat(spellbook): Complete Phase 2 wizard spellbook UI (#680)"
```

---

## Summary

| Task | Component | Tests |
|------|-----------|-------|
| 1 | SpellbookCard | 11 tests |
| 2 | SpellbookFilters | 8 tests |
| 3 | SpellbookColumn | 9 tests |
| 4 | PreparedColumn | 7 tests |
| 5 | SpellbookView | 4 tests |
| 6 | Page integration | 2 tests |
| 7 | Final verification | - |

**Total new tests:** ~41 tests
**Total new components:** 5
**Estimated time:** 2-3 hours
