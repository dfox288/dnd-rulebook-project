# Level-Up Wizard Page-Based Routing Design

**Issue:** #494
**Date:** 2025-12-11
**Status:** Approved

## Problem

The current level-up wizard uses inline component switching within a single page (`/characters/:publicId/level-up`). This causes:

1. **No URL persistence** - Refreshing loses current step
2. **Complex state management** - Step visibility scattered in v-if chains
3. **Vue unmount errors** - Async components unmounting during navigation
4. **Inconsistency** - Different pattern than character creation wizard

## Solution

Refactor to use URL-based routing matching the character creation wizard pattern.

## Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Routing approach | Single dynamic `[step].vue` | Consistency with creation wizard |
| Step granularity | 9 separate steps | Maps to API pending choice types |
| Entry behavior | Preview page with "Begin" button | Prevents accidental level-ups |
| Multi-class selection | On preview page, not separate step | Streamlined UX |

---

## URL Structure

```
/characters/:publicId/level-up/           → Entry/preview page (index.vue)
/characters/:publicId/level-up/:step      → Dynamic step ([step].vue)
```

### Step Slugs

| Slug | Component | Visibility |
|------|-----------|------------|
| `class-selection` | StepClassSelection | Multi-class only |
| `subclass` | StepSubclassChoice | When subclass unlocked |
| `hit-points` | StepHitPoints | Always |
| `asi-feat` | StepAsiFeat | When ASI available |
| `feature-choices` | StepFeatureChoices | Fighting style, expertise, etc. |
| `spells` | StepSpells | Spellcasters |
| `languages` | StepLanguages | When language choices exist |
| `proficiencies` | StepProficiencies | Multi-class proficiency grants |
| `summary` | StepSummary | Always (final) |

---

## File Structure

### Pages

```
app/pages/characters/[publicId]/level-up/
├── index.vue      → Preview page with "Begin Level Up" button
└── [step].vue     → Dynamic step handler with component registry
```

### Components

```
app/components/character/levelup/
├── WizardLayout.vue          # Main layout with sidebar slot
├── WizardSidebar.vue         # Step navigation sidebar
├── PreviewCard.vue           # Preview content for index page
├── StepClassSelection.vue    # Multi-class picker
├── StepSubclassChoice.vue    # Subclass selection
├── StepHitPoints.vue         # HP roll/average choice
├── StepAsiFeat.vue           # ASI/Feat selection
├── StepFeatureChoices.vue    # Fighting style, expertise, etc.
├── StepSpells.vue            # Spell selection
├── StepLanguages.vue         # Language choices
├── StepProficiencies.vue     # Proficiency choices
└── StepSummary.vue           # Level-up summary
```

---

## Entry/Preview Page (`index.vue`)

### Purpose
Show character info and preview of upcoming choices before triggering the level-up API.

### Flow
1. User clicks "Level Up" on character sheet → lands here
2. Page fetches character data + stats
3. Shows preview: "Level 4 → Level 5" with upcoming choices
4. For multi-class: shows class selection dropdown
5. User clicks "Begin Level Up" → triggers `levelUp(classSlug)` API → redirects to first step

### Preview Content
```
┌─────────────────────────────────────────────┐
│  Level Up: Thorn Ironforge                  │
│  Fighter 4 → Fighter 5                      │
│                                             │
│  What's ahead:                              │
│  • Choose HP increase (roll or average)     │
│  • Select 1 Fighting Style                  │
│  • Extra Attack feature unlocked            │
│                                             │
│  [Begin Level Up]                           │
└─────────────────────────────────────────────┘
```

---

## Dynamic Step Page (`[step].vue`)

### Component Registry
```typescript
const stepComponents: Record<string, Component> = {
  'class-selection': defineAsyncComponent(() => import('~/components/character/levelup/StepClassSelection.vue')),
  'subclass': defineAsyncComponent(() => import('~/components/character/levelup/StepSubclassChoice.vue')),
  'hit-points': defineAsyncComponent(() => import('~/components/character/levelup/StepHitPoints.vue')),
  'asi-feat': defineAsyncComponent(() => import('~/components/character/levelup/StepAsiFeat.vue')),
  'feature-choices': defineAsyncComponent(() => import('~/components/character/levelup/StepFeatureChoices.vue')),
  'spells': defineAsyncComponent(() => import('~/components/character/levelup/StepSpells.vue')),
  'languages': defineAsyncComponent(() => import('~/components/character/levelup/StepLanguages.vue')),
  'proficiencies': defineAsyncComponent(() => import('~/components/character/levelup/StepProficiencies.vue')),
  'summary': defineAsyncComponent(() => import('~/components/character/levelup/StepSummary.vue')),
}
```

