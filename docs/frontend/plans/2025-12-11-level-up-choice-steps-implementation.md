# Level-Up Choice Steps Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Add spell, feature, language, and proficiency choice steps to the level-up wizard, with reusable components that also work in the character creation wizard.

**Architecture:** Props-based abstraction for step components - accept `characterId` and `nextStep` as props instead of importing stores directly. This allows the same components to work with both `characterLevelUpStore` and `characterWizardStore`. Core logic remains in composables (`useUnifiedChoices`, `useWizardChoiceSelection`).

**Tech Stack:** Vue 3, Nuxt 4, Pinia, TypeScript, Vitest, NuxtUI 4

**Issues:** #484 (Spells), #485 (Features), #486 (Proficiencies), #489 (Languages)

**Design Doc:** `wrapper/docs/frontend/plans/2025-12-11-level-up-choice-steps-design.md`

---

## Phase 1: Store & Composable Foundation

### Task 1.1: Add Choice Type Groupings to useUnifiedChoices

**Files:**
- Modify: `app/composables/useUnifiedChoices.ts`
- Test: `tests/composables/useUnifiedChoices.spec.ts`

**Step 1: Write the failing test**

```typescript
// tests/composables/useUnifiedChoices.spec.ts
// Add to existing test file

describe('choicesByType groupings', () => {
  it('groups fighting_style choices', async () => {
    const characterId = ref(1)
    const { choicesByType, choices } = useUnifiedChoices(characterId)

    // Simulate choices loaded
    choices.value = [
      { id: 'fs-1', type: 'fighting_style', quantity: 1, source: 'class', source_name: 'Fighter' }
    ]

    expect(choicesByType.value.fightingStyles).toHaveLength(1)
    expect(choicesByType.value.fightingStyles[0].type).toBe('fighting_style')
  })

  it('groups expertise choices', async () => {
    const characterId = ref(1)
    const { choicesByType, choices } = useUnifiedChoices(characterId)

    choices.value = [
      { id: 'exp-1', type: 'expertise', quantity: 2, source: 'class', source_name: 'Rogue' }
    ]

    expect(choicesByType.value.expertise).toHaveLength(1)
    expect(choicesByType.value.expertise[0].quantity).toBe(2)
  })
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- tests/composables/useUnifiedChoices.spec.ts --reporter=verbose`
Expected: FAIL - `choicesByType.value.fightingStyles` is undefined

**Step 3: Write minimal implementation**

```typescript
// app/composables/useUnifiedChoices.ts
// In choicesByType computed, add after existing groupings:

const choicesByType = computed(() => ({
  proficiencies: choices.value.filter(c => c.type === 'proficiency'),
  languages: choices.value.filter(c => c.type === 'language'),
  equipment: choices.value.filter(c => c.type === 'equipment'),
  equipmentMode: choices.value.find(c => c.type === 'equipment_mode') ?? null,
  spells: choices.value.filter(c => c.type === 'spell'),
  asiOrFeat: choices.value.filter(c => c.type === 'asi_or_feat'),
  optionalFeatures: choices.value.filter(c => c.type === 'optional_feature'),
  abilityScores: choices.value.filter(c => c.type === 'ability_score'),
  feats: choices.value.filter(c => c.type === 'feat'),
  sizes: choices.value.filter(c => c.type === 'size'),
  hitPoints: choices.value.filter(c => c.type === 'hit_points'),
  // NEW: Feature choice groupings
  fightingStyles: choices.value.filter(c => c.type === 'fighting_style'),
  expertise: choices.value.filter(c => c.type === 'expertise')
}))
```

**Step 4: Run test to verify it passes**

Run: `docker compose exec nuxt npm run test -- tests/composables/useUnifiedChoices.spec.ts --reporter=verbose`
Expected: PASS

**Step 5: Commit**

```bash
git add app/composables/useUnifiedChoices.ts tests/composables/useUnifiedChoices.spec.ts
git commit -m "feat(composables): add fighting_style and expertise groupings to useUnifiedChoices"
```

---

### Task 1.2: Add Pending Choices State to Level-Up Store

**Files:**
- Modify: `app/stores/characterLevelUp.ts`
- Test: `tests/stores/characterLevelUp.spec.ts`

**Step 1: Write the failing test**

```typescript
// tests/stores/characterLevelUp.spec.ts
// Add to existing test file or create new

import { setActivePinia, createPinia } from 'pinia'
import { useCharacterLevelUpStore } from '~/stores/characterLevelUp'

describe('characterLevelUp store - pending choices', () => {
  beforeEach(() => {
    setActivePinia(createPinia())
  })

  it('initializes with empty pending choices', () => {
    const store = useCharacterLevelUpStore()
    expect(store.pendingChoices).toEqual([])
  })

  it('computes hasSpellChoices from pending choices', () => {
    const store = useCharacterLevelUpStore()
    expect(store.hasSpellChoices).toBe(false)

    store.pendingChoices = [
      { id: 'spell-1', type: 'spell', quantity: 2 }
    ]

    expect(store.hasSpellChoices).toBe(true)
  })

  it('computes hasFeatureChoices for fighting_style', () => {
    const store = useCharacterLevelUpStore()
    store.pendingChoices = [
      { id: 'fs-1', type: 'fighting_style', quantity: 1 }
    ]

    expect(store.hasFeatureChoices).toBe(true)
  })

  it('computes hasFeatureChoices for expertise', () => {
    const store = useCharacterLevelUpStore()
    store.pendingChoices = [
      { id: 'exp-1', type: 'expertise', quantity: 2 }
    ]

    expect(store.hasFeatureChoices).toBe(true)
  })

  it('computes hasFeatureChoices for optional_feature', () => {
    const store = useCharacterLevelUpStore()
    store.pendingChoices = [
      { id: 'of-1', type: 'optional_feature', quantity: 2 }
    ]

    expect(store.hasFeatureChoices).toBe(true)
  })

  it('computes hasLanguageChoices from pending choices', () => {
    const store = useCharacterLevelUpStore()
    store.pendingChoices = [
      { id: 'lang-1', type: 'language', quantity: 3 }
    ]

    expect(store.hasLanguageChoices).toBe(true)
  })

  it('computes hasProficiencyChoices from pending choices', () => {
    const store = useCharacterLevelUpStore()
    store.pendingChoices = [
      { id: 'prof-1', type: 'proficiency', quantity: 3 }
    ]

    expect(store.hasProficiencyChoices).toBe(true)
  })
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- tests/stores/characterLevelUp.spec.ts --reporter=verbose`
Expected: FAIL - `store.pendingChoices` is undefined

