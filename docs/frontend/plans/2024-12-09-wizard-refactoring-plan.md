# Wizard Components Refactoring Plan

**Created:** 2024-12-09
**Status:** Planning
**Estimated Impact:** ~2,500 lines of code reduction (~40% of wizard codebase)

## Overview

This plan addresses significant code duplication identified in the character wizard components. The wizard currently contains 21 components with ~6,500 lines of code and approximately 40% duplication.

## Goals

1. Reduce code duplication from ~40% to ~15%
2. Improve maintainability and consistency
3. Establish reusable patterns for future wizard steps
4. Maintain full backward compatibility (no API changes)

---

## Phase 1: Core Composables (High Priority)

### 1.1 Create `useWizardEntitySelection` Composable

**Purpose:** Consolidate the entity selection pattern used by StepRace, StepClass, StepBackground, and StepSubrace.

**Current Duplication:**
- 4 components with ~70% identical code
- ~800 lines of duplicated logic

**File:** `app/composables/useWizardEntitySelection.ts`

**Interface:**
```typescript
interface EntitySelectionConfig<T extends { id: number; name: string }> {
  // Identification
  entityType: 'race' | 'class' | 'background' | 'subrace' | 'subclass'

  // Data fetching
  fetchFn: () => Promise<T[]>
  cacheKey: string | ComputedRef<string>

  // Store integration
  storeAction: (entity: T) => Promise<void>
  existingSelection?: ComputedRef<T | null>

  // Optional features
  searchableFields?: Array<keyof T>
  confirmChangeMessage?: string  // For race→subrace dependency
}

interface EntitySelectionReturn<T> {
  // Data
  entities: Ref<T[]>
  filtered: ComputedRef<T[]>

  // State
  loading: Ref<boolean>
  error: Ref<string | null>
  searchQuery: Ref<string>
  localSelected: Ref<T | null>

  // Validation
  canProceed: ComputedRef<boolean>

  // Actions
  handleSelect: (entity: T) => void
  confirmSelection: () => Promise<void>

  // Detail modal
  detailModal: {
    open: Ref<boolean>
    item: Ref<T | null>
    show: (entity: T) => void
    close: () => void
  }

  // Change confirmation (for dependent selections)
  changeConfirmation: {
    pending: Ref<T | null>
    modalOpen: Ref<boolean>
    confirm: () => void
    cancel: () => void
  }
}

function useWizardEntitySelection<T extends { id: number; name: string }>(
  config: EntitySelectionConfig<T>
): EntitySelectionReturn<T>
```

**Refactored StepRace Example:**
```vue
<script setup lang="ts">
import type { Race } from '~/types'

const store = useCharacterWizardStore()
const { nextStep } = useCharacterWizard()
const { apiFetch } = useApi()
const { sourceFilterString, selections } = storeToRefs(store)

const {
  entities: races,
  filtered: filteredRaces,
  loading,
  error,
  searchQuery,
  localSelected,
  canProceed,
  handleSelect,
  confirmSelection,
  detailModal,
  changeConfirmation
} = useWizardEntitySelection<Race>({
  entityType: 'race',
  fetchFn: async () => {
    const filter = `is_subrace=false${sourceFilterString.value ? ` AND ${sourceFilterString.value}` : ''}`
    const response = await apiFetch<{ data: Race[] }>(`/races?per_page=50&filter=${encodeURIComponent(filter)}`)
    return response.data
  },
  cacheKey: computed(() => `builder-races-${sourceFilterString.value}`),
  storeAction: (race) => store.selectRace(race),
  existingSelection: computed(() => selections.value.race),
  confirmChangeMessage: 'Changing your race will clear your subrace selection.'
})

// Handle successful selection
watch(canProceed, async (ready) => {
  if (ready && localSelected.value) {
    await confirmSelection()
    nextStep()
  }
})
</script>

<template>
  <WizardEntitySelectionLayout
    title="Choose Your Race"
    description="Your race determines your character's physical traits and natural abilities"
    :search-query="searchQuery"
    search-placeholder="Search races..."
    :loading="loading"
    :error="error"
    :entities="filteredRaces"
    :selected-id="localSelected?.id"
    :can-proceed="canProceed"
    :continue-text="`Continue with ${localSelected?.name || 'Selection'}`"
    @search="searchQuery = $event"
    @confirm="confirmSelection"
  >
    <template #card="{ entity }">
      <CharacterPickerRacePickerCard
        :race="entity"
        :selected="localSelected?.id === entity.id"
        @select="handleSelect"
        @view-details="detailModal.show(entity)"
      />
    </template>

    <template #modal>
      <CharacterPickerRaceDetailModal
        :race="detailModal.item"
        :open="detailModal.open"
        @close="detailModal.close"
      />
    </template>

    <template #change-confirmation>
      <WizardChangeConfirmationModal
        v-model:open="changeConfirmation.modalOpen"
        title="Change Race?"
        :message="changeConfirmation.pending ? 'Changing your race will clear your subrace selection.' : ''"
        @confirm="changeConfirmation.confirm"
        @cancel="changeConfirmation.cancel"
      />
    </template>
  </WizardEntitySelectionLayout>
</template>
```

