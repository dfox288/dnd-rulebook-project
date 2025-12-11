# Monster Traits Polymorphic Migration

**Date:** 2025-12-11
**Status:** Approved
**GitHub Issue:** [#502](https://github.com/dfox288/ledger-of-heroes/issues/502)
**Related:** [#508](https://github.com/dfox288/ledger-of-heroes/issues/508) (attack_data decision)

---

## Goal

Consolidate `monster_traits` into `entity_traits` polymorphic table for:
- **Consistency** - One pattern for traits across all entities
- **Reduced duplication** - Drow as Race and Monster share same trait structure

## Approach

**Re-import strategy:** Update importer first, run `import:all`, drop old table last.

---

## Data Mapping

| `monster_traits` | | `entity_traits` |
|------------------|---|-----------------|
| `monster_id` | → | `reference_id` + `reference_type = 'App\Models\Monster'` |
| `name` | → | `name` |
| `description` | → | `description` |
| `sort_order` | → | `sort_order` |
| `attack_data` | → | **Dropped** (see #508) |
| *(n/a)* | → | `category = 'feature'` |

---

## Files Changed

| File | Action |
|------|--------|
| `Monster.php` | Add `HasEntityTraits` concern, remove `hasMany` |
| `MonsterImporter.php` | Create `CharacterTrait` with `reference_type = Monster` |
| `MonsterResource.php` | Use `TraitResource` instead of `MonsterTraitResource` |
| `MonsterTraitResource.php` | Delete |
| `MonsterTrait.php` | Delete |
| `HasEntityTraits.php` | Update docblock (Monster now included) |
| Migration | Drop `monster_traits` table |

---

## Implementation Steps

### Step 1: Update MonsterImporter

Change `importMonsterTraits()` to create `CharacterTrait` records:

```php
// Before
MonsterTrait::create([
    'monster_id' => $monster->id,
    'name' => $trait['name'],
    'description' => $trait['description'],
    'attack_data' => $trait['attack_data'] ?? null,
    'sort_order' => $sortOrder,
]);

// After
CharacterTrait::create([
    'reference_type' => Monster::class,
    'reference_id' => $monster->id,
    'name' => $trait['name'],
    'description' => $trait['description'],
    'category' => 'feature',
    'sort_order' => $sortOrder,
]);
```

### Step 2: Update Monster Model

```php
// Before
use HasEntitySenses;

public function traits(): HasMany
{
    return $this->hasMany(MonsterTrait::class);
}

// After
use HasEntitySenses;
use HasEntityTraits;  // Add this, remove manual traits() method
```

### Step 3: Update MonsterResource

```php
// Before
'traits' => MonsterTraitResource::collection($this->whenLoaded('traits')),

// After
'traits' => TraitResource::collection($this->whenLoaded('traits')),
```

### Step 4: Cleanup

- Delete `MonsterTrait.php`
- Delete `MonsterTraitResource.php`
- Update `HasEntityTraits.php` docblock
- Create migration to drop `monster_traits` table

---

## Testing & Verification

### Existing Test Coverage

`MonsterImporterTest` covers trait importing. Same assertions work after migration:

```php
expect($monster->traits)->toHaveCount(3);
expect($monster->traits->first()->name)->toBe('Amphibious');
```

### Verification Steps

1. **Pre-migration count:**
   ```bash
   docker compose exec php php artisan tinker --execute="echo App\Models\MonsterTrait::count();"
   ```

2. **Run import:**
   ```bash
   docker compose exec php php artisan import:all
   ```

3. **Post-migration count:**
   ```bash
   docker compose exec php php artisan tinker --execute="echo App\Models\CharacterTrait::where('reference_type', 'App\\\Models\\\Monster')->count();"
   ```

4. **API verification:**
   ```bash
   curl -s http://localhost:8080/api/v1/monsters/aboleth | jq '.data.traits'
   ```

5. **Search verification:**
   ```bash
   curl -s 'http://localhost:8080/api/v1/monsters?filter=has_magic_resistance=true' | jq '.meta.total'
   ```

### Test Suite

```bash
docker compose exec php ./vendor/bin/pest --testsuite=Importers
```

---

## Breaking Changes

| Change | Impact | Mitigation |
|--------|--------|------------|
| `attack_data` removed from API | Minor - frontend doesn't use it | Document in CHANGELOG |
| `category` added to response | Additive - non-breaking | None needed |
| `data_tables` added to response | Additive - non-breaking | None needed |

---

## Execution Order

```
1. Update MonsterImporter.php         # Create CharacterTrait instead
2. Update Monster.php                 # Use HasEntityTraits concern
3. Update MonsterResource.php         # Use TraitResource
4. Update HasEntityTraits.php         # Docblock only
5. Update MonsterImporterTest.php     # If assertions need tweaking
6. Run tests                          # Verify nothing breaks
7. Run import:all                     # Populate entity_traits
8. Verify counts match                # QA check
9. Delete MonsterTrait.php            # Cleanup
10. Delete MonsterTraitResource.php   # Cleanup
11. Create & run drop table migration # Final cleanup
12. Update CHANGELOG.md               # Document breaking change
```

---

## Rollback Plan

If issues arise after import:
- `monster_traits` table still exists until step 11
- Revert code changes and traits work from old table
- Only drop table after full verification
