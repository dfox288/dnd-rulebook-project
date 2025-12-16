# Multiclass Test Character Command Design

**Issue:** #714
**Created:** 2025-12-16
**Status:** Planning

## Overview

Create a command to generate specific multiclass test characters (e.g., Wizard 5 / Cleric 5) for testing multiclass features.

## Current State

### What Works
- API: Full multiclass support via `POST /api/v1/characters/{id}/classes`
- Level-up flow: Can add multiclass at random levels
- Spell slots: Multiclass spellcasting properly calculated
- Validation: Multiclass prerequisites enforced

### What's Missing
- No command to create specific multiclass combinations
- Wizard flow only supports single-class creation
- Level-up `--chaos` picks random classes at random levels

## Proposed Solution

### New Command

```bash
php artisan test:multiclass-combinations \
  --combinations="wizard:5,cleric:5" \
  --count=1 \
  --cleanup
```

**Syntax:** `class:level,class:level` per combination, `|` separates multiple combos

**Examples:**
```bash
# Single multiclass
--combinations="wizard:5,cleric:5"

# Multiple combinations
--combinations="wizard:5,cleric:5|fighter:10,rogue:10"

# Triple class
--combinations="wizard:6,cleric:7,fighter:7"
```

### New Service

**File:** `app/Services/MulticlassCharacterBuilder.php`

```php
<?php

namespace App\Services;

use App\Models\Character;
use App\Models\CharacterClass;

class MulticlassCharacterBuilder
{
    public function __construct(
        private WizardFlowService $wizardFlow,
        private AddClassService $addClassService,
        private LevelUpService $levelUpService,
    ) {}

    /**
     * Build a character with specific class levels.
     *
     * @param array $classLevels [['class' => 'phb:wizard', 'level' => 5], ...]
     */
    public function build(array $classLevels, ?int $seed = null): Character
    {
        // 1. Create character via wizard flow with first class
        $firstClass = array_shift($classLevels);
        $character = $this->createViaWizard($firstClass, $seed);

        // 2. Add additional classes
        foreach ($classLevels as $classSpec) {
            $this->addClassService->addClass(
                $character,
                CharacterClass::where('slug', $classSpec['class'])->firstOrFail(),
                force: true
            );
        }

        // 3. Level each class to target (first class already at level 1)
        $this->levelToTarget($character, $firstClass['class'], $firstClass['level'] - 1);

        foreach ($classLevels as $classSpec) {
            // Additional classes start at level 1
            $this->levelToTarget($character, $classSpec['class'], $classSpec['level'] - 1);
        }

        return $character->fresh();
    }

    private function createViaWizard(array $classSpec, ?int $seed): Character
    {
        // Use existing WizardFlowService or FlowExecutor
        // Force class to $classSpec['class']
    }

    private function levelToTarget(Character $character, string $classSlug, int $levelsToGain): void
    {
        for ($i = 0; $i < $levelsToGain; $i++) {
            $this->levelUpService->levelUp($character, $classSlug);
        }
    }
}
```

### New Command

**File:** `app/Console/Commands/TestMulticlassCombinationsCommand.php`

