# Level-Up Wizard Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Build a modal wizard that allows players to level up existing characters, with HP choice (roll/average), ASI/Feat selection, and celebration summary.

**Architecture:** Full-page modal wizard triggered from Character Sheet header. New Pinia store (`characterLevelUp.ts`) manages level-up session state. Reuses existing step components (StepFeats, StepSpells, StepProficiencies, StepSubclass) and unified choices system. New components for HP selection and celebration summary.

**Tech Stack:** Nuxt 4, NuxtUI 4, Pinia, TypeScript, Vitest

**Related Issue:** #466

---

## Task 1: Nitro Route for Level-Up Endpoint

Create the server proxy route for the backend level-up endpoint.

**Files:**
- Create: `server/api/characters/[id]/classes/[classSlug]/level-up.post.ts`
- Test: Manual verification via curl

**Step 1: Create the Nitro route**

```typescript
// server/api/characters/[id]/classes/[classSlug]/level-up.post.ts
/**
 * Level up character in a class - Proxies to Laravel backend
 *
 * @example POST /api/characters/arcane-phoenix-M7k2/classes/phb:fighter/level-up
 * @returns LevelUpResult with previous_level, new_level, features_gained, etc.
 */
export default defineEventHandler(async (event) => {
  const config = useRuntimeConfig()
  const id = getRouterParam(event, 'id')
  const classSlug = getRouterParam(event, 'classSlug')

  const data = await $fetch(
    `${config.apiBaseServer}/characters/${id}/classes/${classSlug}/level-up`,
    { method: 'POST' }
  )
  return data
})
```

**Step 2: Verify route works**

```bash
# Test with a real character (replace with actual publicId and class)
curl -X POST http://localhost:4000/api/characters/test-char-xxxx/classes/phb:fighter/level-up
# Expected: JSON with previous_level, new_level, features_gained, etc.
```

**Step 3: Commit**

```bash
git add server/api/characters/\[id\]/classes/\[classSlug\]/level-up.post.ts
git commit -m "feat: Add Nitro route for character level-up endpoint"
```

---

## Task 2: TypeScript Types for Level-Up

Add TypeScript types for level-up data structures.

**Files:**
- Modify: `app/types/character.ts`
- Test: TypeScript compilation

**Step 1: Add LevelUpResult type**

Add to `app/types/character.ts`:

```typescript
/**
 * Result from level-up API call
 */
export interface LevelUpResult {
  previous_level: number
  new_level: number
  hp_increase: number
  new_max_hp: number
  features_gained: Array<{
    id: number
    name: string
    description: string | null
  }>
  spell_slots: Record<string, number>
  asi_pending: boolean
  hp_choice_pending: boolean
}

/**
 * Level-up wizard step definition
 */
export interface LevelUpStep {
  name: string
  label: string
  icon: string
  visible: () => boolean
  shouldSkip?: () => boolean
}
```

**Step 2: Run typecheck**

```bash
docker compose exec nuxt npm run typecheck
# Expected: No errors
```

**Step 3: Commit**

```bash
git add app/types/character.ts
git commit -m "feat: Add TypeScript types for level-up result"
```

---

## Task 3: Level-Up Pinia Store

Create the dedicated store for level-up wizard state.

**Files:**
- Create: `app/stores/characterLevelUp.ts`
- Create: `tests/stores/characterLevelUp.test.ts`

**Step 1: Write the store test**

```typescript
// tests/stores/characterLevelUp.test.ts
import { describe, it, expect, beforeEach, vi } from 'vitest'
import { setActivePinia, createPinia } from 'pinia'
import { useCharacterLevelUpStore } from '~/stores/characterLevelUp'

// Mock useApi
vi.mock('~/composables/useApi', () => ({
  useApi: () => ({
    apiFetch: vi.fn()
  })
}))

describe('characterLevelUp store', () => {
  beforeEach(() => {
    setActivePinia(createPinia())
  })

  describe('initial state', () => {
    it('starts with null character data', () => {
      const store = useCharacterLevelUpStore()
      expect(store.characterId).toBeNull()
      expect(store.publicId).toBeNull()
      expect(store.levelUpResult).toBeNull()
    })

    it('starts not open', () => {
      const store = useCharacterLevelUpStore()
      expect(store.isOpen).toBe(false)
    })

    it('starts on class-selection step', () => {
      const store = useCharacterLevelUpStore()
      expect(store.currentStepName).toBe('class-selection')
    })
  })

  describe('openWizard', () => {
    it('sets character data and opens wizard', () => {
      const store = useCharacterLevelUpStore()
      store.openWizard(123, 'shadow-warden-q3x9')

      expect(store.characterId).toBe(123)
      expect(store.publicId).toBe('shadow-warden-q3x9')
      expect(store.isOpen).toBe(true)
    })
  })

  describe('closeWizard', () => {
    it('closes wizard but preserves character data', () => {
      const store = useCharacterLevelUpStore()
      store.openWizard(123, 'shadow-warden-q3x9')
      store.closeWizard()

      expect(store.isOpen).toBe(false)
      expect(store.characterId).toBe(123)
    })
  })

  describe('reset', () => {
    it('clears all state', () => {
      const store = useCharacterLevelUpStore()
      store.openWizard(123, 'shadow-warden-q3x9')
      store.reset()

      expect(store.characterId).toBeNull()
      expect(store.publicId).toBeNull()
      expect(store.isOpen).toBe(false)
      expect(store.levelUpResult).toBeNull()
    })
  })
})
```

**Step 2: Run test to verify it fails**

```bash
docker compose exec nuxt npm run test -- tests/stores/characterLevelUp.test.ts
# Expected: FAIL - store doesn't exist yet
```

**Step 3: Create the store**

