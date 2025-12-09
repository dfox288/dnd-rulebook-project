# Implementation Plan: Character HP Auto-Initialization and Level-Up

**Issue:** #254 - Character HP Auto-Initialization and Level-Up
**Branch:** `feature/issue-254-hp-auto-initialization`
**Date:** 2025-12-08
**Runner:** Sail (`docker compose exec php ...`)

---

## Overview

Implement automatic HP calculation for character creation and level-up:
- Auto-set starting HP when first class is added (hit die max + CON mod)
- Present roll/average choice on level-up (levels 2+)
- Recalculate HP when CON changes
- Track resolved HP choices per level

---

## Phase 1: Foundation (Migration + Model Helpers)

### Task 1.1: Create Migration for HP Tracking Columns

**File:** `database/migrations/2025_12_08_000001_add_hp_tracking_to_characters.php`

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::table('characters', function (Blueprint $table) {
            // JSON array of level numbers with resolved HP choices
            // e.g., [1, 2, 3] means levels 1-3 have HP resolved
            $table->json('hp_levels_resolved')->default('[]')->after('temp_hit_points');

            // Enum: 'calculated' (auto) or 'manual' (legacy/custom)
            $table->string('hp_calculation_method', 10)->default('calculated')->after('hp_levels_resolved');
        });
    }

    public function down(): void
    {
        Schema::table('characters', function (Blueprint $table) {
            $table->dropColumn(['hp_levels_resolved', 'hp_calculation_method']);
        });
    }
};
```

**Test:** Run migration up/down successfully.

```bash
docker compose exec php php artisan migrate
docker compose exec php php artisan migrate:rollback
docker compose exec php php artisan migrate
```

---

### Task 1.2: Add Character Model Helper Methods

**File:** `app/Models/Character.php`

Add casts:
```php
protected function casts(): array
{
    return [
        // ... existing casts
        'hp_levels_resolved' => 'array',
    ];
}
```

Add helper methods:
```php
/**
 * Check if HP has been resolved for a specific level.
 */
public function hasResolvedHpForLevel(int $level): bool
{
    return in_array($level, $this->hp_levels_resolved ?? [], true);
}

/**
 * Mark HP as resolved for a specific level.
 */
public function markHpResolvedForLevel(int $level): void
{
    $resolved = $this->hp_levels_resolved ?? [];
    if (!in_array($level, $resolved, true)) {
        $resolved[] = $level;
        sort($resolved);
        $this->hp_levels_resolved = $resolved;
        $this->save();
    }
}

/**
 * Get all levels that still need HP choices resolved.
 */
public function getPendingHpLevels(): array
{
    $totalLevel = $this->total_level;
    $resolved = $this->hp_levels_resolved ?? [];

    $pending = [];
    for ($level = 1; $level <= $totalLevel; $level++) {
        if (!in_array($level, $resolved, true)) {
            $pending[] = $level;
        }
    }

    return $pending;
}

/**
 * Check if this character uses calculated HP (vs manual).
 */
public function usesCalculatedHp(): bool
{
    return ($this->hp_calculation_method ?? 'calculated') === 'calculated';
}
```

**Test file:** `tests/Unit/Models/CharacterHpTrackingTest.php`

```php
<?php

use App\Models\Character;

