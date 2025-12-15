# Character Creation Wizard - API Flow

**Issue:** #686

This document details the step-by-step API flow for the character creation wizard. Frontend should follow this sequence to create a character through the wizard.

## Overview

The wizard follows this order:
1. Create character shell
2. Select race (and subrace if applicable)
3. Select class (and subclass if applicable at L1)
4. Select background
5. Assign ability scores
6. Resolve pending choices (proficiencies, languages, equipment, spells, etc.)
7. Set name and alignment
8. Validate completion

## Key Concepts

### Pending Choices System

Throughout the wizard, various selections (race, class, background) generate **pending choices** that the user must resolve. These are fetched from:

```
GET /api/v1/characters/{id}/pending-choices
```

Response structure:
```json
{
  "data": {
    "choices": [
      {
        "id": "proficiency:race:phb:elf:2",
        "type": "proficiency",
        "subtype": null,
        "source": "race",
        "source_name": "Elf",
        "level_granted": 1,
        "required": true,
        "quantity": 2,
        "remaining": 2,
        "selected": [],
        "options": [
          { "slug": "perception", "name": "Perception" },
          { "slug": "insight", "name": "Insight" }
        ],
        "options_endpoint": null,
        "metadata": { "choice_group": "race_proficiency" }
      }
    ],
    "summary": {
      "total_pending": 5,
      "required_pending": 3,
      "optional_pending": 2,
      "by_type": { "proficiency": 2, "language": 1, "spell": 2 },
      "by_source": { "race": 2, "class": 2, "background": 1 }
    }
  }
}
```

### Choice Types

| Type | Description | `selected` format |
|------|-------------|-------------------|
| `proficiency` | Skill proficiencies | Array of skill slugs |
| `language` | Language selections | Array of language slugs |
| `spell` | Spell selections (cantrips, known spells) | Array of spell slugs |
| `equipment` | Starting equipment | Array of option letters (a, b, c) |
| `equipment_mode` | Equipment vs. Gold mode | `["equipment"]` or `["gold"]` |
| `size` | Size choice (some races) | Array with size code `["M"]` |
| `ability_score` | Racial ASI choices | Array of ability codes `["STR", "DEX"]` |
| `optional_feature` | Fighting Style, Invocations, etc. | Array of feature slugs |
| `feat` | Feat selection | Array with feat slug |
| `subclass` | Subclass selection (if required) | Array with subclass slug |

---

## Step 1: Create Character Shell

**Endpoint:** `POST /api/v1/characters`

**Minimal Request (shell only):**
```json
{
  "public_id": "brave-warrior-X4kP",
  "name": "Unnamed Hero"
}
```

**Alternative: Include race/class in initial POST:**
```json
{
  "public_id": "brave-warrior-X4kP",
  "name": "Legolas",
  "race_slug": "phb:elf",
  "class_slug": "phb:ranger"
}
```

**Notes:**
- `public_id` should be generated client-side (adjective-noun-4chars pattern)
- `name` can be a placeholder - it's updated in the final step
- You can include `race_slug`, `class_slug`, and/or `background_slug` in the initial POST instead of using separate PATCH requests

**Response:**
```json
{
  "data": {
    "id": 123,
    "public_id": "brave-warrior-X4kP",
    "name": "Unnamed Hero",
    "is_complete": false,
    ...
  }
}
```

**After this step:** Character exists. If race/class were included, those are already set.

---

## Step 2: Select Race

**Endpoint:** `PATCH /api/v1/characters/{id}`

**Request:**
```json
{
  "race_slug": "phb:elf"
}
```

**Notes:**
- Use the parent race slug first (e.g., `phb:elf`)
- This grants racial traits, speed, darkvision, etc.
- **Switching races later clears**: subraces, racial proficiencies, racial languages, racial spells, racial ability score choices

**After this step:** Check `GET /api/v1/characters/{id}/pending-choices` for:
- Racial proficiency choices
- Racial language choices
- Racial ability score choices (if applicable)
- Size choices (if race has variable size)

---

## Step 2b: Select Subrace (if applicable)

**Endpoint:** `PATCH /api/v1/characters/{id}`

**Request:**
```json
{
  "race_slug": "phb:high-elf"
}
```

**Notes:**
- Only call if the parent race has subraces
- Check `GET /api/v1/races/{raceSlug}` - if `subraces` array is non-empty, show subrace selection
- Subrace slug replaces the race slug (not an additional field)

**After this step:** Additional subrace choices may appear in pending choices.

---

## Step 3: Select Class

**Endpoint:** `POST /api/v1/characters/{id}/classes`

