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
**From:** backend | **Issue:** #545 | **Created:** 2024-12-13 12:00

Currency management endpoint is ready for frontend integration.

**What I did:**
- Created `PATCH /api/v1/characters/{id}/currency` endpoint
- Implemented auto-conversion logic ("making change")
- Comprehensive validation and error handling

**What you need to do:**
- Build currency management UI modal as per design doc
- Handle 422 "Insufficient funds" errors (keep modal open)
- Handle 200 success (close modal, update state)

**API Contract:**
```
PATCH /api/v1/characters/{id}/currency

Request:
{
  "gp": "-5",      // subtract 5 gold
  "sp": "+10",     // add 10 silver
  "cp": "25"       // set copper to 25
}

Success Response (200):
{
  "data": {
    "pp": 0,
    "gp": 145,
    "ep": 0,
    "sp": 40,
    "cp": 25
  }
}

Insufficient Funds (422):
{
  "message": "Insufficient funds",
  "errors": {
    "currency": ["Cannot afford this transaction. Need 50 CP but only 20 CP available after conversion."]
  }
}
```

**Test with:**
```bash
# Add 10 gold
curl -X PATCH "http://localhost:8080/api/v1/characters/{id}/currency" \
  -H "Content-Type: application/json" \
  -d '{"gp": "+10"}'

# Subtract 50 copper (will auto-convert if needed)
curl -X PATCH "http://localhost:8080/api/v1/characters/{id}/currency" \
  -H "Content-Type: application/json" \
  -d '{"cp": "-50"}'
```

**Related:**
- PR: https://github.com/dfox288/ledger-of-heroes-backend/pull/142
- Design doc: `docs/frontend/plans/2025-12-13-currency-management-design.md`

---

## For: frontend
**From:** backend | **Issue:** #548 | **Created:** 2024-12-13 14:30

Character revival endpoint is ready. Fixes the bug where "Revive" button didn't work properly when character died from exhaustion level 6.

**What I did:**
- Created `POST /api/v1/characters/{id}/revive` endpoint
- Atomic operation: sets `is_dead=false`, resets death saves, sets HP, clears exhaustion
- All in one request - no more partial state bugs

**What you need to do:**
- Replace current revive logic with single API call
- Handle 422 error if character isn't dead
- Optional: Add modal for different revival spell types (Revivify vs True Resurrection)

**API Contract:**
```
POST /api/v1/characters/{id}/revive

Request (all optional):
{
  "hit_points": 1,           // Default: 1, capped at max HP
  "clear_exhaustion": true,  // Default: true
  "source": "Revivify"       // For future audit trail (not stored yet)
}

Success Response (200): Full CharacterResource

Already Alive (422):
{
  "message": "The given data was invalid.",
  "errors": {
    "character": ["Character is not dead"]
  }
}
```

**Test with:**
```bash
# Kill character first
curl -X PATCH "http://localhost:8080/api/v1/characters/{id}" \
  -H "Content-Type: application/json" \
  -d '{"is_dead": true, "current_hit_points": 0}'

# Revive with defaults (1 HP, clear exhaustion)
curl -X POST "http://localhost:8080/api/v1/characters/{id}/revive"

# Revive with full HP (True Resurrection style)
curl -X POST "http://localhost:8080/api/v1/characters/{id}/revive" \
  -H "Content-Type: application/json" \
  -d '{"hit_points": 999}'
```

**Related:**
- Enhancement to #543 (is_dead flag)
- See also: `app/Http/Controllers/Api/CharacterReviveController.php`

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
