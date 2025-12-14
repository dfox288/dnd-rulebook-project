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
**From:** backend | **Issue:** #589 | **Created:** 2025-12-14

Backend is implementing `equipment_slot` field on items. **Please add `clothes` slot to paperdoll design.**

**What I'm doing (Issue #589):**
- Adding `CLOTHES` case to `EquipmentLocation` enum
- Adding `equipment_slot` column to items table
- Auto-inferring slots from item type and name patterns
- Re-importing all items with slot data

**Slots (13 total):**
```
main_hand, off_hand, head, neck, cloak, armor, clothes, belt, hands, feet, ring_1, ring_2, backpack
```

**NEW: `clothes` slot** for magical clothing that isn't armor:
- Robes (Robe of Eyes, Robe of the Archmagi, etc.)
- Clothes of Mending
- Glamerweave
- Shiftweave

**Auto-assignment by item type:**
| Item Type | Slot |
|-----------|------|
| Light/Medium/Heavy Armor | `armor` |
| Shield | `off_hand` |
| Ring | `ring` |
| Melee/Ranged Weapon, Staff, Rod, Wand | `hand` |

**Wondrous Items pattern matching:**
| Pattern | Slot |
|---------|------|
| Boot, Slipper | `feet` |
| Cloak, Cape, Mantle, Wings | `cloak` |
| Belt, Girdle | `belt` |
| Helm, Hat, Circlet, Crown, Headband, Cap, Goggles, Eyes of | `head` |
| Amulet, Necklace, Periapt, Medallion, Brooch, Scarab, Talisman, Scarf | `neck` |
| Glove, Gauntlet, Bracer | `hands` |
| Robe, Clothes, Glamerweave, Shiftweave | `clothes` |

**Coverage:**
- **1,679 items (67%)** get automatic slot assignment
- **267 Wondrous Items** remain `null` (tattoos, figurines, bags, etc. - correctly have no body slot)
- **562 other items** remain `null` (potions, scrolls, gear)

**What you need to do:**
1. Add `clothes` slot to paperdoll (between `armor` and `belt`)
2. Update `equipmentSlots.ts` utility with new slot
3. Items with `equipment_slot` set can skip the slot picker modal
4. Items with `equipment_slot = null` still need manual picker

**Response shape update:**
```json
{
  "data": {
    "id": 1,
    "slug": "phb:cloak-of-protection",
    "name": "Cloak of Protection",
    "equipment_slot": "cloak",
    ...
  }
}
```

**Related:**
- Frontend issue: #587 (paperdoll UI)
- Design doc needs update: `wrapper/docs/frontend/plans/2025-12-13-equipment-paperdoll-design.md`

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