**Estimated Reduction:** 800 lines → 200 lines (75%)

---

### 1.2 Create `useWizardChoiceSelection` Composable

**Purpose:** Consolidate the choice selection pattern used by StepProficiencies, StepLanguages, StepEquipment, StepSpells, and StepFeats.

**Current Duplication:**
- 5 components with ~50% identical code
- ~1,500 lines of duplicated logic

**File:** `app/composables/useWizardChoiceSelection.ts`

**Interface:**
```typescript
type ChoiceType = 'proficiency' | 'language' | 'equipment' | 'spell' | 'feat'
type SelectionMode = 'multi' | 'single'  // Equipment uses single, others use multi

interface ChoiceSelectionConfig {
  choiceType: ChoiceType
  selectionMode?: SelectionMode  // Default: 'multi'

  // Optional: Custom validation beyond "all choices complete"
  customValidation?: (choices: PendingChoice[], selections: Map<string, Set<string>>) => boolean

  // Optional: Transform options before display
  transformOptions?: (options: unknown[]) => DisplayOption[]

  // Optional: Group choices by source
  groupBySource?: boolean
}

interface DisplayOption {
  id: string
  name: string
  description?: string
  disabled?: boolean
  disabledReason?: string
}

interface ChoiceSelectionReturn {
  // Data from unified choices
  choices: ComputedRef<PendingChoice[]>
  choicesBySource: ComputedRef<GroupedChoices[]>

  // State
  pending: Ref<boolean>
  error: Ref<string | null>
  localSelections: Ref<Map<string, Set<string>>>
  isSaving: Ref<boolean>

  // Helpers
  getSelectedCount: (choiceId: string) => number
  isOptionSelected: (choiceId: string, optionId: string) => boolean
  isOptionDisabled: (choiceId: string, optionId: string) => boolean
  getDisabledReason: (choiceId: string, optionId: string) => string | null

  // Validation
  allComplete: ComputedRef<boolean>
  canProceed: ComputedRef<boolean>

  // Actions
  handleToggle: (choice: PendingChoice, optionId: string) => void
  handleContinue: () => Promise<void>

  // Options fetching (for dynamic endpoints)
  fetchOptionsForChoice: (choice: PendingChoice) => Promise<void>
  getDisplayOptions: (choice: PendingChoice) => DisplayOption[]
}

function useWizardChoiceSelection(
  characterId: ComputedRef<number | null>,
  config: ChoiceSelectionConfig
): ChoiceSelectionReturn
```

**Refactored StepLanguages Example:**
```vue
<script setup lang="ts">
const store = useCharacterWizardStore()
const { nextStep } = useCharacterWizard()

const {
  choices,
  choicesBySource,
  pending,
  error,
  localSelections,
  isSaving,
  getSelectedCount,
  isOptionSelected,
  isOptionDisabled,
  getDisabledReason,
  allComplete,
  handleToggle,
  handleContinue
} = useWizardChoiceSelection(
  computed(() => store.characterId),
  {
    choiceType: 'language',
    groupBySource: true
  }
)

// Fetch already-known languages for cross-choice validation
const { data: knownLanguages } = await useAsyncData(...)

async function onContinue() {
  await handleContinue()
  await store.syncWithBackend()
  nextStep()
}
</script>

<template>
  <WizardChoiceSelectionLayout
    title="Choose Your Languages"
    description="Your race and background grant the following languages"
    :pending="pending"
    :error="error"
    :saving="isSaving"
    :can-proceed="allComplete"
    continue-text="Continue with Languages"
    @continue="onContinue"
  >
    <!-- Known languages section -->
    <template #header>
      <LanguagesKnownSection :languages="knownLanguages" />
    </template>

    <!-- Choice groups -->
    <template #choices>
      <WizardChoiceGroup
        v-for="group in choicesBySource"
        :key="group.source"
        :title="group.label"
        :subtitle="group.entityName"
      >
        <WizardChoiceGrid
          v-for="choice in group.choices"
          :key="choice.id"
          :choice="choice"
          :selected-count="getSelectedCount(choice.id)"
          :options="choice.options"
          :is-selected="(id) => isOptionSelected(choice.id, id)"
          :is-disabled="(id) => isOptionDisabled(choice.id, id)"
          :get-disabled-reason="(id) => getDisabledReason(choice.id, id)"
          @toggle="(id) => handleToggle(choice, id)"
        />
      </WizardChoiceGroup>
    </template>
  </WizardChoiceSelectionLayout>
</template>
```

