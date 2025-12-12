# Character Play Mode - API Analysis & Implementation Plan

**Date:** 2025-12-12
**Status:** Planning
**Epic:** #530

## Overview

This document analyzes the backend API capabilities for "play mode" character management and outlines the frontend implementation strategy.

---

## Backend API Inventory

### 1. HP Management

**Endpoints:**
| Method | Endpoint | Purpose |
|--------|----------|---------|
| POST | `/characters/{id}/death-saves` | Record death save (roll or damage) |
| POST | `/characters/{id}/death-saves/stabilize` | Manually stabilize at 0 HP |
| DELETE | `/characters/{id}/death-saves` | Reset death save counters |

**Database Columns (characters table):**
- `max_hit_points` (smallint unsigned)
- `current_hit_points` (smallint unsigned)
- `temp_hit_points` (smallint unsigned, default 0)
- `death_save_successes` (tinyint unsigned, default 0)
- `death_save_failures` (tinyint unsigned, default 0)

**Gap:** No direct `PATCH /characters/{id}/hp` endpoint for damage/healing.

---

### 2. Conditions

**Endpoints:**
| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/characters/{id}/conditions` | List active conditions |
| POST | `/characters/{id}/conditions` | Apply condition |
| DELETE | `/characters/{id}/conditions/{slug}` | Remove condition |

**Database Table: `character_conditions`**
```
id, character_id, condition_slug, level (exhaustion only),
source, duration, created_at, updated_at
```

**Supported Conditions (15):**
1. Blinded
2. Charmed
3. Deafened
4. Frightened
5. Grappled
6. Incapacitated
7. Invisible
8. Paralyzed
9. Petrified
10. Poisoned
11. Prone
12. Restrained
13. Stunned
14. Unconscious
15. Exhaustion (levels 1-6)

---

### 3. Spell Slots

**Endpoints:**
| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/characters/{id}/spell-slots` | Get current slot status |
| POST | `/characters/{id}/spell-slots/use` | Expend a slot |
| GET | `/characters/{id}/spells` | List known spells |
| PATCH | `/characters/{id}/spells/{slug}/prepare` | Prepare spell |
| PATCH | `/characters/{id}/spells/{slug}/unprepare` | Unprepare spell |

**Database Table: `character_spell_slots`**
```
id, character_id, spell_level (1-9), max_slots, used_slots,
slot_type ("standard" | "pact_magic"), created_at, updated_at
```

**Key Features:**
- Pact magic (Warlock) tracked separately from standard slots
- Multiclass spell slot calculation per RAW
- Preparation status: known, prepared, always_prepared

---

### 4. Class Resources (Limited-Use Features)

**Endpoints:**
| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/characters/{id}/feature-uses` | List features with remaining uses |
| POST | `/characters/{id}/features/{id}/use` | Decrement uses |
| POST | `/characters/{id}/features/{id}/reset` | Reset to max (DM override) |

**Database Table: `character_features`**
```
id, character_id, feature_type, feature_slug, source,
level_acquired, uses_remaining, max_uses, created_at
```

**Reset Timing Categories:**
- Short rest (Action Surge, Second Wind)
- Long rest (Rage, most features)
- Dawn (certain monk abilities)

---

### 5. Inventory

**Endpoints:**
| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/characters/{id}/equipment` | List all items |
| POST | `/characters/{id}/equipment` | Add item |
| PATCH | `/characters/{id}/equipment/{id}` | Equip/unequip, quantity |
| DELETE | `/characters/{id}/equipment/{id}` | Remove item |

**Database Table: `character_equipment`**
```
id, character_id, item_slug (nullable), custom_name (nullable),
custom_description (nullable), quantity, equipped, location, created_at
```

**Locations:** backpack, belt, main_hand, off_hand, armor, etc.

**Key Features:**
- Database items (item_slug) OR custom items (custom_name)
- Stackable items with quantity
- Equipment slots with location tracking

---

### 6. Rests

**Endpoints:**
| Method | Endpoint | Purpose |
|--------|----------|---------|
| POST | `/characters/{id}/short-rest` | Perform short rest |
| POST | `/characters/{id}/long-rest` | Perform long rest |
| GET | `/characters/{id}/hit-dice` | View available/spent hit dice |
| POST | `/characters/{id}/hit-dice/spend` | Spend hit dice |
| POST | `/characters/{id}/hit-dice/recover` | Recover spent hit dice |

**Short Rest Effects:**
- Reset pact magic slots
- Reset features with "short rest" timing
- Allow hit dice spending

**Long Rest Effects:**
- Restore HP to maximum
- Recover half hit dice (minimum 1)
- Reset all spell slots
- Clear death save counters
- Reset all features

---

### 7. Notes/Journal

