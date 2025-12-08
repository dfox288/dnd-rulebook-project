# Issue #355: Personality Characteristics Selection

**Date:** 2025-12-08
**Status:** Approved
**Issue:** [#355](https://github.com/dfox288/dnd-rulebook-project/issues/355)

## Summary

Add UI for selecting personality characteristics (traits, ideals, bonds, flaws) to the character creation wizard. Selection is optional, matching the optional nature of alignment.

## Decision Record

| Decision | Choice | Rationale |
|----------|--------|-----------|
| **Location** | Expand `StepDetails.vue` | Locality of behavior - personality is a "detail" like name/alignment |
| **API** | Notes API (`/characters/{id}/notes`) | Already exists, designed for this use case, no backend changes |
| **Required?** | Optional | Matches D&D flexibility, alignment pattern |

## Architecture

### Component Structure

```
StepDetails.vue (expanded)
├── Existing: Name input + Alignment select
└── New: CharacterWizardPersonalitySection.vue
         ├── CharacterWizardPersonalityTablePicker.vue (×4)
         │    ├── Header: table name + dice type + Roll button
         │    ├── Checkbox/radio list of options
         │    └── Selected count indicator
         └── "Roll All" + "Clear All" buttons
```

### Data Flow

```
1. READ: store.selections.background.data_tables
   → Filter to: Personality Trait, Ideal, Bond, Flaw
   → Render 4 PersonalityTablePicker components

2. LOCAL STATE (during editing):
   personalitySelections = {
     traits: string[],     // max 2
     ideal: string | null,
     bond: string | null,
     flaw: string | null
   }

3. SAVE (on Continue):
   For each non-empty selection:
   POST /api/characters/{id}/notes
   { category: "personality_trait", content: "selected text" }
```

### Selection Rules

| Table | Dice | Pick | UI Control |
|-------|------|------|------------|
| Personality Trait | d8 | 2 | Checkboxes (max 2) |
| Ideal | d6 | 1 | Radio buttons |
| Bond | d6 | 1 | Radio buttons |
| Flaw | d6 | 1 | Radio buttons |

## Files to Create/Modify

### New Files

| File | Description |
|------|-------------|
| `app/components/character/wizard/PersonalitySection.vue` | Container for all 4 personality tables |
| `app/components/character/wizard/PersonalityTablePicker.vue` | Single table picker with roll functionality |
| `server/api/characters/[id]/notes.get.ts` | Nitro route: GET character notes |
| `server/api/characters/[id]/notes.post.ts` | Nitro route: POST new note |
| `tests/components/character/wizard/PersonalitySection.test.ts` | Component tests |
| `tests/components/character/wizard/PersonalityTablePicker.test.ts` | Component tests |

### Modified Files

| File | Changes |
|------|---------|
| `app/components/character/wizard/StepDetails.vue` | Import and render PersonalitySection below alignment |

## API Integration

### GET /api/characters/{id}/notes

```typescript
// server/api/characters/[id]/notes.get.ts
export default defineEventHandler(async (event) => {
  const config = useRuntimeConfig()
  const id = getRouterParam(event, 'id')
  return await $fetch(`${config.apiBaseServer}/characters/${id}/notes`)
})
```

### POST /api/characters/{id}/notes

```typescript
// server/api/characters/[id]/notes.post.ts
export default defineEventHandler(async (event) => {
  const config = useRuntimeConfig()
  const id = getRouterParam(event, 'id')
  const body = await readBody(event)
  return await $fetch(`${config.apiBaseServer}/characters/${id}/notes`, {
    method: 'POST',
    body
  })
})
```

### Request/Response

```typescript
// POST request
{ category: "personality_trait" | "ideal" | "bond" | "flaw", content: string }

// GET response
{
  data: {
    personality_trait: [{ id, content, created_at }],
    ideal: [...],
    bond: [...],
    flaw: [...]
  }
}
```

## UI Behavior

### Roll Button
- Generates random number based on `dice_type` (d6 or d8)
- For traits: rolls twice with different values
- Replaces current selection(s)

### "Roll All" Button
- Rolls all 4 tables at once
- Quick way to generate complete personality

### "Clear All" Button
- Clears all selections
- Returns to unselected state

### Edge Cases

| Scenario | Behavior |
|----------|----------|
| No background selected | Hide personality section |
| Background has no data_tables | Hide personality section |
| Edit existing character | Fetch notes, pre-populate selections |
| Save with partial selections | Only POST non-empty selections |

## Test Cases

### PersonalityTablePicker

- [ ] Renders table name and dice type in header
- [ ] Renders all entries from data table
- [ ] Single-select mode: selecting one deselects others
- [ ] Multi-select mode: allows up to `maxSelections` items
- [ ] Multi-select mode: disables unselected when max reached
- [ ] Roll button selects random entry
- [ ] Roll button in multi-select rolls multiple unique values
- [ ] Emits selection changes to parent

### PersonalitySection

- [ ] Renders all 4 tables when data_tables present
- [ ] Hides when no background selected
- [ ] Hides when background has no personality tables
- [ ] "Roll All" rolls all 4 tables
- [ ] "Clear All" clears all selections
- [ ] Exposes selections for parent to save

### StepDetails Integration

- [ ] Shows personality section below alignment
- [ ] Saves personality notes on Continue
- [ ] Loads existing notes when editing character
- [ ] Can proceed without personality selections (optional)

## Implementation Order

1. Create Nitro routes (notes.get.ts, notes.post.ts)
2. Create PersonalityTablePicker component with tests
3. Create PersonalitySection component with tests
4. Integrate into StepDetails.vue
5. Add edit mode support (load existing notes)