**Estimated Reduction:** 1,500 lines → 500 lines (67%)

---

## Phase 2: Shared UI Components (Medium Priority)

### 2.1 `WizardStepHeader`

**Purpose:** Consistent header across all wizard steps.

**File:** `app/components/character/wizard/WizardStepHeader.vue`

```vue
<script setup lang="ts">
defineProps<{
  title: string
  description?: string
}>()
</script>

<template>
  <div class="text-center">
    <h2 class="text-2xl font-bold text-gray-900 dark:text-white">
      {{ title }}
    </h2>
    <p v-if="description" class="mt-2 text-gray-600 dark:text-gray-400">
      {{ description }}
    </p>
  </div>
</template>
```

### 2.2 `WizardLoadingState`

**Purpose:** Consistent loading spinner with configurable color.

**File:** `app/components/character/wizard/WizardLoadingState.vue`

```vue
<script setup lang="ts">
defineProps<{
  color?: string  // Tailwind color class suffix (e.g., 'race', 'class', 'primary')
}>()
</script>

<template>
  <div class="flex justify-center py-12">
    <UIcon
      name="i-heroicons-arrow-path"
      :class="[
        'w-8 h-8 animate-spin',
        color ? `text-${color}-500` : 'text-primary'
      ]"
    />
  </div>
</template>
```

### 2.3 `WizardEmptyState`

**Purpose:** Consistent empty state display.

**File:** `app/components/character/wizard/WizardEmptyState.vue`

```vue
<script setup lang="ts">
defineProps<{
  icon?: string
  title: string
  description?: string
}>()
</script>

<template>
  <div class="text-center py-12">
    <UIcon
      :name="icon ?? 'i-heroicons-inbox'"
      class="w-12 h-12 text-gray-400 mx-auto mb-4"
    />
    <p class="text-gray-600 dark:text-gray-400">
      {{ title }}
    </p>
    <p v-if="description" class="text-sm text-gray-500 mt-1">
      {{ description }}
    </p>
  </div>
</template>
```

### 2.4 `WizardContinueButton`

**Purpose:** Consistent continue button with loading/disabled states.

**File:** `app/components/character/wizard/WizardContinueButton.vue`

```vue
<script setup lang="ts">
defineProps<{
  disabled?: boolean
  loading?: boolean
  text?: string
}>()

defineEmits<{
  click: []
}>()
</script>

<template>
  <div class="flex justify-center pt-4">
    <UButton
      data-testid="continue-btn"
      size="lg"
      :disabled="disabled || loading"
      :loading="loading"
      @click="$emit('click')"
    >
      {{ text ?? 'Continue' }}
    </UButton>
  </div>
</template>
```

### 2.5 `WizardChoiceGrid`

**Purpose:** Reusable grid for multi-select choice options.

**File:** `app/components/character/wizard/WizardChoiceGrid.vue`

