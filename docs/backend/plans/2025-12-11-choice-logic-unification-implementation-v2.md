# Choice Logic Unification - Implementation Plan (Revised)

**Design Document:** `2025-12-11-choice-logic-unification-design-v2.md`
**GitHub Issue:** [#501](https://github.com/dfox288/ledger-of-heroes/issues/501)
**Runner:** Sail (`sail artisan`, `sail composer`, etc.)

---

## Phase 1: Scaffolding & Data Model

### Task 1.1: Create feature branch

```bash
git checkout -b feature/issue-501-choice-logic-unification-v2
```

---

### Task 1.2: Create migration for new tables

**File:** `database/migrations/2025_12_11_000001_create_choice_group_tables.php`

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('entity_choice_groups', function (Blueprint $table) {
            $table->id();
            $table->string('reference_type', 255);
            $table->unsignedBigInteger('reference_id');
            $table->unsignedTinyInteger('level')->nullable()
                ->comment('NULL for race/background, set for class features');
            $table->string('group_key', 100);
            $table->string('choice_type', 50)
                ->comment('proficiency, spell, language, equipment, ability_score');
            $table->unsignedTinyInteger('choose_count')->default(1);
            $table->boolean('is_optional')->default(false);

            // Ability score specific
            $table->tinyInteger('value')->nullable()
                ->comment('Bonus/penalty value for ability score choices');
            $table->string('choice_constraint', 50)->nullable()
                ->comment('e.g., "different" = cannot pick same option twice');

            $table->index(['reference_type', 'reference_id'], 'idx_ecg_reference');
            $table->index('choice_type', 'idx_ecg_type');
            $table->unique(
                ['reference_type', 'reference_id', 'level', 'group_key'],
                'uk_ecg_group'
            );
        });

        Schema::create('choice_group_options', function (Blueprint $table) {
            $table->id();
            $table->foreignId('choice_group_id')
                ->constrained('entity_choice_groups')
                ->cascadeOnDelete();
            $table->string('option_type', 255)
                ->comment('Model class: Skill, Language, Spell, Item, ProficiencyType, AbilityScore');
            $table->unsignedBigInteger('option_id')->nullable()
                ->comment('NULL = any matching constraints');
            $table->unsignedTinyInteger('quantity')->default(1);
            $table->unsignedTinyInteger('sort_order')->default(0);

            // Equipment: option grouping (a, b, c)
            $table->char('option_letter', 1)->nullable();
            $table->string('option_label', 255)->nullable()
                ->comment('e.g., "a rapier", "two handaxes"');

            // Spell constraints (when option_id is NULL)
            $table->unsignedTinyInteger('max_spell_level')->nullable();
            $table->foreignId('spell_school_id')->nullable()
                ->constrained('spell_schools')->nullOnDelete();
            $table->foreignId('spell_class_id')->nullable()
                ->constrained('classes')->nullOnDelete();
            $table->boolean('is_ritual_only')->default(false);

            // Proficiency constraints (when option_id is NULL)
            $table->string('proficiency_subcategory', 50)->nullable();

            $table->index('choice_group_id', 'idx_cgo_group');
            $table->index(['option_type', 'option_id'], 'idx_cgo_option');
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('choice_group_options');
        Schema::dropIfExists('entity_choice_groups');
    }
};
```

**Verification:** `sail artisan migrate` succeeds

---

### Task 1.3: Create EntityChoiceGroup model

**File:** `app/Models/EntityChoiceGroup.php`

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Relations\HasMany;
use Illuminate\Database\Eloquent\Relations\MorphTo;

/**
 * EntityChoiceGroup - Defines what choices an entity offers.
 *
 * Table: entity_choice_groups
 * Used by: Race, CharacterClass, Background, Feat, ClassFeature
 *
 * Replaces is_choice/choice_group columns scattered across entity_* tables.
 */
class EntityChoiceGroup extends BaseModel
{
    protected $table = 'entity_choice_groups';

    protected $fillable = [
        'reference_type',
        'reference_id',
        'level',
        'group_key',
        'choice_type',
        'choose_count',
        'is_optional',
        'value',
        'choice_constraint',
    ];

    protected $casts = [
        'reference_id' => 'integer',
        'level' => 'integer',
        'choose_count' => 'integer',
        'is_optional' => 'boolean',
        'value' => 'integer',
    ];

    public function reference(): MorphTo
    {
        return $this->morphTo(null, 'reference_type', 'reference_id');
    }

    public function options(): HasMany
    {
        return $this->hasMany(ChoiceGroupOption::class, 'choice_group_id')
            ->orderBy('sort_order');
    }
}
```

---

### Task 1.4: Create ChoiceGroupOption model

**File:** `app/Models/ChoiceGroupOption.php`

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Database\Eloquent\Relations\MorphTo;

/**
 * ChoiceGroupOption - Available options within a choice group.
 *
 * Table: choice_group_options
 *
 * Each row represents either:
 * - A specific option (option_id set)
 * - A constrained "any" option (option_id NULL, constraints set)
 */
class ChoiceGroupOption extends BaseModel
{
    protected $table = 'choice_group_options';

    protected $fillable = [
        'choice_group_id',
        'option_type',
        'option_id',
        'quantity',
        'sort_order',
        'option_letter',
        'option_label',
        'max_spell_level',
        'spell_school_id',
        'spell_class_id',
        'is_ritual_only',
        'proficiency_subcategory',
    ];

    protected $casts = [
        'choice_group_id' => 'integer',
        'option_id' => 'integer',
        'quantity' => 'integer',
        'sort_order' => 'integer',
        'max_spell_level' => 'integer',
        'spell_school_id' => 'integer',
        'spell_class_id' => 'integer',
        'is_ritual_only' => 'boolean',
    ];

    public function choiceGroup(): BelongsTo
    {
        return $this->belongsTo(EntityChoiceGroup::class, 'choice_group_id');
    }

    public function option(): MorphTo
    {
        return $this->morphTo(null, 'option_type', 'option_id');
    }

    public function spellSchool(): BelongsTo
    {
        return $this->belongsTo(SpellSchool::class, 'spell_school_id');
    }

    public function spellClass(): BelongsTo
    {
        return $this->belongsTo(CharacterClass::class, 'spell_class_id');
    }

    /**
     * Check if this option is unrestricted (any of the type).
     */
    public function isUnrestricted(): bool
    {
        return $this->option_id === null;
    }

    /**
     * Check if this option has spell-specific constraints.
     */
    public function hasSpellConstraints(): bool
    {
        return $this->max_spell_level !== null
            || $this->spell_school_id !== null
            || $this->spell_class_id !== null
            || $this->is_ritual_only;
    }
}
```

---

### Task 1.5: Create factories

**File:** `database/factories/EntityChoiceGroupFactory.php`

```php
<?php

namespace Database\Factories;

use App\Models\CharacterClass;
use App\Models\EntityChoiceGroup;
use App\Models\Race;
use Illuminate\Database\Eloquent\Factories\Factory;

/**
 * @extends Factory<EntityChoiceGroup>
 */
class EntityChoiceGroupFactory extends Factory
{
    protected $model = EntityChoiceGroup::class;

    public function definition(): array
    {
        return [
            'reference_type' => CharacterClass::class,
            'reference_id' => CharacterClass::factory(),
            'level' => 1,
            'group_key' => 'skill_choice_1',
            'choice_type' => 'proficiency',
            'choose_count' => 2,
            'is_optional' => false,
        ];
    }

    public function forRace(): static
    {
        return $this->state(fn () => [
            'reference_type' => Race::class,
            'reference_id' => Race::factory(),
            'level' => null,
        ]);
    }

    public function forAbilityScore(int $value = 2): static
    {
        return $this->state(fn () => [
            'choice_type' => 'ability_score',
            'group_key' => 'ability_score_1',
            'choose_count' => 1,
            'value' => $value,
            'choice_constraint' => 'different',
        ]);
    }

    public function forLanguage(): static
    {
        return $this->state(fn () => [
            'choice_type' => 'language',
            'group_key' => 'language_choice_1',
            'choose_count' => 1,
        ]);
    }

    public function forSpell(): static
    {
        return $this->state(fn () => [
            'choice_type' => 'spell',
            'group_key' => 'feature_cantrip',
        ]);
    }

    public function forEquipment(): static
    {
        return $this->state(fn () => [
            'choice_type' => 'equipment',
            'group_key' => 'starting_equipment_1',
            'choose_count' => 1,
        ]);
    }
}
```

**File:** `database/factories/ChoiceGroupOptionFactory.php`

```php
<?php

namespace Database\Factories;

use App\Models\ChoiceGroupOption;
use App\Models\EntityChoiceGroup;
use App\Models\Skill;
use Illuminate\Database\Eloquent\Factories\Factory;

/**
 * @extends Factory<ChoiceGroupOption>
 */
class ChoiceGroupOptionFactory extends Factory
{
    protected $model = ChoiceGroupOption::class;

    public function definition(): array
    {
        return [
            'choice_group_id' => EntityChoiceGroup::factory(),
            'option_type' => Skill::class,
            'option_id' => Skill::factory(),
            'quantity' => 1,
            'sort_order' => 0,
        ];
    }

    public function unrestricted(): static
    {
        return $this->state(fn () => [
            'option_id' => null,
        ]);
    }

    public function forSpell(): static
    {
        return $this->state(fn () => [
            'option_type' => \App\Models\Spell::class,
            'option_id' => \App\Models\Spell::factory(),
        ]);
    }

    public function withSpellConstraints(int $maxLevel, ?int $classId = null): static
    {
        return $this->state(fn () => [
            'option_type' => \App\Models\Spell::class,
            'option_id' => null,
            'max_spell_level' => $maxLevel,
            'spell_class_id' => $classId,
        ]);
    }

    public function forEquipmentOption(string $letter, string $label): static
    {
        return $this->state(fn () => [
            'option_type' => \App\Models\Item::class,
            'option_letter' => $letter,
            'option_label' => $label,
        ]);
    }
}
```

**Verification:** `sail artisan tinker` - factories instantiate without errors

---

### Task 1.6: Add choiceGroups relationship to entities

Add to `Race`, `CharacterClass`, `Background`, `Feat`, `ClassFeature`:

```php
use App\Models\EntityChoiceGroup;

public function choiceGroups(): \Illuminate\Database\Eloquent\Relations\MorphMany
{
    return $this->morphMany(EntityChoiceGroup::class, null, 'reference_type', 'reference_id');
}
```

**Verification:** `$race->choiceGroups` returns collection

---

### Task 1.7: Commit Phase 1

```bash
git add -A
git commit -m "feat(#501): Add choice group tables and models

- Create entity_choice_groups table
- Create choice_group_options table
- Add EntityChoiceGroup, ChoiceGroupOption models
- Add factories for new models
- Add choiceGroups relationship to Race, CharacterClass, Background, Feat, ClassFeature"
```

---

## Phase 2: Resolver Interface & Implementations

### Task 2.1: Create ChoiceResolverInterface

**File:** `app/Services/ChoiceHandlers/Contracts/ChoiceResolverInterface.php`

```php
<?php

namespace App\Services\ChoiceHandlers\Contracts;

use App\DTOs\PendingChoice;
use App\Models\Character;
use Illuminate\Support\Collection;

/**
 * Interface for choice resolvers that read from entity_choice_groups.
 *
 * Resolvers replace the old handlers that read from entity_*.is_choice columns.
 * They must produce PendingChoice DTOs compatible with the existing API contract.
 */
interface ChoiceResolverInterface
{
    /**
     * Get the choice type this resolver handles.
     */
    public function getType(): string;

    /**
     * Get pending choices for a character from entity_choice_groups.
     *
     * @return Collection<int, PendingChoice>
     */
    public function getChoices(Character $character): Collection;

    /**
     * Resolve a choice with the given selection.
     */
    public function resolve(Character $character, PendingChoice $choice, array $selection): void;

    /**
     * Check if a resolved choice can be undone.
     */
    public function canUndo(Character $character, PendingChoice $choice): bool;

    /**
     * Undo a previously resolved choice.
     */
    public function undo(Character $character, PendingChoice $choice): void;
}
```

---

### Task 2.2: Create AbstractChoiceResolver base class

**File:** `app/Services/ChoiceHandlers/Resolvers/AbstractChoiceResolver.php`

```php
<?php

namespace App\Services\ChoiceHandlers\Resolvers;

use App\DTOs\PendingChoice;
use App\Models\Background;
use App\Models\Character;
use App\Models\CharacterClass;
use App\Models\ClassFeature;
use App\Models\EntityChoiceGroup;
use App\Models\Feat;
use App\Models\Race;
use App\Services\ChoiceHandlers\AbstractChoiceHandler;
use App\Services\ChoiceHandlers\Contracts\ChoiceResolverInterface;
use Illuminate\Support\Collection;

/**
 * Base class for choice resolvers.
 *
 * Extends AbstractChoiceHandler to reuse generateChoiceId/parseChoiceId.
 */
abstract class AbstractChoiceResolver extends AbstractChoiceHandler implements ChoiceResolverInterface
{
    abstract public function getType(): string;

    /**
     * Get choice groups applicable to this character for this resolver's type.
     */
    protected function getApplicableChoiceGroups(Character $character): Collection
    {
        return EntityChoiceGroup::query()
            ->where('choice_type', $this->getType())
            ->where(function ($query) use ($character) {
                // Race choices
                if ($character->race) {
                    $query->orWhere(function ($q) use ($character) {
                        $q->where('reference_type', Race::class)
                            ->where('reference_id', $character->race->id);
                    });
                    // Include parent race for subraces
                    if ($character->race->parent_race_id && $character->race->parent) {
                        $query->orWhere(function ($q) use ($character) {
                            $q->where('reference_type', Race::class)
                                ->where('reference_id', $character->race->parent->id);
                        });
                    }
                }

                // Background choices
                if ($character->background) {
                    $query->orWhere(function ($q) use ($character) {
                        $q->where('reference_type', Background::class)
                            ->where('reference_id', $character->background->id);
                    });
                }

                // Class choices at or below character's level
                foreach ($character->characterClasses as $charClass) {
                    $query->orWhere(function ($sub) use ($charClass) {
                        $sub->where('reference_type', CharacterClass::class)
                            ->where('reference_id', $charClass->characterClass->id)
                            ->where(function ($l) use ($charClass) {
                                $l->whereNull('level')
                                    ->orWhere('level', '<=', $charClass->level);
                            });
                    });

                    // Subclass feature choices
                    if ($charClass->subclass) {
                        $featureIds = $charClass->subclass->features->pluck('id');
                        if ($featureIds->isNotEmpty()) {
                            $query->orWhere(function ($sub) use ($featureIds, $charClass) {
                                $sub->where('reference_type', ClassFeature::class)
                                    ->whereIn('reference_id', $featureIds)
                                    ->where(function ($l) use ($charClass) {
                                        $l->whereNull('level')
                                            ->orWhere('level', '<=', $charClass->level);
                                    });
                            });
                        }
                    }
                }
            })
            ->with('options', 'reference')
            ->get();
    }

    /**
     * Get source type string from reference_type.
     */
    protected function getSourceType(string $referenceType): string
    {
        return match ($referenceType) {
            Race::class => 'race',
            CharacterClass::class => 'class',
            Background::class => 'background',
            Feat::class => 'feat',
            ClassFeature::class => 'subclass_feature',
            default => 'unknown',
        };
    }

    /**
     * Get source slug for choice ID generation.
     */
    protected function getSourceSlug(EntityChoiceGroup $group): string
    {
        return $group->reference?->slug ?? '';
    }

    /**
     * Get human-readable source name.
     */
    protected function getSourceName(EntityChoiceGroup $group): string
    {
        return $group->reference?->name ?? 'Unknown';
    }
}
```

---

### Task 2.3: Create ProficiencyChoiceResolver

**File:** `app/Services/ChoiceHandlers/Resolvers/ProficiencyChoiceResolver.php`

```php
<?php

namespace App\Services\ChoiceHandlers\Resolvers;

use App\DTOs\PendingChoice;
use App\Exceptions\InvalidSelectionException;
use App\Models\Character;
use App\Models\CharacterProficiency;
use App\Models\EntityChoiceGroup;
use App\Models\ProficiencyType;
use App\Models\Skill;
use Illuminate\Support\Collection;

class ProficiencyChoiceResolver extends AbstractChoiceResolver
{
    public function getType(): string
    {
        return 'proficiency';
    }

    public function getChoices(Character $character): Collection
    {
        $choiceGroups = $this->getApplicableChoiceGroups($character);
        $choices = collect();

        foreach ($choiceGroups as $group) {
            $choice = $this->buildPendingChoice($character, $group);
            if ($choice) {
                $choices->push($choice);
            }
        }

        return $choices;
    }

    private function buildPendingChoice(Character $character, EntityChoiceGroup $group): ?PendingChoice
    {
        $source = $this->getSourceType($group->reference_type);
        $sourceSlug = $this->getSourceSlug($group);

        if (! $sourceSlug) {
            return null;
        }

        // Build options from choice_group_options
        $options = $this->buildOptions($group);

        // Get existing selections for this choice group
        $existingSelections = $this->getExistingSelections($character, $source, $group->group_key);
        $selected = $existingSelections->pluck('skill_slug')
            ->merge($existingSelections->pluck('proficiency_type_slug'))
            ->filter()
            ->values()
            ->toArray();

        $remaining = max(0, $group->choose_count - count($selected));

        // Determine subtype from first option
        $firstOption = $group->options->first();
        $subtype = $firstOption?->option_type === Skill::class ? 'skill' : 'tool';

        return new PendingChoice(
            id: $this->generateChoiceId('proficiency', $source, $sourceSlug, $group->level ?? 1, $group->group_key),
            type: 'proficiency',
            subtype: $subtype,
            source: $source,
            sourceName: $this->getSourceName($group),
            levelGranted: $group->level ?? 1,
            required: ! $group->is_optional,
            quantity: $group->choose_count,
            remaining: $remaining,
            selected: $selected,
            options: $options,
            optionsEndpoint: $this->buildOptionsEndpoint($group),
            metadata: [
                'choice_group' => $group->group_key,
                'proficiency_subcategory' => $firstOption?->proficiency_subcategory,
            ],
        );
    }

    private function buildOptions(EntityChoiceGroup $group): array
    {
        $options = [];

        foreach ($group->options as $opt) {
            if ($opt->option_id !== null) {
                // Specific option
                $entity = $opt->option;
                if (! $entity) {
                    continue;
                }

                $options[] = $opt->option_type === Skill::class
                    ? ['type' => 'skill', 'slug' => $entity->slug, 'name' => $entity->name]
                    : ['type' => 'proficiency_type', 'slug' => $entity->slug, 'name' => $entity->name];
            } elseif ($opt->proficiency_subcategory) {
                // Unrestricted with subcategory - options come from lookup endpoint
                // Return empty options, frontend uses optionsEndpoint
                return [];
            }
        }

        return $options;
    }

    private function buildOptionsEndpoint(EntityChoiceGroup $group): ?string
    {
        $firstOption = $group->options->first();

        if ($firstOption?->proficiency_subcategory) {
            $subcat = $firstOption->proficiency_subcategory;

            return "/api/v1/lookups/proficiency-types?subcategory={$subcat}";
        }

        return null;
    }

    private function getExistingSelections(Character $character, string $source, string $choiceGroup): Collection
    {
        return $character->proficiencies()
            ->where('source', $source)
            ->where('choice_group', $choiceGroup)
            ->get();
    }

    public function resolve(Character $character, PendingChoice $choice, array $selection): void
    {
        $parsed = $this->parseChoiceId($choice->id);
        $source = $parsed['source'];
        $choiceGroup = $parsed['group'];

        $selected = $selection['selected'] ?? [];
        if (empty($selected)) {
            throw new InvalidSelectionException($choice->id, 'empty', 'Selection cannot be empty');
        }

        if (count($selected) !== $choice->quantity) {
            throw new InvalidSelectionException(
                $choice->id,
                implode(',', $selected),
                "Must select exactly {$choice->quantity} proficiencies"
            );
        }

        // Clear existing and create new
        $character->proficiencies()
            ->where('source', $source)
            ->where('choice_group', $choiceGroup)
            ->delete();

        foreach ($selected as $slug) {
            // Determine if skill or proficiency type
            $skill = Skill::where('slug', $slug)->first();
            $profType = $skill ? null : ProficiencyType::where('slug', $slug)->first();

            CharacterProficiency::create([
                'character_id' => $character->id,
                'skill_slug' => $skill?->slug,
                'proficiency_type_slug' => $profType?->slug,
                'source' => $source,
                'choice_group' => $choiceGroup,
            ]);
        }

        $character->load('proficiencies');
    }

    public function canUndo(Character $character, PendingChoice $choice): bool
    {
        return true;
    }

    public function undo(Character $character, PendingChoice $choice): void
    {
        $parsed = $this->parseChoiceId($choice->id);

        $character->proficiencies()
            ->where('source', $parsed['source'])
            ->where('choice_group', $parsed['group'])
            ->delete();

        $character->load('proficiencies');
    }
}
```

---

### Task 2.4: Create LanguageChoiceResolver

**File:** `app/Services/ChoiceHandlers/Resolvers/LanguageChoiceResolver.php`

Similar structure to ProficiencyChoiceResolver, reads from entity_choice_groups, writes to character_languages.

---

### Task 2.5: Create AbilityScoreChoiceResolver

**File:** `app/Services/ChoiceHandlers/Resolvers/AbilityScoreChoiceResolver.php`

Handles ability score choices from races. Uses `group.value` for bonus amount and `group.choice_constraint` for validation.

---

### Task 2.6: Create EquipmentChoiceResolver

**File:** `app/Services/ChoiceHandlers/Resolvers/EquipmentChoiceResolver.php`

Handles starting equipment choices. Uses `option_letter` and `option_label` for a/b/c options.

---

### Task 2.7: Commit Phase 2

```bash
git add -A
git commit -m "feat(#501): Add choice resolvers

- Add ChoiceResolverInterface contract
- Add AbstractChoiceResolver base class
- Add ProficiencyChoiceResolver
- Add LanguageChoiceResolver
- Add AbilityScoreChoiceResolver
- Add EquipmentChoiceResolver"
```

---

## Phase 3: Parser Trait

### Task 3.1: Create ParsesChoiceGroups trait

**File:** `app/Services/Parsers/Concerns/ParsesChoiceGroups.php`

```php
<?php

namespace App\Services\Parsers\Concerns;

use App\Models\ChoiceGroupOption;
use App\Models\EntityChoiceGroup;

/**
 * Trait ParsesChoiceGroups
 *
 * Provides methods for creating choice groups during XML parsing.
 * Replaces the old pattern of writing is_choice/choice_group to entity_* tables.
 */
trait ParsesChoiceGroups
{
    /**
     * Create a choice group for an entity.
     */
    protected function createChoiceGroup(
        string $referenceType,
        int $referenceId,
        string $choiceType,
        string $groupKey,
        int $chooseCount,
        ?int $level = null,
        array $extra = []
    ): EntityChoiceGroup {
        return EntityChoiceGroup::create([
            'reference_type' => $referenceType,
            'reference_id' => $referenceId,
            'choice_type' => $choiceType,
            'group_key' => $groupKey,
            'choose_count' => $chooseCount,
            'level' => $level,
            'is_optional' => $extra['is_optional'] ?? false,
            'value' => $extra['value'] ?? null,
            'choice_constraint' => $extra['choice_constraint'] ?? null,
        ]);
    }

    /**
     * Add a specific option to a choice group.
     */
    protected function addChoiceOption(
        EntityChoiceGroup $group,
        string $optionType,
        ?int $optionId = null,
        array $attributes = []
    ): ChoiceGroupOption {
        return ChoiceGroupOption::create([
            'choice_group_id' => $group->id,
            'option_type' => $optionType,
            'option_id' => $optionId,
            'quantity' => $attributes['quantity'] ?? 1,
            'sort_order' => $attributes['sort_order'] ?? 0,
            'option_letter' => $attributes['option_letter'] ?? null,
            'option_label' => $attributes['option_label'] ?? null,
            'max_spell_level' => $attributes['max_spell_level'] ?? null,
            'spell_school_id' => $attributes['spell_school_id'] ?? null,
            'spell_class_id' => $attributes['spell_class_id'] ?? null,
            'is_ritual_only' => $attributes['is_ritual_only'] ?? false,
            'proficiency_subcategory' => $attributes['proficiency_subcategory'] ?? null,
        ]);
    }

    /**
     * Add multiple specific options to a choice group.
     */
    protected function addChoiceOptions(
        EntityChoiceGroup $group,
        string $optionType,
        array $optionIds
    ): void {
        foreach ($optionIds as $index => $optionId) {
            $this->addChoiceOption($group, $optionType, $optionId, ['sort_order' => $index]);
        }
    }

    /**
     * Add an unrestricted option with constraints.
     */
    protected function addUnrestrictedOption(
        EntityChoiceGroup $group,
        string $optionType,
        array $constraints = []
    ): ChoiceGroupOption {
        return $this->addChoiceOption($group, $optionType, null, $constraints);
    }
}
```

---

### Task 3.2: Commit Phase 3

```bash
git add -A
git commit -m "feat(#501): Add ParsesChoiceGroups trait

- Provides createChoiceGroup() for parsers
- Provides addChoiceOption() and addChoiceOptions()
- Provides addUnrestrictedOption() for constrained choices"
```

---

## Phase 4: Parser Updates

### Task 4.1: Update RaceXmlParser

Add `use ParsesChoiceGroups;` and update skill/language/ability score choice parsing.

### Task 4.2: Update ClassXmlParser

Add `use ParsesChoiceGroups;` and update skill/equipment choice parsing.

### Task 4.3: Update BackgroundXmlParser

Add `use ParsesChoiceGroups;` and update skill/language/equipment choice parsing.

### Task 4.4: Update FeatXmlParser

Add `use ParsesChoiceGroups;` and update proficiency/language choice parsing.

### Task 4.5: Update SubclassXmlParser / ClassFeatureXmlParser

Add `use ParsesChoiceGroups;` and update spell choice parsing.

### Task 4.6: Commit Phase 4

```bash
git add -A
git commit -m "feat(#501): Update parsers to use choice groups

- RaceXmlParser: proficiency, language, ability_score choices
- ClassXmlParser: proficiency, equipment choices
- BackgroundXmlParser: proficiency, language, equipment choices
- FeatXmlParser: proficiency, language choices
- SubclassXmlParser: spell choices"
```

---

## Phase 5: Update SpellChoiceHandler

### Task 5.1: Update SpellChoiceHandler to read subclass spells from new tables

The handler already reads class cantrips/spells from `class_level_progression`. Update only the subclass feature spell logic to read from `entity_choice_groups` instead of `entity_spells.is_choice`.

---

## Phase 6: Wire Up Resolvers

### Task 6.1: Register resolvers in AppServiceProvider

```php
// In boot() or register()
$choiceService = $this->app->make(CharacterChoiceService::class);

// Register resolvers (replace old handlers)
$choiceService->registerHandler($this->app->make(ProficiencyChoiceResolver::class));
$choiceService->registerHandler($this->app->make(LanguageChoiceResolver::class));
$choiceService->registerHandler($this->app->make(AbilityScoreChoiceResolver::class));
$choiceService->registerHandler($this->app->make(EquipmentChoiceResolver::class));

// Keep SpellChoiceHandler (updated to use new tables for subclass features)
$choiceService->registerHandler($this->app->make(SpellChoiceHandler::class));

// Keep other handlers unchanged
$choiceService->registerHandler($this->app->make(OptionalFeatureChoiceHandler::class));
// ... etc
```

---

## Phase 7: Drop Old Columns

### Task 7.1: Create migration to drop old choice columns

**File:** `database/migrations/2025_12_11_000002_drop_old_choice_columns.php`

```php
Schema::table('entity_proficiencies', function (Blueprint $table) {
    $table->dropIndex('proficiencies_is_choice_index');
    $table->dropIndex('proficiencies_choice_group_index');
    $table->dropColumn(['is_choice', 'choice_group', 'choice_option', 'quantity']);
});

Schema::table('entity_languages', function (Blueprint $table) {
    $table->dropColumn(['is_choice', 'choice_group', 'choice_option', 'quantity']);
});

Schema::table('entity_modifiers', function (Blueprint $table) {
    $table->dropColumn(['is_choice', 'choice_count', 'choice_constraint']);
});

Schema::table('entity_spells', function (Blueprint $table) {
    $table->dropIndex('entity_spells_is_choice_index');
    $table->dropIndex('entity_spells_choice_group_index');
    $table->dropColumn(['is_choice', 'choice_count', 'choice_group', 'max_level', 'is_ritual_only']);
    // Keep school_id and class_id - used for non-choice spell grants
});

Schema::table('entity_items', function (Blueprint $table) {
    $table->dropIndex('entity_items_choice_group_index');
    $table->dropColumn(['is_choice', 'choice_group', 'choice_option', 'choice_description']);
});
```

---

## Phase 8: Reimport

```bash
sail artisan migrate:fresh
sail artisan import:all
```

---

## Phase 9: Delete Old Handlers

```bash
rm app/Services/ChoiceHandlers/ProficiencyChoiceHandler.php
rm app/Services/ChoiceHandlers/LanguageChoiceHandler.php
rm app/Services/ChoiceHandlers/AbilityScoreChoiceHandler.php
rm app/Services/ChoiceHandlers/EquipmentChoiceHandler.php
```

Also delete corresponding tests.

---

## Phase 10: Verify

```bash
sail artisan test --testsuite=Unit-Pure
sail artisan test --testsuite=Unit-DB
sail artisan test --testsuite=Feature-DB
sail artisan test:wizard-flow --count=10 --chaos
sail composer pint
```

---

## Quality Gates

Before merging:
- [ ] All test suites pass
- [ ] Pint formatting clean
- [ ] Wizard flow chaos tests pass
- [ ] Manual test: create character, make choices, switch race
- [ ] Verify entity_choice_groups populated for sample entities

---

## Files Summary

### New Files
- `database/migrations/2025_12_11_000001_create_choice_group_tables.php`
- `database/migrations/2025_12_11_000002_drop_old_choice_columns.php`
- `app/Models/EntityChoiceGroup.php`
- `app/Models/ChoiceGroupOption.php`
- `database/factories/EntityChoiceGroupFactory.php`
- `database/factories/ChoiceGroupOptionFactory.php`
- `app/Services/ChoiceHandlers/Contracts/ChoiceResolverInterface.php`
- `app/Services/ChoiceHandlers/Resolvers/AbstractChoiceResolver.php`
- `app/Services/ChoiceHandlers/Resolvers/ProficiencyChoiceResolver.php`
- `app/Services/ChoiceHandlers/Resolvers/LanguageChoiceResolver.php`
- `app/Services/ChoiceHandlers/Resolvers/AbilityScoreChoiceResolver.php`
- `app/Services/ChoiceHandlers/Resolvers/EquipmentChoiceResolver.php`
- `app/Services/Parsers/Concerns/ParsesChoiceGroups.php`

### Modified Files
- `app/Models/Race.php` - add choiceGroups relationship
- `app/Models/CharacterClass.php` - add choiceGroups relationship
- `app/Models/Background.php` - add choiceGroups relationship
- `app/Models/Feat.php` - add choiceGroups relationship
- `app/Models/ClassFeature.php` - add choiceGroups relationship
- `app/Services/Parsers/RaceXmlParser.php`
- `app/Services/Parsers/ClassXmlParser.php`
- `app/Services/Parsers/BackgroundXmlParser.php`
- `app/Services/Parsers/FeatXmlParser.php`
- `app/Services/Parsers/SubclassXmlParser.php`
- `app/Services/ChoiceHandlers/SpellChoiceHandler.php`
- `app/Providers/AppServiceProvider.php`

### Deleted Files
- `app/Services/ChoiceHandlers/ProficiencyChoiceHandler.php`
- `app/Services/ChoiceHandlers/LanguageChoiceHandler.php`
- `app/Services/ChoiceHandlers/AbilityScoreChoiceHandler.php`
- `app/Services/ChoiceHandlers/EquipmentChoiceHandler.php`
- Corresponding test files