```typescript
// app/stores/characterLevelUp.ts
/**
 * Level-Up Wizard Store
 *
 * Manages state for the level-up modal wizard.
 * Separate from characterWizard store because:
 * 1. Different lifecycle (modal vs. page)
 * 2. Cleaner separation of concerns
 * 3. Level-up has unique data (LevelUpResult, selected class for multiclass)
 */
import { defineStore } from 'pinia'
import type { LevelUpResult } from '~/types/character'
import type { CharacterClass } from '~/types'

export interface CharacterClassEntry {
  class: CharacterClass | null
  level: number
  subclass?: { name: string; slug: string } | null
  is_primary: boolean
}

export const useCharacterLevelUpStore = defineStore('characterLevelUp', () => {
  const { apiFetch } = useApi()

  // ══════════════════════════════════════════════════════════════
  // STATE
  // ══════════════════════════════════════════════════════════════

  /** Character being leveled up */
  const characterId = ref<number | null>(null)
  const publicId = ref<string | null>(null)

  /** Is the wizard modal open? */
  const isOpen = ref(false)

  /** Current wizard step */
  const currentStepName = ref<string>('class-selection')

  /** Result from level-up API call */
  const levelUpResult = ref<LevelUpResult | null>(null)

  /** Class selected for level-up (for multiclass scenarios) */
  const selectedClassSlug = ref<string | null>(null)

  /** Character's current classes (for multiclass selection) */
  const characterClasses = ref<CharacterClassEntry[]>([])

  /** Character's total level */
  const totalLevel = ref<number>(0)

  /** Loading state */
  const isLoading = ref(false)

  /** Error state */
  const error = ref<string | null>(null)

  // ══════════════════════════════════════════════════════════════
  // COMPUTED
  // ══════════════════════════════════════════════════════════════

  /** Is this a multiclass character? */
  const isMulticlass = computed(() => characterClasses.value.length > 1)

  /** Is this the first opportunity to multiclass (level 1 -> 2)? */
  const isFirstMulticlassOpportunity = computed(() => totalLevel.value === 1)

  /** Should we show class selection step? */
  const needsClassSelection = computed(() =>
    isMulticlass.value || isFirstMulticlassOpportunity.value
  )

  /** Is level-up complete (all choices resolved)? */
  const isComplete = computed(() => {
    if (!levelUpResult.value) return false
    return !levelUpResult.value.hp_choice_pending && !levelUpResult.value.asi_pending
  })

  // ══════════════════════════════════════════════════════════════
  // ACTIONS
  // ══════════════════════════════════════════════════════════════

  /**
   * Open the level-up wizard for a character
   */
  function openWizard(charId: number, charPublicId: string, classes: CharacterClassEntry[] = [], level: number = 1) {
    characterId.value = charId
    publicId.value = charPublicId
    characterClasses.value = classes
    totalLevel.value = level
    currentStepName.value = 'class-selection'
    levelUpResult.value = null
    selectedClassSlug.value = null
    error.value = null
    isOpen.value = true
  }

  /**
   * Close the wizard (but preserve state for potential reopening)
   */
  function closeWizard() {
    isOpen.value = false
  }

  /**
   * Trigger level-up for a class
   */
  async function levelUp(classSlug: string): Promise<LevelUpResult> {
    if (!characterId.value) throw new Error('No character selected')

    isLoading.value = true
    error.value = null

    try {
      const result = await apiFetch<LevelUpResult>(
        `/characters/${publicId.value}/classes/${classSlug}/level-up`,
        { method: 'POST' }
      )

      levelUpResult.value = result
      selectedClassSlug.value = classSlug
      return result
    } catch (e) {
      error.value = e instanceof Error ? e.message : 'Failed to level up'
      throw e
    } finally {
      isLoading.value = false
    }
  }

  /**
   * Navigate to a specific step
   */
  function goToStep(stepName: string) {
    currentStepName.value = stepName
  }

  /**
   * Reset all wizard state
   */
  function reset() {
    characterId.value = null
    publicId.value = null
    characterClasses.value = []
    totalLevel.value = 0
    isOpen.value = false
    currentStepName.value = 'class-selection'
    levelUpResult.value = null
    selectedClassSlug.value = null
    isLoading.value = false
    error.value = null
  }

  // ══════════════════════════════════════════════════════════════
  // RETURN
  // ══════════════════════════════════════════════════════════════

  return {
    // State
    characterId,
    publicId,
    isOpen,
    currentStepName,
    levelUpResult,
    selectedClassSlug,
    characterClasses,
    totalLevel,
    isLoading,
    error,

    // Computed
    isMulticlass,
    isFirstMulticlassOpportunity,
    needsClassSelection,
    isComplete,

    // Actions
    openWizard,
    closeWizard,
    levelUp,
    goToStep,
    reset
  }
})
```

**Step 4: Run tests**

```bash
docker compose exec nuxt npm run test -- tests/stores/characterLevelUp.test.ts
# Expected: PASS
```

**Step 5: Commit**

```bash
git add app/stores/characterLevelUp.ts tests/stores/characterLevelUp.test.ts
git commit -m "feat: Add Pinia store for level-up wizard state"
```

---

## Task 4: Level-Up Wizard Composable

Create the navigation composable for level-up wizard steps.

**Files:**
- Create: `app/composables/useLevelUpWizard.ts`
- Create: `tests/composables/useLevelUpWizard.test.ts`

**Step 1: Write the composable test**

```typescript
// tests/composables/useLevelUpWizard.test.ts
import { describe, it, expect, beforeEach, vi } from 'vitest'
import { setActivePinia, createPinia } from 'pinia'

// Mock the store
vi.mock('~/stores/characterLevelUp', () => ({
  useCharacterLevelUpStore: vi.fn(() => ({
    levelUpResult: null,
    needsClassSelection: true,
    currentStepName: 'class-selection',
    goToStep: vi.fn()
  }))
}))

describe('useLevelUpWizard', () => {
  beforeEach(() => {
    setActivePinia(createPinia())
    vi.resetModules()
  })

  it('provides step definitions', async () => {
    const { useLevelUpWizard } = await import('~/composables/useLevelUpWizard')
    const { stepRegistry } = useLevelUpWizard()

    expect(stepRegistry).toBeDefined()
    expect(stepRegistry.length).toBeGreaterThan(0)
    expect(stepRegistry[0].name).toBe('class-selection')
  })
})
```

**Step 2: Run test to verify it fails**

```bash
docker compose exec nuxt npm run test -- tests/composables/useLevelUpWizard.test.ts
# Expected: FAIL - composable doesn't exist yet
```

**Step 3: Create the composable**