describe('Character HP Tracking', function () {
    it('tracks resolved HP levels as array', function () {
        $character = Character::factory()->create([
            'hp_levels_resolved' => [1, 2],
        ]);

        expect($character->hp_levels_resolved)->toBe([1, 2]);
    });

    it('checks if HP is resolved for a level', function () {
        $character = Character::factory()->create([
            'hp_levels_resolved' => [1, 2],
        ]);

        expect($character->hasResolvedHpForLevel(1))->toBeTrue();
        expect($character->hasResolvedHpForLevel(2))->toBeTrue();
        expect($character->hasResolvedHpForLevel(3))->toBeFalse();
    });

    it('marks HP as resolved for a level', function () {
        $character = Character::factory()->create([
            'hp_levels_resolved' => [1],
        ]);

        $character->markHpResolvedForLevel(2);

        expect($character->fresh()->hp_levels_resolved)->toBe([1, 2]);
    });

    it('does not duplicate resolved levels', function () {
        $character = Character::factory()->create([
            'hp_levels_resolved' => [1, 2],
        ]);

        $character->markHpResolvedForLevel(2);

        expect($character->fresh()->hp_levels_resolved)->toBe([1, 2]);
    });

    it('keeps resolved levels sorted', function () {
        $character = Character::factory()->create([
            'hp_levels_resolved' => [3],
        ]);

        $character->markHpResolvedForLevel(1);

        expect($character->fresh()->hp_levels_resolved)->toBe([1, 3]);
    });

    it('gets pending HP levels', function () {
        $character = Character::factory()
            ->withClass('fighter', 3)
            ->create([
                'hp_levels_resolved' => [1],
            ]);

        expect($character->getPendingHpLevels())->toBe([2, 3]);
    });

    it('returns empty array when all levels resolved', function () {
        $character = Character::factory()
            ->withClass('fighter', 2)
            ->create([
                'hp_levels_resolved' => [1, 2],
            ]);

        expect($character->getPendingHpLevels())->toBe([]);
    });

    it('checks calculated vs manual HP mode', function () {
        $calculated = Character::factory()->create([
            'hp_calculation_method' => 'calculated',
        ]);
        $manual = Character::factory()->create([
            'hp_calculation_method' => 'manual',
        ]);

        expect($calculated->usesCalculatedHp())->toBeTrue();
        expect($manual->usesCalculatedHp())->toBeFalse();
    });

    it('defaults to calculated HP mode', function () {
        $character = Character::factory()->create();

        expect($character->usesCalculatedHp())->toBeTrue();
    });
});
```

**Run:** `docker compose exec php ./vendor/bin/pest tests/Unit/Models/CharacterHpTrackingTest.php`

---

### Task 1.3: Create HitPointService

**File:** `app/Services/HitPointService.php`

```php
<?php

namespace App\Services;

use App\Models\Character;
use App\Models\CharacterClass;
use InvalidArgumentException;

class HitPointService
{
    public function __construct(
        private CharacterStatCalculator $calculator
    ) {}

    /**
     * Calculate starting HP for level 1 character.
     * Formula: Hit Die Maximum + CON Modifier
     */
    public function calculateStartingHp(Character $character, CharacterClass $class): int
    {
        $hitDieMax = $class->hit_die;
        $conMod = $this->calculator->abilityModifier($character->constitution ?? 10);

        // Minimum 1 HP even with negative CON
        return max(1, $hitDieMax + $conMod);
    }

    /**
     * Calculate average HP gain for a level-up.
     * Formula: (Hit Die / 2 + 1) + CON Modifier
     */
    public function calculateAverageHpGain(int $hitDie, int $constitution): int
    {
        $average = intdiv($hitDie, 2) + 1;
        $conMod = $this->calculator->abilityModifier($constitution);

        // Minimum 1 HP per level
        return max(1, $average + $conMod);
    }

    /**
     * Calculate HP gain from a roll.
     * Formula: Roll Result + CON Modifier
     */
    public function calculateRolledHpGain(int $rollResult, int $hitDie, int $constitution): int
    {
        if ($rollResult < 1 || $rollResult > $hitDie) {
            throw new InvalidArgumentException(
                "Roll result {$rollResult} is invalid for d{$hitDie} (must be 1-{$hitDie})"
            );
        }

        $conMod = $this->calculator->abilityModifier($constitution);

        // Minimum 1 HP per level
        return max(1, $rollResult + $conMod);
    }

    /**
     * Recalculate HP after CON change.
     * Adjusts HP by (new modifier - old modifier) * total level.
     */
    public function recalculateForConChange(
        Character $character,
        int $oldConstitution,
        int $newConstitution
    ): array {
        $oldMod = $this->calculator->abilityModifier($oldConstitution);
        $newMod = $this->calculator->abilityModifier($newConstitution);
        $diff = $newMod - $oldMod;

        if ($diff === 0) {
            return [
                'adjustment' => 0,
                'new_max_hp' => $character->max_hit_points,
                'new_current_hp' => $character->current_hit_points,
            ];
        }

        $adjustment = $diff * $character->total_level;
        $newMaxHp = max(1, ($character->max_hit_points ?? 0) + $adjustment);

        // Current HP adjusts but caps at new max
        $newCurrentHp = $character->current_hit_points ?? 0;
        if ($adjustment > 0) {
            // CON increased: gain HP
            $newCurrentHp += $adjustment;
        } else {
            // CON decreased: lose HP but not below 1
            $newCurrentHp = max(1, min($newCurrentHp, $newMaxHp));
        }

        return [
            'adjustment' => $adjustment,
            'new_max_hp' => $newMaxHp,
            'new_current_hp' => $newCurrentHp,
        ];
    }

