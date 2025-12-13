# Inventory Tab Redesign

**Issue:** #555
**Created:** 2025-12-13
**Status:** Design Complete

## Overview

Redesign the character inventory page to improve usability, scannability, and visual clarity. Replace the current expandable list with a grouped table layout.

## Problems with Current Design

1. **Sidebar**: Shows 3 "Empty" slots for attuned items unnecessarily
2. **Currency**: Buried in sidebar, should be more prominent
3. **Item list**: Too wide, quantities not clearly visible (`×N` on far right)
4. **Item expansion**: Shows raw JSON for description
5. **Equip buttons**: Shown for ALL items, even non-equippable ones; duplicated in expanded view AND dropdown menu
6. **Action buttons**: Add Loot / Shop buried at bottom of page

## Design Decisions

### Layout

```
┌─────────────────────────────────────────────────────────────────┐
│ CharacterPageHeader (existing)                                  │
├───────────────────────────────────────────┬─────────────────────┤
│  Items (12)           [+ Add Loot] [Shop] │  Currency           │
│  [Search items...]                        │  Wielded            │
│  ▼ Weapons (3)                            │  Armor              │
│    │ Longsword           Main Hand        │  Attuned (1/3)      │
│  ▼ Armor (2)                              │  Encumbrance        │
│  ▼ Consumables (5)                        │                     │
│  ▼ Gear (2)                               │                     │
└───────────────────────────────────────────┴─────────────────────┘
```

- Two-column layout: Item table (left), Sidebar (right)
- Header row with title + action buttons
- Search on dedicated row below header
- Sidebar reordered: Currency at top, no empty attuned slots

### Item Table

| Qty | Icon | Name | Location | Actions |
|-----|------|------|----------|---------|
| 3×  | 🧪   | Potion of Healing | — | [−][+] [⋮] |
|     | ⚔️   | Longsword | Main Hand | [Unequip] [⋮] |

**Columns:**
- **Qty**: `N×` format on left (blank if 1)
- **Icon**: Visual type indicator
- **Name**: Clickable → opens detail modal
- **Location**: Badge if equipped, dash if not
- **Actions**: Inline buttons + overflow menu

**No weight column** — visible in detail modal instead.

**Row Styling:**
- Equipped rows: Left border accent (primary color)
- Click anywhere: Opens detail modal

### Item Grouping

Items grouped by type with collapsible section headers:
1. Weapons
2. Armor
3. Consumables
4. Gear
5. Miscellaneous (fallback)

**Backend provides `group` field** — no frontend mapping logic.

Empty groups are hidden. Search filters across all groups.

### Actions (Play Mode Only)

**Inline in table row:**
- `[Equip]` / `[Unequip]` — only for equippable items (weapons, armor, shields, attuneable)
- `[−][+]` — quantity buttons for stackable items (qty > 1)
- `[⋮]` — overflow menu: Sell, Drop, Edit Qty

**Smart auto-slot for Equip:**
- Weapon → Main Hand (prompt if off-hand available)
- Armor → Worn
- Shield → Off Hand
- Magic w/ attunement → Attuned

### Item Detail Modal

Read-only modal triggered by clicking item name/row:

```
┌────────────────────────────────────────────┐
│ Longsword                              [×] │
│ [Weapon] [Common]                          │
├────────────────────────────────────────────┤
│ Weight: 3 lb  •  Value: 15 gp  •  Qty: 1   │
├────────────────────────────────────────────┤
│ A versatile martial weapon favored by...   │
├────────────────────────────────────────────┤
│ Properties: Versatile (1d10), Martial      │
│ Damage: 1d8 slashing                       │
└────────────────────────────────────────────┘
```

**No action buttons** — actions are in table row only.

### Sidebar

```
┌─────────────────────────────┐
│ Currency            [edit]  │  ← Moved to top
│ 150 GP  23 SP  8 CP  0 EP   │
├─────────────────────────────┤
│ Wielded                     │
│ Main Hand: Longsword        │
│ Off Hand: Shield            │
├─────────────────────────────┤
│ Armor                       │
│ Chain Mail (AC 16)          │
├─────────────────────────────┤
│ Attuned (1/3)               │  ← Count in header
│ Ring of Protection          │  ← Only actual items, no empties
├─────────────────────────────┤
│ Encumbrance                 │
│ [████████░░░] 85/150 lb     │
└─────────────────────────────┘
```

**Changes:**
- Currency at top (most frequently checked)
- Attuned: Shows count in header, lists only actual items
- No "Empty" slot placeholders for attuned
- Clicking equipped items scrolls to them in table

## Files Changed

**Modify:**
| File | Changes |
|------|---------|
| `inventory.vue` | New layout, header with buttons, sidebar reorder |
| `ItemList.vue` | Replace with grouped table |
| `EquipmentStatus.vue` | Reorder sections, remove empty attuned slots |

**Delete:**
| File | Reason |
|------|--------|
| `ItemRow.vue` | Replaced by table rows |

**Create:**
| File | Purpose |
|------|---------|
| `ItemTable.vue` | Grouped table with collapsible sections |
| `ItemDetailModal.vue` | Read-only item details modal |

## Backend Requirements

**New Issue Required:**
- Add `group` field to equipment endpoint response
- Values: `Weapons`, `Armor`, `Consumables`, `Gear`, `Miscellaneous`

## Out of Scope

- Sell modal implementation (existing stub remains)
- Edit Qty modal implementation (existing stub remains)
- Shop modal improvements
- Add Loot modal improvements

These will be addressed in future sessions.

## Testing Strategy

- Unit tests for new components (ItemTable, ItemDetailModal)
- Update existing inventory page tests
- Test grouping with various item type combinations
- Test equipped row highlighting
- Test action button visibility (equippable vs non-equippable)
