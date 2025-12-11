# Implementation Plan: Monster Traits Polymorphic Migration

**Design:** [2025-12-11-monster-traits-polymorphic-design.md](./2025-12-11-monster-traits-polymorphic-design.md)
**GitHub Issue:** [#502](https://github.com/dfox288/ledger-of-heroes/issues/502)
**Runner:** Sail (`docker compose exec php`)

---

## Pre-Flight Checklist

- [ ] Confirm Sail running: `docker compose ps`
- [ ] Branch created: `git checkout -b feature/issue-502-monster-traits-polymorphic`
- [ ] Record baseline count: `sail artisan tinker --execute="echo App\Models\MonsterTrait::count();"`

---

## Phase 1: Update Importer (TDD)

### Task 1.1: Update MonsterImporterTest for polymorphic traits

**File:** `tests/Feature/Importers/MonsterImporterTest.php`

**Changes:**
1. Find test that verifies trait import (search for `traits` assertions)
2. Add assertion that traits are `CharacterTrait` instances:
   ```php
   expect($monster->traits->first())->toBeInstanceOf(\App\Models\CharacterTrait::class);
   ```
3. Add assertion for `category`:
   ```php
   expect($monster->traits->first()->category)->toBe('feature');
   ```
4. Add assertion for `reference_type`:
   ```php
   expect($monster->traits->first()->reference_type)->toBe(\App\Models\Monster::class);
   ```

**Verify:** Test FAILS (still using MonsterTrait)
```bash
sail artisan test --filter=MonsterImporterTest
```

**Commit:** `test: Add polymorphic trait assertions to MonsterImporterTest (#502)`

---

### Task 1.2: Update MonsterImporter to create CharacterTrait

**File:** `app/Services/Importers/MonsterImporter.php`

**Changes:**
1. Replace import at top:
   ```php
   // Remove: use App\Models\MonsterTrait;
   // Add:
   use App\Models\CharacterTrait;
   use App\Models\Monster;
   ```

2. Update `importMonsterTraits()` method (~line 247):
   ```php
   protected function importMonsterTraits(Monster $monster, array $traits): void
   {
       // Clear existing traits (polymorphic)
       $monster->traits()->delete();

       $sortOrder = 0;
       foreach ($traits as $trait) {
           $sortOrder++;
           CharacterTrait::create([
               'reference_type' => Monster::class,
               'reference_id' => $monster->id,
               'name' => $trait['name'],
               'description' => $trait['description'] ?? null,
               'category' => 'feature',
               'sort_order' => $sortOrder,
           ]);
       }
   }
   ```

**Note:** Do NOT change Monster model yet - test will still fail because `$monster->traits()` returns MonsterTrait.

**Commit:** `refactor: Update MonsterImporter to create CharacterTrait (#502)`

---

## Phase 2: Update Monster Model

### Task 2.1: Switch Monster to HasEntityTraits concern

**File:** `app/Models/Monster.php`

**Changes:**
1. Add import at top:
   ```php
   use App\Models\Concerns\HasEntityTraits;
   ```

2. Add trait to class (after existing traits ~line 15):
   ```php
   use HasEntityTraits;
   ```

3. Remove the manual `traits()` method (~line 98-101):
   ```php
   // DELETE this entire method:
   public function traits(): HasMany
   {
       return $this->hasMany(MonsterTrait::class);
   }
   ```

4. Remove `MonsterTrait` import if present:
   ```php
   // Remove: use App\Models\MonsterTrait;
   ```

5. Update `HasMany` import if no longer needed (check other relationships first)

**Verify:** Test PASSES
```bash
sail artisan test --filter=MonsterImporterTest
```

**Commit:** `refactor: Use HasEntityTraits concern in Monster model (#502)`

---

### Task 2.2: Update HasEntityTraits docblock

**File:** `app/Models/Concerns/HasEntityTraits.php`

**Changes:**
1. Update docblock (~line 12-15):
   ```php
   /**
    * Trait HasEntityTraits
    *
    * Provides the polymorphic traits relationship for entities that can have character traits.
    * Named HasEntityTraits to avoid conflict with PHP's trait keyword.
    * Used by: CharacterClass, Race, Background, Monster
    */
   ```

**Commit:** `docs: Update HasEntityTraits docblock to include Monster (#502)`

---

## Phase 3: Update API Resource

### Task 3.1: Update MonsterResource to use TraitResource

**File:** `app/Http/Resources/MonsterResource.php`

**Changes:**
1. Replace import:
   ```php
   // Remove: use App\Http\Resources\MonsterTraitResource;
   // Add (if not present):
   use App\Http\Resources\TraitResource;
   ```

2. Update traits line (~line 45):
   ```php
   // Before:
   'traits' => MonsterTraitResource::collection($this->whenLoaded('traits')),

   // After:
   'traits' => TraitResource::collection($this->whenLoaded('traits')),
   ```

**Verify:** Run API test
```bash
sail artisan test --filter=MonsterApiTest
```

**Commit:** `refactor: Use TraitResource for monster traits (#502)`

---

## Phase 4: Cleanup

### Task 4.1: Delete MonsterTrait model

**File:** `app/Models/MonsterTrait.php`

**Action:** Delete entire file

```bash
rm app/Models/MonsterTrait.php
```

**Verify:** Tests still pass
```bash
sail artisan test --testsuite=Importers
```

**Commit:** `chore: Delete MonsterTrait model (#502)`

---

### Task 4.2: Delete MonsterTraitResource

**File:** `app/Http/Resources/MonsterTraitResource.php`

**Action:** Delete entire file

```bash
rm app/Http/Resources/MonsterTraitResource.php
```

**Verify:** Tests still pass
```bash
sail artisan test --testsuite=Feature-DB
```

**Commit:** `chore: Delete MonsterTraitResource (#502)`

---

## Phase 5: Data Migration & Table Cleanup

### Task 5.1: Run full import to populate entity_traits

**Action:** Run import command

```bash
# Record pre-import count
sail artisan tinker --execute="echo 'MonsterTrait: ' . App\Models\MonsterTrait::count();"

# This will fail now - MonsterTrait is deleted. Use raw query:
sail artisan tinker --execute="echo 'monster_traits: ' . DB::table('monster_traits')->count();"

# Run import
sail artisan import:all

# Verify new count matches
sail artisan tinker --execute="echo 'Monster CharacterTraits: ' . App\Models\CharacterTrait::where('reference_type', 'App\\\Models\\\Monster')->count();"
```

**Verify:** Counts should match (approximately - may vary slightly if XML changed)

**No commit** - this is a data operation

---

### Task 5.2: Create migration to drop monster_traits table

**Action:** Create migration

```bash
sail artisan make:migration drop_monster_traits_table
```

**File:** `database/migrations/YYYY_MM_DD_HHMMSS_drop_monster_traits_table.php`

**Content:**
```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::dropIfExists('monster_traits');
    }

    public function down(): void
    {
        Schema::create('monster_traits', function (Blueprint $table) {
            $table->id();
            $table->foreignId('monster_id')->constrained()->cascadeOnDelete();
            $table->string('name');
            $table->text('description')->nullable();
            $table->json('attack_data')->nullable();
            $table->unsignedSmallInteger('sort_order')->default(0);
            $table->timestamps();

            $table->index('monster_id');
        });
    }
};
```

**Run migration:**
```bash
sail artisan migrate
```

**Commit:** `chore: Drop monster_traits table (#502)`

---

## Phase 6: Quality Gates & Documentation

### Task 6.1: Run full test suite

```bash
sail artisan test --testsuite=Unit-Pure
sail artisan test --testsuite=Unit-DB
sail artisan test --testsuite=Feature-DB
sail artisan test --testsuite=Importers
```

**All must pass.**

---

### Task 6.2: Run Pint

```bash
sail php ./vendor/bin/pint
```

**Commit any fixes:** `style: Apply Pint formatting (#502)`

---

### Task 6.3: Update CHANGELOG.md

**File:** `CHANGELOG.md`

**Add under `[Unreleased]`:**
```markdown
### Changed
- Monster traits now use polymorphic `entity_traits` table (consistency with races/classes/backgrounds) (#502)

### Removed
- `attack_data` field removed from monster trait API responses (#508)
- `MonsterTrait` model deleted (use `CharacterTrait` with `reference_type = Monster`)
- `MonsterTraitResource` deleted (use `TraitResource`)
```

**Commit:** `docs: Update CHANGELOG for monster traits migration (#502)`

---

## Phase 7: Final Verification

### Task 7.1: API smoke test

```bash
# Get a monster with traits
curl -s http://localhost:8080/api/v1/monsters/aboleth | jq '.data.traits'

# Should return array with: id, name, category, description, sort_order, data_tables
# Should NOT contain: attack_data
```

### Task 7.2: Search smoke test

```bash
# Verify computed flags still work
curl -s 'http://localhost:8080/api/v1/monsters?filter=has_magic_resistance=true' | jq '.meta.total'

# Should return count > 0
```

---

## Rollback Plan

If issues arise:
1. Restore `MonsterTrait.php` and `MonsterTraitResource.php` from git
2. Revert Monster.php to use `hasMany(MonsterTrait::class)`
3. Revert MonsterImporter.php to create `MonsterTrait`
4. Revert MonsterResource.php to use `MonsterTraitResource`
5. Run `sail artisan migrate:rollback` to restore `monster_traits` table
6. Re-import: `sail artisan import:all`

---

## Summary

| Phase | Tasks | Estimated Commits |
|-------|-------|-------------------|
| 1. Update Importer | 2 | 2 |
| 2. Update Model | 2 | 2 |
| 3. Update Resource | 1 | 1 |
| 4. Cleanup | 2 | 2 |
| 5. Data & Table | 2 | 1 |
| 6. Quality Gates | 3 | 2 |
| **Total** | **12** | **10** |