    /**
     * Get the hit die for a specific level (handles multiclass).
     */
    public function getHitDieForLevel(Character $character, int $level): int
    {
        // Get class pivots ordered by when they were added
        $pivots = $character->characterClasses()
            ->orderBy('order')
            ->get();

        $currentLevel = 0;
        foreach ($pivots as $pivot) {
            $classLevels = $pivot->level;
            if ($currentLevel + $classLevels >= $level) {
                // This class covers the requested level
                return $pivot->class->hit_die;
            }
            $currentLevel += $classLevels;
        }

        // Fallback to primary class
        return $character->primaryClass?->hit_die ?? 8;
    }
}
```

**Test file:** `tests/Unit/Services/HitPointServiceTest.php`

```php
<?php

use App\Models\Character;
use App\Models\CharacterClass;
use App\Services\CharacterStatCalculator;
use App\Services\HitPointService;

beforeEach(function () {
    $this->calculator = app(CharacterStatCalculator::class);
    $this->service = new HitPointService($this->calculator);
});

describe('calculateStartingHp', function () {
    it('calculates fighter starting HP with positive CON', function () {
        $character = Character::factory()->create(['constitution' => 14]); // +2
        $class = CharacterClass::factory()->create(['hit_die' => 10]);

        $hp = $this->service->calculateStartingHp($character, $class);

        expect($hp)->toBe(12); // 10 + 2
    });

    it('calculates wizard starting HP with low CON', function () {
        $character = Character::factory()->create(['constitution' => 8]); // -1
        $class = CharacterClass::factory()->create(['hit_die' => 6]);

        $hp = $this->service->calculateStartingHp($character, $class);

        expect($hp)->toBe(5); // 6 - 1
    });

    it('enforces minimum 1 HP with very low CON', function () {
        $character = Character::factory()->create(['constitution' => 3]); // -4
        $class = CharacterClass::factory()->create(['hit_die' => 6]);

        $hp = $this->service->calculateStartingHp($character, $class);

        expect($hp)->toBe(2); // max(1, 6 - 4) = 2
    });

    it('handles null constitution as 10', function () {
        $character = Character::factory()->create(['constitution' => null]);
        $class = CharacterClass::factory()->create(['hit_die' => 8]);

        $hp = $this->service->calculateStartingHp($character, $class);

        expect($hp)->toBe(8); // 8 + 0
    });
});

describe('calculateAverageHpGain', function () {
    it('calculates d10 average with +2 CON', function () {
        $hp = $this->service->calculateAverageHpGain(10, 14);

        expect($hp)->toBe(8); // (10/2 + 1) + 2 = 6 + 2
    });

    it('calculates d6 average with +1 CON', function () {
        $hp = $this->service->calculateAverageHpGain(6, 12);

        expect($hp)->toBe(5); // (6/2 + 1) + 1 = 4 + 1
    });

    it('calculates d12 average with -1 CON', function () {
        $hp = $this->service->calculateAverageHpGain(12, 8);

        expect($hp)->toBe(6); // (12/2 + 1) - 1 = 7 - 1
    });

    it('enforces minimum 1 HP', function () {
        $hp = $this->service->calculateAverageHpGain(6, 3); // -4 CON mod

        expect($hp)->toBe(1); // max(1, 4 - 4)
    });
});

describe('calculateRolledHpGain', function () {
    it('calculates HP from roll with CON modifier', function () {
        $hp = $this->service->calculateRolledHpGain(7, 10, 14);

        expect($hp)->toBe(9); // 7 + 2
    });

    it('enforces minimum 1 HP on low roll', function () {
        $hp = $this->service->calculateRolledHpGain(1, 6, 3); // -4 CON

        expect($hp)->toBe(1); // max(1, 1 - 4)
    });

    it('rejects roll below 1', function () {
        expect(fn () => $this->service->calculateRolledHpGain(0, 10, 10))
            ->toThrow(InvalidArgumentException::class);
    });

    it('rejects roll above hit die', function () {
        expect(fn () => $this->service->calculateRolledHpGain(11, 10, 10))
            ->toThrow(InvalidArgumentException::class);
    });
});