**Request:**
```json
{
  "class_slug": "phb:wizard",
  "force": true
}
```

**Notes:**
- `force: true` allows setting class even if one exists (for switching)
- This grants class proficiencies, features, and spellcasting setup
- **Switching classes later clears**: equipment mode, equipment choices, class proficiencies, class features, spells

**If replacing an existing class:**

**Endpoint:** `PUT /api/v1/characters/{id}/classes/{currentClassSlug}`

**Request:**
```json
{
  "class_slug": "phb:sorcerer"
}
```

**After this step:** Check pending choices for:
- Class skill proficiency choices
- Equipment mode choice
- Spell choices (if spellcaster)

---

## Step 3b: Select Subclass (if applicable at L1)

**Endpoint:** `PUT /api/v1/characters/{id}/classes/{classSlug}/subclass`

**Request:**
```json
{
  "subclass_slug": "phb:school-of-evocation"
}
```

**Notes:**
- Only some classes get subclass at level 1 (Cleric, Sorcerer, Warlock)
- Check `GET /api/v1/classes/{classSlug}` - if `subclass_level === 1`, show subclass selection
- Subclasses are listed in the class response under `subclasses` array

**After this step:** Subclass features and spells may generate additional choices.

---

## Step 4: Select Background

**Endpoint:** `PATCH /api/v1/characters/{id}`

**Request:**
```json
{
  "background_slug": "phb:acolyte"
}
```

**Notes:**
- Grants background proficiencies, languages, equipment, and feature
- **Switching backgrounds later clears**: background proficiencies, background languages, background equipment

**After this step:** Check pending choices for:
- Background skill proficiency choices
- Background language choices
- Background equipment (appears as equipment choices if equipment mode is set)

---

## Step 5: Assign Ability Scores

**Endpoint:** `PATCH /api/v1/characters/{id}`

**Request:**
```json
{
  "strength": 15,
  "dexterity": 14,
  "constitution": 13,
  "intelligence": 12,
  "wisdom": 10,
  "charisma": 8
}
```

**Notes:**
- These are the BASE scores before racial bonuses
- Racial bonuses are applied automatically based on race
- Standard Array: [15, 14, 13, 12, 10, 8]
- Point Buy: Calculate based on rules
- Rolled: Whatever the player rolled

**After this step:** Character has ability scores. If race has ASI choices (e.g., Half-Elf +1 to two abilities), those appear in pending choices.

---

## Step 6: Resolve Pending Choices

This is the core of the wizard. Iterate through pending choices and resolve them.

### 6.1 Check Pending Choices

**Endpoint:** `GET /api/v1/characters/{id}/pending-choices`

### 6.2 Resolve a Choice

**Endpoint:** `POST /api/v1/characters/{id}/choices/{choiceId}`

#### Proficiency/Language/Spell Choices

**Request:**
```json
{
  "selected": ["perception", "stealth"]
}
```

#### Equipment Mode Choice

**Request:**
```json
{
  "selected": ["equipment"]
}
```
or
```json
{
  "selected": ["gold"]
}
```

**Notes:**
- `equipment` mode shows equipment choices to resolve
- `gold` mode gives starting gold based on class (no equipment choices)

#### Equipment Choices

Equipment choices have options like `a`, `b`, `c`:

**Simple equipment (non-category):**
```json
{
  "selected": ["a"]
}
```

**Category equipment (user picks item from category):**
```json
{
  "selected": ["a"],
  "item_selections": {
    "a": ["phb:longsword"]
  }
}
```

**Notes:**
- Check `is_category` on the option
- If `is_category === true`, filter `items` to those where `is_fixed === false` and let user pick
- `item_selections` maps option letter to array of selected item slugs

#### Ability Score Choices

**Request:**
```json
{
  "selected": ["STR", "CHA"]
}
```

**Notes:**
- For ASI choices, the `selected` array should contain ALL selections, not just new ones
- If partially resolved and adding more, merge existing `selected` with new picks

#### Optional Feature Choices (Fighting Style, etc.)

**Request:**
```json
{
  "selected": ["phb:defense"]
}
```

### 6.3 Resolve Order

Resolve choices in this order for best UX:
1. **Equipment Mode** - Determines if equipment choices appear
2. **Proficiencies** - Often have collisions to avoid (same skill from race + class)
3. **Languages** - Similar collision handling
4. **Equipment** - After mode is set
5. **Spells** - Cantrips and known spells
6. **Other** - Size, ability scores, optional features

### 6.4 Loop Until Complete

Keep checking `GET /pending-choices` and resolving until `summary.required_pending === 0`.

---

## Step 7: Set Name and Alignment

