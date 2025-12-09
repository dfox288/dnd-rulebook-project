# Plan: Character Ability Bonuses Endpoint

**Issue:** #403
**Date:** 2025-12-09
**Status:** Draft

## Overview

Create a centralized endpoint that returns all ability score bonuses for a character from all sources (race, feats), with metadata about whether each bonus is from a choice that can be changed.

## Current State

- **`entity_modifiers`** stores ASI templates for Race, Feat, Item, CharacterClass
  - `modifier_category = 'ability_score'`
  - `is_choice`, `choice_count`, `choice_constraint` for choice modifiers
  - `ability_score_id` → FK to lookup table
- **`character_ability_scores`** stores resolved choices
  - `source` field (`race`, future: `feat`)
  - `modifier_id` links back to the choice template
- **`Character` model** has `getRacialAbilityBonuses()` and `getChosenAbilityBonuses()` methods
- **`CharacterFeature`** stores selected feats with polymorphic `feature` relationship

## API Design

### Endpoint

```
GET /api/v1/characters/{id}/ability-bonuses
```

### Response

```json
{
  "data": {
    "bonuses": [
      {
        "source_type": "race",
        "source_name": "Half-Elf",
        "source_slug": "phb:half-elf",
        "ability_code": "CHA",
        "ability_name": "Charisma",
        "value": 2,
        "is_choice": false
      },
      {
        "source_type": "race",
        "source_name": "Half-Elf",
        "source_slug": "phb:half-elf",
        "ability_code": "STR",
        "ability_name": "Strength",
        "value": 1,
        "is_choice": true,
        "choice_resolved": true,
        "modifier_id": 123
      },
      {
        "source_type": "feat",
        "source_name": "Actor",
        "source_slug": "phb:actor",
        "ability_code": "CHA",
        "ability_name": "Charisma",
        "value": 1,
        "is_choice": false
      }
    ],
    "totals": {
      "STR": 1,
      "DEX": 0,
      "CON": 0,
      "INT": 0,
      "WIS": 0,
      "CHA": 3
    }
  }
}
```

### Field Descriptions

| Field | Type | Description |
|-------|------|-------------|
| `source_type` | string | `race` or `feat` (extensible for `class_feature`, `item` later) |
| `source_name` | string | Display name of the source entity |
| `source_slug` | string | Full slug for linking/identification |
| `ability_code` | string | `STR`, `DEX`, `CON`, `INT`, `WIS`, `CHA` |
| `ability_name` | string | Full ability name |
| `value` | int | Bonus amount (typically 1 or 2) |
| `is_choice` | bool | Whether this came from a choice modifier |
| `choice_resolved` | bool | Only present when `is_choice=true` - whether slot has been picked |
| `modifier_id` | int | Only present when `is_choice=true` - links to choice for undo/redo |
| `totals` | object | Sum of all resolved bonuses per ability |

## Implementation Tasks

### Task 1: Create AbilityBonusService

**File:** `app/Services/AbilityBonusService.php`

```php
<?php

declare(strict_types=1);

namespace App\Services;

use App\Models\Character;
use App\Models\Modifier;
use Illuminate\Support\Collection;

class AbilityBonusService
{
    /**
     * Get all ability bonuses for a character.
     *
     * @return array{bonuses: Collection, totals: array<string, int>}
     */
    public function getBonuses(Character $character): array
    {
        $bonuses = collect();

        // Collect from all sources
        $bonuses = $bonuses->merge($this->getRaceBonuses($character));
        $bonuses = $bonuses->merge($this->getFeatBonuses($character));

        // Calculate totals from resolved bonuses only
        $totals = $this->calculateTotals($bonuses);

        return [
            'bonuses' => $bonuses,
            'totals' => $totals,
        ];
    }

    private function getRaceBonuses(Character $character): Collection
    {
        // Implementation: Query race modifiers + character_ability_scores
    }

    private function getFeatBonuses(Character $character): Collection
    {
        // Implementation: Query feat modifiers via CharacterFeature
    }

    private function calculateTotals(Collection $bonuses): array
    {
        // Sum only resolved bonuses (is_choice=false OR choice_resolved=true)
    }
}
```

### Task 2: Create AbilityBonusResource

**File:** `app/Http/Resources/AbilityBonusResource.php`

Single bonus item resource.

### Task 3: Create AbilityBonusCollectionResource

**File:** `app/Http/Resources/AbilityBonusCollectionResource.php`

Wraps the collection with totals.

### Task 4: Add Controller Method

**File:** `app/Http/Controllers/Api/CharacterController.php`

```php
/**
 * Get all ability score bonuses for a character.
 *
 * Returns bonuses from race (fixed and choice) and feats,
 * with metadata about source and whether the bonus can be changed.
 *
 * @operationId getCharacterAbilityBonuses
 * @tags Characters
 */
public function abilityBonuses(Character $character): AbilityBonusCollectionResource
{
    $result = $this->abilityBonusService->getBonuses($character);

    return new AbilityBonusCollectionResource($result);
}
```

### Task 5: Add Route

**File:** `routes/api.php`