describe('recalculateForConChange', function () {
    it('increases HP when CON increases', function () {
        $character = Character::factory()
            ->withClass('fighter', 5)
            ->create([
                'constitution' => 14,
                'max_hit_points' => 50,
                'current_hit_points' => 50,
            ]);

        // CON 14 (+2) -> CON 16 (+3) = +1 per level = +5 HP
        $result = $this->service->recalculateForConChange($character, 14, 16);

        expect($result['adjustment'])->toBe(5);
        expect($result['new_max_hp'])->toBe(55);
        expect($result['new_current_hp'])->toBe(55);
    });

    it('decreases HP when CON decreases', function () {
        $character = Character::factory()
            ->withClass('fighter', 5)
            ->create([
                'constitution' => 16,
                'max_hit_points' => 55,
                'current_hit_points' => 55,
            ]);

        // CON 16 (+3) -> CON 14 (+2) = -1 per level = -5 HP
        $result = $this->service->recalculateForConChange($character, 16, 14);

        expect($result['adjustment'])->toBe(-5);
        expect($result['new_max_hp'])->toBe(50);
        expect($result['new_current_hp'])->toBe(50);
    });

    it('caps current HP at new max when CON decreases', function () {
        $character = Character::factory()
            ->withClass('fighter', 5)
            ->create([
                'constitution' => 16,
                'max_hit_points' => 55,
                'current_hit_points' => 30, // Already damaged
            ]);

        $result = $this->service->recalculateForConChange($character, 16, 14);

        expect($result['new_max_hp'])->toBe(50);
        expect($result['new_current_hp'])->toBe(30); // Stays at 30 (below new max)
    });

    it('returns no change when CON modifier unchanged', function () {
        $character = Character::factory()
            ->withClass('fighter', 5)
            ->create([
                'constitution' => 14,
                'max_hit_points' => 50,
                'current_hit_points' => 50,
            ]);

        // CON 14 -> 15 both have +2 modifier
        $result = $this->service->recalculateForConChange($character, 14, 15);

        expect($result['adjustment'])->toBe(0);
    });
});

describe('getHitDieForLevel', function () {
    it('returns primary class hit die for single class', function () {
        $character = Character::factory()
            ->withClass('fighter', 5)
            ->create();

        // Fighter has d10
        expect($this->service->getHitDieForLevel($character, 1))->toBe(10);
        expect($this->service->getHitDieForLevel($character, 5))->toBe(10);
    });

    it('returns correct hit die for multiclass levels', function () {
        // Fighter 3 / Wizard 2
        $character = Character::factory()
            ->withClass('fighter', 3) // Levels 1-3: d10
            ->withClass('wizard', 2)  // Levels 4-5: d6
            ->create();

        expect($this->service->getHitDieForLevel($character, 1))->toBe(10);
        expect($this->service->getHitDieForLevel($character, 3))->toBe(10);
        expect($this->service->getHitDieForLevel($character, 4))->toBe(6);
        expect($this->service->getHitDieForLevel($character, 5))->toBe(6);
    });
});
```

**Run:** `docker compose exec php ./vendor/bin/pest tests/Unit/Services/HitPointServiceTest.php`

---

## Phase 2: First Class HP Initialization

### Task 2.1: Update AddClassService to Auto-Initialize HP

**File:** `app/Services/AddClassService.php`

Add to constructor:
```php
public function __construct(
    private MulticlassValidationService $multiclassValidator,
    private HitPointService $hitPointService, // ADD THIS
) {}
```

In the `addClass` method, after creating the pivot, add:
```php
// Auto-initialize HP for first class (level 1)
if ($character->usesCalculatedHp() && $character->characterClasses()->count() === 1) {
    $startingHp = $this->hitPointService->calculateStartingHp($character, $class);
    $character->update([
        'max_hit_points' => $startingHp,
        'current_hit_points' => $startingHp,
        'hp_levels_resolved' => [1],
    ]);
}
```

**Test file:** `tests/Feature/Services/AddClassServiceHpTest.php`

```php
<?php

