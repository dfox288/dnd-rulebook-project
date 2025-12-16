# Character Counters Implementation Plan

**Design:** [2025-12-16-character-counters-design.md](./2025-12-16-character-counters-design.md)
**Issue:** #722

## Implementation Order

### Phase 1: Database Migration

**Task 1.1: Create migration**
```bash
php artisan make:migration create_character_counters_table
```

Migration should:
- Create `character_counters` table with schema from design doc
- Drop `max_uses`, `uses_remaining` from `character_features`
- Drop `max_uses`, `uses_remaining` from `feature_selections`

**File:** `database/migrations/2025_12_16_XXXXXX_create_character_counters_table.php`

**Verification:** `php artisan migrate` succeeds, columns dropped

---

### Phase 2: Model Layer

**Task 2.1: Create CharacterCounter model**

```php
// app/Models/CharacterCounter.php
class CharacterCounter extends BaseModel
{
    protected $fillable = [
        'character_id',
        'source_type',
        'source_slug',
        'counter_name',
        'current_uses',
        'max_uses',
        'reset_timing',
    ];

    protected $casts = [
        'current_uses' => 'integer',
        'max_uses' => 'integer',
    ];

    public function character(): BelongsTo;

    // Decrement uses, return false if none remaining
    public function use(): bool;

    // Reset to full (null)
    public function reset(): void;

    // Check if unlimited (-1)
    public function isUnlimited(): bool;
}
```

**Task 2.2: Add relationship to Character model**

```php
// app/Models/Character.php
public function counters(): HasMany
{
    return $this->hasMany(CharacterCounter::class);
}
```

**Task 2.3: Update CharacterFeature model**
- Remove `max_uses`, `uses_remaining` from `$fillable`
- Remove related methods

**Verification:** Unit tests for model methods

---

### Phase 3: CounterService

**Task 3.1: Create CounterService**

```php
// app/Services/CounterService.php
class CounterService
{
    /**
     * Sync counters for character based on current classes, feats, race.
     * Creates new counters, updates max_uses for existing.
     */
    public function syncCountersForCharacter(Character $character): void;

    /**
     * Get counter value from entity_counters for given source at level.
     */
    private function getCounterDefinition(
        string $referenceType,
        int $referenceId,
        string $counterName,
        int $level
    ): ?EntityCounter;

    /**
     * Use a counter (decrement current_uses).
     */
    public function useCounter(Character $character, int $counterId): bool;

    /**
     * Reset counters matching given timings.
     */
    public function resetByTiming(Character $character, string ...$timings): int;

    /**
     * Get all counters formatted for API response.
     */
    public function getCountersForCharacter(Character $character): Collection;
}
```

**Verification:** Unit tests with mocked EntityCounter data

---

### Phase 4: Integration

**Task 4.1: Update LevelUpService**
- After level up, call `CounterService::syncCountersForCharacter()`

**Task 4.2: Update FeatChoiceService**
- After feat granted, call `CounterService::syncCountersForCharacter()`
- Remove call to `FeatureUseService::initializeUsesForFeature()`

**Task 4.3: Update AsiChoiceService**
- Remove call to `FeatureUseService::initializeUsesForFeature()`

**Task 4.4: Update CharacterFeatureService**
- Remove call to `FeatureUseService::initializeUsesForFeature()`

**Task 4.5: Update RestService** (if exists)
- Call `CounterService::resetByTiming()` instead of FeatureUseService

**Verification:** Integration tests - level up Barbarian, verify Rage counter created

---

### Phase 5: Export/Import

**Task 5.1: Update CharacterExportService**
- Export counters from `character_counters` table
- Format: `{ source_type, source_slug, counter_name, current_uses, max_uses, reset_timing }`

**Task 5.2: Update CharacterImportService**
- Import counters to `character_counters` table
- Handle missing counters gracefully (sync will recreate)

**Task 5.3: Update CharacterImportRequest**
- Add validation for counters array

**Verification:** Export → Import round-trip test

---

### Phase 6: API Response

**Task 6.1: Update CharacterResource**
- Change `counters` to use `CounterService::getCountersForCharacter()`

**Task 6.2: Create CharacterCounterResource** (optional)
- If response format changes, create dedicated resource

**Verification:** API test - character with counters returns expected format

---

### Phase 7: Cleanup

**Task 7.1: Simplify FeatureUseService**
- Remove counter-related methods (moved to CounterService)
- Keep only feature-specific methods if any remain
- Or delete entirely if empty

**Task 7.2: Remove unused code**
- Remove `CharacterFeature::useFeature()`, `resetUses()` methods
- Remove unused imports

**Task 7.3: Update tests**
- Update/remove tests that reference old columns
- Add comprehensive CounterService tests

---

## Test Plan

### Unit Tests
- [ ] CharacterCounter model methods (use, reset, isUnlimited)
- [ ] CounterService::syncCountersForCharacter with various class combinations
- [ ] CounterService::getCounterDefinition lookup logic
- [ ] CounterService::resetByTiming with mixed timings

### Integration Tests
- [ ] Barbarian L1 → Rage counter created with max=2
- [ ] Barbarian L3 → Rage counter updated to max=3
- [ ] Multiclass Fighter/Rogue with Psi Warrior/Soulknife → separate Psionic Energy counters
- [ ] Feat selection → Lucky creates Luck Points counter
- [ ] Export/Import round-trip preserves counters

### API Tests
- [ ] GET /characters/{id} includes counters array
- [ ] Counter format matches expected schema

---

## Files Changed

**New:**
- `database/migrations/2025_12_16_XXXXXX_create_character_counters_table.php`
- `app/Models/CharacterCounter.php`
- `app/Services/CounterService.php`
- `tests/Unit/Services/CounterServiceTest.php`
- `tests/Unit/Models/CharacterCounterTest.php`

**Modified:**
- `app/Models/Character.php` (add counters relationship)
- `app/Models/CharacterFeature.php` (remove max_uses)
- `app/Services/LevelUpService.php`
- `app/Services/FeatChoiceService.php`
- `app/Services/AsiChoiceService.php`
- `app/Services/CharacterFeatureService.php`
- `app/Services/CharacterExportService.php`
- `app/Services/CharacterImportService.php`
- `app/Services/FeatureUseService.php` (simplify/remove)
- `app/Http/Resources/CharacterResource.php`

**Deleted (potentially):**
- `app/Services/FeatureUseService.php` (if empty after cleanup)
