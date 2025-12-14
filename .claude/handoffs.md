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
**From:** backend | **Issue:** #616 | **Created:** 2025-12-14 20:00

Spell slot tracking and preparation toggle endpoints are now available (PR #170).

**What I did:**
- Implemented `PATCH /characters/{id}/spells/{characterSpellId}` - preparation toggle
- Implemented `PATCH /characters/{id}/spell-slots/{level}` - slot usage management
- Long rest already resets spell slots (no changes needed)
- 29 new tests covering all edge cases

**New endpoints:**

### Toggle Spell Preparation
```bash
# Prepare a spell (use characterSpell.id, not spell.id)
curl -X PATCH "http://localhost:8080/api/v1/characters/{id}/spells/42" \
  -H "Content-Type: application/json" \
  -d '{"is_prepared": true}'

# Response includes full CharacterSpell resource
```

### Update Spell Slot Usage
```bash
# Action-based (recommended for UI clicks)
curl -X PATCH "http://localhost:8080/api/v1/characters/{id}/spell-slots/1" \
  -H "Content-Type: application/json" \
  -d '{"action": "use"}'      # or "restore" or "reset"

# Absolute value (for drag/drop interfaces)
curl -X PATCH "http://localhost:8080/api/v1/characters/{id}/spell-slots/1" \
  -H "Content-Type: application/json" \
  -d '{"spent": 2}'

# For Warlocks
curl -X PATCH "http://localhost:8080/api/v1/characters/{id}/spell-slots/3" \
  -H "Content-Type: application/json" \
  -d '{"action": "use", "slot_type": "pact_magic"}'

# Response: { "data": { "level": 1, "total": 4, "spent": 2, "available": 2, "slot_type": "standard" } }
```

**Error codes:**
- 422: Validation errors (limit reached, always-prepared, no slots available, spent > total)
- 404: Non-existent character spell or spell level

**Related:**
- Backend PR: dfox288/ledger-of-heroes-backend#170
- Frontend issue: #616
- Original request: #612

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
