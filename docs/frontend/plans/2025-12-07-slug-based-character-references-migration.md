# Frontend Migration Plan: Slug-Based Character References

**Related Backend Issue:** #288 (Epic: Slug-based character references)
**Created:** 2025-12-07
**Status:** ✅ Implementation Complete
**PR:** https://github.com/dfox288/dnd-rulebook-frontend/pull/30

## Overview

This plan documents the frontend changes required to support the backend's migration from ID-based to slug-based character references. The backend is converting all `*_id` foreign keys in character tables to `*_slug` columns using the format `{source_code}:{slug}` (e.g., `phb:high-elf`, `xge:shadow-blade`).

### Why This Change?

1. **Portability** - Characters can be exported/imported without ID collisions
2. **Reimport Safety** - Sourcebook data can be reimported without breaking existing characters
3. **Human Readability** - `phb:high-elf` is self-documenting vs opaque `race_id: 42`

### Breaking Change Notice

All existing character data will be invalidated by the backend migration. There is no backwards compatibility path - this is a clean break.

---

## Backend Progress Tracker

| Issue | Description | Status |
|-------|-------------|--------|
| #289 | Add `full_slug` to entity tables | ✅ Complete |
| #290 | Update importers to generate `full_slug` | ✅ Complete |
| #291 | Migrate character tables to slug-based references | ✅ Complete |
| #292 | Update API layer (requests, controllers, resources) | ✅ Complete |
| #293 | Update service layer | ✅ Complete |
| #294 | Add character validation endpoint | ✅ Complete |
| #295 | Add character export/import feature | 🔴 Not Started |
| #296 | Update test suite | ✅ Complete |
| #297 | Quality gates and documentation | 🔴 Not Started |
| #300 | Update fixture seeders | ✅ Complete |

**Frontend migration can begin now! All API changes are live.**

---

## Phase Overview

| Phase | Description | Backend Dependency | Status |
|-------|-------------|-------------------|--------|
| **Phase 1** | API Types Sync | After #292 merged | ✅ Complete |
| **Phase 2** | Character Wizard Store | After #292 merged | ✅ Complete |
| **Phase 3** | Nitro API Routes | After #292 merged | ✅ Complete |
| **Phase 4** | Test Fixtures & Mocks | After Phase 2-3 | ✅ Complete |
| **Phase 5** | Validation Endpoint | After #294 merged | ⏭️ Skipped (optional) |

---

## Phase 1: API Types Sync

### Trigger
Backend PR for #292 merged and deployed.

### Steps

1. **Regenerate OpenAPI types:**
   ```bash
   docker compose exec nuxt npm run types:sync
   ```

2. **Expected changes in `generated.ts`:**
   - `race_id: number` → `race_slug: string`
   - `class_id: number` → `class_slug: string`
   - `background_id: number` → `background_slug: string`
   - `spell_id: number` → `spell_slug: string`
   - `item_id: number` → `item_slug: string`
   - `subclass_id: number` → `subclass_slug: string`
   - `language_id: number` → `language_slug: string`
   - `proficiency_type_id: number` → `proficiency_type_slug: string`
   - `condition_id: number` → `condition_slug: string`
   - `optional_feature_id: number` → `optional_feature_slug: string`

3. **Manual type updates:**

#### `app/types/character.ts`

```typescript
// Lines 136-146: Update CharacterClassEntry
// Before:
export interface CharacterClassEntry {
  classId: number
  subclassId: number | null
  level: number
  isPrimary: boolean
  order: number
  classData: CharacterClass | null
  subclassData?: CharacterClass | null
}

// After:
export interface CharacterClassEntry {
  classSlug: string           // Changed from classId
  subclassSlug: string | null // Changed from subclassId
  level: number
  isPrimary: boolean
  order: number
  classData: CharacterClass | null
  subclassData?: CharacterClass | null
}
```

#### `app/types/proficiencies.ts`

```typescript
// Before:
export interface SkillProficiency {
  skill_id: number
  // ...
}

export interface ToolProficiency {
  proficiency_type_id: number
  // ...
}

// After:
export interface SkillProficiency {
  skill_slug: string
  // ...
}

export interface ToolProficiency {
  proficiency_type_slug: string
  // ...
}
```

#### `app/types/api/entities.ts`

```typescript
// Line 125: Update parent_class reference
// Before:
parent_class_id?: number | null

// After:
parent_class_slug?: string | null
```

4. **Verify compilation:**
   ```bash
   docker compose exec nuxt npm run typecheck
   ```

---

## Phase 2: Character Wizard Store

### File
`app/stores/characterWizard.ts`

### Interface Updates

