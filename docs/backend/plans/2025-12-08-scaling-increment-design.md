# Scaling Increment Parser Design

**Issue:** #198 - Parse spell scaling from 'At Higher Levels' text
**Date:** 2025-12-08
**Status:** Approved

## Overview

Parse the `scaling_increment` value from spell "At Higher Levels" text to enable automatic damage/effect scaling calculations.

## Current State

- `higher_levels` text is stored on Spell model (e.g., "the damage increases by 1d6 for each slot level above 3rd")
- `scaling_increment` field exists in `spell_effects` table but is always `null`
- `ParsesProjectileScaling` trait handles target/projectile scaling (separate concern)

## Data Analysis

157 spells have `higher_levels` text:

| Pattern | Count | Example |
|---------|-------|---------|
| Damage dice | 71 | "increases by **1d6** for each slot level" |
| Target scaling | 19 | "one additional creature" (handled by ParsesProjectileScaling) |
| Flat value | 8 | "increases by **5** for each slot level" |
| Duration | 7 | "duration is 8 hours" |
| Complex/other | 52 | Multi-conditional, unique patterns |

## Solution

### New Trait: `ParsesScalingIncrement`

**File:** `app/Services/Parsers/Concerns/ParsesScalingIncrement.php`

```php
trait ParsesScalingIncrement
{
    /**
     * Parse scaling increment from "At Higher Levels" text.
     *
     * Handles patterns:
     * - "increases by 1d6 for each slot level" → "1d6"
     * - "increases by 3d6 for each slot level" → "3d6"
     * - "increase by 5 for each slot level" → "5"
     *
     * @param string|null $higherLevels The "At Higher Levels" text
     * @return string|null Dice notation (e.g., "1d6") or flat value (e.g., "5")
     */
    protected function parseScalingIncrement(?string $higherLevels): ?string
    {
        if (empty($higherLevels)) {
            return null;
        }

        // Pattern 1: Dice notation - "increases by 1d6 for each"
        if (preg_match('/increases?\s+by\s+(\d+d\d+)\s+for\s+each/i', $higherLevels, $matches)) {
            return $matches[1];
        }

        // Pattern 2: Flat value - "increase by 5 for each"
        if (preg_match('/increases?\s+by\s+(\d+)\s+for\s+each/i', $higherLevels, $matches)) {
            return $matches[1];
        }

        return null;
    }
}
```

### Integration into SpellXmlParser

1. Add `use ParsesScalingIncrement` to SpellXmlParser
2. Pass `$higherLevels` to `parseRollElements()` method
3. Apply parsed increment to all damage effects

```php
// In parseSpell():
$effects = $this->parseRollElements($element, $higherLevels);

// In parseRollElements():
private function parseRollElements(SimpleXMLElement $element, ?string $higherLevels = null): array
{
    $effects = [];
    // ... existing effect parsing ...

    // Apply scaling increment to damage effects
    $scalingIncrement = $this->parseScalingIncrement($higherLevels);

    if ($scalingIncrement !== null) {
        foreach ($effects as &$effect) {
            if ($effect['effect_type'] === 'damage') {
                $effect['scaling_increment'] = $scalingIncrement;
            }
        }
    }

    return $effects;
}
```

## Coverage

| Spell Type | Handled |
|------------|---------|
| Fireball ("1d6") | Yes |
| Disintegrate ("3d6") | Yes |
| Armor of Agathys ("5") | Yes |
| Magic Missile (projectile) | No (handled by ParsesProjectileScaling) |
| Bestow Curse (duration tiers) | No (complex conditional) |

## Test Plan

### Unit Tests (`ParsesScalingIncrementTest.php`)

| Test | Input | Expected |
|------|-------|----------|
| Parses 1d6 | "increases by 1d6 for each slot level" | "1d6" |
| Parses 3d6 | "increases by 3d6 for each slot level" | "3d6" |
| Parses 1d8 | "the cold damage increases by 1d8 for each" | "1d8" |
| Parses flat 5 | "increase by 5 for each slot level" | "5" |
| Returns null for target scaling | "one additional creature for each" | null |
| Returns null for duration | "duration is 8 hours" | null |
| Returns null for empty | null | null |
| Returns null for empty string | "" | null |

### Integration Tests

- Verify Fireball effect has `scaling_increment = "1d6"` after import
- Verify spells without scaling have `scaling_increment = null`

## Acceptance Criteria

- [x] Design approved
- [ ] `ParsesScalingIncrement` trait created with regex patterns
- [ ] Unit tests for trait (8 tests)
- [ ] SpellXmlParser updated to use trait
- [ ] Integration test verifying imported data
- [ ] Re-import populates `scaling_increment` for ~79 spells

## Future Enhancements (Out of Scope)

- Parse "every two slot levels" pattern (Spiritual Weapon) - requires `scaling_per_levels` field
- Parse duration scaling patterns
- Parse target count from text (currently uses ParsesProjectileScaling)