```php
Route::get('characters/{character}/ability-bonuses', [CharacterController::class, 'abilityBonuses']);
```

### Task 6: Write Tests

**File:** `tests/Feature/Api/CharacterAbilityBonusesApiTest.php`

Test cases:
1. Returns empty bonuses for character with no race
2. Returns fixed racial bonuses
3. Returns resolved racial choice bonuses with metadata
4. Returns unresolved racial choice bonuses (is_choice=true, choice_resolved=false)
5. Returns feat bonuses (fixed ASI from feats like Actor)
6. Calculates totals correctly (only resolved bonuses)
7. Returns 404 for non-existent character
8. Handles subrace inheritance (parent + subrace modifiers)

### Task 7: Unit Tests for Service

**File:** `tests/Unit/Services/AbilityBonusServiceTest.php`

Test the service in isolation with mocked data.

## Detailed Implementation Notes

### Race Bonuses Logic

```php
private function getRaceBonuses(Character $character): Collection
{
    if (!$character->race) {
        return collect();
    }

    $bonuses = collect();
    $race = $character->race;

    // Load modifiers from race and parent race (if subrace)
    $raceModifiers = $this->getAbilityModifiers($race);
    if ($race->parent_race_id && $race->parent) {
        $raceModifiers = $raceModifiers->merge($this->getAbilityModifiers($race->parent));
    }

    // Get resolved choices
    $resolvedChoices = $character->abilityScores()
        ->where('source', 'race')
        ->get()
        ->keyBy('modifier_id');

    foreach ($raceModifiers as $modifier) {
        if ($modifier->is_choice) {
            // Handle choice modifiers
            $resolved = $resolvedChoices->where('modifier_id', $modifier->id);
            foreach ($resolved as $choice) {
                $bonuses->push($this->buildBonus(
                    sourceType: 'race',
                    sourceName: $race->name,
                    sourceSlug: $race->full_slug,
                    abilityCode: $choice->ability_score_code,
                    value: $choice->bonus,
                    isChoice: true,
                    choiceResolved: true,
                    modifierId: $modifier->id,
                ));
            }
            // Note: Unresolved choices are NOT included here -
            // they appear in the /choices endpoint
        } else {
            // Fixed modifier
            $bonuses->push($this->buildBonus(
                sourceType: 'race',
                sourceName: $race->name,
                sourceSlug: $race->full_slug,
                abilityCode: $modifier->abilityScore->code,
                value: (int) $modifier->value,
                isChoice: false,
            ));
        }
    }

    return $bonuses;
}
```

### Feat Bonuses Logic

```php
private function getFeatBonuses(Character $character): Collection
{
    $bonuses = collect();

    // Get feats via CharacterFeature
    $featFeatures = $character->features()
        ->where('feature_type', Feat::class)
        ->with('feature.modifiers.abilityScore')
        ->get();

    foreach ($featFeatures as $characterFeature) {
        $feat = $characterFeature->feature;
        if (!$feat) continue;

        $abilityModifiers = $feat->modifiers
            ->where('modifier_category', 'ability_score')
            ->where('is_choice', false); // Only fixed for now

        foreach ($abilityModifiers as $modifier) {
            if (!$modifier->abilityScore) continue;

            $bonuses->push($this->buildBonus(
                sourceType: 'feat',
                sourceName: $feat->name,
                sourceSlug: $feat->full_slug,
                abilityCode: $modifier->abilityScore->code,
                value: (int) $modifier->value,
                isChoice: false,
            ));
        }
    }

    return $bonuses;
}
```

## Open Questions (Resolved)

1. ~~Include unresolved choices?~~ **No** - unresolved choices appear in `/choices` endpoint, not here. This endpoint shows what the character *has*, not what they *could have*.

2. ~~Choice replacement during character creation?~~ **Handled by choice system** - when race changes, old `character_ability_scores` become orphaned (different `modifier_id`s). The service only returns bonuses that have valid references.

3. ~~Future sources?~~ **Add when needed** - class_feature, item, condition can be added later without API changes.

## Acceptance Criteria

- [ ] Endpoint returns bonuses from race (fixed + resolved choices)
- [ ] Endpoint returns bonuses from feats
- [ ] `is_choice` and `choice_resolved` metadata included for choice bonuses
- [ ] `modifier_id` included for linkage to choice system
- [ ] Totals calculated from resolved bonuses only
- [ ] Subrace inheritance works (parent + subrace modifiers)
- [ ] 404 for non-existent character
- [ ] OpenAPI docs generated correctly
- [ ] All tests pass

## File Changes Summary

| File | Action |
|------|--------|
| `app/Services/AbilityBonusService.php` | Create |
| `app/Http/Resources/AbilityBonusResource.php` | Create |
| `app/Http/Resources/AbilityBonusCollectionResource.php` | Create |
| `app/Http/Controllers/Api/CharacterController.php` | Modify (add method) |
| `routes/api.php` | Modify (add route) |
| `tests/Feature/Api/CharacterAbilityBonusesApiTest.php` | Create |
| `tests/Unit/Services/AbilityBonusServiceTest.php` | Create |
