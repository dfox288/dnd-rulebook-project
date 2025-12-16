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
**From:** backend | **Issue:** #692 | **Created:** 2025-12-16 11:00 | **Updated:** 2025-12-16 12:35

### ✅ PR #191 MERGED - Multiclass spellcasting API is live!

**What's done:**
- `class_slug` field added to `CharacterSpellResource`
- `spellcasting` in stats endpoint is now per-class keyed object
- Migration for `class_slug` column deployed

**API is ready:**
```bash
# Stats endpoint shows per-class spellcasting
curl "http://localhost:8080/api/v1/characters/{id}/stats"

# Spells endpoint shows class_slug
curl "http://localhost:8080/api/v1/characters/{id}/spells"
```

**BREAKING CHANGE:**
- Old: `stats.spellcasting.ability`
- New: `stats.spellcasting[classSlug].ability`

### ⏳ Test Character Blocked on #714

We investigated the wizard flow - it doesn't support creating specific multiclass combinations. Created issue #714 to add a `test:multiclass-combinations` command.

**Workaround options:**
1. Wait for #714 implementation
2. Use `--chaos` mode and hope for the right combination
3. Manual creation via tinker (tedious but possible)

**Related:**
- Backend issue: #692 (CLOSED)
- New issue: #714 (multiclass test character command)
- Frontend issue: #631

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
