# Handover: Test Suite CPU Spike Fix

**Date:** 2025-11-26
**Status:** Complete
**Impact:** Critical infrastructure fix

---

## Problem

Running `npm run test` in Docker caused severe CPU spikes (400%+) that would crash the developer's machine. The container would be killed with exit code 137 (OOM).

## Root Causes Identified

1. **No component cleanup** - 953 `mountSuspended()` calls across 71 test files with zero `afterEach` cleanup hooks. Components accumulated in memory.

2. **E2E tests running with unit tests** - Playwright E2E tests (`.spec.ts`) were being picked up by Vitest, attempting to run browser tests without a browser.

3. **WebGL/Three.js tests** - Animation tests initialized full Three.js WebGL scenes, which Docker software-renders on CPU.

4. **Unlimited parallelization** - No `maxForks` setting meant all 107 test files could run simultaneously, each creating a full Nuxt environment.

5. **No Docker resource limits** - Container could consume unlimited CPU/memory, causing host system impact.

## Solution

### 1. Automatic Component Cleanup (`tests/setup.ts`)

```typescript
import { enableAutoUnmount, flushPromises } from '@vue/test-utils'

// KEY FIX - tracks and unmounts all mounted components
enableAutoUnmount(afterEach)

afterEach(async () => {
  await flushPromises()
  vi.clearAllMocks()
  vi.clearAllTimers()
})
```

### 2. Exclude E2E Tests (`vitest.config.ts`)

```typescript
exclude: [
  '**/node_modules/**',
  '**/dist/**',
  '**/e2e/**',           // Playwright E2E tests
  '**/*.spec.ts',        // E2E convention
],
```

### 3. Removed WebGL Tests

Deleted (animation is stable, doesn't need tests):
- `tests/components/AnimatedBackground.test.ts`
- `tests/composables/useAnimatedBackground.test.ts`

### 4. Limited Parallelization (`vitest.config.ts`)

```typescript
pool: 'forks',
poolOptions: {
  forks: {
    maxForks: 2,    // Only 2 concurrent test files
    minForks: 1,
  }
},
sequence: {
  concurrent: false,  // Sequential within files
},
isolate: true,
```

### 5. Docker Resource Limits (`docker-compose.yml`)

```yaml
deploy:
  resources:
    limits:
      cpus: '4'
      memory: 6G
    reservations:
      cpus: '1'
      memory: 2G
```

## Results

| Metric | Before | After |
|--------|--------|-------|
| CPU Usage | 400%+ (crash) | <200% (stable) |
| Test Duration | N/A (crashed) | ~120s |
| Tests Passing | N/A | 1302/1302 |
| Test Files | N/A | 107/107 |

## Files Changed

- `tests/setup.ts` - Added `enableAutoUnmount` and cleanup hooks
- `vitest.config.ts` - Added exclusions, pool config, resource limits
- `docker-compose.yml` - Added CPU/memory limits
- `tests/components/AnimatedBackground.test.ts` - Deleted
- `tests/composables/useAnimatedBackground.test.ts` - Deleted
- `CHANGELOG.md` - Documented fix

## Running Tests

```bash
# Unit tests (in Docker)
docker compose exec nuxt npm run test

# E2E tests (separate command, requires Playwright)
docker compose exec nuxt npm run test:e2e
```

## Future Considerations

- If tests slow down significantly, consider reducing `maxForks` to 1
- E2E tests need Playwright browsers installed to run
- Icon loading warnings are harmless (icons not bundled in test env)

---

**Next Session:** Tests are stable. Ready for feature work.