```typescript
// app/composables/useLevelUpWizard.ts
/**
 * Level-Up Wizard Navigation Composable
 *
 * Manages wizard step navigation, visibility, and validation for level-up flow.
 * Similar to useCharacterWizard but adapted for level-up's different step structure.
 */
import { useCharacterLevelUpStore } from '~/stores/characterLevelUp'
import type { LevelUpStep } from '~/types/character'

/**
 * Create the step registry with visibility functions
 */
function createStepRegistry(store: ReturnType<typeof useCharacterLevelUpStore>): LevelUpStep[] {
  return [
    {
      name: 'class-selection',
      label: 'Class',
      icon: 'i-heroicons-shield-check',
      visible: () => store.needsClassSelection,
      shouldSkip: () => !store.needsClassSelection
    },
    {
      name: 'subclass',
      label: 'Subclass',
      icon: 'i-heroicons-star',
      visible: () => false, // Will be set based on level-up result
      shouldSkip: () => true // Skip unless subclass choice pending
    },
    {
      name: 'hit-points',
      label: 'Hit Points',
      icon: 'i-heroicons-heart',
      visible: () => true,
      shouldSkip: () => !(store.levelUpResult?.hp_choice_pending ?? true)
    },
    {
      name: 'asi-feat',
      label: 'ASI / Feat',
      icon: 'i-heroicons-arrow-trending-up',
      visible: () => true,
      shouldSkip: () => !(store.levelUpResult?.asi_pending ?? false)
    },
    {
      name: 'spells',
      label: 'Spells',
      icon: 'i-heroicons-sparkles',
      visible: () => false, // Will be determined by pending spell choices
      shouldSkip: () => true // Skip unless spell choices pending
    },
    {
      name: 'summary',
      label: 'Summary',
      icon: 'i-heroicons-trophy',
      visible: () => true
    }
  ]
}

export function useLevelUpWizard() {
  const store = useCharacterLevelUpStore()

  // Create step registry with store access
  const stepRegistry = createStepRegistry(store)

  // ══════════════════════════════════════════════════════════════
  // COMPUTED: Active Steps
  // ══════════════════════════════════════════════════════════════

  /**
   * Only visible steps (filtered by visibility functions)
   */
  const activeSteps = computed(() =>
    stepRegistry.filter(step => step.visible())
  )

  /**
   * Current step index within active steps
   */
  const currentStepIndex = computed(() =>
    activeSteps.value.findIndex(s => s.name === store.currentStepName)
  )

  /**
   * Current step object
   */
  const currentStep = computed(() =>
    activeSteps.value[currentStepIndex.value] ?? null
  )

  /**
   * Total number of visible steps
   */
  const totalSteps = computed(() => activeSteps.value.length)

  /**
   * Is this the first step?
   */
  const isFirstStep = computed(() => currentStepIndex.value === 0)

  /**
   * Is this the last step (summary)?
   */
  const isLastStep = computed(() =>
    currentStepIndex.value === totalSteps.value - 1
  )

  /**
   * Progress percentage (0-100)
   */
  const progressPercent = computed(() => {
    if (totalSteps.value <= 1) return 100
    return Math.round((currentStepIndex.value / (totalSteps.value - 1)) * 100)
  })

  // ══════════════════════════════════════════════════════════════
  // NAVIGATION
  // ══════════════════════════════════════════════════════════════

  /**
   * Navigate to next step, skipping any steps that should be skipped
   */
  function nextStep(): void {
    let nextIndex = currentStepIndex.value + 1

    while (nextIndex < activeSteps.value.length) {
      const next = activeSteps.value[nextIndex]
      if (next && !next.shouldSkip?.()) {
        store.goToStep(next.name)
        return
      }
      nextIndex++
    }
  }

  /**
   * Navigate to previous step
   */
  function previousStep(): void {
    let prevIndex = currentStepIndex.value - 1

    while (prevIndex >= 0) {
      const prev = activeSteps.value[prevIndex]
      if (prev && !prev.shouldSkip?.()) {
        store.goToStep(prev.name)
        return
      }
      prevIndex--
    }
  }

  /**
   * Navigate to specific step by name
   */
  function goToStep(stepName: string): void {
    store.goToStep(stepName)
  }

  // ══════════════════════════════════════════════════════════════
  // RETURN
  // ══════════════════════════════════════════════════════════════

  return {
    // Step registry
    stepRegistry,
    activeSteps,

    // Current step
    currentStep,
    currentStepIndex,

    // Progress
    totalSteps,
    progressPercent,
    isFirstStep,
    isLastStep,

    // Navigation
    nextStep,
    previousStep,
    goToStep
  }
}
```

**Step 4: Run tests**

```bash
docker compose exec nuxt npm run test -- tests/composables/useLevelUpWizard.test.ts
# Expected: PASS
```

**Step 5: Commit**

```bash
git add app/composables/useLevelUpWizard.ts tests/composables/useLevelUpWizard.test.ts
git commit -m "feat: Add composable for level-up wizard navigation"
```

---

## Task 5: Level-Up Button in Character Sheet Header

Add the "Level Up" button to the character sheet header.

**Files:**
- Modify: `app/components/character/sheet/Header.vue`
- Modify: `tests/components/character/sheet/Header.test.ts`

**Step 1: Write the test for level-up button**

Add to existing Header tests:

```typescript
// Add to tests/components/character/sheet/Header.test.ts

describe('level-up button', () => {
  it('shows level-up button for complete characters under level 20', async () => {
    const character = createMockCharacter({
      is_complete: true,
      classes: [{ class: { name: 'Fighter', slug: 'fighter' }, level: 5 }]
    })

    const wrapper = await mountSuspended(Header, {
      props: { character }
    })

    expect(wrapper.find('[data-testid="level-up-button"]').exists()).toBe(true)
  })

  it('hides level-up button for incomplete characters', async () => {
    const character = createMockCharacter({
      is_complete: false
    })

    const wrapper = await mountSuspended(Header, {
      props: { character }
    })

    expect(wrapper.find('[data-testid="level-up-button"]').exists()).toBe(false)
  })

  it('hides level-up button at max level (20)', async () => {
    const character = createMockCharacter({
      is_complete: true,
      total_level: 20
    })

    const wrapper = await mountSuspended(Header, {
      props: { character }
    })

    expect(wrapper.find('[data-testid="level-up-button"]').exists()).toBe(false)
  })

  it('emits level-up event when clicked', async () => {
    const character = createMockCharacter({
      is_complete: true,
      classes: [{ class: { name: 'Fighter', slug: 'fighter' }, level: 5 }]
    })

    const wrapper = await mountSuspended(Header, {
      props: { character }
    })

    await wrapper.find('[data-testid="level-up-button"]').trigger('click')
    expect(wrapper.emitted('level-up')).toBeTruthy()
  })
})
```

**Step 2: Run test to verify it fails**

```bash
docker compose exec nuxt npm run test -- tests/components/character/sheet/Header.test.ts
# Expected: FAIL - button doesn't exist yet
```

**Step 3: Update Header component**

```vue
<!-- app/components/character/sheet/Header.vue -->
<script setup lang="ts">
import type { Character } from '~/types/character'

const props = defineProps<{
  character: Character
}>()

const emit = defineEmits<{
  'level-up': []
}>()

// ... existing computed properties ...

/**
 * Calculate total character level from classes
 */
const totalLevel = computed(() => {
  if (!props.character.classes?.length) return 1
  return props.character.classes.reduce((sum, c) => sum + (c.level || 0), 0)
})

/**
 * Can this character level up?
 * - Must be complete (not a draft)
 * - Must be under max level (20)
 */
const canLevelUp = computed(() => {
  return props.character.is_complete && totalLevel.value < 20
})

function handleLevelUp() {
  emit('level-up')
}
</script>

<template>
  <div class="flex flex-col sm:flex-row sm:items-center sm:justify-between gap-4">
    <!-- Portrait and Name Section (unchanged) -->
    <div class="flex items-center gap-4">
      <!-- ... existing portrait and name markup ... -->
    </div>

    <!-- Right: Badges and actions -->
    <div class="flex items-center gap-2 flex-wrap">
      <!-- Inspiration Badge (unchanged) -->
      <!-- ... -->

      <!-- Status Badge (unchanged) -->
      <!-- ... -->

      <!-- Level Up Button (NEW) -->
      <UButton
        v-if="canLevelUp"
        data-testid="level-up-button"
        variant="soft"
        color="primary"
        size="sm"
        icon="i-heroicons-arrow-trending-up"
        @click="handleLevelUp"
      >
        Level Up
      </UButton>

      <!-- Edit Button (unchanged) -->
      <!-- ... -->
    </div>
  </div>
</template>
```

