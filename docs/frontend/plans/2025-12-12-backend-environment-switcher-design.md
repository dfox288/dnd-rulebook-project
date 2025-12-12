# Backend Environment Switcher Design

**Date:** 2025-12-12
**Status:** Approved
**Scope:** Frontend configuration + Claude Code status line

## Overview

Add environment variable-based switching between stable and bleeding-edge backend environments, enabling parallel frontend/backend development workflows.

## Problem

When the backend has breaking changes in progress, frontend development is blocked or unstable. A stable backend environment (port 8081) now exists alongside the dev environment (port 8080), but there's no easy way to switch between them.

## Solution

A named profile environment variable (`NUXT_BACKEND_ENV`) that maps to backend URLs, with visibility in the Claude Code status line.

## Configuration

### Environment Variable

**File:** `.env`

```bash
# Backend Environment: dev | stable
# - dev:    Bleeding-edge (port 8080) - active development, may break
# - stable: Stable API (port 8081) - for frontend testing against known-good backend
NUXT_BACKEND_ENV=dev
```

### Nuxt Runtime Config

**File:** `nuxt.config.ts`

```typescript
runtimeConfig: {
  // Determine backend URL from environment profile
  apiBaseServer: (() => {
    const env = process.env.NUXT_BACKEND_ENV || 'dev'
    const urls: Record<string, string> = {
      dev: 'http://host.docker.internal:8080/api/v1',
      stable: 'http://host.docker.internal:8081/api/v1'
    }
    return urls[env] || urls.dev
  })(),

  public: {
    backendEnv: process.env.NUXT_BACKEND_ENV || 'dev',
    // ... existing config
  }
}
```

### URL Mapping

| `NUXT_BACKEND_ENV` | Port | Backend |
|--------------------|------|---------|
| `dev` (default) | 8080 | Bleeding-edge development |
| `stable` | 8081 | Stable, known-good API |

## Claude Code Status Line

Add a custom command widget to show the active backend environment.

**File:** `~/.config/ccstatusline/settings.json`

Add to the first line's widget array:

```json
{
  "id": "backend-env",
  "type": "custom-command",
  "commandPath": "grep -h NUXT_BACKEND_ENV .env 2>/dev/null | cut -d= -f2 | tr '[:lower:]' '[:upper:]' || echo ''",
  "color": "blue",
  "timeout": 500
}
```

**Behavior:**
- Shows `DEV` or `STABLE` in the status line when in frontend directory
- Shows nothing when in other directories (graceful fallback)
- Updates on each status line refresh

## CLAUDE.md Documentation Update

Add new section to `CLAUDE.md`:

```markdown
## Backend Environment

Switch between backend environments using `NUXT_BACKEND_ENV`:

| Value | Port | Use Case |
|-------|------|----------|
| `dev` (default) | 8080 | Active backend development, bleeding-edge |
| `stable` | 8081 | Frontend work against stable, known-good API |

**To switch:**
1. Edit `.env`: `NUXT_BACKEND_ENV=stable`
2. Restart dev server: `docker compose restart nuxt`

**When to use stable:**
- Frontend feature work that doesn't need latest backend changes
- Testing against consistent API behavior
- Parallel work while backend has breaking changes in progress

**When to use dev:**
- Testing new backend features
- Full-stack development
- Verifying frontend/backend integration
```

## Implementation Tasks

1. Update `nuxt.config.ts` with profile-based URL mapping
2. Update `.env` with new variable (remove old `NUXT_API_BASE_SERVER`)
3. Update `CLAUDE.md` with backend environment documentation
4. Add widget to `~/.config/ccstatusline/settings.json`

## Testing

After implementation:
1. Set `NUXT_BACKEND_ENV=stable`, restart, verify API calls go to port 8081
2. Set `NUXT_BACKEND_ENV=dev`, restart, verify API calls go to port 8080
3. Remove variable entirely, verify default is `dev` (port 8080)
4. Check Claude Code status line shows correct environment

## Future Extensions

Additional environments can be added by extending the URL mapping:
- `staging` - Pre-production testing
- `production` - Production API (read-only)