```typescript
// Update Subclass interface to include full_slug
export interface Subclass {
  id: number
  name: string
  slug: string
  full_slug: string  // NEW: "phb:evoker"
  description?: string
  source?: { code: string, name: string }
}
```

### API Call Changes

| Location | Current | New |
|----------|---------|-----|
| Line 138-141 (createCharacterWithRetry) | `race_id: raceId` | `race_slug: race.full_slug` |
| Line 374 (selectRace - update) | `body: { race_id: race.id }` | `body: { race_slug: race.full_slug }` |
| Line 407 (selectSubrace) | `body: { race_id: raceId }` | `body: { race_slug: subrace?.full_slug ?? selections.value.race?.full_slug }` |
| Line 441 (selectClass - replace) | `body: { class_id: cls.id }` | `body: { class_slug: cls.full_slug }` |
| Line 447 (selectClass - add) | `body: { class_id: cls.id }` | `body: { class_slug: cls.full_slug }` |
| Line 482 (selectSubclass) | `body: { subclass_id: subclass.id }` | `body: { subclass_slug: subclass.full_slug }` |
| Line 508 (selectBackground) | `body: { background_id: background.id }` | `body: { background_slug: background.full_slug }` |

### URL Path Changes

The store uses class IDs in URL paths. Clarify with backend whether these change:

```typescript
// Current (line 439):
`/characters/${characterId.value}/classes/${selections.value.class.id}`

// Potential new format (TBD - depends on backend decision):
`/characters/${characterId.value}/classes/${selections.value.class.slug}`
// OR
`/characters/${characterId.value}/classes/${selections.value.class.full_slug}`
```

### Code Example

```typescript
// selectRace() - lines 351-390
async function selectRace(race: Race): Promise<void> {
  isLoading.value = true
  error.value = null

  try {
    if (!characterId.value) {
      const defaultName = selections.value.name || `New ${race.name}`

      // CHANGED: Use race_slug instead of race_id
      const response = await createCharacterWithRetry(defaultName, race.full_slug)
      characterId.value = response.id
      publicId.value = response.public_id

      if (!selections.value.name) {
        selections.value.name = defaultName
      }
    } else {
      // CHANGED: Use race_slug instead of race_id
      await apiFetch(`/characters/${characterId.value}`, {
        method: 'PATCH',
        body: { race_slug: race.full_slug }
      })
    }
    // ... rest unchanged
  }
}

// Update createCharacterWithRetry signature
async function createCharacterWithRetry(
  name: string,
  raceSlug: string,  // CHANGED: was raceId: number
  maxRetries = 3
): Promise<{ id: number, public_id: string }> {
  // ...
  body: {
    public_id: newPublicId,
    name,
    race_slug: raceSlug  // CHANGED
  }
}
```

---

## Phase 3: Nitro API Routes

### Routes Requiring Body Changes

| File | Current Body Fields | New Body Fields |
|------|---------------------|-----------------|
| `server/api/characters/index.post.ts` | `race_id` | `race_slug` |
| `server/api/characters/[id].patch.ts` | `race_id`, `background_id` | `race_slug`, `background_slug` |
| `server/api/characters/[id]/classes/index.post.ts` | `class_id` | `class_slug` |
| `server/api/characters/[id]/classes/[classId].put.ts` | `class_id` | `class_slug` |
| `server/api/characters/[id]/classes/[classId]/subclass.put.ts` | `subclass_id` | `subclass_slug` |
| `server/api/characters/[id]/spells.post.ts` | `spell_id` | `spell_slug` |
| `server/api/characters/[id]/equipment.post.ts` | `item_id` | `item_slug` |
| `server/api/characters/[id]/languages/sync.post.ts` | `language_ids` | `language_slugs` |

### Routes with Potential Path Parameter Changes

**Clarify with backend team:**

| Current Route | Potential New Route | Notes |
|---------------|---------------------|-------|
| `[classId].put.ts` | `[classSlug].put.ts` | If backend uses slug in URL |
| `[classId].delete.ts` | `[classSlug].delete.ts` | If backend uses slug in URL |
| `[spellId].delete.ts` | `[spellSlug].delete.ts` | Or may stay as pivot ID |
| `[equipmentId].delete.ts` | Likely unchanged | Pivot table ID, not entity reference |

### Example Route Update