**Step 3: Write minimal implementation**

```typescript
// app/stores/characterLevelUp.ts
// Add to STATE section:

import type { components } from '~/types/api/generated'
type PendingChoice = components['schemas']['PendingChoiceResource']

// In the store function, add to state:
const pendingChoices = ref<PendingChoice[]>([])

// Add to COMPUTED section:

/** Does character have pending spell choices? */
const hasSpellChoices = computed(() =>
  pendingChoices.value.some(c => c.type === 'spell')
)

/** Does character have pending feature choices (fighting_style, expertise, optional_feature)? */
const hasFeatureChoices = computed(() =>
  pendingChoices.value.some(c =>
    ['fighting_style', 'expertise', 'optional_feature'].includes(c.type)
  )
)

/** Does character have pending language choices? */
const hasLanguageChoices = computed(() =>
  pendingChoices.value.some(c => c.type === 'language')
)

/** Does character have pending proficiency choices? */
const hasProficiencyChoices = computed(() =>
  pendingChoices.value.some(c => c.type === 'proficiency')
)

// Add to RETURN section:
return {
  // ... existing
  pendingChoices,
  hasSpellChoices,
  hasFeatureChoices,
  hasLanguageChoices,
  hasProficiencyChoices
}
```

**Step 4: Run test to verify it passes**

Run: `docker compose exec nuxt npm run test -- tests/stores/characterLevelUp.spec.ts --reporter=verbose`
Expected: PASS

**Step 5: Commit**

```bash
git add app/stores/characterLevelUp.ts tests/stores/characterLevelUp.spec.ts
git commit -m "feat(store): add pending choices state and visibility computeds to level-up store"
```

---

### Task 1.3: Add fetchPendingChoices and refreshChoices Actions

**Files:**
- Modify: `app/stores/characterLevelUp.ts`
- Test: `tests/stores/characterLevelUp.spec.ts`

**Step 1: Write the failing test**

```typescript
// tests/stores/characterLevelUp.spec.ts
// Add to existing describe block

describe('fetchPendingChoices', () => {
  it('fetches and stores pending choices', async () => {
    const store = useCharacterLevelUpStore()
    store.characterId = 1
    store.publicId = 'test-char-abc'

    // Mock will be handled by test setup
    await store.fetchPendingChoices()

    // Verify it attempted to fetch (actual mock setup in test helper)
    expect(store.pendingChoices).toBeDefined()
  })
})

describe('refreshChoices', () => {
  it('calls fetchPendingChoices', async () => {
    const store = useCharacterLevelUpStore()
    store.characterId = 1
    store.publicId = 'test-char-abc'

    const fetchSpy = vi.spyOn(store, 'fetchPendingChoices')
    await store.refreshChoices()

    expect(fetchSpy).toHaveBeenCalled()
  })
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- tests/stores/characterLevelUp.spec.ts --reporter=verbose`
Expected: FAIL - `store.fetchPendingChoices` is not a function

**Step 3: Write minimal implementation**

```typescript
// app/stores/characterLevelUp.ts
// Add to ACTIONS section:

/**
 * Fetch pending choices for the character
 * Called after level-up and after each choice is resolved
 */
async function fetchPendingChoices(): Promise<void> {
  if (!publicId.value) return

  try {
    const response = await apiFetch<{ data: PendingChoice[] }>(
      `/characters/${publicId.value}/pending-choices`
    )
    pendingChoices.value = response.data
  } catch (e) {
    // Don't fail silently - log but don't throw
    console.error('Failed to fetch pending choices:', e)
  }
}

/**
 * Refresh choices - alias for fetchPendingChoices
 * Used after completing a step to update visibility of downstream steps
 */
async function refreshChoices(): Promise<void> {
  await fetchPendingChoices()
}

// Update levelUp action to fetch choices after success:
async function levelUp(classSlug: string): Promise<LevelUpResult> {
  // ... existing code ...

  levelUpResult.value = result
  selectedClassSlug.value = classSlug

  // NEW: Fetch pending choices after level-up
  await fetchPendingChoices()

  return result
}

// Add to RETURN:
return {
  // ... existing
  fetchPendingChoices,
  refreshChoices
}
```

**Step 4: Run test to verify it passes**

Run: `docker compose exec nuxt npm run test -- tests/stores/characterLevelUp.spec.ts --reporter=verbose`
Expected: PASS

**Step 5: Commit**

```bash
git add app/stores/characterLevelUp.ts tests/stores/characterLevelUp.spec.ts
git commit -m "feat(store): add fetchPendingChoices and refreshChoices actions"
```

---

### Task 1.4: Update Level-Up Step Registry

**Files:**
- Modify: `app/composables/useLevelUpWizard.ts`
- Test: `tests/composables/useLevelUpWizard.spec.ts`