```php
<?php

namespace App\Console\Commands;

use App\Services\MulticlassCharacterBuilder;
use Illuminate\Console\Command;

class TestMulticlassCombinationsCommand extends Command
{
    protected $signature = 'test:multiclass-combinations
        {--combinations= : Class:level pairs, e.g., "wizard:5,cleric:5"}
        {--count=1 : Number of characters to create}
        {--seed= : Random seed for reproducibility}
        {--cleanup : Delete characters after creation}
        {--force : Bypass multiclass prerequisites}';

    protected $description = 'Create specific multiclass test characters';

    public function handle(MulticlassCharacterBuilder $builder): int
    {
        $combinations = $this->parseCombinations($this->option('combinations'));
        $count = (int) $this->option('count');
        $seed = $this->option('seed') ? (int) $this->option('seed') : null;

        $this->info("Creating {$count} character(s) for each combination...\n");

        foreach ($combinations as $combo) {
            $this->createCombination($builder, $combo, $count, $seed);
        }

        return Command::SUCCESS;
    }

    private function parseCombinations(?string $spec): array
    {
        if (!$spec) {
            $this->error('--combinations is required');
            exit(1);
        }

        $combinations = [];
        foreach (explode('|', $spec) as $combo) {
            $classes = [];
            foreach (explode(',', $combo) as $classLevel) {
                [$class, $level] = explode(':', $classLevel);
                $classes[] = [
                    'class' => $this->resolveClassSlug($class),
                    'level' => (int) $level,
                ];
            }
            $combinations[] = $classes;
        }

        return $combinations;
    }

    private function resolveClassSlug(string $class): string
    {
        // Handle shorthand: "wizard" -> "phb:wizard"
        if (!str_contains($class, ':')) {
            return "phb:{$class}";
        }
        return $class;
    }

    private function createCombination(
        MulticlassCharacterBuilder $builder,
        array $combo,
        int $count,
        ?int $seed
    ): void {
        $comboName = collect($combo)
            ->map(fn($c) => "{$c['class']}:{$c['level']}")
            ->join(' / ');

        $this->info("Combination: {$comboName}");

        for ($i = 0; $i < $count; $i++) {
            $iterSeed = $seed !== null ? $seed + $i : null;

            try {
                $character = $builder->build($combo, $iterSeed);
                $this->line("  ✓ Created: {$character->public_id} ({$character->name})");

                if ($this->option('cleanup')) {
                    $character->delete();
                    $this->line("    (cleaned up)");
                }
            } catch (\Exception $e) {
                $this->error("  ✗ Failed: {$e->getMessage()}");
            }
        }

        $this->newLine();
    }
}
```

## Implementation Steps

### Phase 1: Core Service
1. [ ] Create `MulticlassCharacterBuilder` service
2. [ ] Implement `build()` method with wizard flow integration
3. [ ] Handle class slug resolution (shorthand support)
4. [ ] Add proper error handling for invalid combinations

### Phase 2: Command
1. [ ] Create `TestMulticlassCombinationsCommand`
2. [ ] Implement combination parsing
3. [ ] Add `--cleanup`, `--seed`, `--force` options
4. [ ] Output character public_ids

### Phase 3: Integration
1. [ ] Wire up service in `AppServiceProvider`
2. [ ] Add tests for the builder service
3. [ ] Test with real combinations (Wizard/Cleric, Fighter/Rogue)

### Phase 4: Documentation
1. [ ] Update `11-commands-reference.md` with new command
2. [ ] Add examples to issue #714

## Validation

### Test Cases
1. **Dual caster:** Wizard 5 / Cleric 5 - different abilities (INT/WIS)
2. **Caster + martial:** Wizard 5 / Fighter 5 - one caster, one non-caster
3. **Triple class:** Fighter 6 / Rogue 7 / Wizard 7 - level 20 total
4. **Same ability:** Sorcerer 5 / Warlock 5 - both CHA casters
5. **Half casters:** Paladin 5 / Ranger 5 - shared half-caster slots

### Success Criteria
- [ ] Command creates exact class/level combinations
- [ ] Total level doesn't exceed 20
- [ ] Spells assigned correctly for each class
- [ ] public_id outputted for frontend testing
- [ ] `--cleanup` removes characters after

## Files to Create/Modify

| File | Action |
|------|--------|
| `app/Services/MulticlassCharacterBuilder.php` | Create |
| `app/Console/Commands/TestMulticlassCombinationsCommand.php` | Create |
| `tests/Feature/Services/MulticlassCharacterBuilderTest.php` | Create |
| `.claude/rules/11-commands-reference.md` | Update |

## Dependencies

- `AddClassService` - existing, adds class to character
- `LevelUpService` - existing, handles level-up flow
- `WizardFlowService` / `FlowExecutor` - existing, creates base character

## Related

- Issue #714 - This plan
- Issue #692 - Multiclass spellcasting API (merged)
- Issue #631 - Frontend multiclass UI (needs test character)