```typescript
// server/api/characters/[id]/classes/index.post.ts
// Before:
export default defineEventHandler(async (event) => {
  const config = useRuntimeConfig()
  const id = getRouterParam(event, 'id')
  const body = await readBody(event)
  // body = { class_id: 1 }

  return await $fetch(`${config.apiBaseServer}/characters/${id}/classes`, {
    method: 'POST',
    body
  })
})

// After:
export default defineEventHandler(async (event) => {
  const config = useRuntimeConfig()
  const id = getRouterParam(event, 'id')
  const body = await readBody(event)
  // body = { class_slug: "phb:fighter" }

  return await $fetch(`${config.apiBaseServer}/characters/${id}/classes`, {
    method: 'POST',
    body
  })
})
```

---

## Phase 4: Test Fixtures & Mocks

### Files Requiring Updates

| Category | Files |
|----------|-------|
| **Mock Factories** | `tests/helpers/mockFactories.ts` |
| **Fixtures** | `tests/fixtures/equipment.ts` |
| **Store Tests** | `tests/stores/characterWizard.test.ts` |
| **Setup** | `tests/setup.ts` |
| **Component Tests** | Various files using mock data with `*_id` fields |

### Mock Factory Updates

```typescript
// tests/helpers/mockFactories.ts

export function createMockClass(overrides = {}) {
  return {
    id: 1,
    name: 'Fighter',
    slug: 'fighter',
    full_slug: 'phb:fighter',      // NEW
    parent_class_id: null,
    parent_class_slug: null,        // NEW (if backend adds this)
    is_base_class: true,
    // ...
    ...overrides
  }
}

export function createMockRace(overrides = {}) {
  return {
    id: 1,
    name: 'Human',
    slug: 'human',
    full_slug: 'phb:human',         // NEW
    parent_race_id: null,
    parent_race_slug: null,          // NEW (if backend adds this)
    // ...
    ...overrides
  }
}

export function createMockBackground(overrides = {}) {
  return {
    id: 1,
    name: 'Acolyte',
    slug: 'acolyte',
    full_slug: 'phb:acolyte',       // NEW
    // ...
    ...overrides
  }
}
```

### Store Test Updates

```typescript
// tests/stores/characterWizard.test.ts

// Before (line 367):
expect.objectContaining({ method: 'PATCH', body: { race_id: mockHuman.id } })

// After:
expect.objectContaining({ method: 'PATCH', body: { race_slug: mockHuman.full_slug } })

// Before (line 514):
expect.objectContaining({ method: 'POST', body: { class_id: mockCleric.id } })

// After:
expect.objectContaining({ method: 'POST', body: { class_slug: mockCleric.full_slug } })
```

### Equipment Fixture Updates

```typescript
// tests/fixtures/equipment.ts

// Before:
{
  id: 1,
  item_id: 729,
  quantity: 1,
  // ...
}

// After:
{
  id: 1,
  item_slug: 'phb:longsword',
  quantity: 1,
  // ...
}
```

---

## Phase 5: Validation Endpoint (Future)

### Trigger
Backend #294 merged.

### New Composable

```typescript
// app/composables/useCharacterValidation.ts
export function useCharacterValidation(characterId: Ref<number | null>) {
  const { apiFetch } = useApi()

  const validationResult = ref<{
    valid: boolean
    dangling_references: Array<{
      field: string
      slug: string
      message: string
    }>
  } | null>(null)

  const isValidating = ref(false)

  async function validateReferences() {
    if (!characterId.value) return null

    isValidating.value = true
    try {
      const response = await apiFetch<{ data: typeof validationResult.value }>(
        `/characters/${characterId.value}/validate`
      )
      validationResult.value = response.data
      return response.data
    } finally {
      isValidating.value = false
    }
  }

  return {
    validationResult,
    isValidating,
    validateReferences
  }
}
```

### UI Integration

Add validation warning to character sheet when references are invalid:

```vue
<!-- app/pages/characters/[publicId]/index.vue -->
<template>
  <div v-if="validationResult && !validationResult.valid" class="mb-4">
    <UAlert
      color="warning"
      icon="i-heroicons-exclamation-triangle"
      title="Character has invalid references"
    >
      <template #description>
        <p>Some sourcebook content this character uses is no longer available:</p>
        <ul class="list-disc list-inside mt-2">
          <li v-for="ref in validationResult.dangling_references" :key="ref.slug">
            {{ ref.message }}
          </li>
        </ul>
      </template>
    </UAlert>
  </div>
</template>
```

---

## Migration Checklist

### Pre-Migration
- [ ] Backend #292 (API layer) merged and deployed
- [ ] Backend API docs updated at http://localhost:8080/docs/api
- [ ] Confirm with backend: URL path format (slug vs full_slug vs pivot ID)

### Phase 1: Types
- [ ] Run `npm run types:sync`
- [ ] Update `app/types/character.ts` (CharacterClassEntry)
- [ ] Update `app/types/proficiencies.ts`
- [ ] Update `app/types/api/entities.ts`
- [ ] Run `npm run typecheck` - fix compilation errors

