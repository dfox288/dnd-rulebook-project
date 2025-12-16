# Character Counters Redesign

**Issue:** #722 (supersedes #720)
**Date:** 2025-12-16
**Status:** Approved

## Problem

Counter names in `entity_counters` don't match feature names in `class_features` (96 mismatches). The current approach of populating `character_features.max_uses` by matching names fails.

More fundamentally, **counters and features are different concepts**:
- **Features** = abilities (Rage, Font of Magic, Infuse Item)
- **Counters** = resource pools (Rage uses, Sorcery Points, Infusions Known)

## Solution

Create a dedicated `character_counters` table to track resource usage separately from features.

## Schema

```sql
CREATE TABLE character_counters (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    character_id BIGINT UNSIGNED NOT NULL,
    source_type ENUM('class', 'feat', 'race') NOT NULL,
    source_slug VARCHAR(255) NOT NULL,
    counter_name VARCHAR(255) NOT NULL,
    current_uses INT NULL,              -- NULL = full
    max_uses INT NOT NULL,              -- -1 = unlimited
    reset_timing CHAR(1) NULL,          -- S/L/D
    created_at TIMESTAMP,
    updated_at TIMESTAMP,

    FOREIGN KEY (character_id) REFERENCES characters(id) ON DELETE CASCADE,
    UNIQUE KEY (character_id, source_type, source_slug, counter_name)
);
```

**Dropped columns:**
- `character_features.max_uses`
- `character_features.uses_remaining`
- `feature_selections.max_uses`
- `feature_selections.uses_remaining`

**Kept columns:**
- `character_spells.max_uses` / `uses_remaining` (for spell-specific limits like racial spells)

## Counter Lifecycle

### Creation

| Source | When Created |
|--------|--------------|
| Class | On level up, when `entity_counters` has entry for class at that level |
| Subclass | On subclass selection + level ups with new counters |
| Feat | On feat selection |
| Race | On race selection (future, #717) |

### Updates

- **Level up** → Recalculate `max_uses` from `entity_counters`
- **Use ability** → Decrement `current_uses`
- **Rest** → Reset `current_uses` to NULL based on `reset_timing`

### Deletion

- Character deleted → CASCADE
- Class removed → Delete counters for that source_slug

## Multiclass Handling

Separate counters per class, even with same name:
```
Fighter 6 / Rogue 4 with Psi Warrior + Soulknife:
- source_slug: phb:fighter-psi-warrior, counter_name: Psionic Energy, max: 6
- source_slug: phb:rogue-soulknife, counter_name: Psionic Energy, max: 4
```

This is RAW-accurate (separate pools).

## Service Layer

**New:** `CounterService`
- `syncCountersForCharacter(Character)` - Create/update counters based on current state
- `recalculateMaxUses(Character)` - Update max_uses on level up
- `useCounter(Character, int $counterId)` - Decrement usage
- `resetByTiming(Character, string ...$timings)` - Reset on rest
- `getCountersForCharacter(Character)` - API response format

**Modified:**
- `LevelUpService` → Call `CounterService::syncCountersForCharacter()`
- `FeatChoiceService` → Call `CounterService::syncCountersForCharacter()`
- `RestService` → Call `CounterService::resetByTiming()`
- `CharacterExportService` → Export from new table
- `CharacterImportService` → Import to new table

**Removed from `FeatureUseService`:**
- `initializeUsesForFeature()`
- `recalculateMaxUses()`
- `getCountersForCharacter()` (moved to CounterService)
- `getFeaturesWithUses()` (no longer needed)

## API Response

Unchanged format via `CharacterResource.counters`:
```json
{
  "counters": [
    {
      "id": 1,
      "slug": "phb:barbarian:rage",
      "name": "Rage",
      "current": 3,
      "max": 3,
      "reset_on": "long_rest",
      "source": "Barbarian",
      "source_type": "class",
      "unlimited": false
    }
  ]
}
```

## Migration Strategy

1. Create `character_counters` table
2. Drop unused columns from `character_features` and `feature_selections`
3. No data migration needed (columns were unpopulated)