**Endpoints:**
| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/characters/{id}/notes` | List notes by category |
| POST | `/characters/{id}/notes` | Create note |
| GET | `/characters/{id}/notes/{id}` | Get single note |
| PUT | `/characters/{id}/notes/{id}` | Update note |
| DELETE | `/characters/{id}/notes/{id}` | Delete note |

**Categories:**
- `personality_trait` - Character personality
- `ideal` - What character believes in
- `bond` - Connections to people/places
- `flaw` - Character weaknesses
- `backstory` - Character history (has title)
- `custom` - Free-form notes (has title)

---

### 8. Other Features

| Feature | Status | Notes |
|---------|--------|-------|
| Inspiration | ✅ `has_inspiration` boolean | Simple toggle |
| Experience Points | ✅ `experience_points` integer | For non-milestone |
| Attunement | ❌ Not implemented | Need 3-slot tracking |
| Currency | ⚠️ Indirect | Gold as equipment item |
| Concentration | ❌ Not implemented | Spell concentration state |

---

## Implementation Phases

### Phase 1: Core Combat Support (Issues #531-534)

| Issue | Component | Priority |
|-------|-----------|----------|
| #531 | HP Management Panel | High |
| #532 | Conditions Panel | High |
| #533 | Death Saves Tracker | High |
| #534 | Rest Actions | High |

**Estimated effort:** 1-2 weeks

### Phase 2: Resource Management

| Feature | Component | Priority |
|---------|-----------|----------|
| Spell Slots | SpellSlotsPanel.vue | Medium |
| Class Resources | ResourcesPanel.vue | Medium |
| Hit Dice Spending | HitDiceSpender.vue | Medium |

**Estimated effort:** 1 week

### Phase 3: Inventory & Spells

| Feature | Component | Priority |
|---------|-----------|----------|
| Equipment List | EquipmentPanel.vue | Medium |
| Add Item Modal | AddItemModal.vue | Medium |
| Spell Preparation | SpellPreparation.vue | Medium |

**Estimated effort:** 1 week

### Phase 4: Polish

| Feature | Component | Priority |
|---------|-----------|----------|
| Notes/Journal | NotesPanel.vue | Low |
| Inspiration Toggle | InspirationBadge.vue | Low |
| XP Tracker | ExperiencePanel.vue | Low |

**Estimated effort:** 3-5 days

---

## Backend Requests

### Needed Endpoints

1. **Direct HP Modification**
   ```
   PATCH /characters/{id}/hp
   Body: { damage?: number, healing?: number, temp_hp?: number }
   ```
   Currently must calculate delta and update via character update.

2. **Attunement Slots** (future)
   ```
   GET /characters/{id}/attunements
   POST /characters/{id}/attunements
   DELETE /characters/{id}/attunements/{id}
   ```

### Schema Additions (future)

```sql
CREATE TABLE character_attunements (
  id BIGINT UNSIGNED PRIMARY KEY,
  character_id BIGINT UNSIGNED,
  item_slug VARCHAR(150),
  attuned_at TIMESTAMP,
  FOREIGN KEY (character_id) REFERENCES characters(id)
);
-- Max 3 per character per D&D 5e rules
```

---

## UX Principles

1. **Non-destructive by default** - Confirm dialogs for major actions
2. **Quick actions** - Common operations should be one-click
3. **Mobile-first** - Players use phones at the table
4. **Dark mode essential** - Sessions happen in dim lighting
5. **Offline resilience** - Queue actions if connection drops (future)
6. **Clear feedback** - Visual confirmation of all state changes

---

## Component Hierarchy

```
CharacterSheet.vue
├── CharacterSheetHeader.vue (existing)
├── PlayModePanel.vue (new - container)
│   ├── HpPanel.vue (#531)
│   │   └── DeathSavesTracker.vue (#533)
│   ├── ConditionsPanel.vue (#532)
│   ├── RestPanel.vue (#534)
│   │   ├── ShortRestModal.vue
│   │   ├── LongRestModal.vue
│   │   └── HitDiceSpender.vue
│   ├── SpellSlotsPanel.vue (Phase 2)
│   ├── ResourcesPanel.vue (Phase 2)
│   └── EquipmentPanel.vue (Phase 3)
├── AbilityScoreBlock.vue (existing)
├── SkillsList.vue (existing)
└── ... other existing components
```

---

## Success Metrics

- [ ] Player can run entire combat using only character sheet
- [ ] All resource tracking persists correctly
- [ ] Rest mechanics reset appropriate resources
- [ ] Conditions show mechanical effects
- [ ] Mobile users can manage character comfortably
- [ ] < 500ms response time for all actions
- [ ] Works offline with sync on reconnect (future)

---

## Related Issues

- **Epic:** #530 (Character Play Mode)
- **Phase 1:** #531, #532, #533, #534
- **Prerequisite:** #488 (Level-Up Wizard) - building before playing
