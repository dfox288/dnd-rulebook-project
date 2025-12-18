# Migration Guide: Issue #755 - Ability Scores Format Unification

**PR:** dfox288/ledger-of-heroes-backend#224
**Issue:** #755
**Date:** 2025-12-18

## Summary

This migration unifies the ability scores format in `CharacterResource` to match `CharacterStatsResource` and removes duplicate fields.

## Breaking Changes

### 1. `level` field removed from CharacterResource

**Before:**
```php
'level' => $this->total_level,
'total_level' => $this->total_level,
```

**After:**
```php
'total_level' => $this->total_level,
```

### 2. `ability_scores` format changed (flat → nested)

**Before:**
```php
'ability_scores' => $finalScores, // ['STR' => 15, 'DEX' => 14, ...]
'modifiers' => $modifiers,        // ['STR' => 2, 'DEX' => 2, ...]
```

**After:**
```php
'ability_scores' => $this->formatAbilityScores($finalScores),
// ['STR' => ['score' => 15, 'modifier' => 2], ...]
```

### 3. `base_ability_scores` format changed (same nested format)

### 4. `modifiers` field removed (now nested in ability_scores)

---

## Files Changed

### Resource
- `app/Http/Resources/CharacterResource.php`
  - Removed `'level'` field
  - Added `formatAbilityScores()` method
  - Changed `ability_scores` and `base_ability_scores` to nested format
  - Removed `calculateModifiers()` method
  - Updated currency PHPDoc

### Tests Updated
- `tests/Feature/Api/CharacterControllerTest.php`
- `tests/Feature/Api/CharacterAbilityScoreMethodTest.php`
- `tests/Feature/Api/CharacterCreationFlowTest.php`

---

## Files NOT Changed (Intentional)

### Different Resources (different `level` meaning)
These use `level` for different purposes and were NOT changed:

| Resource/Endpoint | `level` Meaning | Action |
|-------------------|-----------------|--------|
| `CharacterClassPivotResource` | Class level (1-20 per class) | Keep as is |
| `ExperienceResource` | Character level in XP context | Keep as is |
| `CharacterConditionResource` | Condition level (e.g., exhaustion 1-6) | Keep as is |
| `SpellSlotResource` | Spell slot level (1st, 2nd, etc.) | Keep as is |

### Internal Services (different format for different purpose)
- `CharacterExportService.php` - Uses lowercase keys (`'strength'`, `'dexterity'`) for portable format
- `CharacterImportService.php` - Matches export format
- These are for import/export, NOT API responses

---

## Issue #807: Additional Alignment (Completed)

The following resources were updated to use `total_level` for consistency:

| Resource | Before | After |
|----------|--------|-------|
| `CharacterStatsResource` | `'level' => $this->resource->level` | `'total_level' => $this->resource->level` |
| `PartyCharacterStatsResource` | `'level' => (int) $this->total_level` | `'total_level' => (int) $this->total_level` |
| `CharacterListResource` | `'level' => (int) $this->total_level` | `'total_level' => (int) $this->total_level` |
| `PartyCharacterResource` | `'level' => (int) $this->total_level` | `'total_level' => (int) $this->total_level` |

**Note:** `CharacterListResource.classes[].level` remains unchanged as it represents class-specific level (e.g., Fighter 5, Wizard 3), not total character level.

**Files Changed:**
- `app/Http/Resources/CharacterStatsResource.php` (line 25)
- `app/Http/Resources/PartyCharacterStatsResource.php` (line 36)
- `app/Http/Resources/CharacterListResource.php` (line 37)
- `app/Http/Resources/PartyCharacterResource.php` (line 22)

**Tests Updated:**
- `tests/Feature/Api/CharacterCreationFlowTest.php`
- `tests/Feature/Api/PartyApiTest.php`
- `tests/Feature/Api/CharacterListResourceTest.php`
- `tests/Feature/Api/CharacterControllerTest.php`

### No Action Needed

- `RaceResource`, `FeatResource`, `ItemResource` - These have `modifiers` for racial/feat modifiers (different concept)
- `MonsterResource` - Has ability scores in a different format for monsters

---

## Testing Checklist

After this migration, verify:

- [x] `GET /api/v1/characters/{id}` returns nested ability scores
- [x] `GET /api/v1/characters/{id}` does NOT have `level` field
- [x] `GET /api/v1/characters/{id}` does NOT have `modifiers` field
- [x] `GET /api/v1/characters/{id}/stats` uses `total_level` (Issue #807)
- [x] `GET /api/v1/parties/{id}/stats` uses `total_level` (Issue #807)
- [x] `GET /api/v1/characters` (list) uses `total_level` (Issue #807)
- [x] `GET /api/v1/parties/{id}` (characters) uses `total_level` (Issue #807)
- [x] Character creation flow works
- [x] Character update (ability scores) works
- [x] All 690 feature tests pass
- [x] All 1390 unit tests pass

---

## Rollback Plan

If issues arise, revert PR #224:

```bash
git revert <commit-hash>
```

Then re-add the old format while investigating.

---

## Related

- Frontend migration issue: #806
- Backend PR: dfox288/ledger-of-heroes-backend#224
- Original issue: #755
- Additional alignment issue: #807