**Step 4: Run tests**

```bash
docker compose exec nuxt npm run test -- tests/components/character/sheet/Header.test.ts
# Expected: PASS
```

**Step 5: Commit**

```bash
git add app/components/character/sheet/Header.vue tests/components/character/sheet/Header.test.ts
git commit -m "feat: Add level-up button to character sheet header"
```

---

## Task 6: Hit Points Step Component

Create the HP choice step (roll vs. average).

**Files:**
- Create: `app/components/character/levelup/StepHitPoints.vue`
- Create: `app/components/character/levelup/HitDieRoller.vue`
- Create: `tests/components/character/levelup/StepHitPoints.test.ts`

**Step 1: Write the test**

```typescript
// tests/components/character/levelup/StepHitPoints.test.ts
import { describe, it, expect, beforeEach, vi } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import { setActivePinia, createPinia } from 'pinia'
import StepHitPoints from '~/components/character/levelup/StepHitPoints.vue'

// Mock the stores
vi.mock('~/stores/characterLevelUp', () => ({
  useCharacterLevelUpStore: vi.fn(() => ({
    characterId: 1,
    selectedClassSlug: 'phb:fighter'
  }))
}))

vi.mock('~/composables/useUnifiedChoices', () => ({
  useUnifiedChoices: vi.fn(() => ({
    choices: ref([]),
    choicesByType: computed(() => ({ hitPoints: [] })),
    pending: ref(false),
    fetchChoices: vi.fn(),
    resolveChoice: vi.fn()
  }))
}))

describe('StepHitPoints', () => {
  beforeEach(() => {
    setActivePinia(createPinia())
  })

  it('renders HP choice options', async () => {
    const wrapper = await mountSuspended(StepHitPoints, {
      props: {
        hitDie: 10,
        conModifier: 2
      }
    })

    expect(wrapper.text()).toContain('Hit Point Increase')
    expect(wrapper.find('[data-testid="roll-button"]').exists()).toBe(true)
    expect(wrapper.find('[data-testid="average-button"]').exists()).toBe(true)
  })

  it('shows hit die type', async () => {
    const wrapper = await mountSuspended(StepHitPoints, {
      props: {
        hitDie: 10,
        conModifier: 2
      }
    })

    expect(wrapper.text()).toContain('d10')
  })

  it('shows average value correctly (rounded up)', async () => {
    const wrapper = await mountSuspended(StepHitPoints, {
      props: {
        hitDie: 10, // Average = 5.5, rounded up = 6
        conModifier: 2
      }
    })

    expect(wrapper.text()).toContain('6') // Average for d10
  })

  it('calculates total with CON modifier', async () => {
    const wrapper = await mountSuspended(StepHitPoints, {
      props: {
        hitDie: 10,
        conModifier: 3
      }
    })

    // Average 6 + CON 3 = 9
    await wrapper.find('[data-testid="average-button"]').trigger('click')
    expect(wrapper.text()).toContain('9')
  })
})
```

**Step 2: Run test to verify it fails**

```bash
docker compose exec nuxt npm run test -- tests/components/character/levelup/StepHitPoints.test.ts
# Expected: FAIL - component doesn't exist
```

**Step 3: Create HitDieRoller component**

```vue
<!-- app/components/character/levelup/HitDieRoller.vue -->
<script setup lang="ts">
const props = defineProps<{
  dieSize: number
  rolling?: boolean
}>()

const emit = defineEmits<{
  'roll-complete': [result: number]
}>()

const displayValue = ref<number | null>(null)
const isRolling = ref(false)

// Animation frames for dice roll effect
const rollFrames = ref(0)
const rollInterval = ref<NodeJS.Timeout | null>(null)

function rollDie() {
  isRolling.value = true
  rollFrames.value = 0

  // Clear any existing interval
  if (rollInterval.value) clearInterval(rollInterval.value)

  // Animate random values for ~1 second
  rollInterval.value = setInterval(() => {
    displayValue.value = Math.floor(Math.random() * props.dieSize) + 1
    rollFrames.value++

    if (rollFrames.value >= 20) {
      // Stop rolling, generate final result
      if (rollInterval.value) clearInterval(rollInterval.value)
      const finalResult = Math.floor(Math.random() * props.dieSize) + 1
      displayValue.value = finalResult
      isRolling.value = false
      emit('roll-complete', finalResult)
    }
  }, 50)
}

// Expose roll function for parent
defineExpose({ rollDie })

onUnmounted(() => {
  if (rollInterval.value) clearInterval(rollInterval.value)
})
</script>

<template>
  <div
    class="relative w-24 h-24 flex items-center justify-center"
    :class="{ 'animate-bounce': isRolling }"
  >
    <!-- Die background -->
    <div
      class="absolute inset-0 bg-gradient-to-br from-primary-400 to-primary-600 rounded-xl shadow-lg transform"
      :class="{ 'rotate-12': isRolling }"
    />

    <!-- Die value -->
    <span
      class="relative text-3xl font-bold text-white z-10"
      :class="{ 'animate-pulse': isRolling }"
    >
      <template v-if="displayValue !== null">
        {{ displayValue }}
      </template>
      <template v-else>
        d{{ dieSize }}
      </template>
    </span>
  </div>
</template>
```

**Step 4: Create StepHitPoints component**

