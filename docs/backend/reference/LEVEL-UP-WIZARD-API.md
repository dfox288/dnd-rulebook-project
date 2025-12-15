# Level-Up Wizard - API Flow

**Issue:** #687

This document details the step-by-step API flow for leveling up a character. Frontend should follow this sequence when a user levels up their character.

## Overview

Level-up is simpler than character creation:
1. Verify character is complete
2. Call level-up endpoint
3. Resolve any pending choices (HP, ASI/Feat, subclass, spells, etc.)
4. Repeat until desired level reached

## Prerequisites

A character must be **complete** (`is_complete: true`) before leveling up. Incomplete characters cannot level up.

---

## Step 1: Check Character State

**Endpoint:** `GET /api/v1/characters/{id}`

Verify:
- `is_complete === true`
- Note `total_level` (current level)
- Note `classes` array (for multiclassing decisions)

**Response excerpt:**
```json
{
  "data": {
    "id": 123,
    "is_complete": true,
    "total_level": 5,
    "classes": [
      {
        "class": { "slug": "phb:wizard", "name": "Wizard" },
        "subclass": { "slug": "phb:school-of-evocation", "name": "School of Evocation" },
        "level": 5,
        "is_primary": true
      }
    ]
  }
}
```

---

## Step 2: Level Up a Class

**Endpoint:** `POST /api/v1/characters/{id}/classes/{classSlug}/level-up`

**Request:** No body required

**Response:**
```json
{
  "data": {
    "message": "Leveled up to Wizard 6",
    "new_level": 6,
    "total_level": 6,
    "features_gained": [
      {
        "slug": "phb:wizard:arcane-tradition-feature-6",
        "name": "Arcane Tradition Feature",
        "description": "At 6th level, you gain a feature from your Arcane Tradition."
      }
    ],
    "pending_choices": 2
  }
}
```

**Notes:**
- `classSlug` is the class to level (e.g., `phb:wizard`)
- For single-class characters, use the only class
- For multiclass characters, specify which class gets the level
- `features_gained` lists new features from this level
- `pending_choices` indicates how many choices need resolution

---

## Step 3: Resolve Pending Choices

After level-up, check for pending choices:

**Endpoint:** `GET /api/v1/characters/{id}/pending-choices`

### Level-Up Choice Types

| Type | When It Appears | Description |
|------|-----------------|-------------|
| `hit_points` | Every level | Roll, average, or manual HP |
| `asi_or_feat` | Levels 4, 8, 12, 16, 19 | ASI (+2 to one or +1 to two) or Feat |
| `subclass` | Class-specific level | Choose subclass (if not yet chosen) |
| `spell` / `cantrip` | Spellcaster levels | New spells known |
| `optional_feature` | Various levels | Fighting Style, Invocations, etc. |
| `expertise` | Rogue 1, 6; Bard 3, 10 | Double proficiency bonus |
| `ability_score` | Some features | Ability score choices |

---

## Hit Points Choice

Every level requires an HP choice:

```json
{
  "id": "hit_points:level:6",
  "type": "hit_points",
  "source": "level_up",
  "level_granted": 6,
  "required": true,
  "options": [
    { "id": "roll", "name": "Roll", "description": "Roll 1d6 for HP" },
    { "id": "average", "name": "Average", "description": "Take average (4 HP)" },
    { "id": "manual", "name": "Manual", "description": "Enter HP manually" }
  ]
}
```

**Resolve:**
```json
POST /api/v1/characters/{id}/choices/hit_points:level:6
{ "selected": ["average"] }
```

**Notes:**
- `roll` - Backend rolls dice, result is applied
- `average` - Uses fixed average (rounded up): d6=4, d8=5, d10=6, d12=7
- `manual` - Requires additional `value` parameter: `{ "selected": ["manual"], "value": 5 }`

---

## ASI or Feat Choice

At levels 4, 8, 12, 16, 19 (varies by class):

