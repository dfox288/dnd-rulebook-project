# Prepared Caster Spell Preparation Design

**Issue:** #727 - API: Support preparing any class spell for prepared casters
**Date:** 2025-12-16
**Status:** Approved

## Summary

Prepared casters (Cleric, Druid, Paladin, Artificer) should be able to prepare any spell from their class spell list without first "learning" it. This matches D&D 5e rules where divine casters have access to their entire class list.

## D&D 5e Casting Patterns

| Pattern | Classes | Repertoire | Preparation |
|---------|---------|------------|-------------|
| **Known** | Bard, Sorcerer, Ranger, Warlock | Fixed known spells | N/A (always ready) |
| **Divine Prepared** | Cleric, Druid, Paladin, Artificer | Entire class list | Daily subset |
| **Spellbook Prepared** | Wizard | Copied spells only | Daily subset from book |

## Solution

Add a new source value `prepared_from_list` to distinguish ephemeral preparations from permanent spell acquisitions.

### Source Values

| Source | Meaning | On Unprepare |
|--------|---------|--------------|
| `class` | Learned via class feature (wizard copying, bard selecting) | Keep as 'known' |
| `race` | Granted by racial trait | Keep as 'known' |
| `feat` | Granted by feat | Keep as 'known' |
| `item` | Granted by magic item | Keep as 'known' |
| `subclass` | Always-prepared domain/oath spells | Cannot unprepare |
| `prepared_from_list` | Prepared from full class list (divine casters) | Delete row |

## Prepare Endpoint Flow

```
1. Is spell level 0 (cantrip)?
   → YES: Reject ("Cantrips cannot be prepared")

2. Is spell already in character_spells?
   → YES: Toggle preparation status (existing flow)
   → NO: Continue to step 3

3. Is character a 'prepared' caster? (spell_preparation_method = 'prepared')
   → NO: Reject ("Spell not known")
   → YES: Continue to step 4

4. Is spell on the character's class spell list?
   → NO: Reject ("Spell not on class list")
   → YES: Continue to step 5

5. Is spell level accessible? (character level high enough)
   → NO: Reject ("Spell level too high")
   → YES: Continue to step 6

6. Is preparation limit reached?
   → YES: Reject ("Preparation limit reached")
   → NO: Create row with source='prepared_from_list', status='prepared'
```

## Unprepare Endpoint Flow

```
1. Find spell in character_spells
   → NOT FOUND: Reject ("Spell not known")

2. Is spell always_prepared? (domain/oath spells)
   → YES: Reject ("Cannot unprepare always-prepared spell")

3. Is spell a cantrip?
   → YES: Reject ("Cantrips cannot be unprepared")

4. Check source value:
   → source = 'prepared_from_list': DELETE the row entirely
   → source = anything else: Set preparation_status = 'known'
```

## API Contract

No API changes needed. Existing endpoints handle everything:

| Endpoint | Current Behavior | New Behavior |
|----------|------------------|--------------|
| `PATCH .../spells/{id}/prepare` | Requires spell in character_spells | Auto-adds for prepared casters if on class list |
| `PATCH .../spells/{id}/unprepare` | Sets status to 'known' | Deletes row if `source = prepared_from_list` |

### Frontend Workflow (Cleric Example)

1. `GET /characters/1/available-spells` → Shows full Cleric spell list
2. User clicks "Prepare" on Cure Wounds
3. `PATCH /characters/1/spells/phb:cure-wounds/prepare` → Backend auto-adds + prepares
4. `GET /characters/1/spells` → Now includes Cure Wounds with `is_prepared: true`
5. User clicks "Unprepare" on Cure Wounds
6. `PATCH /characters/1/spells/phb:cure-wounds/unprepare` → Backend deletes the row
7. `GET /characters/1/spells` → Cure Wounds is gone

## Implementation Changes

### Files to Modify

| File | Change |
|------|--------|
| `SpellManagerService.php` | Modify `prepareSpell()` to auto-add for prepared casters |
| `SpellManagerService.php` | Modify `unprepareSpell()` to delete `prepared_from_list` rows |
| `CharacterSpellResource.php` | Verify `source` is exposed |

### prepareSpell() Changes

```php
public function prepareSpell(Character $character, Spell $spell): CharacterSpell
{
    // Cantrip check (move to top)
    if ($spell->level === 0) {
        throw SpellManagementException::cannotPrepareCantrip($spell);
    }

    $characterSpell = $character->spells()->where('spell_slug', $spell->slug)->first();

    // NEW: Auto-add for prepared casters
    if (!$characterSpell) {
        if (!$this->isPreparedCaster($character)) {
            throw SpellManagementException::spellNotKnown($spell, $character);
        }
        if (!$this->isSpellOnClassList($character, $spell)) {
            throw SpellManagementException::spellNotOnClassList($spell, $character);
        }
        if ($spell->level > $this->getMaxSpellLevelForCharacter($character)) {
            throw SpellManagementException::spellLevelTooHigh(...);
        }
        // Check prep limit, then create
        $characterSpell = CharacterSpell::create([
            'source' => 'prepared_from_list',
            'preparation_status' => 'prepared',
            // ... other fields
        ]);
        return $characterSpell;
    }

    // Existing flow for already-known spells...
}
```

### unprepareSpell() Changes

```php
// After finding $characterSpell and checking always_prepared...

if ($characterSpell->source === 'prepared_from_list') {
    $characterSpell->delete();
    return null;
} else {
    $characterSpell->update(['preparation_status' => 'known']);
    return $characterSpell->fresh();
}
```

## Test Cases

### Prepare Endpoint Tests

| Test | Scenario |
|------|----------|
| `it prepares spell from class list for cleric` | Cleric prepares Cure Wounds (not in character_spells) → success |
| `it rejects cantrip preparation for prepared caster` | Cleric tries to prepare Sacred Flame → 422 |
| `it rejects spell not on class list` | Cleric tries to prepare Fireball → 422 |
| `it rejects spell level too high` | Level 1 Cleric tries to prepare Revivify → 422 |
| `it rejects when preparation limit reached` | Cleric at limit tries to prepare → 422 |
| `it does not auto-add for known casters` | Bard tries to prepare unknown spell → 422 |
| `it does not auto-add for wizard` | Wizard tries to prepare spell not in spellbook → 422 |

### Unprepare Endpoint Tests

| Test | Scenario |
|------|----------|
| `it deletes prepared_from_list spell on unprepare` | Unprepare Cleric's daily prep → row deleted |
| `it keeps class-source spell on unprepare` | Unprepare Wizard's spellbook spell → row kept |
| `it rejects unprepare cantrip` | Try to unprepare cantrip → 422 |

## Multiclass Edge Case

A Cleric 5 / Wizard 5 could have:
- Wizard spells with `source = 'class'` (spellbook - keep on unprepare)
- Cleric spells with `source = 'prepared_from_list'` (divine - delete on unprepare)
- Domain spells with `preparation_status = 'always_prepared'` (cannot unprepare)

The `source` field correctly distinguishes these cases.
