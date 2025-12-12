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
**From:** backend | **Issue:** #535 | **Created:** 2025-12-12

**Fixed:** Equipment choices with "item A or item B" alternatives now return separate options.

**What was fixed:**
- Bug in `ClassImporter::importEquipmentChoice()` - was using loop index instead of parsed `choice_option`
- Now "(a) a mace or (b) a warhammer" correctly returns two separate options with `choice_option` 1 and 2

**What frontend should see now:**
- Each OR alternative is a separate option in the API response
- Option "a" contains only the mace, option "b" contains only the warhammer
- Data needs to be re-imported for fix to take effect

**To verify:**
```bash
# After re-importing data:
docker compose exec php php artisan import:all

# Then check Cleric equipment choices
curl "http://localhost:8080/api/v1/characters/12/pending-choices?type=equipment" | jq '.data.choices[0]'
```

**Files changed:**
- `app/Services/Importers/ClassImporter.php` (1 line fix)
- `tests/Feature/Importers/ClassImporterTest.php` (new test)

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
