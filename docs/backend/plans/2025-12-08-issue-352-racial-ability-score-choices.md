# Issue #352: Racial Ability Score Choices

**Issue:** [#352 - Backend: Add ability_score pending choice type for racial modifier choices](https://github.com/dfox288/dnd-rulebook-project/issues/352)
**Branch:** `feature/issue-352-racial-ability-score-choices`
**Blocking:** Frontend issue #219

## Overview

Half-Elf and similar races have ability score modifiers with `is_choice: true`. These need to:
1. Appear in `GET /characters/{id}/pending-choices`
2. Be resolvable via `POST /characters/{id}/choices/{choiceId}`
3. Apply to final ability score calculations

## Architecture

Follow the `LanguageChoiceHandler` + `CharacterLanguage` pattern exactly:

| Component | Reference | New File |
|-----------|-----------|----------|
| Handler | `LanguageChoiceHandler` | `AbilityScoreChoiceHandler` |
| Model | `CharacterLanguage` | `CharacterAbilityScore` |
| Table | `character_languages` | `character_ability_scores` |
| Service | `CharacterLanguageService` | (inline in handler) |

## Tasks

### Task 1: Migration + Model

**Create migration:** `create_character_ability_scores_table`

```php
Schema::create('character_ability_scores', function (Blueprint $table) {
    $table->id();
    $table->foreignId('character_id')->constrained()->cascadeOnDelete();
    $table->string('ability_score_code', 3);  // STR, DEX, CON, INT, WIS, CHA
    $table->tinyInteger('bonus')->unsigned(); // +1, +2
    $table->string('source', 20)->default('race');
    $table->unsignedBigInteger('modifier_id')->nullable(); // Links to entity_modifiers
    $table->timestamp('created_at')->nullable();

    $table->unique(['character_id', 'ability_score_code', 'source']);
    $table->index('ability_score_code');
});
```

**Create model:** `app/Models/CharacterAbilityScore.php`

```php
class CharacterAbilityScore extends Model
{
    use HasFactory;

    public $timestamps = false;

    protected $fillable = [
        'character_id',
        'ability_score_code',
        'bonus',
        'source',
        'modifier_id',
    ];

    protected $casts = [
        'bonus' => 'integer',
        'created_at' => 'datetime',
    ];

    public function character(): BelongsTo
    {
        return $this->belongsTo(Character::class);
    }

    public function abilityScore(): BelongsTo
    {
        return $this->belongsTo(AbilityScore::class, 'ability_score_code', 'code');
    }
}
```

**Files:**
- `database/migrations/2025_12_08_000001_create_character_ability_scores_table.php`
- `app/Models/CharacterAbilityScore.php`
- `database/factories/CharacterAbilityScoreFactory.php`

---

### Task 2: Character Model Relationship + Stats Update

**Add relationship to Character.php:**

```php
public function abilityScores(): HasMany
{
    return $this->hasMany(CharacterAbilityScore::class);
}
```

**Update `getFinalAbilityScoresArray()` to include chosen bonuses:**

```php
public function getFinalAbilityScoresArray(): array
{
    $baseScores = $this->getAbilityScoresArray();

    if (! $this->race_slug) {
        return $baseScores;
    }

    if (! $this->relationLoaded('race')) {
        $this->load('race.modifiers.abilityScore', 'race.parent.modifiers.abilityScore');
    }

    // Fixed racial bonuses (is_choice = false)
    $racialBonuses = $this->getRacialAbilityBonuses();

    // Chosen racial bonuses (from character_ability_scores)
    $chosenBonuses = $this->getChosenAbilityBonuses();

    // Apply all bonuses
    foreach ($baseScores as $code => $baseScore) {
        if ($baseScore !== null) {
            $total = ($racialBonuses[$code] ?? 0) + ($chosenBonuses[$code] ?? 0);
            if ($total > 0) {
                $baseScores[$code] = $baseScore + $total;
            }
        }
    }

    return $baseScores;
}

private function getChosenAbilityBonuses(): array
{
    if (! $this->relationLoaded('abilityScores')) {
        $this->load('abilityScores');
    }

    $bonuses = [];
    foreach ($this->abilityScores as $choice) {
        $bonuses[$choice->ability_score_code] =
            ($bonuses[$choice->ability_score_code] ?? 0) + $choice->bonus;
    }

    return $bonuses;
}
```

**Files:**
- `app/Models/Character.php` (add relationship + update stats methods)

---

### Task 3: AbilityScoreChoiceHandler

**Create handler:** `app/Services/ChoiceHandlers/AbilityScoreChoiceHandler.php`

```php
class AbilityScoreChoiceHandler extends AbstractChoiceHandler
{
    public function getType(): string
    {
        return 'ability_score';
    }

    public function getChoices(Character $character): Collection
    {
        $choices = collect();

        if (! $character->race_slug) {
            return $choices;
        }

        // Load race with modifiers
        if (! $character->relationLoaded('race')) {
            $character->load('race.modifiers.abilityScore', 'race.parent.modifiers.abilityScore');
        }

        // Get choice modifiers from race (and parent race if subrace)
        $modifiers = $this->getChoiceModifiers($character);

        foreach ($modifiers as $modifier) {
            $choice = $this->buildPendingChoice($character, $modifier);
            if ($choice) {
                $choices->push($choice);
            }
        }

        return $choices;
    }

    private function getChoiceModifiers(Character $character): Collection
    {
        $modifiers = collect();

        // Get from race
        $raceModifiers = $character->race->modifiers()
            ->where('modifier_category', 'ability_score')
            ->where('is_choice', true)
            ->get();
        $modifiers = $modifiers->merge($raceModifiers);

        // Get from parent race if subrace
        if ($character->race->parent_race_id && $character->race->parent) {
            $parentModifiers = $character->race->parent->modifiers()
                ->where('modifier_category', 'ability_score')
                ->where('is_choice', true)
                ->get();
            $modifiers = $modifiers->merge($parentModifiers);
        }

        return $modifiers;
    }

    private function buildPendingChoice(Character $character, Modifier $modifier): ?PendingChoice
    {
        // Get all ability score options
        $allAbilities = AbilityScore::all();

        // Get already-selected ability codes for this modifier
        $selected = $character->abilityScores()
            ->where('modifier_id', $modifier->id)
            ->pluck('ability_score_code')
            ->toArray();

        $quantity = $modifier->choice_count ?? 1;
        $remaining = $quantity - count($selected);

        // Get fixed racial bonuses to potentially exclude
        $fixedAbilityCodes = $this->getFixedAbilityCodes($character);

        $options = $allAbilities
            ->reject(fn ($ability) => in_array($ability->code, $fixedAbilityCodes))
            ->map(fn ($ability) => [
                'code' => $ability->code,
                'name' => $ability->name,
            ])
            ->values()
            ->all();

        return new PendingChoice(
            id: $this->generateChoiceId(
                'ability_score',
                'race',
                $character->race->full_slug,
                1,
                'modifier_' . $modifier->id
            ),
            type: 'ability_score',
            subtype: null,
            source: 'race',
            sourceName: $character->race->name,
            levelGranted: 1,
            required: true,
            quantity: $quantity,
            remaining: $remaining,
            selected: $selected,
            options: $options,
            optionsEndpoint: null,
            metadata: [
                'modifier_id' => $modifier->id,
                'bonus_value' => (int) $modifier->value,
                'choice_constraint' => $modifier->choice_constraint,
            ],
        );
    }

    private function getFixedAbilityCodes(Character $character): array
    {
        // Get ability codes that already have fixed racial bonuses
        // (optional: D&D RAW is unclear on whether you can stack)
        return [];
    }

    public function resolve(Character $character, PendingChoice $choice, array $selection): void
    {
        $parsed = $this->parseChoiceId($choice->id);
        $modifierId = (int) str_replace('modifier_', '', $parsed['group']);

        $selected = $selection['selected'] ?? [];
        if (empty($selected)) {
            throw new InvalidSelectionException($choice->id, 'empty', 'Selection cannot be empty');
        }

        // Validate quantity
        if (count($selected) !== $choice->quantity) {
            throw new InvalidSelectionException(
                $choice->id,
                implode(',', $selected),
                "Must select exactly {$choice->quantity} ability scores"
            );
        }

        // Validate constraint (e.g., 'different' means no duplicates)
        $constraint = $choice->metadata['choice_constraint'] ?? null;
        if ($constraint === 'different' && count($selected) !== count(array_unique($selected))) {
            throw new InvalidSelectionException(
                $choice->id,
                implode(',', $selected),
                'Selected ability scores must be different'
            );
        }

        // Validate ability codes exist
        $validCodes = AbilityScore::pluck('code')->toArray();
        foreach ($selected as $code) {
            if (! in_array($code, $validCodes)) {
                throw new InvalidSelectionException($choice->id, $code, "Invalid ability score code: {$code}");
            }
        }

        $bonusValue = $choice->metadata['bonus_value'] ?? 1;

        // Delete existing choices for this modifier
        $character->abilityScores()->where('modifier_id', $modifierId)->delete();

        // Create new choices
        foreach ($selected as $code) {
            $character->abilityScores()->create([
                'ability_score_code' => $code,
                'bonus' => $bonusValue,
                'source' => 'race',
                'modifier_id' => $modifierId,
            ]);
        }

        $character->load('abilityScores');
    }

    public function canUndo(Character $character, PendingChoice $choice): bool
    {
        return true;
    }

    public function undo(Character $character, PendingChoice $choice): void
    {
        $parsed = $this->parseChoiceId($choice->id);
        $modifierId = (int) str_replace('modifier_', '', $parsed['group']);

        $character->abilityScores()->where('modifier_id', $modifierId)->delete();
        $character->load('abilityScores');
    }
}
```

**Files:**
- `app/Services/ChoiceHandlers/AbilityScoreChoiceHandler.php`

---

### Task 4: Register Handler

**Update AppServiceProvider.php:**

```php
// In boot() method, after other handler registrations
$service->registerHandler($app->make(\App\Services\ChoiceHandlers\AbilityScoreChoiceHandler::class));
```

**Files:**
- `app/Providers/AppServiceProvider.php`

---

### Task 5: Unit Tests for Handler

**Create test:** `tests/Unit/Services/ChoiceHandlers/AbilityScoreChoiceHandlerTest.php`

Test cases:
- `getChoices()` returns empty when character has no race
- `getChoices()` returns choices for race with `is_choice` ability modifiers
- `getChoices()` includes parent race modifiers for subraces
- `getChoices()` shows correct `remaining` count based on selections
- `resolve()` stores ability score selections correctly
- `resolve()` validates exact quantity required
- `resolve()` validates 'different' constraint
- `resolve()` rejects invalid ability codes
- `undo()` removes selections for the modifier

**Files:**
- `tests/Unit/Services/ChoiceHandlers/AbilityScoreChoiceHandlerTest.php`

---

### Task 6: Feature Tests for API

**Create test:** `tests/Feature/Api/AbilityScoreChoiceApiTest.php`

Test cases:
- `GET /pending-choices` includes ability_score choices for Half-Elf
- `POST /choices/{id}` resolves ability score selection
- `POST /choices/{id}` returns validation errors for wrong quantity
- `DELETE /choices/{id}` undoes ability score selection
- Final ability scores reflect chosen bonuses

**Files:**
- `tests/Feature/Api/AbilityScoreChoiceApiTest.php`

---

### Task 7: Update CharacterStatsDTO (if needed)

If `CharacterStatsDTO` caches ability scores, ensure it calls the updated `getFinalAbilityScoresArray()`.

**Files:**
- `app/DTOs/CharacterStatsDTO.php` (verify/update)

---

### Task 8: Quality Gates

```bash
# Run Pint
docker compose exec php ./vendor/bin/pint

# Run relevant test suites
docker compose exec php ./vendor/bin/pest --testsuite=Unit-DB
docker compose exec php ./vendor/bin/pest --testsuite=Feature-DB

# Verify specific tests
docker compose exec php ./vendor/bin/pest tests/Unit/Services/ChoiceHandlers/AbilityScoreChoiceHandlerTest.php
docker compose exec php ./vendor/bin/pest tests/Feature/Api/AbilityScoreChoiceApiTest.php
```

---

## Summary

| # | Task | Files |
|---|------|-------|
| 1 | Migration + Model + Factory | 3 new files |
| 2 | Character relationship + stats | 1 modified |
| 3 | AbilityScoreChoiceHandler | 1 new file |
| 4 | Register handler | 1 modified |
| 5 | Unit tests | 1 new file |
| 6 | Feature tests | 1 new file |
| 7 | CharacterStatsDTO | 1 verify/modify |
| 8 | Quality gates | - |

**Total:** 6 new files, 3 modified files

## Verification

After implementation, verify with Half-Elf character:
1. `GET /characters/{id}/pending-choices` shows `ability_score` choice
2. `POST /choices/{choiceId}` with `{"selected": ["STR", "DEX"]}` succeeds
3. `GET /characters/{id}/stats` shows +1 to chosen abilities