### Template Structure
```vue
<CharacterLevelupWizardLayout>
  <template #sidebar>
    <CharacterLevelupWizardSidebar :active-steps="activeSteps" :current-step="currentStep" />
  </template>

  <Suspense>
    <component :is="stepComponent" />
    <template #fallback>
      <StepSkeleton />
    </template>
  </Suspense>
</CharacterLevelupWizardLayout>
```

---

## Navigation Composable

### `useLevelUpWizard.ts`

```typescript
const stepRegistry = [
  { name: 'class-selection', label: 'Class', isVisible: () => store.isMulticlass },
  { name: 'subclass', label: 'Subclass', isVisible: () => store.hasSubclassChoice },
  { name: 'hit-points', label: 'Hit Points', isVisible: () => true },
  { name: 'asi-feat', label: 'ASI/Feat', isVisible: () => store.hasAsiChoice },
  { name: 'feature-choices', label: 'Features', isVisible: () => store.hasFeatureChoices },
  { name: 'spells', label: 'Spells', isVisible: () => store.hasSpellChoices },
  { name: 'languages', label: 'Languages', isVisible: () => store.hasLanguageChoices },
  { name: 'proficiencies', label: 'Proficiencies', isVisible: () => store.hasProficiencyChoices },
  { name: 'summary', label: 'Summary', isVisible: () => true },
]

// URL-based navigation
const nextStep = () => {
  const next = activeSteps.value[currentStepIndex.value + 1]
  if (next) navigateTo(`/characters/${publicId}/level-up/${next.name}`)
}

const previousStep = () => {
  const prev = activeSteps.value[currentStepIndex.value - 1]
  if (prev) navigateTo(`/characters/${publicId}/level-up/${prev.name}`)
}

const goToStep = (stepName: string) => {
  navigateTo(`/characters/${publicId}/level-up/${stepName}`)
}
```

---

## Store Changes (`characterLevelUp.ts`)

### Retained State
```typescript
// Character context
characterId: number | null
publicId: string | null
characterClasses: CharacterClass[]
totalLevel: number
selectedClassSlug: string | null

// Level-up state
levelUpResult: LevelUpResult | null
pendingChoices: PendingChoice[]
isLevelUpInProgress: boolean

// Computed visibility flags
hasSubclassChoice: computed(...)
hasAsiChoice: computed(...)
hasFeatureChoices: computed(...)
hasSpellChoices: computed(...)
hasLanguageChoices: computed(...)
hasProficiencyChoices: computed(...)
isMulticlass: computed(...)
```

### Removed
- `currentStepName` - Now from URL
- `goToStep()` - Now in composable

### Actions
- `initialize(publicId)` - Fetch character data
- `levelUp(classSlug)` - Trigger level-up API
- `fetchPendingChoices()` - Refresh after each step
- `reset()` - Clear state when leaving

---

## Edge Case Handling

| Scenario | Behavior |
|----------|----------|
| Direct URL to step, no level-up started | Redirect to `index.vue` (preview) |
| Direct URL to completed step | Skip to next visible step or summary |
| Direct URL to invalid step slug | Redirect to first valid step |
| Browser back from summary | Returns to previous step |
| Refresh mid-wizard | Resumes at same step (URL persisted) |

### Guard Logic
```typescript
// On mount / route change
if (!store.isLevelUpInProgress) {
  return navigateTo(`/characters/${publicId}/level-up`)
}

if (!activeSteps.value.find(s => s.name === stepName)) {
  return navigateTo(`/characters/${publicId}/level-up/${activeSteps.value[0].name}`)
}
```

---

## Migration Notes

### Files to Create
- `app/pages/characters/[publicId]/level-up/[step].vue`
- `app/components/character/levelup/WizardLayout.vue`
- `app/components/character/levelup/WizardSidebar.vue`
- `app/components/character/levelup/PreviewCard.vue`
- `app/components/character/levelup/StepClassSelection.vue`
- `app/components/character/levelup/StepAsiFeat.vue`

### Files to Modify
- `app/pages/characters/[publicId]/level-up/index.vue` - Replace with preview page
- `app/stores/characterLevelUp.ts` - Remove currentStepName, goToStep
- `app/composables/useLevelUpWizard.ts` - URL-based navigation
- Rename existing step components (drop `CharacterLevelup` prefix)

### Files to Delete
- None (existing index.vue is replaced)

---

## Benefits

1. **URL persistence** - Refresh preserves step
2. **Browser navigation** - Back/forward buttons work naturally
3. **Consistency** - Same pattern as character creation wizard
4. **Cleaner code** - No v-if chains, one component per step
5. **Better async** - Page-level Suspense handles loading
6. **Debuggability** - Step visible in URL