```vue
<!-- app/components/character/levelup/StepHitPoints.vue -->
<script setup lang="ts">
import { useCharacterLevelUpStore } from '~/stores/characterLevelUp'
import { useLevelUpWizard } from '~/composables/useLevelUpWizard'

const props = defineProps<{
  hitDie: number
  conModifier: number
}>()

const emit = defineEmits<{
  'choice-made': [hpGained: number]
}>()

const store = useCharacterLevelUpStore()
const { nextStep } = useLevelUpWizard()

// Use unified choices for HP choice resolution
const { resolveChoice, fetchChoices, choicesByType } = useUnifiedChoices(
  computed(() => store.characterId)
)

// Local state
const selectedMethod = ref<'roll' | 'average' | null>(null)
const rollResult = ref<number | null>(null)
const hpGained = ref<number | null>(null)
const isSaving = ref(false)

// Reference to die roller component
const dieRoller = ref<{ rollDie: () => void } | null>(null)

// Calculate average (rounded up per 5e rules)
const averageValue = computed(() => Math.ceil((props.hitDie + 1) / 2))

// Calculate total HP gained
const totalHpGained = computed(() => {
  if (selectedMethod.value === 'average') {
    return averageValue.value + props.conModifier
  }
  if (selectedMethod.value === 'roll' && rollResult.value !== null) {
    return rollResult.value + props.conModifier
  }
  return null
})

function handleRollClick() {
  selectedMethod.value = 'roll'
  dieRoller.value?.rollDie()
}

function handleRollComplete(result: number) {
  rollResult.value = result
  hpGained.value = result + props.conModifier
}

function handleAverageClick() {
  selectedMethod.value = 'average'
  rollResult.value = averageValue.value
  hpGained.value = averageValue.value + props.conModifier
}

async function handleConfirm() {
  if (hpGained.value === null) return

  isSaving.value = true
  try {
    // Find the HP choice and resolve it
    await fetchChoices('hit_points')
    const hpChoices = choicesByType.value.hitPoints
    if (hpChoices && hpChoices.length > 0) {
      await resolveChoice(hpChoices[0].id, {
        hit_point_increase: hpGained.value
      })
    }

    emit('choice-made', hpGained.value)
    nextStep()
  } finally {
    isSaving.value = false
  }
}

// Fetch HP choices on mount
onMounted(() => {
  fetchChoices('hit_points')
})
</script>

<template>
  <div class="space-y-8">
    <!-- Header -->
    <div class="text-center">
      <h2 class="text-2xl font-bold text-gray-900 dark:text-white">
        Hit Point Increase
      </h2>
      <p class="mt-2 text-gray-600 dark:text-gray-400">
        Your hit die: <span class="font-semibold">d{{ hitDie }}</span>
        &bull; Constitution modifier: <span class="font-semibold">{{ conModifier >= 0 ? '+' : '' }}{{ conModifier }}</span>
      </p>
    </div>

    <!-- Choice Options -->
    <div class="flex flex-col sm:flex-row justify-center items-center gap-6">
      <!-- Roll Option -->
      <div
        class="flex flex-col items-center p-6 rounded-xl border-2 transition-all cursor-pointer"
        :class="selectedMethod === 'roll'
          ? 'border-primary bg-primary-50 dark:bg-primary-900/20'
          : 'border-gray-200 dark:border-gray-700 hover:border-primary-300'"
        data-testid="roll-button"
        @click="handleRollClick"
      >
        <CharacterLevelupHitDieRoller
          ref="dieRoller"
          :die-size="hitDie"
          @roll-complete="handleRollComplete"
        />
        <span class="mt-3 font-semibold text-gray-900 dark:text-white">
          Roll d{{ hitDie }}
        </span>
        <span class="text-sm text-gray-500">
          Take your chances
        </span>
      </div>

      <!-- Average Option -->
      <div
        class="flex flex-col items-center p-6 rounded-xl border-2 transition-all cursor-pointer"
        :class="selectedMethod === 'average'
          ? 'border-primary bg-primary-50 dark:bg-primary-900/20'
          : 'border-gray-200 dark:border-gray-700 hover:border-primary-300'"
        data-testid="average-button"
        @click="handleAverageClick"
      >
        <div class="w-24 h-24 flex items-center justify-center bg-gray-100 dark:bg-gray-800 rounded-xl">
          <span class="text-3xl font-bold text-gray-700 dark:text-gray-300">
            {{ averageValue }}
          </span>
        </div>
        <span class="mt-3 font-semibold text-gray-900 dark:text-white">
          Take Average
        </span>
        <span class="text-sm text-gray-500">
          Guaranteed value
        </span>
      </div>
    </div>

    <!-- Result Display -->
    <div
      v-if="totalHpGained !== null"
      class="text-center p-6 bg-success-50 dark:bg-success-900/20 rounded-xl"
    >
      <p class="text-lg text-gray-700 dark:text-gray-300">
        <span v-if="selectedMethod === 'roll'">
          You rolled <span class="font-bold text-primary">{{ rollResult }}</span>
        </span>
        <span v-else>
          Average: <span class="font-bold">{{ averageValue }}</span>
        </span>
        + {{ conModifier }} (CON) =
        <span class="text-2xl font-bold text-success-600 dark:text-success-400">
          {{ totalHpGained }} HP
        </span>
      </p>
    </div>

    <!-- Confirm Button -->
    <div class="flex justify-center pt-4">
      <UButton
        data-testid="confirm-hp-btn"
        size="lg"
        :disabled="totalHpGained === null || isSaving"
        :loading="isSaving"
        @click="handleConfirm"
      >
        Confirm HP Increase
      </UButton>
    </div>
  </div>
</template>
```

**Step 5: Run tests**

```bash
docker compose exec nuxt npm run test -- tests/components/character/levelup/StepHitPoints.test.ts
# Expected: PASS
```

**Step 6: Commit**

```bash
git add app/components/character/levelup/
git add tests/components/character/levelup/
git commit -m "feat: Add HP choice step with dice roller animation"
```

---

## Task 7: Level-Up Summary Step

Create the celebration summary step.

**Files:**
- Create: `app/components/character/levelup/StepLevelUpSummary.vue`
- Create: `tests/components/character/levelup/StepLevelUpSummary.test.ts`

**Step 1: Write the test**

```typescript
// tests/components/character/levelup/StepLevelUpSummary.test.ts
import { describe, it, expect, beforeEach, vi } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import { setActivePinia, createPinia } from 'pinia'
import StepLevelUpSummary from '~/components/character/levelup/StepLevelUpSummary.vue'

describe('StepLevelUpSummary', () => {
  beforeEach(() => {
    setActivePinia(createPinia())
  })

  const mockLevelUpResult = {
    previous_level: 3,
    new_level: 4,
    hp_increase: 9,
    new_max_hp: 38,
    features_gained: [
      { id: 1, name: 'Ability Score Improvement', description: 'Increase ability scores' }
    ],
    spell_slots: {},
    asi_pending: false,
    hp_choice_pending: false
  }

  it('shows level up complete message', async () => {
    const wrapper = await mountSuspended(StepLevelUpSummary, {
      props: {
        levelUpResult: mockLevelUpResult,
        className: 'Fighter',
        hpGained: 9
      }
    })

    expect(wrapper.text()).toContain('Level Up Complete')
  })

  it('shows level transition', async () => {
    const wrapper = await mountSuspended(StepLevelUpSummary, {
      props: {
        levelUpResult: mockLevelUpResult,
        className: 'Fighter',
        hpGained: 9
      }
    })

    expect(wrapper.text()).toContain('Fighter 3')
    expect(wrapper.text()).toContain('Fighter 4')
  })

  it('shows HP gained', async () => {
    const wrapper = await mountSuspended(StepLevelUpSummary, {
      props: {
        levelUpResult: mockLevelUpResult,
        className: 'Fighter',
        hpGained: 9
      }
    })

    expect(wrapper.text()).toContain('+9')
    expect(wrapper.text()).toContain('38')
  })

  it('shows features gained', async () => {
    const wrapper = await mountSuspended(StepLevelUpSummary, {
      props: {
        levelUpResult: mockLevelUpResult,
        className: 'Fighter',
        hpGained: 9
      }
    })

    expect(wrapper.text()).toContain('Ability Score Improvement')
  })

  it('has button to return to character sheet', async () => {
    const wrapper = await mountSuspended(StepLevelUpSummary, {
      props: {
        levelUpResult: mockLevelUpResult,
        className: 'Fighter',
        hpGained: 9
      }
    })

    expect(wrapper.find('[data-testid="view-sheet-button"]').exists()).toBe(true)
  })
})
```