```json
{
  "id": "asi_or_feat:class:phb:wizard:4",
  "type": "asi_or_feat",
  "source": "class",
  "level_granted": 4,
  "required": true,
  "options": [
    { "value": "asi", "name": "Ability Score Improvement" },
    { "value": "feat", "name": "Feat" }
  ]
}
```

### Option 1: ASI

**Resolve with ASI:**
```json
POST /api/v1/characters/{id}/choices/asi_or_feat:class:phb:wizard:4
{ "selected": ["asi"] }
```

This generates a follow-up `ability_score` choice:

```json
{
  "id": "ability_score:asi:phb:wizard:4",
  "type": "ability_score",
  "quantity": 2,
  "remaining": 2,
  "options": [
    { "code": "STR", "name": "Strength" },
    { "code": "DEX", "name": "Dexterity" },
    { "code": "CON", "name": "Constitution" },
    { "code": "INT", "name": "Intelligence" },
    { "code": "WIS", "name": "Wisdom" },
    { "code": "CHA", "name": "Charisma" }
  ]
}
```

**Resolve ASI allocation:**
```json
// +2 to one ability
{ "selected": ["INT", "INT"] }

// OR +1 to two abilities
{ "selected": ["INT", "CON"] }
```

### Option 2: Feat

**Resolve with Feat:**
```json
POST /api/v1/characters/{id}/choices/asi_or_feat:class:phb:wizard:4
{ "selected": ["feat"] }
```

This generates a follow-up `feat` choice:

```json
{
  "id": "feat:asi:phb:wizard:4",
  "type": "feat",
  "options_endpoint": "/api/v1/characters/123/available-feats"
}
```

**Fetch available feats:**
```
GET /api/v1/characters/123/available-feats
```

**Resolve feat selection:**
```json
{ "selected": ["phb:war-caster"] }
```

---

## Subclass Choice

When a class reaches its subclass level:

| Class | Subclass Level |
|-------|---------------|
| Barbarian | 3 |
| Bard | 3 |
| Cleric | 1 |
| Druid | 2 |
| Fighter | 3 |
| Monk | 3 |
| Paladin | 3 |
| Ranger | 3 |
| Rogue | 3 |
| Sorcerer | 1 |
| Warlock | 1 |
| Wizard | 2 |
| Artificer | 3 |

**Choice example:**
```json
{
  "id": "subclass:class:phb:fighter:3",
  "type": "subclass",
  "source": "class",
  "level_granted": 3,
  "required": true,
  "options": [
    { "slug": "phb:champion", "name": "Champion" },
    { "slug": "phb:battle-master", "name": "Battle Master" },
    { "slug": "phb:eldritch-knight", "name": "Eldritch Knight" }
  ]
}
```

**Resolve:**
```json
{ "selected": ["phb:battle-master"] }
```

---

## Spell Choices

Spellcasters gain spells at various levels:

### Cantrips

```json
{
  "id": "spell:class:phb:wizard:cantrip:4",
  "type": "spell",
  "subtype": "cantrip",
  "quantity": 1,
  "remaining": 1,
  "options_endpoint": "/api/v1/characters/123/available-spells?level=0&class=phb:wizard"
}
```

### Spells Known (Sorcerer, Bard, Warlock, Ranger)

```json
{
  "id": "spell:class:phb:sorcerer:known:5",
  "type": "spell",
  "subtype": "spells_known",
  "quantity": 1,
  "remaining": 1,
  "options_endpoint": "/api/v1/characters/123/available-spells?max_level=3&class=phb:sorcerer"
}
```

### Spellbook Spells (Wizard)

```json
{
  "id": "spell:class:phb:wizard:spellbook:3",
  "type": "spell",
  "subtype": "spellbook",
  "quantity": 2,
  "remaining": 2,
  "options_endpoint": "/api/v1/characters/123/available-spells?max_level=2&class=phb:wizard"
}
```

**Resolve spell choice:**
```json
{ "selected": ["phb:fireball", "phb:counterspell"] }
```

---

## Optional Feature Choices

### Fighting Style (Fighter 1, Paladin 2, Ranger 2)