```vue
<script setup lang="ts">
interface Option {
  id: string
  name: string
  description?: string
}

defineProps<{
  options: Option[]
  selectedIds: Set<string>
  disabledIds?: Set<string>
  getDisabledReason?: (id: string) => string | null
  columns?: 2 | 3 | 4
}>()

defineEmits<{
  toggle: [id: string]
}>()
</script>

<template>
  <div :class="[
    'grid gap-3',
    columns === 2 ? 'grid-cols-2' : '',
    columns === 3 ? 'grid-cols-2 md:grid-cols-3' : '',
    columns === 4 ? 'grid-cols-2 md:grid-cols-3 lg:grid-cols-4' : 'grid-cols-2 md:grid-cols-3 lg:grid-cols-4'
  ]">
    <button
      v-for="option in options"
      :key="option.id"
      type="button"
      class="p-3 rounded-lg border text-left transition-all"
      :class="{
        'border-primary bg-primary/10': selectedIds.has(option.id),
        'border-gray-200 dark:border-gray-700 hover:border-primary/50': !selectedIds.has(option.id) && !disabledIds?.has(option.id),
        'border-gray-200 dark:border-gray-700 opacity-50 cursor-not-allowed': disabledIds?.has(option.id)
      }"
      :disabled="disabledIds?.has(option.id)"
      @click="$emit('toggle', option.id)"
    >
      <div class="flex items-center gap-2">
        <UIcon
          :name="selectedIds.has(option.id) ? 'i-heroicons-check-circle-solid' : 'i-heroicons-circle'"
          class="w-5 h-5"
          :class="{
            'text-primary': selectedIds.has(option.id),
            'text-gray-400': !selectedIds.has(option.id)
          }"
        />
        <span class="font-medium">{{ option.name }}</span>
      </div>
      <p
        v-if="getDisabledReason?.(option.id)"
        class="text-xs text-gray-400 dark:text-gray-500 mt-1 ml-7"
      >
        {{ getDisabledReason(option.id) }}
      </p>
      <p
        v-else-if="option.description"
        class="text-xs text-gray-500 dark:text-gray-400 mt-1 ml-7"
      >
        {{ option.description }}
      </p>
    </button>
  </div>
</template>
```

### 2.6 `WizardChangeConfirmationModal`

**Purpose:** Reusable confirmation modal for dependent selection changes.

**File:** `app/components/character/wizard/WizardChangeConfirmationModal.vue`

```vue
<script setup lang="ts">
defineProps<{
  open: boolean
  title: string
  message: string
  confirmText?: string
  cancelText?: string
}>()

defineEmits<{
  'update:open': [value: boolean]
  confirm: []
  cancel: []
}>()
</script>

<template>
  <UModal :open="open" @update:open="$emit('update:open', $event)">
    <template #content>
      <div class="p-6">
        <div class="flex items-center gap-3 mb-4">
          <div class="flex-shrink-0 w-10 h-10 rounded-full bg-amber-100 dark:bg-amber-900/30 flex items-center justify-center">
            <UIcon
              name="i-heroicons-exclamation-triangle"
              class="w-5 h-5 text-amber-600 dark:text-amber-400"
            />
          </div>
          <h3 class="text-lg font-semibold text-gray-900 dark:text-white">
            {{ title }}
          </h3>
        </div>

        <p class="text-gray-600 dark:text-gray-400 mb-6">
          {{ message }}
        </p>

        <div class="flex justify-end gap-3">
          <UButton variant="outline" @click="$emit('cancel')">
            {{ cancelText ?? 'Cancel' }}
          </UButton>
          <UButton color="primary" @click="$emit('confirm')">
            {{ confirmText ?? 'Continue' }}
          </UButton>
        </div>
      </div>
    </template>
  </UModal>
</template>
```

---

## Phase 3: Standardization (Medium Priority)

### 3.1 Error Handling Utilities

**File:** `app/utils/wizardErrors.ts`

```typescript
import { logger } from '~/utils/logger'

interface WizardErrorOptions {
  title?: string
  description?: string
  logContext?: string
}

/**
 * Standard error handler for wizard operations
 * Logs error and shows toast notification
 */
export function handleWizardError(
  error: unknown,
  toast: ReturnType<typeof useToast>,
  options: WizardErrorOptions = {}
) {
  const {
    title = 'Save Failed',
    description = 'Unable to save your selection. Please try again.',
    logContext = 'Wizard error'
  } = options

  logger.error(`${logContext}:`, error)

  toast.add({
    title,
    description,
    color: 'error',
    icon: 'i-heroicons-exclamation-circle'
  })
}

/**
 * Pre-configured error handlers for common wizard operations
 */
export const wizardErrors = {
  saveFailed: (error: unknown, toast: ReturnType<typeof useToast>) =>
    handleWizardError(error, toast, { logContext: 'Failed to save selection' }),

  loadFailed: (error: unknown, toast: ReturnType<typeof useToast>, entity: string) =>
    handleWizardError(error, toast, {
      title: `Failed to load ${entity}`,
      description: 'Please try again or refresh the page.',
      logContext: `Failed to load ${entity}`
    }),

  choiceResolveFailed: (error: unknown, toast: ReturnType<typeof useToast>, choiceType: string) =>
    handleWizardError(error, toast, {
      title: `Failed to save ${choiceType}`,
      description: 'Please try again.',
      logContext: `Failed to resolve ${choiceType} choice`
    })
}
```

