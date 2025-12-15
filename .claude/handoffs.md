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
**From:** backend | **Issue:** #647 | **Created:** 2025-12-15 18:30

Class counters API is ready! Characters now expose Rage, Ki Points, Sorcery Points, etc.

**IMPORTANT NAMING CHANGE:** We're using `counters` instead of `class_resources` - aligns with existing `class_counters` table.

**What I did:**
- Added `counters` array to character response
- Created dedicated counter endpoints for listing and updating
- Counters auto-initialize from class features when characters are created
- Integrated with existing short/long rest mechanics

**What you need to do:**
- Update character sheet to display `counters` array (not `class_resources`)
- Add use/restore controls for each counter
- Implement DM Screen resource tracker using same data

**API contract:**
| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/characters/{id}` | Includes `counters` array in response |
| GET | `/characters/{id}/counters` | List all counters |
| PATCH | `/characters/{id}/counters/{slug}` | Update counter |

**Response shape (in character response):**
```json
{
  "counters": [
    {
      "id": 123,
      "slug": "phb:barbarian:rage",
      "name": "Rage",
      "current": 2,
      "max": 3,
      "reset_on": "long_rest",
      "source": "Barbarian",
      "source_type": "class",
      "unlimited": false
    }
  ]
}
```

**Update counter:**
```bash
# Use action (increment/decrement)
curl -X PATCH "http://localhost:8080/api/v1/characters/1/counters/phb:barbarian:rage" \
  -H "Content-Type: application/json" \
  -d '{"action": "use"}'   # or "restore" or "reset"

# Set absolute spent value
curl -X PATCH "http://localhost:8080/api/v1/characters/1/counters/phb:barbarian:rage" \
  -H "Content-Type: application/json" \
  -d '{"spent": 2}'   # sets current = max - spent
```

**Test with:**
```bash
curl "http://localhost:8080/api/v1/characters/1" | jq '.data.counters'
```

**Related:**
- Closes: #647 (Backend class resources API)
- Unblocks: #632 (Frontend class resource counters)
- Also enables: #606 (DM Screen resource tracker)
- Note: `/feature-uses` endpoint deprecated, use `/counters`

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