use App\Models\Character;
use App\Models\CharacterClass;
use App\Services\AddClassService;

describe('AddClassService HP Initialization', function () {
    beforeEach(function () {
        $this->service = app(AddClassService::class);
        $this->fighter = CharacterClass::factory()->create([
            'slug' => 'fighter',
            'name' => 'Fighter',
            'hit_die' => 10,
        ]);
        $this->wizard = CharacterClass::factory()->create([
            'slug' => 'wizard',
            'name' => 'Wizard',
            'hit_die' => 6,
        ]);
    });

    it('auto-initializes HP when first class is added', function () {
        $character = Character::factory()->create([
            'constitution' => 14, // +2 modifier
            'max_hit_points' => null,
            'current_hit_points' => null,
        ]);

        $this->service->addClass($character, $this->fighter);

        $character->refresh();
        expect($character->max_hit_points)->toBe(12); // 10 + 2
        expect($character->current_hit_points)->toBe(12);
        expect($character->hp_levels_resolved)->toBe([1]);
    });

    it('does not change HP when second class is added (multiclass)', function () {
        $character = Character::factory()
            ->withClass('fighter', 5)
            ->create([
                'constitution' => 14,
                'max_hit_points' => 52,
                'current_hit_points' => 52,
                'hp_levels_resolved' => [1, 2, 3, 4, 5],
            ]);

        $this->service->addClass($character, $this->wizard);

        $character->refresh();
        expect($character->max_hit_points)->toBe(52); // Unchanged
        expect($character->current_hit_points)->toBe(52);
    });

    it('does not auto-initialize HP for manual characters', function () {
        $character = Character::factory()->create([
            'constitution' => 14,
            'max_hit_points' => 20, // Manually set
            'current_hit_points' => 20,
            'hp_calculation_method' => 'manual',
        ]);

        $this->service->addClass($character, $this->fighter);

        $character->refresh();
        expect($character->max_hit_points)->toBe(20); // Unchanged
    });
});
```

**Run:** `docker compose exec php ./vendor/bin/pest tests/Feature/Services/AddClassServiceHpTest.php`

---

## Phase 3: Level-Up HP Choice Integration

### Task 3.1: Update HitPointRollChoiceHandler

**File:** `app/Services/ChoiceHandlers/HitPointRollChoiceHandler.php`

Replace the placeholder `hasPendingHpChoice` method:
```php
private function hasPendingHpChoice(Character $character, int $level): bool
{
    // Level 1 HP is auto-set, no choice needed
    if ($level === 1) {
        return false;
    }

    // Check if this level's HP has been resolved
    return !$character->hasResolvedHpForLevel($level);
}
```

Update `resolve` method to mark level as resolved:
```php
public function resolve(Character $character, PendingChoice $choice, array $selection): void
{
    // ... existing validation and HP calculation ...

    // Update character HP
    $character->update([
        'max_hit_points' => $character->max_hit_points + $hpGain,
        'current_hit_points' => $character->current_hit_points + $hpGain,
    ]);

    // Mark this level's HP as resolved
    $level = $choice->levelGranted;
    $character->markHpResolvedForLevel($level);
}
```

**Test updates:** Fix existing tests in `tests/Unit/Services/ChoiceHandlers/HitPointRollChoiceHandlerTest.php` to use new tracking:

```php
it('returns no choices when all levels have HP resolved', function () {
    $character = Character::factory()
        ->withClass('fighter', 3)
        ->create([
            'hp_levels_resolved' => [1, 2, 3],
        ]);

    $choices = $this->handler->getChoices($character);

    expect($choices)->toBeEmpty();
});

it('returns choices for unresolved levels only', function () {
    $character = Character::factory()
        ->withClass('fighter', 3)
        ->create([
            'hp_levels_resolved' => [1], // Only level 1 resolved
        ]);

    $choices = $this->handler->getChoices($character);

    expect($choices)->toHaveCount(2); // Levels 2 and 3 need HP
});

