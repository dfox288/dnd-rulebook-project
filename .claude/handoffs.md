# Agent Handoffs

<!--
This file enables async communication between Claude Code agents working in different repos.

HOW IT WORKS:
- When an agent creates cross-repo work, they append a handoff here with full context
- When an agent starts a session, they check for handoffs addressed to them
- After processing a handoff, DELETE that section from this file

LOCATION: wrapper/.claude/handoffs.md
-->

## Pending Handoffs

<!-- Agents: Add new handoffs below this line. Delete handoffs after processing. -->

## For: frontend
**From:** backend | **Issue:** #564 | **Created:** 2025-12-13

Party System API + DM Screen data is complete. Frontend needs UI for party management and DM dashboard.

**What I did (Issues #559 + #563):**
- Created Party model with user ownership
- 8 API endpoints for full CRUD + character management + stats
- **Expanded DM Dashboard** with combat, senses, capabilities, equipment, party summary
- 42 tests covering all functionality

**What you need to do:**
- Party list page (`/parties`) - cards with name, description, character count
- Party detail page (`/parties/{id}`) - edit party, manage characters
- **DM Screen** (`/parties/{id}/dashboard`) - comprehensive character stats
- Add character modal - search/select from DM's characters

**API Endpoints:**
| Method | Endpoint | Purpose |
|--------|----------|---------|
| POST | `/api/v1/parties` | Create party `{ name, description? }` |
| GET | `/api/v1/parties` | List DM's parties |
| GET | `/api/v1/parties/{id}` | Get party with characters |
| PUT | `/api/v1/parties/{id}` | Update party |
| DELETE | `/api/v1/parties/{id}` | Delete party |
| POST | `/api/v1/parties/{id}/characters` | Add character `{ character_id }` |
| DELETE | `/api/v1/parties/{id}/characters/{characterId}` | Remove character |
| GET | `/api/v1/parties/{id}/stats` | **DM Screen data** |

**Stats response shape (EXPANDED):**
```json
{
  "data": {
    "party": { "id": 1, "name": "Dragon Heist" },
    "characters": [{
      "id": 1, "public_id": "...", "name": "Gandalf",
      "level": 5, "class_name": "Wizard",
      "hit_points": { "current": 28, "max": 35, "temp": 0 },
      "armor_class": 15, "proficiency_bonus": 3,
      "combat": {
        "initiative_modifier": 2,
        "speeds": { "walk": 30, "fly": null, "swim": null, "climb": null },
        "death_saves": { "successes": 0, "failures": 0 },
        "concentration": { "active": false, "spell": null }
      },
      "senses": {
        "passive_perception": 14, "passive_investigation": 12,
        "passive_insight": 14, "darkvision": 60
      },
      "capabilities": {
        "languages": ["Common", "Elvish"],
        "size": "Medium",
        "tool_proficiencies": ["Thieves' Tools"]
      },
      "equipment": {
        "armor": { "name": "Leather", "type": "light", "stealth_disadvantage": false },
        "weapons": [{ "name": "Longsword", "damage": "1d8 slashing", "range": null }],
        "shield": false
      },
      "saving_throws": { "STR": 0, "DEX": 2, "CON": 1, "INT": 4, "WIS": 2, "CHA": -1 },
      "conditions": [{ "name": "Poisoned", "slug": "poisoned", "level": null }],
      "spell_slots": { "1": { "current": 4, "max": 4 } }
    }],
    "party_summary": {
      "all_languages": ["Common", "Dwarvish", "Elvish"],
      "darkvision_count": 2,
      "no_darkvision": ["Aldric"],
      "has_healer": true,
      "healers": ["Mira (Cleric)"],
      "has_detect_magic": true,
      "has_dispel_magic": false,
      "has_counterspell": true
    }
  }
}
```

**Key UI Components:**
- Initiative tracker (quick reference for combat)
- HP bars with temp HP indicator
- Passive skills display (Perception is key for DMs!)
- Death save tracker
- Party-wide language/darkvision summary
- "Who has detect magic?" type quick checks

**Test with:**
```bash
curl "http://localhost:8080/api/v1/parties/1/stats" -H "Authorization: Bearer $TOKEN" | jq
```

**Related:**
- Frontend issue: #564
- See also: `app/Http/Controllers/Api/PartyController.php`

---

## For: frontend
**From:** backend | **Issue:** #566 | **Created:** 2025-12-13 17:55

Equipment location slots are now supported. Your inventory tab UI can integrate equip/unequip functionality.

**What I did (Issue #565):**
- Added `EquipmentLocation` enum: `main_hand`, `off_hand`, `worn`, `attuned`, `backpack`
- `PATCH /equipment/{id}` now accepts `location` parameter
- Auto-sets `equipped` and `is_attuned` based on location
- Slot enforcement with auto-unequip of previous items
- 18 new tests, all passing

**What you need to do:**
- Wire `useInventoryActions.equipItem(id, location)` to the endpoint
- Update ItemRow action menus to call equip/unequip
- EquipmentStatus sidebar should display items by slot
- Handle 422 errors (attunement limit, invalid location, etc.)

**API Contract:**
```bash
# Equip to main hand
PATCH /api/v1/characters/{id}/equipment/{equipmentId}
{"location": "main_hand"}

# Response
{
  "data": {
    "id": 176,
    "location": "main_hand",
    "equipped": true,
    "is_attuned": false
  }
}
```

**Valid locations:** `main_hand` (1), `off_hand` (1), `worn` (1), `attuned` (max 3), `backpack` (unlimited)

**Test with:**
```bash
curl -X PATCH "http://localhost:8080/api/v1/characters/1/equipment/1" \
  -H "Content-Type: application/json" \
  -d '{"location": "main_hand"}'
```

**Related:**
- Backend PR: dfox288/ledger-of-heroes-backend#146
- Frontend branch ready: `feature/issue-555-inventory-tab`

---

<!-- HANDOFF TEMPLATE (copy this when creating a new handoff):

## For: frontend|backend
**From:** backend|frontend | **Issue:** #N | **Created:** YYYY-MM-DD HH:MM

[Brief description]

**What I did:**
- [Key changes made]

**What you need to do:**
- [Specific tasks]

**Technical details:**
- Endpoint: `GET /api/v1/example`
- Request: `?filter=field=value`
- Response: `{ data: { id, name, slug } }`

**Test with:**
```bash
curl "http://localhost:8080/api/v1/example"
```

**Related:**
- Closes/Follows from: #N
- See also: `path/to/relevant/file`

---

-->
