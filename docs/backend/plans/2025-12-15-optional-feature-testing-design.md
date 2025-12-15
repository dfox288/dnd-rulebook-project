# Optional Feature & Spellcaster Progression Testing Design

**Date:** 2025-12-15
**Status:** Approved
**Related Issues:** TBD (will create after audit)

## Overview

Create a comprehensive test suite that validates character progression for optional features (invocations, metamagic, maneuvers, etc.) and spellcasting, ensuring the level-up flow generates correct choices according to D&D 5e rules.

## Problem Statement

1. Existing spellcaster test fixtures (18 characters) are incomplete "shells" - they have class levels but no choices were made (no skills, languages, subclass, or features selected)
2. No automated validation that optional feature counts match official D&D 5e rules at each level
3. Database counter names are inconsistent (e.g., "Eldritch Invocations Known" vs "Eldritch Invocations")

## Goals

1. **Level-up flow validation** - Verify optional features are offered and resolved correctly during progression
2. **Reference data audit** - Compare database counters against official D&D rules to catch data bugs
3. **Complete fixtures** - Create properly populated character fixtures for import/export testing

## Design

### 1. Config File: Official D&D Rules Reference

**File:** `config/dnd-rules.php`

```php
return [
    'optional_features' => [
        'eldritch_invocation' => [
            'class' => 'phb:warlock',
            'subclass' => null,
            'progression' => [
                2 => 2, 5 => 3, 7 => 4, 9 => 5, 12 => 6, 15 => 7, 18 => 8
            ],
        ],
        'metamagic' => [
            'class' => 'phb:sorcerer',
            'subclass' => null,
            'progression' => [
                3 => 2, 10 => 3, 17 => 4
            ],
        ],
        'artificer_infusion' => [
            'class' => 'erlw:artificer',
            'subclass' => null,
            'progression' => [
                2 => 4, 6 => 6, 10 => 8, 14 => 10, 18 => 12
            ],
        ],
        'maneuver' => [
            'class' => 'phb:fighter',
            'subclass' => 'phb:fighter-battle-master',
            'progression' => [
                3 => 3, 7 => 5, 10 => 7, 15 => 9
            ],
        ],
        'arcane_shot' => [
            'class' => 'phb:fighter',
            'subclass' => 'xge:fighter-arcane-archer',
            'progression' => [
                3 => 2, 7 => 3, 10 => 4, 15 => 5, 18 => 6
            ],
        ],
        'rune' => [
            'class' => 'phb:fighter',
            'subclass' => 'tce:fighter-rune-knight',
            'progression' => [
                3 => 2, 7 => 3, 10 => 4, 15 => 5
            ],
        ],
        'elemental_discipline' => [
            'class' => 'phb:monk',
            'subclass' => 'phb:monk-way-of-the-four-elements',
            'progression' => [
                3 => 1, 6 => 2, 11 => 3, 17 => 4
            ],
        ],
        'fighting_style' => [
            // Multiple classes - handled separately
        ],
    ],
];
```

### 2. Data Audit Command

**File:** `app/Console/Commands/AuditOptionalFeatureCountersCommand.php`

Purpose: Compare database counters against config reference, report mismatches.

```bash
php artisan audit:optional-feature-counters
```

- Reports mismatches between database and official rules
- Returns failure exit code if issues found
- Does NOT auto-fix - issues tracked in GitHub and fixed in importer/parser

### 3. Test Suite Structure

**File:** `tests/Feature/OptionalFeatures/OptionalFeatureProgressionTest.php`

```php
describe('Optional Feature Progression', function () {
    describe('Warlock Eldritch Invocations', function () {
        it('gains 2 invocations at level 2', function () { });
        it('gains 3 invocations at level 5', function () { });
        // ... through level 18
    });

    describe('Sorcerer Metamagic', function () { });
    describe('Fighter Battle Master Maneuvers', function () { });
    describe('Fighter Arcane Archer Arcane Shots', function () { });
    describe('Fighter Rune Knight Runes', function () { });
    describe('Monk Four Elements Disciplines', function () { });
    describe('Artificer Infusions', function () { });
});
```

**Suite assignment:** `Feature-DB`

### 4. Helper Trait

**File:** `tests/Traits/CreatesTestCharacters.php`

```php
trait CreatesTestCharacters
{
    protected function createAndLevelCharacter(
        string $classSlug,
        ?string $subclassSlug = null,
        int $targetLevel = 1
    ): Character {
        // 1. Create via wizard flow (FlowExecutor)
        // 2. Force subclass if specified
        // 3. Level up to target (LevelUpFlowExecutor)
        // 4. Return fresh character
    }
}
```

### 5. Fixture Structure

```
storage/fixtures/
├── spellcaster-characters/
│   ├── wizard-l5.json
│   ├── wizard-l20.json
│   ├── cleric-l5.json
│   ├── cleric-l20.json
│   └── ... (9 classes × 2-3 levels)
├── optional-feature-characters/
│   ├── warlock-l5.json
│   ├── warlock-l12.json
│   ├── warlock-l20.json
│   ├── sorcerer-l10.json
│   ├── sorcerer-l20.json
│   ├── fighter-battle-master-l10.json
│   ├── fighter-battle-master-l20.json
│   ├── fighter-arcane-archer-l20.json
│   ├── fighter-rune-knight-l20.json
│   ├── monk-four-elements-l20.json
│   ├── artificer-l10.json
│   └── artificer-l20.json
└── combined-test-characters.json
```

Characters with both spellcasting AND optional features (Sorcerer, Warlock, Artificer) are combined into single fixtures.

## Implementation Phases

### Phase 1: Foundation
1. Create `config/dnd-rules.php` with official progression tables
2. Create `AuditOptionalFeatureCountersCommand`
3. Run audit, create GitHub issues for any counter mismatches

### Phase 2: Test Infrastructure
4. Create `tests/Traits/CreatesTestCharacters.php` helper trait
5. Create `tests/Feature/OptionalFeatures/OptionalFeatureProgressionTest.php`
6. Add tests for each class/subclass combo at milestone levels

### Phase 3: Spellcaster Tests (Recreate)
7. Delete existing incomplete fixtures and database records
8. Create `tests/Feature/Spellcasting/SpellcasterProgressionTest.php`
9. Tests verify spell slots, cantrips known, spells known

### Phase 4: Fixture Export
10. Add export capability to test suite
11. Export passing characters to `storage/fixtures/`
12. Update `ImportFixtureCharactersCommand` for new structure

### Cleanup
- Fix data issues discovered by audit (separate PRs)
- Remove old incomplete fixtures

## Known Issues to Address

1. **Warlock counter name mismatch**: "Eldritch Invocations Known" (L2) vs "Eldritch Invocations" (L5+)
   - Handler only recognizes the "Known" variant
   - Needs importer fix or handler update

## Test Classes/Subclasses Matrix

| Class | Subclass | Feature Type | Key Levels |
|-------|----------|--------------|------------|
| Warlock | - | Eldritch Invocations | 2, 5, 7, 9, 12, 15, 18 |
| Sorcerer | - | Metamagic | 3, 10, 17 |
| Artificer | - | Infusions | 2, 6, 10, 14, 18 |
| Fighter | Battle Master | Maneuvers | 3, 7, 10, 15 |
| Fighter | Arcane Archer | Arcane Shots | 3, 7, 10, 15, 18 |
| Fighter | Rune Knight | Runes | 3, 7, 10, 15 |
| Monk | Way of Four Elements | Disciplines | 3, 6, 11, 17 |
| Fighter/Paladin/Ranger | - | Fighting Styles | varies |
