# D&D Rulebook Project Coordination

Central hub for cross-cutting issues, API contracts, and coordination between frontend and backend.

## Quick Links

- **Frontend:** [dnd-rulebook-frontend](https://github.com/dfox288/dnd-rulebook-frontend)
- **Backend:** [dnd-rulebook-parser](https://github.com/dfox288/dnd-rulebook-parser)
- **Issues:** [View all issues](https://github.com/dfox288/dnd-rulebook-project/issues)

---

## How to Use

### Creating Issues

**From browser:** Click "New Issue" above

**From CLI:**
```bash
# Quick issue
gh issue create --repo dfox288/dnd-rulebook-project \
  --title "Spell components not returned from API" \
  --label "backend,bug,from:manual-testing"

# With body
gh issue create --repo dfox288/dnd-rulebook-project \
  --title "Add pagination to /spells" \
  --label "backend,feature" \
  --body "Currently returns all spells. Need limit/offset params."
```

### Checking Your Inbox

```bash
# Frontend team
gh issue list --repo dfox288/dnd-rulebook-project --label "frontend" --state open

# Backend team
gh issue list --repo dfox288/dnd-rulebook-project --label "backend" --state open

# All open issues
gh issue list --repo dfox288/dnd-rulebook-project --state open
```

### Resolving Issues

```bash
gh issue close 42 --repo dfox288/dnd-rulebook-project \
  --comment "Fixed in frontend PR #123"
```

---

## Label System

### Assignee (who should fix it)
| Label | Description |
|-------|-------------|
| `frontend` | Frontend team should handle |
| `backend` | Backend team should handle |
| `both` | Requires coordination |

### Type (what kind of issue)
| Label | Description |
|-------|-------------|
| `bug` | Something isn't working |
| `feature` | New feature or enhancement |
| `api-contract` | API design discussion |
| `data-issue` | Missing or incorrect data |
| `performance` | Performance improvement |

### Status
| Label | Description |
|-------|-------------|
| `blocked` | Waiting on dependency |
| `needs-discussion` | Needs team input |

### Source (who found it)
| Label | Description |
|-------|-------------|
| `from:manual-testing` | Found during manual testing |
| `from:frontend` | Discovered by frontend agent |
| `from:backend` | Discovered by backend agent |

---

## Workflow

1. **Anyone** discovers an issue (manual testing, agent analysis, code review)
2. **Create issue** with appropriate labels (assignee + type + source)
3. **Assignee checks inbox** at session start
4. **Work on issue**, reference in commits/PRs
5. **Close with comment** explaining resolution

---

## Issue Templates

### Bug Report
```markdown
**What's broken:**
[Description]

**Where:**
[Endpoint, component, or flow]

**Expected:**
[What should happen]

**Actual:**
[What happens instead]

**Steps to reproduce:**
1. ...
```

### API Contract Request
```markdown
**Endpoint:** `GET /api/v1/...`

**Current response:**
```json
{ ... }
```

**Requested change:**
[What you need added/changed]

**Why:**
[Frontend use case]
```