**Endpoint:** `PATCH /api/v1/characters/{id}`

**Request:**
```json
{
  "name": "Elara Moonwhisper",
  "alignment": "Neutral Good"
}
```

**Valid alignments:**
- Lawful Good, Neutral Good, Chaotic Good
- Lawful Neutral, True Neutral, Chaotic Neutral
- Lawful Evil, Neutral Evil, Chaotic Evil

---

## Step 8: Check Completion Status

### Option A: Character Resource (recommended)

**Endpoint:** `GET /api/v1/characters/{id}`

The character response includes completion status:
```json
{
  "data": {
    "id": 123,
    "name": "Elara Moonwhisper",
    "is_complete": true,
    "validation_status": {
      "is_complete": true,
      "missing": []
    },
    ...
  }
}
```

When incomplete:
```json
{
  "data": {
    "is_complete": false,
    "validation_status": {
      "is_complete": false,
      "missing": ["race", "class", "background"]
    },
    ...
  }
}
```

### Option B: Summary Endpoint

**Endpoint:** `GET /api/v1/characters/{id}/summary`

```json
{
  "data": {
    "character": { "id": 123, "name": "Elara", "total_level": 1 },
    "pending_choices": { "proficiencies": 0, "languages": 0, ... },
    "creation_complete": true,
    "missing_required": []
  }
}
```

### Validate Endpoint (Data Integrity Only)

**Endpoint:** `GET /api/v1/characters/{id}/validate`

> **Note:** This endpoint checks for **dangling references** (data integrity), not character creation completion. Use this to verify all slug-based references resolve after reimports.

**Response:**
```json
{
  "data": {
    "valid": true,
    "dangling_references": {},
    "summary": {
      "total_references": 22,
      "valid_references": 22,
      "dangling_count": 0
    }
  }
}
```

**When dangling references exist:**
```json
{
  "data": {
    "valid": false,
    "dangling_references": {
      "race": "phb:high-elf",
      "spells": ["phb:wish", "phb:meteor-swarm"]
    },
    "summary": {
      "total_references": 15,
      "valid_references": 12,
      "dangling_count": 3
    }
  }
}
```

**Notes:**
- Use `is_complete` from the character resource or `creation_complete` from summary for wizard completion
- The `/validate` endpoint is for data integrity checks, not wizard flow

---

## Handling Switches (Changing Selections)

When the user goes back and changes race/class/background, the backend handles cascading resets:

### Switching Race

- Clears: subrace, racial proficiencies, racial languages, racial spells, racial ASI choices
- Keeps: class, background, equipment mode, equipment, class proficiencies

### Switching Class

- Clears: subclass, class proficiencies, class features, equipment mode, equipment, spells
- Keeps: race, background, background proficiencies, background languages

### Switching Background

- Clears: background proficiencies, background languages, background equipment
- Keeps: race, class, class proficiencies, spells, equipment mode

**Important:** After any switch, re-fetch pending choices - new ones will have been generated.

---

## Complete Flow Example

### Option A: Step-by-step (separate requests)
```
1. POST /api/v1/characters
   { "public_id": "brave-warrior-X4kP", "name": "Temp" }

2. PATCH /api/v1/characters/123
   { "race_slug": "phb:high-elf" }

3. POST /api/v1/characters/123/classes
   { "class_slug": "phb:wizard", "force": true }

4. PATCH /api/v1/characters/123
   { "background_slug": "phb:sage" }

5. PATCH /api/v1/characters/123
   { "strength": 8, "dexterity": 14, "constitution": 14,
     "intelligence": 15, "wisdom": 12, "charisma": 10 }

6. GET /api/v1/characters/123/pending-choices
   → Loop through and resolve each required choice

7. PATCH /api/v1/characters/123
   { "name": "Elara Moonwhisper", "alignment": "Neutral Good" }

8. GET /api/v1/characters/123
   → Check: is_complete === true
```

### Option B: Combined initial POST (fewer requests)
```
1. POST /api/v1/characters
   { "public_id": "brave-warrior-X4kP", "name": "Temp",
     "race_slug": "phb:high-elf", "class_slug": "phb:wizard",
     "background_slug": "phb:sage" }

2. PATCH /api/v1/characters/123
   { "strength": 8, "dexterity": 14, "constitution": 14,
     "intelligence": 15, "wisdom": 12, "charisma": 10 }

3. GET /api/v1/characters/123/pending-choices
   → Loop through and resolve each required choice

4. PATCH /api/v1/characters/123
   { "name": "Elara Moonwhisper", "alignment": "Neutral Good" }

5. GET /api/v1/characters/123
   → Check: is_complete === true
```