### 3.2 Standardize Continue Button Text

Create constants for consistent button text:

**File:** `app/constants/wizard.ts`

```typescript
export const WIZARD_BUTTON_TEXT = {
  continueWith: (name: string) => `Continue with ${name}`,
  saveAndContinue: 'Save & Continue',
  continueToReview: 'Continue to Review',
  createCharacter: 'Create Character'
} as const

export const WIZARD_STEP_TITLES = {
  race: 'Choose Your Race',
  class: 'Choose Your Class',
  background: 'Choose Your Background',
  // ... etc
} as const
```

---

## Phase 4: Cleanup (Low Priority)

### 4.1 Rename Components

| Current Name | New Name |
|--------------|----------|
| `CharacterPickerRacePickerCard` | `CharacterRaceCard` |
| `CharacterPickerClassPickerCard` | `CharacterClassCard` |
| `CharacterPickerBackgroundPickerCard` | `CharacterBackgroundCard` |
| `CharacterPickerSpellPickerCard` | `CharacterSpellCard` |

**Migration Strategy:**
1. Create new components with new names
2. Have old components re-export new ones (deprecation period)
3. Update all imports
4. Remove old components

### 4.2 Refactor StepSpells Modal

Replace manual modal management with `useDetailModal` composable:

```typescript
// Before (StepSpells.vue:234-251)
const detailModalOpen = ref(false)
const detailSpell = ref<Spell | null>(null)

function handleViewDetails(spell: Spell) {
  detailSpell.value = spell
  detailModalOpen.value = true
}

function handleCloseModal() {
  detailModalOpen.value = false
  detailSpell.value = null
}

// After
const { open: detailModalOpen, item: detailSpell, show: handleViewDetails, close: handleCloseModal } = useDetailModal<Spell>()
```

---

## Implementation Order

| Phase | Task | Priority | Estimated Effort | Dependencies |
|-------|------|----------|------------------|--------------|
| 1.1 | `useWizardEntitySelection` composable | High | 3-4 hours | None |
| 1.2 | `useWizardChoiceSelection` composable | High | 4-5 hours | None |
| 2.1-2.6 | Shared UI components | Medium | 2-3 hours | None |
| 3.1 | Error handling utilities | Medium | 1 hour | None |
| 3.2 | Button text constants | Medium | 30 min | None |
| 4.1 | Rename components | Low | 2 hours | All above |
| 4.2 | StepSpells modal refactor | Low | 30 min | None |

**Total Estimated Effort:** 13-16 hours

---

## Testing Strategy

1. **Unit Tests:** Test new composables in isolation
2. **Component Tests:** Update existing wizard step tests
3. **Integration Tests:** Run character builder stress test after each phase
4. **Manual Testing:** Full character creation flow in browser

---

## Success Metrics

| Metric | Before | Target |
|--------|--------|--------|
| Lines of Code | ~6,500 | ~4,000 |
| Duplication % | ~40% | ~15% |
| Unique Patterns | 5+ | 2 |
| Inconsistencies | 8 | 0 |

---

## Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| Breaking existing functionality | High | Extensive test coverage, incremental refactoring |
| Type safety regressions | Medium | Strict TypeScript, no `any` types |
| Performance impact | Low | Composables are lightweight, no runtime overhead |

---

## Related Issues

**Epic:** [#426 - Wizard Components Refactoring](https://github.com/dfox288/dnd-rulebook-project/issues/426)

### Phase 1: Core Composables
- [#420 - Create useWizardEntitySelection composable](https://github.com/dfox288/dnd-rulebook-project/issues/420)
- [#421 - Create useWizardChoiceSelection composable](https://github.com/dfox288/dnd-rulebook-project/issues/421)

### Phase 2: Shared UI Components
- [#422 - Create shared wizard UI components](https://github.com/dfox288/dnd-rulebook-project/issues/422)

### Phase 3: Standardization
- [#423 - Standardize wizard error handling](https://github.com/dfox288/dnd-rulebook-project/issues/423)

### Phase 4: Cleanup
- [#424 - Rename wizard picker card components](https://github.com/dfox288/dnd-rulebook-project/issues/424)
- [#425 - Refactor StepSpells to use useDetailModal](https://github.com/dfox288/dnd-rulebook-project/issues/425)