```json
{
  "id": "optional_feature:class:phb:fighter:fighting-style:1",
  "type": "optional_feature",
  "options": [
    { "slug": "phb:defense", "name": "Defense" },
    { "slug": "phb:dueling", "name": "Dueling" },
    { "slug": "phb:great-weapon-fighting", "name": "Great Weapon Fighting" },
    { "slug": "phb:archery", "name": "Archery" }
  ]
}
```

### Eldritch Invocations (Warlock 2+)

```json
{
  "id": "optional_feature:class:phb:warlock:invocation:2",
  "type": "optional_feature",
  "quantity": 2,
  "remaining": 2,
  "options_endpoint": "/api/v1/characters/123/available-feature-selections?type=invocation"
}
```

### Metamagic (Sorcerer 3+)

```json
{
  "id": "optional_feature:class:phb:sorcerer:metamagic:3",
  "type": "optional_feature",
  "quantity": 2,
  "remaining": 2,
  "options": [
    { "slug": "phb:careful-spell", "name": "Careful Spell" },
    { "slug": "phb:distant-spell", "name": "Distant Spell" },
    { "slug": "phb:empowered-spell", "name": "Empowered Spell" },
    { "slug": "phb:quickened-spell", "name": "Quickened Spell" },
    { "slug": "phb:twinned-spell", "name": "Twinned Spell" }
  ]
}
```

**Resolve:**
```json
{ "selected": ["phb:quickened-spell", "phb:twinned-spell"] }
```

---

## Expertise Choice (Rogue, Bard)

```json
{
  "id": "expertise:class:phb:rogue:1",
  "type": "expertise",
  "quantity": 2,
  "remaining": 2,
  "options": [
    { "slug": "stealth", "name": "Stealth" },
    { "slug": "perception", "name": "Perception" },
    { "slug": "thieves-tools", "name": "Thieves' Tools" }
  ]
}
```

**Notes:**
- Only skills/tools the character is proficient in appear as options
- Expertise doubles the proficiency bonus for that skill

**Resolve:**
```json
{ "selected": ["stealth", "perception"] }
```

---

## Multiclassing

### Adding a New Class

**Endpoint:** `POST /api/v1/characters/{id}/classes`

**Request:**
```json
{
  "class_slug": "phb:fighter",
  "force": false
}
```

**Notes:**
- `force: false` respects multiclass prerequisites (13+ in required abilities)
- Returns error if prerequisites not met
- Adding a class sets it at level 1
- This counts as "taking a level" - the character's total level increases by 1

**Prerequisites by class:**
| Class | Requirement |
|-------|-------------|
| Barbarian | STR 13 |
| Bard | CHA 13 |
| Cleric | WIS 13 |
| Druid | WIS 13 |
| Fighter | STR 13 or DEX 13 |
| Monk | DEX 13 and WIS 13 |
| Paladin | STR 13 and CHA 13 |
| Ranger | DEX 13 and WIS 13 |
| Rogue | DEX 13 |
| Sorcerer | CHA 13 |
| Warlock | CHA 13 |
| Wizard | INT 13 |

### Leveling a Specific Class (Multiclass)

When multiclassed, specify which class to level:

```
POST /api/v1/characters/{id}/classes/phb:wizard/level-up
```

or

```
POST /api/v1/characters/{id}/classes/phb:fighter/level-up
```

---

## Complete Level-Up Flow Example

```
// Character is Wizard 5, wants to reach Wizard 6

1. GET /api/v1/characters/123
   → Verify is_complete: true, total_level: 5

2. POST /api/v1/characters/123/classes/phb:wizard/level-up
   → { "new_level": 6, "pending_choices": 1 }

3. GET /api/v1/characters/123/pending-choices
   → Returns HP choice

4. POST /api/v1/characters/123/choices/hit_points:level:6
   { "selected": ["average"] }

5. GET /api/v1/characters/123/pending-choices
   → summary.required_pending === 0

Done! Character is now Wizard 6.
```

---

## Multi-Level Flow Example (Wizard 1 → 4)