**Step 1: Write the failing test**

```typescript
// tests/composables/useLevelUpWizard.spec.ts
import { setActivePinia, createPinia } from 'pinia'
import { useLevelUpWizard } from '~/composables/useLevelUpWizard'
import { useCharacterLevelUpStore } from '~/stores/characterLevelUp'

describe('useLevelUpWizard - step registry', () => {
  beforeEach(() => {
    setActivePinia(createPinia())
  })

  it('includes feature-choices step in registry', () => {
    const { stepRegistry } = useLevelUpWizard()
    const step = stepRegistry.find(s => s.name === 'feature-choices')

    expect(step).toBeDefined()
    expect(step?.label).toBe('Features')
    expect(step?.icon).toBe('i-heroicons-puzzle-piece')
  })

  it('includes languages step in registry', () => {
    const { stepRegistry } = useLevelUpWizard()
    const step = stepRegistry.find(s => s.name === 'languages')

    expect(step).toBeDefined()
    expect(step?.label).toBe('Languages')
  })

  it('includes proficiencies step in registry', () => {
    const { stepRegistry } = useLevelUpWizard()
    const step = stepRegistry.find(s => s.name === 'proficiencies')

    expect(step).toBeDefined()
    expect(step?.label).toBe('Proficiencies')
  })

  it('shows spells step when hasSpellChoices is true', () => {
    const store = useCharacterLevelUpStore()
    store.pendingChoices = [{ id: 'spell-1', type: 'spell', quantity: 2 }]

    const { stepRegistry } = useLevelUpWizard()
    const spellStep = stepRegistry.find(s => s.name === 'spells')

    expect(spellStep?.visible()).toBe(true)
    expect(spellStep?.shouldSkip?.()).toBe(false)
  })

  it('hides spells step when hasSpellChoices is false', () => {
    const store = useCharacterLevelUpStore()
    store.pendingChoices = []

    const { stepRegistry } = useLevelUpWizard()
    const spellStep = stepRegistry.find(s => s.name === 'spells')

    expect(spellStep?.visible()).toBe(false)
    expect(spellStep?.shouldSkip?.()).toBe(true)
  })

  it('orders steps correctly', () => {
    const { stepRegistry } = useLevelUpWizard()
    const names = stepRegistry.map(s => s.name)

    const hitPointsIdx = names.indexOf('hit-points')
    const asiIdx = names.indexOf('asi-feat')
    const featuresIdx = names.indexOf('feature-choices')
    const spellsIdx = names.indexOf('spells')
    const languagesIdx = names.indexOf('languages')
    const proficienciesIdx = names.indexOf('proficiencies')
    const summaryIdx = names.indexOf('summary')

    // Verify order: hit-points < asi-feat < feature-choices < spells < languages < proficiencies < summary
    expect(hitPointsIdx).toBeLessThan(asiIdx)
    expect(asiIdx).toBeLessThan(featuresIdx)
    expect(featuresIdx).toBeLessThan(spellsIdx)
    expect(spellsIdx).toBeLessThan(languagesIdx)
    expect(languagesIdx).toBeLessThan(proficienciesIdx)
    expect(proficienciesIdx).toBeLessThan(summaryIdx)
  })
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- tests/composables/useLevelUpWizard.spec.ts --reporter=verbose`
Expected: FAIL - step 'feature-choices' not found

**Step 3: Write minimal implementation**

```typescript
// app/composables/useLevelUpWizard.ts
// Replace the step registry with updated version:

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
      visible: () => false, // Deferred - will be set based on level-up result
      shouldSkip: () => true
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
      name: 'feature-choices',
      label: 'Features',
      icon: 'i-heroicons-puzzle-piece',
      visible: () => store.hasFeatureChoices,
      shouldSkip: () => !store.hasFeatureChoices
    },
    {
      name: 'spells',
      label: 'Spells',
      icon: 'i-heroicons-sparkles',
      visible: () => store.hasSpellChoices,
      shouldSkip: () => !store.hasSpellChoices
    },
    {
      name: 'languages',
      label: 'Languages',
      icon: 'i-heroicons-language',
      visible: () => store.hasLanguageChoices,
      shouldSkip: () => !store.hasLanguageChoices
    },
    {
      name: 'proficiencies',
      label: 'Proficiencies',
      icon: 'i-heroicons-academic-cap',
      visible: () => store.hasProficiencyChoices,
      shouldSkip: () => !store.hasProficiencyChoices
    },
    {
      name: 'summary',
      label: 'Summary',
      icon: 'i-heroicons-trophy',
      visible: () => true
    }
  ]
}
```

**Step 4: Run test to verify it passes**

Run: `docker compose exec nuxt npm run test -- tests/composables/useLevelUpWizard.spec.ts --reporter=verbose`
Expected: PASS

**Step 5: Commit**

```bash
git add app/composables/useLevelUpWizard.ts tests/composables/useLevelUpWizard.spec.ts
git commit -m "feat(composables): add feature, language, proficiency steps to level-up registry"
```

---

## Phase 2: Component Refactoring (Props-Based)

### Task 2.1: Refactor StepSpells to Accept Props

**Files:**
- Modify: `app/components/character/wizard/StepSpells.vue`
- Modify: `tests/components/character/wizard/StepSpells.spec.ts`

**Step 1: Write the failing test**