it('marks level as resolved after choice', function () {
    $character = Character::factory()
        ->withClass('fighter', 2)
        ->create([
            'constitution' => 14,
            'max_hit_points' => 12,
            'current_hit_points' => 12,
            'hp_levels_resolved' => [1],
        ]);

    $choices = $this->handler->getChoices($character);
    $this->handler->resolve($character, $choices->first(), ['option' => 'average']);

    expect($character->fresh()->hp_levels_resolved)->toBe([1, 2]);
});
```

**Run:** `docker compose exec php ./vendor/bin/pest tests/Unit/Services/ChoiceHandlers/HitPointRollChoiceHandlerTest.php`

---

### Task 3.2: Update LevelUpService to Not Auto-Apply HP

**File:** `app/Services/LevelUpService.php`

Remove the automatic HP calculation from `levelUp()`. The HP should be handled via the choice system.

Current code to REMOVE or comment:
```php
// REMOVE THIS BLOCK - HP is now handled via choice system
// $hpIncrease = $this->calculateHpIncrease($character);
// $character->update([
//     'max_hit_points' => $character->max_hit_points + $hpIncrease,
//     'current_hit_points' => $character->current_hit_points + $hpIncrease,
// ]);
```

Update `LevelUpResult` DTO to indicate pending HP choice:

**File:** `app/DTOs/LevelUpResult.php`

```php
public function __construct(
    public readonly int $previousLevel,
    public readonly int $newLevel,
    public readonly int $hpIncrease, // Will be 0, calculated later via choice
    public readonly int $newMaxHp,
    public readonly array $featuresGained,
    public readonly array $spellSlots,
    public readonly bool $asiPending,
    public readonly bool $hpChoicePending = true, // ADD THIS
) {}
```

**Test file:** `tests/Unit/Services/LevelUpServiceTest.php`

Update tests to expect `hpChoicePending` instead of automatic HP increase:

```php
it('does not auto-apply HP on level up', function () {
    $character = Character::factory()
        ->withClass('fighter', 1)
        ->create([
            'constitution' => 14,
            'max_hit_points' => 12,
            'current_hit_points' => 12,
            'hp_levels_resolved' => [1],
        ]);

    $result = $this->service->levelUp($character);

    expect($result->hpIncrease)->toBe(0);
    expect($result->hpChoicePending)->toBeTrue();
    expect($character->fresh()->max_hit_points)->toBe(12); // Unchanged
});
```

**Run:** `docker compose exec php ./vendor/bin/pest tests/Unit/Services/LevelUpServiceTest.php`

---

## Phase 4: CON Change Recalculation

### Task 4.1: Create Character Observer for CON Changes

**File:** `app/Observers/CharacterObserver.php`

```php
<?php

namespace App\Observers;

use App\Models\Character;
use App\Services\HitPointService;

class CharacterObserver
{
    public function __construct(
        private HitPointService $hitPointService
    ) {}

    public function updating(Character $character): void
    {
        // Only recalculate if CON changed and character uses calculated HP
        if (!$character->isDirty('constitution')) {
            return;
        }

        if (!$character->usesCalculatedHp()) {
            return;
        }

        $oldCon = $character->getOriginal('constitution');
        $newCon = $character->constitution;

        if ($oldCon === null || $newCon === null) {
            return;
        }

        $result = $this->hitPointService->recalculateForConChange(
            $character,
            $oldCon,
            $newCon
        );

        if ($result['adjustment'] !== 0) {
            $character->max_hit_points = $result['new_max_hp'];
            $character->current_hit_points = $result['new_current_hp'];
        }
    }
}
```

Register in `AppServiceProvider`:

**File:** `app/Providers/AppServiceProvider.php`

```php
use App\Models\Character;
use App\Observers\CharacterObserver;

public function boot(): void
{
    Character::observe(CharacterObserver::class);
    // ... existing code
}
```

**Test file:** `tests/Feature/Observers/CharacterObserverTest.php`

```php
<?php

use App\Models\Character;