### Phase 2: Character Wizard Store
- [ ] Add `full_slug` to `Subclass` interface
- [ ] Update `createCharacterWithRetry()` signature and body
- [ ] Update `selectRace()` to use `race_slug`
- [ ] Update `selectSubrace()` to use `race_slug`
- [ ] Update `selectClass()` to use `class_slug`
- [ ] Update `selectSubclass()` to use `subclass_slug`
- [ ] Update `selectBackground()` to use `background_slug`
- [ ] Run `npm run test:character` - fix failures

### Phase 3: Nitro Routes
- [ ] Update `server/api/characters/index.post.ts`
- [ ] Update `server/api/characters/[id].patch.ts`
- [ ] Update `server/api/characters/[id]/classes/index.post.ts`
- [ ] Update `server/api/characters/[id]/classes/[classId].put.ts`
- [ ] Update `server/api/characters/[id]/classes/[classId]/subclass.put.ts`
- [ ] Update `server/api/characters/[id]/spells.post.ts`
- [ ] Update `server/api/characters/[id]/equipment.post.ts`
- [ ] Update `server/api/characters/[id]/languages/sync.post.ts`
- [ ] Rename route files if backend changes URL structure

### Phase 4: Tests
- [ ] Update `tests/helpers/mockFactories.ts` with `full_slug` fields
- [ ] Update `tests/fixtures/equipment.ts`
- [ ] Update `tests/stores/characterWizard.test.ts`
- [ ] Update `tests/setup.ts`
- [ ] Update component tests with `*_id` mocks
- [ ] Run full test suite: `npm run test`

### Phase 5: Validation (After #294)
- [ ] Create `app/composables/useCharacterValidation.ts`
- [ ] Add validation UI to character sheet
- [ ] Add tests for validation composable

### Post-Migration
- [ ] Manual E2E test of character creation wizard
- [ ] Manual E2E test of character sheet view
- [ ] Update CHANGELOG.md
- [ ] Create PR with reference to backend #288

---

## Questions for Backend Team (ANSWERED)

1. **URL path format:** Will routes like `/characters/{id}/classes/{classId}` use slugs in the path, or continue using pivot table IDs?

   **Answer:** Routes use `{classIdOrSlug}` - they accept **either format**. Recommendation: Use pivot table IDs for character-specific routes since it's what you get back from the API.

2. **Response format:** Will API responses include both `id` and `full_slug`, or only `full_slug`?

   **Answer:** Responses include **both** `id` and `full_slug`, plus an `is_dangling` flag when the reference can't be resolved.

3. **Subclass handling:** For subclass routes, should we use `full_slug` (e.g., `phb:evoker`) or just `slug` (`evoker`)?

   **Answer:** Use `full_slug` format (e.g., `phb:evoker`). The backend accepts it in the `subclass_slug` field.

4. **Equipment routes:** Does `equipment_id` stay as pivot ID, or does it become `item_slug`?

   **Answer:** Equipment routes use **pivot IDs** in the URL path. The POST body uses `item_slug`.

5. **Entity responses:** Will entity responses (races, classes, etc.) include `full_slug` in list views, or only detail views?

   **Answer:** Yes! `full_slug` is included in **both list and detail views** for all entities (Race, Class, Background, Spell, Item, Feat, Monster, Language, Condition, OptionalFeature).

---

## Risk Assessment

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| Breaking existing characters | High | Certain | Backend handles data migration; frontend just needs to use new format |
| Type sync fails | Medium | Low | Manual type updates as fallback |
| Route parameter confusion | Medium | Medium | Document clearly; coordinate with backend |
| Test coverage gaps | Medium | Medium | Run full test suite; manual E2E testing |
| Partial migration causes bugs | High | Low | Execute all phases in single PR |

---

## Recommended Approach

**Big Bang Migration** - Execute all phases in a single PR after backend #292 is ready.

**Rationale:**
- Backend is already invalidating existing character data
- No benefit to backwards compatibility on frontend
- Cleaner git history and easier rollback if needed
- Reduces coordination complexity

---

## Timeline Estimate

| Phase | Estimated Time |
|-------|----------------|
| Phase 1 (Types) | 1-2 hours |
| Phase 2 (Store) | 2-3 hours |
| Phase 3 (Routes) | 1-2 hours |
| Phase 4 (Tests) | 3-4 hours |
| Phase 5 (Validation) | 2-3 hours |
| **Total** | **9-14 hours** |

Note: Phase 5 can be done separately after #294 is ready.