**Step 2: Run test to verify it fails**

```bash
docker compose exec nuxt npm run test -- tests/components/character/levelup/StepLevelUpSummary.test.ts
# Expected: FAIL - component doesn't exist
```

**Step 3: Create the component**

```vue
<!-- app/components/character/levelup/StepLevelUpSummary.vue -->
<script setup lang="ts">
import type { LevelUpResult } from '~/types/character'
import { useCharacterLevelUpStore } from '~/stores/characterLevelUp'

const props = defineProps<{
  levelUpResult: LevelUpResult
  className: string
  hpGained: number
  asiChoice?: string
  featName?: string
}>()

const emit = defineEmits<{
  'complete': []
}>()

const store = useCharacterLevelUpStore()

// Trigger celebration animation on mount
const showConfetti = ref(false)
const showContent = ref(false)

onMounted(() => {
  // Start confetti
  showConfetti.value = true

  // Fade in content after brief delay
  setTimeout(() => {
    showContent.value = true
  }, 300)

  // Stop confetti after a few seconds
  setTimeout(() => {
    showConfetti.value = false
  }, 3000)
})

function handleComplete() {
  store.closeWizard()
  store.reset()
  emit('complete')
}
</script>

<template>
  <div class="space-y-8 relative">
    <!-- Confetti Animation (CSS-based for simplicity) -->
    <div
      v-if="showConfetti"
      class="absolute inset-0 pointer-events-none overflow-hidden"
    >
      <div
        v-for="i in 50"
        :key="i"
        class="confetti-piece"
        :style="{
          left: `${Math.random() * 100}%`,
          animationDelay: `${Math.random() * 2}s`,
          backgroundColor: ['#fbbf24', '#34d399', '#60a5fa', '#f472b6', '#a78bfa'][i % 5]
        }"
      />
    </div>

    <!-- Header with animation -->
    <div
      class="text-center transition-all duration-500"
      :class="showContent ? 'opacity-100 translate-y-0' : 'opacity-0 translate-y-4'"
    >
      <div class="text-5xl mb-4">
        🎉
      </div>
      <h2 class="text-3xl font-bold text-gray-900 dark:text-white">
        Level Up Complete!
      </h2>
      <p class="mt-4 text-xl text-primary-600 dark:text-primary-400 font-semibold">
        {{ className }} {{ levelUpResult.previous_level }} → {{ className }} {{ levelUpResult.new_level }}
      </p>
    </div>

    <!-- Summary Cards -->
    <div
      class="space-y-4 transition-all duration-500 delay-200"
      :class="showContent ? 'opacity-100 translate-y-0' : 'opacity-0 translate-y-4'"
    >
      <!-- HP Card -->
      <div class="bg-white dark:bg-gray-800 rounded-xl p-4 shadow-sm border border-gray-200 dark:border-gray-700">
        <div class="flex items-center justify-between">
          <div class="flex items-center gap-3">
            <UIcon
              name="i-heroicons-heart"
              class="w-6 h-6 text-red-500"
            />
            <span class="font-medium text-gray-900 dark:text-white">Hit Points</span>
          </div>
          <div class="text-right">
            <span class="text-lg font-bold text-success-600 dark:text-success-400">+{{ hpGained }}</span>
            <span class="text-gray-500 dark:text-gray-400 ml-2">(now {{ levelUpResult.new_max_hp }} max)</span>
          </div>
        </div>
      </div>

      <!-- ASI/Feat Card (if applicable) -->
      <div
        v-if="asiChoice || featName"
        class="bg-white dark:bg-gray-800 rounded-xl p-4 shadow-sm border border-gray-200 dark:border-gray-700"
      >
        <div class="flex items-center justify-between">
          <div class="flex items-center gap-3">
            <UIcon
              name="i-heroicons-arrow-trending-up"
              class="w-6 h-6 text-primary-500"
            />
            <span class="font-medium text-gray-900 dark:text-white">
              {{ featName ? 'Feat' : 'Ability Score Increase' }}
            </span>
          </div>
          <span class="font-semibold text-gray-900 dark:text-white">
            {{ featName || asiChoice }}
          </span>
        </div>
      </div>

      <!-- Features Gained Card -->
      <div
        v-if="levelUpResult.features_gained.length > 0"
        class="bg-white dark:bg-gray-800 rounded-xl p-4 shadow-sm border border-gray-200 dark:border-gray-700"
      >
        <div class="flex items-center gap-3 mb-3">
          <UIcon
            name="i-heroicons-star"
            class="w-6 h-6 text-yellow-500"
          />
          <span class="font-medium text-gray-900 dark:text-white">New Features</span>
        </div>
        <ul class="space-y-2">
          <li
            v-for="feature in levelUpResult.features_gained"
            :key="feature.id"
            class="pl-4 border-l-2 border-primary-300 dark:border-primary-700"
          >
            <p class="font-medium text-gray-900 dark:text-white">
              {{ feature.name }}
            </p>
            <p
              v-if="feature.description"
              class="text-sm text-gray-500 dark:text-gray-400 line-clamp-2"
            >
              {{ feature.description }}
            </p>
          </li>
        </ul>
      </div>
    </div>

    <!-- Complete Button -->
    <div
      class="flex justify-center pt-4 transition-all duration-500 delay-300"
      :class="showContent ? 'opacity-100 translate-y-0' : 'opacity-0 translate-y-4'"
    >
      <UButton
        data-testid="view-sheet-button"
        size="lg"
        @click="handleComplete"
      >
        View Character Sheet
      </UButton>
    </div>
  </div>
</template>

<style scoped>
.confetti-piece {
  position: absolute;
  width: 10px;
  height: 10px;
  top: -10px;
  animation: confetti-fall 3s ease-out forwards;
}

@keyframes confetti-fall {
  0% {
    transform: translateY(0) rotate(0deg);
    opacity: 1;
  }
  100% {
    transform: translateY(100vh) rotate(720deg);
    opacity: 0;
  }
}
</style>
```

**Step 4: Run tests**

```bash
docker compose exec nuxt npm run test -- tests/components/character/levelup/StepLevelUpSummary.test.ts
# Expected: PASS
```

**Step 5: Commit**

```bash
git add app/components/character/levelup/StepLevelUpSummary.vue
git add tests/components/character/levelup/StepLevelUpSummary.test.ts
git commit -m "feat: Add celebration summary step with confetti animation"
```

---

## Task 8: Level-Up Wizard Modal Container

Create the main wizard modal that orchestrates all steps.

**Files:**
- Create: `app/components/character/levelup/LevelUpWizard.vue`
- Create: `app/components/character/levelup/LevelUpSidebar.vue`
- Create: `tests/components/character/levelup/LevelUpWizard.test.ts`

**Step 1: Write the test**