---

## Fetching Options

For some choices, options aren't inline but require a separate fetch:

```json
{
  "id": "spell:class:phb:wizard:cantrip:3",
  "type": "spell",
  "options": null,
  "options_endpoint": "/api/v1/characters/123/available-spells?level=0&class=phb:wizard"
}
```

When `options` is null/empty but `options_endpoint` is set, fetch from that endpoint to get available options.

---

## Error Handling

### Validation Errors (422)

```json
{
  "message": "The selected field is required.",
  "errors": {
    "selected": ["The selected field is required."]
  }
}
```

### Not Found (404)

```json
{
  "message": "Choice not found: invalid-choice-id"
}
```

### Invalid Selection (422)

```json
{
  "message": "Invalid selection",
  "errors": {
    "selected": ["Option 'invalid-slug' is not available for this choice"]
  }
}
```

---

## Edge Cases & Special Scenarios

### 1. Proficiency Collisions

When the same proficiency is available from multiple sources (race + class + background), the backend filters duplicates from options.

**Example:** Elf grants Perception proficiency. If the character is a Ranger (which also offers Perception), the Ranger's proficiency choice won't include Perception as an option.

**Frontend handling:** No special handling needed. Just display what `options` contains - the backend has already filtered.

### 2. Subrace Required

Some races have subraces and require one to be selected.

**How to detect:**
```
GET /api/v1/races/{raceSlug}
```
Check if `subraces` array is non-empty. If so, show subrace selection step.