```
// Level 1 → 2
POST /api/v1/characters/123/classes/phb:wizard/level-up
→ Resolve: HP, possibly subclass (Wizard gets subclass at 2)

// Level 2 → 3
POST /api/v1/characters/123/classes/phb:wizard/level-up
→ Resolve: HP, new spells

// Level 3 → 4
POST /api/v1/characters/123/classes/phb:wizard/level-up
→ Resolve: HP, new spells, ASI or Feat, plus ASI allocation if ASI chosen

// After each level-up:
GET /api/v1/characters/123/pending-choices
→ Loop until required_pending === 0
```

---

## Edge Cases

### 1. Character Not Complete

**Error:**
```json
{
  "message": "Character is not complete and cannot level up",
  "errors": {}
}
```

**Solution:** Complete character creation first.

### 2. Already at Max Level

**Error:**
```json
{
  "message": "Character has reached maximum level (20)",
  "errors": {}
}
```

### 3. Class Not Found on Character

**Error:**
```json
{
  "message": "Character does not have class: phb:paladin",
  "errors": {}
}
```

**Solution:** Either level an existing class or add the class first via multiclassing.

### 4. Multiclass Prerequisites Not Met

**Error:**
```json
{
  "message": "Character does not meet multiclass prerequisites for Fighter",
  "errors": {
    "prerequisites": ["Requires Strength 13 or Dexterity 13"]
  }
}
```

### 5. Spell Selection When Spell Slots Change

When a character gains access to new spell levels, the `options_endpoint` respects the new maximum:

- Wizard 3 → 4: Can now prepare 2nd-level spells
- `available-spells?max_level=2` returns 2nd-level spells

### 6. Warlock Pact Magic

Warlocks have Pact Magic (different from standard spellcasting):
- All slots are the same level
- Slots refresh on short rest
- Spell choices respect Pact Magic progression

### 7. Replacing Spells (Sorcerer, Bard, Warlock, Ranger)

Some classes can replace one known spell when leveling:

```json
{
  "id": "spell_swap:class:phb:sorcerer:6",
  "type": "spell_swap",
  "required": false,
  "current_spells": ["phb:burning-hands", "phb:magic-missile"],
  "options_endpoint": "/api/v1/characters/123/available-spells?max_level=3"
}
```

**Resolve (optional):**
```json
{
  "selected": ["phb:fireball"],
  "swap_out": "phb:burning-hands"
}
```

### 8. Fighter Extra ASIs

Fighters get ASIs at levels 4, 6, 8, 12, 14, 16, 19 (7 total vs. 5 for most classes).

### 9. Rogue Expertise Timing

- Level 1: Choose 2 expertise
- Level 6: Choose 2 more expertise

The level 6 choice only shows skills not already expertise.

### 10. Multiclass Spellcasting

When multiclassing spellcasters:
- Spell slots combine using multiclass table
- Spells known are per-class
- Cantrips are per-class
- Spell choice endpoints filter by class

Example: Wizard 3 / Cleric 2
- Has spell slots of a 3rd-level caster (multiclass calculation)
- Wizard spells: INT-based, from wizard list
- Cleric spells: WIS-based, from cleric list (prepared, not chosen)

### 11. Subclass Already Selected

If a character already has a subclass and reaches another "subclass level" (for features):
- No subclass choice appears
- Subclass features are auto-granted

### 12. HP After Level 1

After level 1:
- Never add Constitution modifier again (it's already factored into max HP)
- HP gained = roll/average + CON modifier for THAT level

Wait, that's wrong. Let me correct:
- Level 1: Max hit die + CON mod
- Level 2+: Roll/average + CON mod (added each level)

---

## Debugging

### Check Current Level
```
GET /api/v1/characters/{id}
→ total_level, classes[].level
```

### Check Pending After Level-Up
```
GET /api/v1/characters/{id}/pending-choices
→ summary.required_pending should be 0 when done
```

### Validate Character
```
GET /api/v1/characters/{id}/validate
→ Should show is_valid: true, is_complete: true
```

### Character Stats
```
GET /api/v1/characters/{id}/stats
→ Full computed stats including HP, spell slots, etc.
```