```typescript
// tests/components/character/levelup/LevelUpWizard.test.ts
import { describe, it, expect, beforeEach, vi } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import { setActivePinia, createPinia } from 'pinia'
import LevelUpWizard from '~/components/character/levelup/LevelUpWizard.vue'

// Mock the store
const mockStore = {
  isOpen: true,
  characterId: 1,
  publicId: 'test-char-xxxx',
  currentStepName: 'hit-points',
  levelUpResult: null,
  closeWizard: vi.fn(),
  needsClassSelection: false
}

vi.mock('~/stores/characterLevelUp', () => ({
  useCharacterLevelUpStore: vi.fn(() => mockStore)
}))

describe('LevelUpWizard', () => {
  beforeEach(() => {
    setActivePinia(createPinia())
    vi.clearAllMocks()
  })

  it('renders when open', async () => {
    const wrapper = await mountSuspended(LevelUpWizard)

    expect(wrapper.find('[data-testid="level-up-wizard"]').exists()).toBe(true)
  })

  it('shows sidebar with steps', async () => {
    const wrapper = await mountSuspended(LevelUpWizard)

    expect(wrapper.find('[data-testid="level-up-sidebar"]').exists()).toBe(true)
  })

  it('closes when close button clicked', async () => {
    const wrapper = await mountSuspended(LevelUpWizard)

    await wrapper.find('[data-testid="close-button"]').trigger('click')
    expect(mockStore.closeWizard).toHaveBeenCalled()
  })
})
```

**Step 2: Run test to verify it fails**

```bash
docker compose exec nuxt npm run test -- tests/components/character/levelup/LevelUpWizard.test.ts
# Expected: FAIL - component doesn't exist
```

**Step 3: Create LevelUpSidebar component**

```vue
<!-- app/components/character/levelup/LevelUpSidebar.vue -->
<script setup lang="ts">
import { useLevelUpWizard } from '~/composables/useLevelUpWizard'
import { useCharacterLevelUpStore } from '~/stores/characterLevelUp'

const store = useCharacterLevelUpStore()
const { activeSteps, currentStepIndex, progressPercent, goToStep } = useLevelUpWizard()

function getStepStatus(stepIndex: number): 'completed' | 'current' | 'future' {
  if (stepIndex < currentStepIndex.value) return 'completed'
  if (stepIndex === currentStepIndex.value) return 'current'
  return 'future'
}

function handleStepClick(stepName: string, stepIndex: number) {
  // Only allow navigating to completed steps
  if (stepIndex < currentStepIndex.value) {
    goToStep(stepName)
  }
}
</script>

<template>
  <aside
    data-testid="level-up-sidebar"
    class="bg-white dark:bg-gray-800 border-r border-gray-200 dark:border-gray-700 flex flex-col h-full"
  >
    <!-- Header -->
    <div class="p-4 border-b border-gray-200 dark:border-gray-700">
      <h2 class="text-lg font-semibold text-gray-900 dark:text-white">
        Level Up
      </h2>
      <div class="mt-2">
        <div class="h-2 bg-gray-200 dark:bg-gray-700 rounded-full overflow-hidden">
          <div
            class="h-full bg-primary transition-all duration-300"
            :style="{ width: `${progressPercent}%` }"
          />
        </div>
        <p class="text-xs text-gray-500 dark:text-gray-400 mt-1">
          {{ progressPercent }}% complete
        </p>
      </div>
    </div>

    <!-- Step List -->
    <nav class="flex-1 overflow-y-auto p-2">
      <ul class="space-y-1">
        <li
          v-for="(step, index) in activeSteps"
          :key="step.name"
        >
          <button
            type="button"
            class="w-full flex items-center gap-3 px-3 py-2 rounded-lg text-left transition-colors"
            :class="{
              'bg-primary-50 dark:bg-primary-900/20 text-primary-700 dark:text-primary-300': getStepStatus(index) === 'current',
              'text-gray-900 dark:text-white hover:bg-gray-100 dark:hover:bg-gray-700 cursor-pointer': getStepStatus(index) === 'completed',
              'text-gray-400 dark:text-gray-500 cursor-not-allowed': getStepStatus(index) === 'future'
            }"
            :disabled="getStepStatus(index) === 'future'"
            @click="handleStepClick(step.name, index)"
          >
            <!-- Step indicator -->
            <span
              class="flex-shrink-0 w-6 h-6 flex items-center justify-center rounded-full text-xs font-medium"
              :class="{
                'bg-primary text-white': getStepStatus(index) === 'completed',
                'bg-primary-100 dark:bg-primary-900 text-primary-700 dark:text-primary-300 ring-2 ring-primary': getStepStatus(index) === 'current',
                'bg-gray-200 dark:bg-gray-700 text-gray-500': getStepStatus(index) === 'future'
              }"
            >
              <UIcon
                v-if="getStepStatus(index) === 'completed'"
                name="i-heroicons-check"
                class="w-4 h-4"
              />
              <span v-else>{{ index + 1 }}</span>
            </span>

            <!-- Step label -->
            <span class="flex items-center gap-2 flex-1 min-w-0">
              <UIcon
                :name="step.icon"
                class="w-4 h-4 flex-shrink-0"
              />
              <span class="truncate">{{ step.label }}</span>
            </span>
          </button>
        </li>
      </ul>
    </nav>
  </aside>
</template>
```

**Step 4: Create LevelUpWizard component**