describe('CharacterObserver CON Change', function () {
    it('recalculates HP when CON increases', function () {
        $character = Character::factory()
            ->withClass('fighter', 5)
            ->create([
                'constitution' => 14, // +2
                'max_hit_points' => 50,
                'current_hit_points' => 50,
                'hp_calculation_method' => 'calculated',
            ]);

        $character->update(['constitution' => 16]); // +3

        expect($character->max_hit_points)->toBe(55); // +5 (1 per level)
        expect($character->current_hit_points)->toBe(55);
    });

    it('recalculates HP when CON decreases', function () {
        $character = Character::factory()
            ->withClass('fighter', 5)
            ->create([
                'constitution' => 16,
                'max_hit_points' => 55,
                'current_hit_points' => 55,
                'hp_calculation_method' => 'calculated',
            ]);

        $character->update(['constitution' => 14]);

        expect($character->max_hit_points)->toBe(50);
        expect($character->current_hit_points)->toBe(50);
    });

    it('does not recalculate HP for manual characters', function () {
        $character = Character::factory()
            ->withClass('fighter', 5)
            ->create([
                'constitution' => 14,
                'max_hit_points' => 100, // Custom value
                'current_hit_points' => 100,
                'hp_calculation_method' => 'manual',
            ]);

        $character->update(['constitution' => 16]);

        expect($character->max_hit_points)->toBe(100); // Unchanged
    });
});
```

**Run:** `docker compose exec php ./vendor/bin/pest tests/Feature/Observers/CharacterObserverTest.php`

---

## Phase 5: Quality Gates & Polish

### Task 5.1: Run Full Test Suite

```bash
# Unit tests
docker compose exec php ./vendor/bin/pest --testsuite=Unit-Pure
docker compose exec php ./vendor/bin/pest --testsuite=Unit-DB

# Feature tests
docker compose exec php ./vendor/bin/pest --testsuite=Feature-DB
```

### Task 5.2: Code Formatting

```bash
docker compose exec php ./vendor/bin/pint
```

### Task 5.3: Update CHANGELOG.md

Add under `[Unreleased]`:

```markdown
### Added
- Auto-initialize HP when first class is added to character (#254)
- HP choice (roll/average) system for level-ups (levels 2+)
- Automatic HP recalculation when CON changes
- `hp_levels_resolved` tracking column on characters
- `hp_calculation_method` column for calculated vs manual HP modes
- `HitPointService` for centralized HP calculations
- `CharacterObserver` for CON change detection

### Changed
- `LevelUpService` no longer auto-applies HP; creates pending choice instead
- `HitPointRollChoiceHandler` now uses proper level tracking
```

---

## Files Summary

### New Files
- `database/migrations/2025_12_08_000001_add_hp_tracking_to_characters.php`
- `app/Services/HitPointService.php`
- `app/Observers/CharacterObserver.php`
- `tests/Unit/Models/CharacterHpTrackingTest.php`
- `tests/Unit/Services/HitPointServiceTest.php`
- `tests/Feature/Services/AddClassServiceHpTest.php`
- `tests/Feature/Observers/CharacterObserverTest.php`

### Modified Files
- `app/Models/Character.php` (casts + helper methods)
- `app/Services/AddClassService.php` (HP initialization)
- `app/Services/LevelUpService.php` (remove auto HP)
- `app/Services/ChoiceHandlers/HitPointRollChoiceHandler.php` (fix tracking)
- `app/DTOs/LevelUpResult.php` (add hpChoicePending)
- `app/Providers/AppServiceProvider.php` (register observer)
- `tests/Unit/Services/ChoiceHandlers/HitPointRollChoiceHandlerTest.php` (fix tests)
- `tests/Unit/Services/LevelUpServiceTest.php` (update expectations)
- `CHANGELOG.md`

---

## Acceptance Criteria Checklist

- [ ] Adding first class auto-sets starting HP (hit die max + CON mod)
- [ ] Level-up creates HP choice (average or roll) instead of auto-applying
- [ ] HP choice resolution updates max/current HP correctly
- [ ] CON changes adjust HP correctly (calculated mode only)
- [ ] Multiclass HP uses correct hit die for each level
- [ ] Manual HP mode bypasses all auto-calculation
- [ ] All existing tests pass
- [ ] New tests cover all edge cases
- [ ] Code formatted with Pint

---

## Future Enhancements (Not in Scope)

- **Tough Feat**: Retroactive +2 HP per level (separate issue)
- **Migration Script**: Convert existing characters to calculated mode
- **HP History**: Track individual level HP choices for audit

---

**Ready for execution.** Use `laravel:executing-plans` to proceed in batches.
