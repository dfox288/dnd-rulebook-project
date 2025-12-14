# Party Management UI Design

**Issue:** #560
**Date:** 2025-12-14
**Status:** Approved

## Overview

Frontend UI for DMs to create, manage, and view parties. CRUD-first approach - party list and detail pages now, DM Dashboard in a future phase.

## Scope

### In Scope (This Phase)
- Party list page with card grid
- Party detail page with character management
- Create/edit party modal
- Add character modal with search

### Out of Scope (Future)
- DM Dashboard (`/parties/{id}/dashboard`)
- Stats endpoint integration
- Real-time updates

## Routes

| Route | Page | Purpose |
|-------|------|---------|
| `/parties` | PartyListPage | Card grid of user's parties |
| `/parties/[id]` | PartyDetailPage | Party info + character management |
| `/parties/[id]/dashboard` | (future) | DM Dashboard with stats |

## Nitro API Routes

```
server/api/parties/
├── index.get.ts              # GET /api/parties
├── index.post.ts             # POST /api/parties
├── [id].get.ts               # GET /api/parties/:id
├── [id].put.ts               # PUT /api/parties/:id
├── [id].delete.ts            # DELETE /api/parties/:id
└── [id]/
    └── characters/
        ├── index.post.ts     # POST /api/parties/:id/characters
        └── [characterId].delete.ts  # DELETE
```

## Components

```
app/components/party/
├── Card.vue              # Party card for list page
├── CreateModal.vue       # Create/edit party modal
├── CharacterList.vue     # Character list on detail page
└── AddCharacterModal.vue # Search & add characters modal
```

## Page Designs

### Party List Page (`/parties`)

**Layout:**
- Page header: "My Parties" + "New Party" button
- Card grid: 1 col mobile, 2 col tablet, 3 col desktop
- Empty state when no parties

**Party Card:**
```
┌─────────────────────────────────┐
│ Dragon Heist Campaign        ⋮  │  ← Name + actions dropdown
│ Weekly Thursday game            │  ← Description (truncated)
│                                 │
│ 👤👤👤👤 +2                      │  ← Character avatars (max 4)
│ 6 characters                    │  ← Count
└─────────────────────────────────┘
```

**Interactions:**
- Click card → navigate to `/parties/{id}` (detail page)
- Actions dropdown: Edit Party, Delete Party, View Dashboard (future/disabled)

**Empty State:**
- "No parties yet"
- "Create your first party to start tracking your adventuring group"
- Primary "New Party" button

### Create/Edit Modal

**Fields:**
- Name (required, text input)
- Description (optional, textarea)

**Behavior:**
- Create mode: "New Party" title, "Create" button
- Edit mode: "Edit Party" title, "Save" button
- On create success: redirect to `/parties/{id}`
- On edit success: close modal, refresh data

### Party Detail Page (`/parties/[id]`)

**Layout:**
```
┌─────────────────────────────────────────────────────┐
│ ← Back to Parties                                   │
├─────────────────────────────────────────────────────┤
│ Dragon Heist Campaign                    [Edit] [⋮] │
│ Weekly Thursday game with the crew                  │
├─────────────────────────────────────────────────────┤
│ Characters (6)                    [+ Add Character] │
├─────────────────────────────────────────────────────┤
│ ┌─────────┐ ┌─────────┐ ┌─────────┐                │
│ │ Thorin  │ │ Elara   │ │ Grimble │                │
│ │ Fighter │ │ Wizard  │ │ Rogue   │                │
│ │ Lvl 5   │ │ Lvl 5   │ │ Lvl 4   │                │
│ │    [×]  │ │    [×]  │ │    [×]  │                │
│ └─────────┘ └─────────┘ └─────────┘                │
└─────────────────────────────────────────────────────┘
```

**Header Actions:**
- Edit button → Opens CreateModal in edit mode
- Dropdown: Delete Party (with confirmation), View Dashboard (future)

**Character Cards:**
- Portrait/avatar, name, class, level
- Remove button (×) with confirmation
- Click → navigate to character sheet

**Empty State:**
- "No characters in this party yet"
- "Add characters to track their stats together"
- Primary "Add Character" button

### Add Character Modal

**Layout:**
```
┌─────────────────────────────────────────────────────┐
│ Add Characters to Party                         [×] │
├─────────────────────────────────────────────────────┤
│ [🔍 Search characters...                          ] │
├─────────────────────────────────────────────────────┤
│ ☑ Thorin Ironforge         Fighter 5              │
│   └─ Already in: Waterdeep Campaign               │
│ ☐ Elara Moonwhisper        Wizard 5               │
│ ☑ Grimble Thornfoot        Rogue 4                │
├─────────────────────────────────────────────────────┤
│                              [Cancel] [Add Selected]│
└─────────────────────────────────────────────────────┘
```

**Behavior:**
- Fetches user's characters on open
- Filters out characters already in THIS party
- Search filters by name (client-side)
- Multi-select with checkboxes
- Characters in other parties shown with indicator (not blocked)
- "Add Selected" adds all checked, closes modal, refreshes list
- Toast: "Added 3 characters to party"

## Types

```typescript
interface Party {
  id: number
  name: string
  description: string | null
  character_count: number
  characters?: PartyCharacter[]
  created_at: string
}

interface PartyCharacter {
  id: number
  public_id: string
  name: string
  class_name: string
  level: number
  portrait: { thumb: string } | null
  parties?: { id: number; name: string }[]
}
```

## Error Handling

| Scenario | Handling |
|----------|----------|
| 404 on party detail | Redirect to `/parties` with error toast |
| Create/update validation | Inline error in modal |
| Delete failure | Toast error |
| Add character failure | Toast error, keep modal open |

## Test Coverage

| Component | Tests |
|-----------|-------|
| `PartyCard.vue` | Renders name, description, avatars, actions |
| `CreateModal.vue` | Validation, create/edit modes, submit |
| `AddCharacterModal.vue` | Search, multi-select, party indicator |
| `CharacterList.vue` | Renders characters, remove action |
| List page | Load parties, empty state, create flow |
| Detail page | Load party, character management |