**Races with subraces:** Elf, Dwarf, Halfling, Gnome, Tiefling (some sources), Dragonborn (Fizban's)

**Races without subraces:** Human, Half-Orc, Half-Elf, etc.

### 3. Subclass at Level 1

Only some classes get their subclass at level 1:

| Class | Subclass Level |
|-------|---------------|
| Cleric | 1 |
| Sorcerer | 1 |
| Warlock | 1 |
| All others | 2 or 3 |

**How to detect:**
```
GET /api/v1/classes/{classSlug}
```
Check `subclass_level`. If `1`, show subclass selection immediately after class selection.

### 4. Variable Size Races

Some races allow choosing size (Small or Medium):

- Fairy
- Harengon
- Owlin
- Custom Lineage

**How to detect:** A pending choice with `type: "size"` will appear.

**Response example:**
```json
{
  "id": "size:race:phb:fairy:1",
  "type": "size",
  "options": [
    { "code": "S", "name": "Small" },
    { "code": "M", "name": "Medium" }
  ]
}
```

**Resolve:**
```json
{ "selected": ["M"] }
```

### 5. Racial Ability Score Choices

Some races let you choose which ability scores get bonuses:

- Half-Elf: +2 CHA fixed, +1 to two other abilities
- Custom Lineage: +2 to one ability
- Tasha's variants: Many races can swap ASIs

**How to detect:** A pending choice with `type: "ability_score"` will appear.

**Important:** For ability score choices, `selected` must include ALL selections, not just new ones. If partially resolved:
```json
// Choice has: quantity: 2, selected: ["STR"], remaining: 1
// To add DEX, send the FULL list:
{ "selected": ["STR", "DEX"] }
```

### 6. Equipment vs Gold Mode

After selecting class, an `equipment_mode` choice appears.

| Mode | What happens |
|------|--------------|
| `equipment` | Equipment choices appear; resolve each to get items |
| `gold` | Starting gold granted based on class; no equipment choices |

**Gold amounts by class:**
- Monk: 5d4 GP
- Barbarian: 2d4 × 10 GP
- Bard, Cleric, Druid, Warlock: 5d4 × 10 GP
- Fighter, Paladin, Ranger: 5d4 × 10 GP
- Rogue: 4d4 × 10 GP
- Sorcerer, Wizard: 4d4 × 10 GP

### 7. Equipment Category Choices

Some equipment choices have categories where the user picks from a list:

```json
{
  "id": "equipment:class:phb:fighter:1",
  "type": "equipment",
  "options": [
    {
      "option": "a",
      "is_category": false,
      "items": [{ "slug": "phb:chain-mail", "name": "Chain Mail" }]
    },
    {
      "option": "b",
      "is_category": true,
      "items": [
        { "slug": "phb:leather-armor", "name": "Leather Armor", "is_fixed": true },
        { "slug": "phb:longbow", "name": "Longbow", "is_fixed": false },
        { "slug": "phb:shortbow", "name": "Shortbow", "is_fixed": false }
      ]
    }
  ]
}
```

**For option "a":** Just send `{ "selected": ["a"] }`

**For option "b" (category):** User must also pick the non-fixed item:
```json
{
  "selected": ["b"],
  "item_selections": {
    "b": ["phb:longbow"]
  }
}
```

**Tip:** Items with `is_fixed: true` are always granted; items with `is_fixed: false` are selectable.

### 8. Spellcaster Spell Choices

Spellcasters get spell choices. These often have `options_endpoint` instead of inline `options`:

```json
{
  "id": "spell:class:phb:wizard:cantrip:3",
  "type": "spell",
  "subtype": "cantrip",
  "quantity": 3,
  "remaining": 3,
  "options": null,
  "options_endpoint": "/api/v1/characters/123/available-spells?level=0&class=phb:wizard"
}
```

**Fetch available spells:**
```
GET /api/v1/characters/123/available-spells?level=0&class=phb:wizard
```

**Resolve:**
```json
{
  "selected": ["phb:fire-bolt", "phb:mage-hand", "phb:prestidigitation"]
}
```

### 9. Switching Class Clears Equipment Mode

When switching class:
- Equipment mode is cleared (resets to unset)
- All equipment choices are cleared
- New equipment_mode choice appears
- User must re-select equipment mode and equipment

**Why:** Different classes have different equipment options and gold amounts.

### 10. Incomplete Characters

Characters with `is_complete: false` can exist indefinitely. They:
- Can be loaded and edited
- Cannot be used in parties or encounters
- Show up in character lists with incomplete status

**Resume wizard:** Fetch `GET /pending-choices` - if any required choices remain, the wizard isn't done.

### 11. Racial Spells

Some races grant spellcasting (Tiefling, Drow, etc.). These appear as:
- Fixed spells (automatically granted) - no choice needed
- Or spell choices with `source: "race"`

**Switching races clears racial spells.**

### 12. Background Tool Proficiencies

Some backgrounds grant tool proficiency choices:

```json
{
  "id": "proficiency:background:phb:guild-artisan:1",
  "type": "proficiency",
  "subtype": "artisan_tool",
  "options": [
    { "slug": "phb:alchemists-supplies", "name": "Alchemist's Supplies" },
    { "slug": "phb:brewers-supplies", "name": "Brewer's Supplies" }
  ]
}
```

These are resolved the same way as skill proficiencies.

### 13. Optional Feature Choices (Fighting Style, etc.)

Classes like Fighter and Ranger get Fighting Style at level 1:

```json
{
  "id": "optional_feature:class:phb:fighter:fighting-style:1",
  "type": "optional_feature",
  "options": [
    { "slug": "phb:defense", "name": "Defense" },
    { "slug": "phb:dueling", "name": "Dueling" },
    { "slug": "phb:great-weapon-fighting", "name": "Great Weapon Fighting" }
  ]
}
```

**Resolve:**
```json
{ "selected": ["phb:defense"] }
```

### 14. Order Independence

While the recommended order is Race → Class → Background → Ability Scores, the API doesn't strictly enforce this. However:

- **Equipment mode** requires a class to be set first
- **Subclass** requires a class to be set first
- **Subrace** requires a race to be set first

Fetching pending choices when these aren't set returns limited choices.

### 15. Undoing Choices

Resolved choices can be undone:

```
DELETE /api/v1/characters/{id}/choices/{choiceId}
```

This reverts the choice and returns the selected items/proficiencies. Not all choice types support undo - check the response.

### 16. Multiple Spell Choice Groups

A wizard might have multiple spell choices:
- Cantrips (3 at level 1)
- Spellbook spells (6 at level 1)
- High Elf bonus cantrip (if high elf)

Each appears as a separate choice with different `id` values. Resolve each independently.

### 17. Feat Choices (Variant Human, Custom Lineage)

Variant Human and Custom Lineage get a feat at level 1:

```json
{
  "id": "feat:race:phb:variant-human:1",
  "type": "feat",
  "options_endpoint": "/api/v1/characters/123/available-feats"
}
```

**Important:** Available feats are filtered by prerequisites. The endpoint only returns feats the character qualifies for.

---

## Debugging Tips

### Character State

Get full character state anytime:
```
GET /api/v1/characters/{id}
```

### What's Missing?

Check the character resource's `validation_status.missing` array:
```
GET /api/v1/characters/{id}
```

Or use the summary endpoint's `missing_required` array:
```
GET /api/v1/characters/{id}/summary
```

These return specific fields that are incomplete (e.g., `["race", "class", "background"]`).

### Choice Details

Get details about a specific choice:
```
GET /api/v1/characters/{id}/pending-choices/{choiceId}
```

Returns full choice details including all options.