```typescript
// tests/components/character/wizard/StepSpells.spec.ts
// Add new test for props-based usage

describe('StepSpells - props-based usage', () => {
  it('accepts characterId as prop', async () => {
    const wrapper = await mountSuspended(StepSpells, {
      props: {
        characterId: 123,
        nextStep: vi.fn()
      }
    })

    expect(wrapper.props('characterId')).toBe(123)
  })

  it('accepts nextStep function as prop', async () => {
    const nextStepFn = vi.fn()
    const wrapper = await mountSuspended(StepSpells, {
      props: {
        characterId: 123,
        nextStep: nextStepFn
      }
    })

    expect(wrapper.props('nextStep')).toBe(nextStepFn)
  })

  it('uses characterId prop for useUnifiedChoices', async () => {
    // This tests that the component uses props.characterId, not store.characterId
    const wrapper = await mountSuspended(StepSpells, {
      props: {
        characterId: 456,
        nextStep: vi.fn()
      }
    })

    // Component should initialize with the prop value
    expect(wrapper.vm).toBeDefined()
  })
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- tests/components/character/wizard/StepSpells.spec.ts --reporter=verbose`
Expected: FAIL - props validation error or characterId not accepted

**Step 3: Write minimal implementation**

```typescript
// app/components/character/wizard/StepSpells.vue
// Update script setup section:

<script setup lang="ts">
import { storeToRefs } from 'pinia'
import type { Spell } from '~/types'
import type { components } from '~/types/api/generated'
import { useCharacterWizardStore } from '~/stores/characterWizard'
import { useCharacterWizard } from '~/composables/useCharacterWizard'
import { useWizardChoiceSelection } from '~/composables/useWizardChoiceSelection'
import { normalizeEndpoint } from '~/composables/useApi'
import { wizardErrors } from '~/utils/wizardErrors'

type PendingChoice = components['schemas']['PendingChoiceResource']

// Props for store-agnostic usage
const props = withDefaults(defineProps<{
  characterId?: number
  nextStep?: () => void
  spellcastingStats?: {
    ability: string
    spell_save_dc: number
    spell_attack_bonus: number
  } | null
}>(), {
  characterId: undefined,
  nextStep: undefined,
  spellcastingStats: null
})

// Fallback to store if props not provided (backward compatibility)
const store = useCharacterWizardStore()
const wizardNav = useCharacterWizard()

// Use prop or store value
const effectiveCharacterId = computed(() => props.characterId ?? store.characterId)
const effectiveNextStep = computed(() => props.nextStep ?? wizardNav.nextStep)

const {
  selections,
  stats,
  isLoading,
  error: storeError
} = storeToRefs(store)

// API client for fetching spell options
const { apiFetch } = useApi()

// Use unified choices composable with effective character ID
const {
  choicesByType,
  pending: loadingChoices,
  error: choicesError,
  fetchChoices,
  resolveChoice
} = useUnifiedChoices(effectiveCharacterId)

// ... rest of component unchanged, but update handleContinue:

async function handleContinue() {
  try {
    await saveAllChoices()
    effectiveNextStep.value()
  } catch (e) {
    wizardErrors.choiceResolveFailed(e, toast, 'spell')
  }
}
</script>
```

**Step 4: Run test to verify it passes**

Run: `docker compose exec nuxt npm run test -- tests/components/character/wizard/StepSpells.spec.ts --reporter=verbose`
Expected: PASS

**Step 5: Commit**

```bash
git add app/components/character/wizard/StepSpells.vue tests/components/character/wizard/StepSpells.spec.ts
git commit -m "refactor(StepSpells): accept characterId and nextStep as props for reusability"
```

---

### Task 2.2: Refactor StepLanguages to Accept Props

**Files:**
- Modify: `app/components/character/wizard/StepLanguages.vue`
- Test: `tests/components/character/wizard/StepLanguages.spec.ts`

**Step 1: Read current implementation**

Run: Read `app/components/character/wizard/StepLanguages.vue` to understand current structure.

**Step 2: Write the failing test**

```typescript
// tests/components/character/wizard/StepLanguages.spec.ts

describe('StepLanguages - props-based usage', () => {
  it('accepts characterId as prop', async () => {
    const wrapper = await mountSuspended(StepLanguages, {
      props: {
        characterId: 123,
        nextStep: vi.fn()
      }
    })

    expect(wrapper.props('characterId')).toBe(123)
  })

  it('accepts nextStep function as prop', async () => {
    const nextStepFn = vi.fn()
    const wrapper = await mountSuspended(StepLanguages, {
      props: {
        characterId: 123,
        nextStep: nextStepFn
      }
    })

    expect(wrapper.props('nextStep')).toBe(nextStepFn)
  })
})
```

**Step 3: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- tests/components/character/wizard/StepLanguages.spec.ts --reporter=verbose`
Expected: FAIL

**Step 4: Write minimal implementation**

Apply same pattern as StepSpells:
- Add props with defaults
- Create `effectiveCharacterId` and `effectiveNextStep` computed
- Use effective values throughout

**Step 5: Run test to verify it passes**

Run: `docker compose exec nuxt npm run test -- tests/components/character/wizard/StepLanguages.spec.ts --reporter=verbose`
Expected: PASS

**Step 6: Commit**

```bash
git add app/components/character/wizard/StepLanguages.vue tests/components/character/wizard/StepLanguages.spec.ts
git commit -m "refactor(StepLanguages): accept characterId and nextStep as props for reusability"
```

---

### Task 2.3: Refactor StepProficiencies to Accept Props and Add Feat Source

**Files:**
- Modify: `app/components/character/wizard/StepProficiencies.vue`
- Test: `tests/components/character/wizard/StepProficiencies.spec.ts`

**Step 1: Write the failing test**

```typescript
// tests/components/character/wizard/StepProficiencies.spec.ts

describe('StepProficiencies - props-based usage', () => {
  it('accepts characterId as prop', async () => {
    const wrapper = await mountSuspended(StepProficiencies, {
      props: {
        characterId: 123,
        nextStep: vi.fn()
      }
    })

    expect(wrapper.props('characterId')).toBe(123)
  })
})