```vue
<!-- app/components/character/levelup/LevelUpWizard.vue -->
<script setup lang="ts">
import { storeToRefs } from 'pinia'
import { useCharacterLevelUpStore } from '~/stores/characterLevelUp'
import { useLevelUpWizard } from '~/composables/useLevelUpWizard'

const store = useCharacterLevelUpStore()
const {
  isOpen,
  characterId,
  publicId,
  currentStepName,
  levelUpResult,
  selectedClassSlug,
  characterClasses,
  isLoading,
  error
} = storeToRefs(store)

const { nextStep } = useLevelUpWizard()

// Track HP gained for summary
const hpGained = ref<number>(0)

// Get character stats for HP calculation (CON modifier, hit die)
const { apiFetch } = useApi()
const characterStats = ref<{ constitution_modifier: number } | null>(null)

// Fetch character stats when wizard opens
watch(isOpen, async (open) => {
  if (open && publicId.value) {
    try {
      const response = await apiFetch<{ data: { constitution_modifier: number } }>(
        `/characters/${publicId.value}/stats`
      )
      characterStats.value = response.data
    } catch (e) {
      console.error('Failed to fetch character stats:', e)
    }
  }
})

// Get hit die for selected class
const hitDie = computed(() => {
  if (!selectedClassSlug.value || !characterClasses.value.length) return 8
  const cls = characterClasses.value.find(
    c => c.class?.full_slug === selectedClassSlug.value || c.class?.slug === selectedClassSlug.value
  )
  return cls?.class?.hit_die ?? 8
})

const conModifier = computed(() => characterStats.value?.constitution_modifier ?? 0)

// Get class name for summary
const className = computed(() => {
  if (!selectedClassSlug.value || !characterClasses.value.length) return 'Character'
  const cls = characterClasses.value.find(
    c => c.class?.full_slug === selectedClassSlug.value || c.class?.slug === selectedClassSlug.value
  )
  return cls?.class?.name ?? 'Character'
})

function handleHpChoice(hp: number) {
  hpGained.value = hp
}

function handleComplete() {
  // Emit event so parent can refresh character data
  emit('level-up-complete')
}

const emit = defineEmits<{
  'level-up-complete': []
}>()
</script>

<template>
  <UModal
    v-model:open="isOpen"
    :ui="{ width: 'max-w-6xl' }"
    fullscreen
    prevent-close
  >
    <div
      data-testid="level-up-wizard"
      class="flex h-full"
    >
      <!-- Sidebar -->
      <div class="w-64 flex-shrink-0">
        <CharacterLevelupLevelUpSidebar />
      </div>

      <!-- Main Content -->
      <div class="flex-1 flex flex-col">
        <!-- Header with close button -->
        <div class="flex items-center justify-between p-4 border-b border-gray-200 dark:border-gray-700">
          <h1 class="text-xl font-semibold text-gray-900 dark:text-white">
            Level Up
          </h1>
          <UButton
            data-testid="close-button"
            variant="ghost"
            color="gray"
            icon="i-heroicons-x-mark"
            @click="store.closeWizard()"
          />
        </div>

        <!-- Step Content -->
        <div class="flex-1 overflow-y-auto p-8">
          <!-- Error State -->
          <UAlert
            v-if="error"
            color="error"
            icon="i-heroicons-exclamation-circle"
            :title="error"
            class="mb-6"
          />

          <!-- Loading State -->
          <div
            v-if="isLoading"
            class="flex justify-center py-12"
          >
            <UIcon
              name="i-heroicons-arrow-path"
              class="w-8 h-8 animate-spin text-primary"
            />
          </div>

          <!-- Step Components -->
          <template v-else>
            <!-- Class Selection Step -->
            <div v-if="currentStepName === 'class-selection'">
              <!-- TODO: Implement class selection for multiclass -->
              <p class="text-center text-gray-500">
                Class selection coming soon...
              </p>
            </div>

            <!-- Hit Points Step -->
            <CharacterLevelupStepHitPoints
              v-else-if="currentStepName === 'hit-points'"
              :hit-die="hitDie"
              :con-modifier="conModifier"
              @choice-made="handleHpChoice"
            />

            <!-- ASI/Feat Step (reuse existing) -->
            <CharacterWizardStepFeats
              v-else-if="currentStepName === 'asi-feat'"
            />

            <!-- Summary Step -->
            <CharacterLevelupStepLevelUpSummary
              v-else-if="currentStepName === 'summary' && levelUpResult"
              :level-up-result="levelUpResult"
              :class-name="className"
              :hp-gained="hpGained"
              @complete="handleComplete"
            />
          </template>
        </div>
      </div>
    </div>
  </UModal>
</template>
```

**Step 5: Run tests**

```bash
docker compose exec nuxt npm run test -- tests/components/character/levelup/LevelUpWizard.test.ts
# Expected: PASS
```

**Step 6: Commit**

```bash
git add app/components/character/levelup/LevelUpWizard.vue
git add app/components/character/levelup/LevelUpSidebar.vue
git add tests/components/character/levelup/LevelUpWizard.test.ts
git commit -m "feat: Add level-up wizard modal container with sidebar"
```

---

## Task 9: Wire Up Character Sheet Page

Connect the level-up button to open the wizard modal.

**Files:**
- Modify: `app/pages/characters/[publicId]/index.vue`

**Step 1: Update character sheet page**

Add the wizard modal and wire up the level-up button:

```vue
<!-- Add to app/pages/characters/[publicId]/index.vue -->
<script setup lang="ts">
// Add import
import { useCharacterLevelUpStore } from '~/stores/characterLevelUp'

// Add store reference
const levelUpStore = useCharacterLevelUpStore()

// Add handler function
function handleLevelUp() {
  if (!character.value?.data) return

  const char = character.value.data
  levelUpStore.openWizard(
    char.id,
    char.public_id,
    char.classes ?? [],
    char.classes?.reduce((sum, c) => sum + (c.level || 0), 0) ?? 1
  )
}

// Add refresh handler for after level-up
async function handleLevelUpComplete() {
  // Refresh character data
  await refresh()
}
</script>

<template>
  <!-- Add @level-up handler to Header -->
  <CharacterSheetHeader
    :character="character.data"
    @level-up="handleLevelUp"
  />

  <!-- Add wizard modal at end of template -->
  <CharacterLevelupLevelUpWizard
    @level-up-complete="handleLevelUpComplete"
  />
</template>
```

**Step 2: Test manually in browser**

1. Navigate to a complete character sheet
2. Click "Level Up" button
3. Verify wizard modal opens
4. Test HP roll/average selection
5. Verify summary shows
6. Click "View Character Sheet"
7. Verify modal closes and data refreshes

**Step 3: Commit**

```bash
git add app/pages/characters/\[publicId\]/index.vue
git commit -m "feat: Wire up level-up wizard to character sheet"
```

---

## Task 10: Integration Testing

Run the full test suite and fix any issues.

**Step 1: Run all level-up tests**

```bash
docker compose exec nuxt npm run test -- tests/components/character/levelup/ tests/stores/characterLevelUp.test.ts tests/composables/useLevelUpWizard.test.ts
# Expected: All PASS
```

**Step 2: Run full test suite**

```bash
docker compose exec nuxt npm run test
# Expected: All tests pass
```

**Step 3: Run typecheck**

```bash
docker compose exec nuxt npm run typecheck
# Expected: No errors
```

**Step 4: Run linter**

```bash
docker compose exec nuxt npm run lint:fix
# Expected: No errors
```

**Step 5: Final commit**

```bash
git add -A
git commit -m "test: Add integration tests for level-up wizard"
```

---

## Summary

**New Files Created:**
- `server/api/characters/[id]/classes/[classSlug]/level-up.post.ts`
- `app/stores/characterLevelUp.ts`
- `app/composables/useLevelUpWizard.ts`
- `app/components/character/levelup/LevelUpWizard.vue`
- `app/components/character/levelup/LevelUpSidebar.vue`
- `app/components/character/levelup/StepHitPoints.vue`
- `app/components/character/levelup/HitDieRoller.vue`
- `app/components/character/levelup/StepLevelUpSummary.vue`
- `tests/stores/characterLevelUp.test.ts`
- `tests/composables/useLevelUpWizard.test.ts`
- `tests/components/character/levelup/*.test.ts`

**Files Modified:**
- `app/types/character.ts`
- `app/components/character/sheet/Header.vue`
- `app/pages/characters/[publicId]/index.vue`

**Reused Components (no changes needed):**
- `app/components/character/wizard/StepFeats.vue`
- `app/components/character/wizard/StepSpells.vue`
- `app/components/character/wizard/StepSubclass.vue`
- `app/components/character/wizard/StepProficiencies.vue`