describe('StepProficiencies - feat source support', () => {
  it('handles proficiency choices with source feat', async () => {
    // This test verifies 'feat' is a valid source type
    const wrapper = await mountSuspended(StepProficiencies, {
      props: {
        characterId: 123,
        nextStep: vi.fn()
      }
    })

    // Component should not error when receiving feat-sourced choices
    expect(wrapper.vm).toBeDefined()
  })
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- tests/components/character/wizard/StepProficiencies.spec.ts --reporter=verbose`
Expected: FAIL

**Step 3: Write minimal implementation**

```typescript
// app/components/character/wizard/StepProficiencies.vue
// Update ProficiencySource type to include 'feat':

type ProficiencySource = 'class' | 'race' | 'background' | 'subclass_feature' | 'feat'

// Update proficiencyChoicesBySource computed to handle feat:
const proficiencyChoicesBySource = computed<ChoicesBySource[]>(() => {
  const choices = choicesByType.value.proficiencies
  if (!choices || choices.length === 0) return []

  const sources: ChoicesBySource[] = []

  const bySource = {
    class: choices.filter(c => c.source === 'class'),
    race: choices.filter(c => c.source === 'race'),
    background: choices.filter(c => c.source === 'background'),
    subclass_feature: choices.filter(c => c.source === 'subclass_feature'),
    feat: choices.filter(c => c.source === 'feat')  // NEW
  }

  // ... existing class, race, background, subclass_feature handling ...

  // NEW: Handle feat source
  if (bySource.feat.length > 0) {
    sources.push({
      source: 'feat',
      label: 'From Feat',
      entityName: bySource.feat[0]?.source_name ?? 'Feat',
      choices: bySource.feat
    })
  }

  return sources
})

// Also update getSourceLabel:
function getSourceLabel(source: string): string {
  const labels: Record<string, string> = {
    class: 'Class',
    race: 'Race',
    background: 'Background',
    subclass_feature: 'Subclass',
    feat: 'Feat'  // NEW
  }
  return labels[source] ?? source
}
```

Also apply the same props pattern as previous components.

**Step 4: Run test to verify it passes**

Run: `docker compose exec nuxt npm run test -- tests/components/character/wizard/StepProficiencies.spec.ts --reporter=verbose`
Expected: PASS

**Step 5: Commit**

```bash
git add app/components/character/wizard/StepProficiencies.vue tests/components/character/wizard/StepProficiencies.spec.ts
git commit -m "refactor(StepProficiencies): accept props and add feat source support"
```

---

## Phase 3: New StepFeatureChoices Component

### Task 3.1: Create StepFeatureChoices Component Skeleton

**Files:**
- Create: `app/components/character/wizard/StepFeatureChoices.vue`
- Create: `tests/components/character/wizard/StepFeatureChoices.spec.ts`

**Step 1: Write the failing test**

```typescript
// tests/components/character/wizard/StepFeatureChoices.spec.ts
import { mountSuspended } from '@nuxt/test-utils/runtime'
import { setActivePinia, createPinia } from 'pinia'
import StepFeatureChoices from '~/components/character/wizard/StepFeatureChoices.vue'

describe('StepFeatureChoices', () => {
  beforeEach(() => {
    setActivePinia(createPinia())
  })

  it('renders without error', async () => {
    const wrapper = await mountSuspended(StepFeatureChoices, {
      props: {
        characterId: 123,
        nextStep: vi.fn()
      }
    })

    expect(wrapper.exists()).toBe(true)
  })

  it('accepts required props', async () => {
    const nextStepFn = vi.fn()
    const wrapper = await mountSuspended(StepFeatureChoices, {
      props: {
        characterId: 456,
        nextStep: nextStepFn
      }
    })

    expect(wrapper.props('characterId')).toBe(456)
    expect(wrapper.props('nextStep')).toBe(nextStepFn)
  })

  it('displays header', async () => {
    const wrapper = await mountSuspended(StepFeatureChoices, {
      props: {
        characterId: 123,
        nextStep: vi.fn()
      }
    })

    expect(wrapper.text()).toContain('Feature Choices')
  })
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- tests/components/character/wizard/StepFeatureChoices.spec.ts --reporter=verbose`
Expected: FAIL - component not found

**Step 3: Write minimal implementation**

```vue
<!-- app/components/character/wizard/StepFeatureChoices.vue -->
<script setup lang="ts">
import type { components } from '~/types/api/generated'
import { useWizardChoiceSelection } from '~/composables/useWizardChoiceSelection'
import { wizardErrors } from '~/utils/wizardErrors'

type PendingChoice = components['schemas']['PendingChoiceResource']

const props = defineProps<{
  characterId: number
  nextStep: () => void
}>()

const toast = useToast()

// Fetch choices for this character
const {
  choicesByType,
  pending: loadingChoices,
  error: choicesError,
  fetchChoices,
  resolveChoice
} = useUnifiedChoices(computed(() => props.characterId))

// Feature choice categories
const fightingStyleChoices = computed(() => choicesByType.value.fightingStyles ?? [])
const expertiseChoices = computed(() => choicesByType.value.expertise ?? [])
const optionalFeatureChoices = computed(() => choicesByType.value.optionalFeatures ?? [])

// Combined: any feature choices exist?
const hasAnyChoices = computed(() =>
  fightingStyleChoices.value.length > 0
  || expertiseChoices.value.length > 0
  || optionalFeatureChoices.value.length > 0
)

// Fetch on mount
onMounted(async () => {
  await fetchChoices(['fighting_style', 'expertise', 'optional_feature'])
})

// Continue handler
async function handleContinue() {
  props.nextStep()
}
</script>

<template>
  <div class="space-y-8">
    <!-- Header -->
    <div class="text-center">
      <h2 class="text-2xl font-bold text-gray-900 dark:text-white">
        Feature Choices
      </h2>
      <p class="mt-2 text-gray-600 dark:text-gray-400">
        Select your class features
      </p>
    </div>

    <!-- Error State -->
    <UAlert
      v-if="choicesError"
      color="error"
      icon="i-heroicons-exclamation-circle"
      :title="choicesError"
    />

    <!-- Loading State -->
    <div
      v-if="loadingChoices"
      class="flex justify-center py-12"
    >
      <UIcon
        name="i-heroicons-arrow-path"
        class="w-8 h-8 animate-spin text-primary"
      />
    </div>

    <!-- Empty State -->
    <div
      v-else-if="!hasAnyChoices"
      class="text-center py-12"
    >
      <p class="text-gray-600 dark:text-gray-400">
        No feature choices available.
      </p>
    </div>

    <!-- Continue Button -->
    <div class="flex justify-center pt-4">
      <UButton
        data-testid="continue-btn"
        size="lg"
        :loading="loadingChoices"
        @click="handleContinue"
      >
        Continue
      </UButton>
    </div>
  </div>
</template>
```

**Step 4: Run test to verify it passes**

Run: `docker compose exec nuxt npm run test -- tests/components/character/wizard/StepFeatureChoices.spec.ts --reporter=verbose`
Expected: PASS

**Step 5: Commit**

```bash
git add app/components/character/wizard/StepFeatureChoices.vue tests/components/character/wizard/StepFeatureChoices.spec.ts
git commit -m "feat(StepFeatureChoices): create component skeleton with props"
```

---

### Task 3.2: Add Fighting Style Selection UI

**Files:**
- Modify: `app/components/character/wizard/StepFeatureChoices.vue`
- Modify: `tests/components/character/wizard/StepFeatureChoices.spec.ts`

**Step 1: Write the failing test**

```typescript
// tests/components/character/wizard/StepFeatureChoices.spec.ts

describe('StepFeatureChoices - Fighting Style', () => {
  it('displays fighting style section when choices exist', async () => {
    // Mock the composable to return fighting style choices
    const wrapper = await mountSuspended(StepFeatureChoices, {
      props: {
        characterId: 123,
        nextStep: vi.fn()
      }
    })

    // Simulate choices loaded
    // Note: actual test will need proper mocking of useUnifiedChoices

    expect(wrapper.find('[data-testid="fighting-style-section"]').exists()).toBe(true)
  })

  it('shows fighting style options as selectable cards', async () => {
    const wrapper = await mountSuspended(StepFeatureChoices, {
      props: {
        characterId: 123,
        nextStep: vi.fn()
      }
    })

    // Should render option buttons/cards
    const options = wrapper.findAll('[data-testid^="fighting-style-option-"]')
    expect(options.length).toBeGreaterThan(0)
  })
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- tests/components/character/wizard/StepFeatureChoices.spec.ts --reporter=verbose`
Expected: FAIL - section not found

**Step 3: Write minimal implementation**

```vue
<!-- Add to StepFeatureChoices.vue template, after loading state -->

<!-- Fighting Style Section -->
<section
  v-if="fightingStyleChoices.length > 0"
  data-testid="fighting-style-section"
  class="space-y-4"
>
  <div
    v-for="choice in fightingStyleChoices"
    :key="choice.id"
    class="space-y-4"
  >
    <div class="flex items-center justify-between border-b pb-2">
      <h3 class="text-lg font-semibold text-gray-900 dark:text-white">
        Fighting Style ({{ choice.source_name }})
      </h3>
      <UBadge
        :color="getSelectedCount(choice.id) >= choice.quantity ? 'success' : 'warning'"
        variant="subtle"
        size="md"
      >
        {{ getSelectedCount(choice.id) }} of {{ choice.quantity }}
      </UBadge>
    </div>

    <div class="grid grid-cols-1 md:grid-cols-2 gap-4">
      <button
        v-for="option in getDisplayOptions(choice)"
        :key="option.id"
        :data-testid="`fighting-style-option-${option.id}`"
        type="button"
        class="p-4 rounded-lg border text-left transition-all"
        :class="{
          'border-primary bg-primary/10': isOptionSelected(choice.id, option.id),
          'border-gray-200 dark:border-gray-700 hover:border-primary/50': !isOptionSelected(choice.id, option.id) && !isOptionDisabled(choice.id, option.id),
          'opacity-50 cursor-not-allowed': isOptionDisabled(choice.id, option.id)
        }"
        :disabled="isOptionDisabled(choice.id, option.id)"
        @click="handleOptionToggle(choice, option.id)"
      >
        <div class="flex items-center gap-3">
          <UIcon
            :name="isOptionSelected(choice.id, option.id) ? 'i-heroicons-check-circle-solid' : 'i-heroicons-circle'"
            class="w-6 h-6 flex-shrink-0"
            :class="{
              'text-primary': isOptionSelected(choice.id, option.id),
              'text-gray-400': !isOptionSelected(choice.id, option.id)
            }"
          />
          <div>
            <p class="font-semibold">{{ option.name }}</p>
            <p
              v-if="option.description"
              class="text-sm text-gray-600 dark:text-gray-400 mt-1"
            >
              {{ option.description }}
            </p>
          </div>
        </div>
      </button>
    </div>
  </div>
</section>
```

Also add to script setup:

```typescript
// Use choice selection composable for fighting styles
const {
  getSelectedCount: getFightingStyleSelectedCount,
  isOptionSelected: isFightingStyleSelected,
  isOptionDisabled: isFightingStyleDisabled,
  handleToggle: handleFightingStyleToggle,
  getDisplayOptions: getFightingStyleOptions,
  fetchOptionsIfNeeded: fetchFightingStyleOptions,
  allComplete: allFightingStylesComplete,
  saveAllChoices: saveFightingStyleChoices
} = useWizardChoiceSelection(
  fightingStyleChoices,
  { resolveChoice }
)

// Load options on mount
watch(fightingStyleChoices, async (choices) => {
  for (const choice of choices) {
    await fetchFightingStyleOptions(choice)
  }
}, { immediate: true })
```

**Step 4: Run test to verify it passes**

Run: `docker compose exec nuxt npm run test -- tests/components/character/wizard/StepFeatureChoices.spec.ts --reporter=verbose`
Expected: PASS

**Step 5: Commit**

```bash
git add app/components/character/wizard/StepFeatureChoices.vue tests/components/character/wizard/StepFeatureChoices.spec.ts
git commit -m "feat(StepFeatureChoices): add fighting style selection UI"
```

---

### Task 3.3: Add Expertise Selection UI

**Files:**
- Modify: `app/components/character/wizard/StepFeatureChoices.vue`
- Modify: `tests/components/character/wizard/StepFeatureChoices.spec.ts`

**Step 1: Write the failing test**

```typescript
describe('StepFeatureChoices - Expertise', () => {
  it('displays expertise section when choices exist', async () => {
    const wrapper = await mountSuspended(StepFeatureChoices, {
      props: {
        characterId: 123,
        nextStep: vi.fn()
      }
    })

    expect(wrapper.find('[data-testid="expertise-section"]').exists()).toBe(true)
  })

  it('shows expertise skill options', async () => {
    const wrapper = await mountSuspended(StepFeatureChoices, {
      props: {
        characterId: 123,
        nextStep: vi.fn()
      }
    })

    const options = wrapper.findAll('[data-testid^="expertise-option-"]')
    expect(options.length).toBeGreaterThan(0)
  })
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- tests/components/character/wizard/StepFeatureChoices.spec.ts --reporter=verbose`
Expected: FAIL

**Step 3: Write minimal implementation**

Add expertise section similar to fighting style, with skill-specific UI (checkboxes for selecting which skills get expertise).

**Step 4: Run test to verify it passes**

**Step 5: Commit**

```bash
git add app/components/character/wizard/StepFeatureChoices.vue tests/components/character/wizard/StepFeatureChoices.spec.ts
git commit -m "feat(StepFeatureChoices): add expertise selection UI"
```

---

### Task 3.4: Add Optional Feature Selection UI (Invocations, Metamagic)

**Files:**
- Modify: `app/components/character/wizard/StepFeatureChoices.vue`
- Modify: `tests/components/character/wizard/StepFeatureChoices.spec.ts`

Similar pattern to above - add section for optional features with dynamic header based on `choice.source_name` (e.g., "Eldritch Invocations", "Metamagic Options").

---

### Task 3.5: Add Choice Completion Validation and Save

**Files:**
- Modify: `app/components/character/wizard/StepFeatureChoices.vue`
- Modify: `tests/components/character/wizard/StepFeatureChoices.spec.ts`

**Step 1: Write the failing test**

```typescript
describe('StepFeatureChoices - Completion', () => {
  it('disables continue button until all choices complete', async () => {
    const wrapper = await mountSuspended(StepFeatureChoices, {
      props: {
        characterId: 123,
        nextStep: vi.fn()
      }
    })

    const continueBtn = wrapper.find('[data-testid="continue-btn"]')
    expect(continueBtn.attributes('disabled')).toBeDefined()
  })

  it('enables continue button when all choices complete', async () => {
    // Setup with complete choices
    const wrapper = await mountSuspended(StepFeatureChoices, {
      props: {
        characterId: 123,
        nextStep: vi.fn()
      }
    })

    // After selections complete
    const continueBtn = wrapper.find('[data-testid="continue-btn"]')
    expect(continueBtn.attributes('disabled')).toBeUndefined()
  })

  it('calls nextStep after saving choices', async () => {
    const nextStepFn = vi.fn()
    const wrapper = await mountSuspended(StepFeatureChoices, {
      props: {
        characterId: 123,
        nextStep: nextStepFn
      }
    })

    await wrapper.find('[data-testid="continue-btn"]').trigger('click')

    expect(nextStepFn).toHaveBeenCalled()
  })
})
```

**Implementation:** Add `allComplete` computed that combines all choice types, update continue button to check it, save all choices before calling nextStep.

---

## Phase 4: Wire Up Level-Up Wizard

### Task 4.1: Add Step Components to LevelUpWizard.vue

**Files:**
- Modify: `app/components/character/levelup/LevelUpWizard.vue`
- Modify: `tests/components/character/levelup/LevelUpWizard.spec.ts`

**Step 1: Write the failing test**

```typescript
// tests/components/character/levelup/LevelUpWizard.spec.ts

describe('LevelUpWizard - new steps', () => {
  it('renders StepFeatureChoices when on feature-choices step', async () => {
    const store = useCharacterLevelUpStore()
    store.isOpen = true
    store.currentStepName = 'feature-choices'
    store.pendingChoices = [{ id: 'fs-1', type: 'fighting_style', quantity: 1 }]

    const wrapper = await mountSuspended(LevelUpWizard)

    expect(wrapper.findComponent({ name: 'StepFeatureChoices' }).exists()).toBe(true)
  })

  it('renders StepSpells when on spells step', async () => {
    const store = useCharacterLevelUpStore()
    store.isOpen = true
    store.currentStepName = 'spells'
    store.pendingChoices = [{ id: 'spell-1', type: 'spell', quantity: 2 }]

    const wrapper = await mountSuspended(LevelUpWizard)

    expect(wrapper.findComponent({ name: 'StepSpells' }).exists()).toBe(true)
  })

  it('passes characterId prop to step components', async () => {
    const store = useCharacterLevelUpStore()
    store.isOpen = true
    store.characterId = 999
    store.currentStepName = 'spells'

    const wrapper = await mountSuspended(LevelUpWizard)
    const stepSpells = wrapper.findComponent({ name: 'StepSpells' })

    expect(stepSpells.props('characterId')).toBe(999)
  })
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- tests/components/character/levelup/LevelUpWizard.spec.ts --reporter=verbose`
Expected: FAIL

**Step 3: Write minimal implementation**

```vue
<!-- app/components/character/levelup/LevelUpWizard.vue -->
<!-- Update template step rendering section -->

<!-- Feature Choices Step -->
<CharacterWizardStepFeatureChoices
  v-else-if="store.currentStepName === 'feature-choices'"
  :character-id="store.characterId"
  :next-step="handleFeatureChoicesComplete"
/>

<!-- Spells Step -->
<CharacterWizardStepSpells
  v-else-if="store.currentStepName === 'spells'"
  :character-id="store.characterId"
  :next-step="handleSpellsComplete"
  :spellcasting-stats="characterStats?.spellcasting"
/>

<!-- Languages Step -->
<CharacterWizardStepLanguages
  v-else-if="store.currentStepName === 'languages'"
  :character-id="store.characterId"
  :next-step="handleLanguagesComplete"
/>

<!-- Proficiencies Step -->
<CharacterWizardStepProficiencies
  v-else-if="store.currentStepName === 'proficiencies'"
  :character-id="store.characterId"
  :next-step="handleProficienciesComplete"
/>
```

Add handlers in script:

```typescript
const { nextStep } = useLevelUpWizard()

// Handlers that refresh choices before advancing
async function handleFeatureChoicesComplete() {
  await store.refreshChoices()
  nextStep()
}

async function handleSpellsComplete() {
  await store.refreshChoices()
  nextStep()
}

async function handleLanguagesComplete() {
  await store.refreshChoices()
  nextStep()
}

async function handleProficienciesComplete() {
  await store.refreshChoices()
  nextStep()
}
```

**Step 4: Run test to verify it passes**

**Step 5: Commit**

```bash
git add app/components/character/levelup/LevelUpWizard.vue tests/components/character/levelup/LevelUpWizard.spec.ts
git commit -m "feat(LevelUpWizard): wire up feature, spell, language, proficiency steps"
```

---

## Phase 5: Character Creation Wizard Integration

### Task 5.1: Add hasFeatureChoices to Character Wizard Store

**Files:**
- Modify: `app/stores/characterWizard.ts`
- Test: `tests/stores/characterWizard.spec.ts`

Add `hasFeatureChoices` computed similar to `hasProficiencyChoices`.

---

### Task 5.2: Add Feature Choices Step to Creation Wizard Registry

**Files:**
- Modify: `app/composables/useCharacterWizard.ts`
- Test: `tests/composables/useCharacterWizard.spec.ts`

Add step after proficiencies, before languages.

---

### Task 5.3: Wire Up StepFeatureChoices in Character Creation Flow

**Files:**
- Modify: `app/pages/characters/new.vue` (or equivalent wizard page)
- Test: Existing character creation tests

---

## Phase 6: Integration Testing

### Task 6.1: Run Full Test Suite

Run: `docker compose exec nuxt npm run test`
Expected: All tests pass

### Task 6.2: Manual Testing Checklist

- [ ] Level up a Sorcerer to L3 - verify Metamagic choices appear
- [ ] Level up a Bard to L3 - verify Expertise choices appear
- [ ] Level up a Fighter to L2 - verify Fighting Style choices appear
- [ ] Level up a Wizard to L4, take Linguist feat - verify Language choices appear
- [ ] Level up a Rogue to L4, take Skilled feat - verify Proficiency choices appear
- [ ] Create a new Rogue - verify Expertise step appears at L1

### Task 6.3: Final Commit and PR

```bash
git push origin feature/issue-484-485-486-489-level-up-choice-steps

gh pr create --title "feat: Add level-up choice steps (spells, features, languages, proficiencies)" \
  --body "## Summary
- Add spell selection step to level-up wizard (#484)
- Add feature choices step (fighting style, expertise, invocations) (#485)
- Add language choices step for feats like Linguist (#489)
- Add proficiency choices step for feats like Skilled (#486)
- Refactor step components to accept props for reusability
- Add feature choices step to character creation wizard (fixes Rogue L1 Expertise)

## Test Plan
- [x] Unit tests for all new/modified components
- [x] Unit tests for store changes
- [x] Unit tests for composable changes
- [ ] Manual testing per checklist in implementation plan

Closes #484, #485, #486, #489"
```

---

## Appendix: Test Helpers

If mocking `useUnifiedChoices` becomes repetitive, create a helper:

```typescript
// tests/helpers/mockChoices.ts
export function mockUnifiedChoices(choices: PendingChoice[]) {
  return {
    choices: ref(choices),
    choicesByType: computed(() => ({
      fightingStyles: choices.filter(c => c.type === 'fighting_style'),
      expertise: choices.filter(c => c.type === 'expertise'),
      optionalFeatures: choices.filter(c => c.type === 'optional_feature'),
      // ... etc
    })),
    pending: ref(false),
    error: ref(null),
    fetchChoices: vi.fn(),
    resolveChoice: vi.fn()
  }
}
```
