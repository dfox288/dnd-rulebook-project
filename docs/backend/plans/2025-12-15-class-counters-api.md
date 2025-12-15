# Implementation Plan: Class Counters API (#647)

**Issue:** [#647 - Add class resources API (Rage, Ki, Sorcery Points, etc.)](https://github.com/dfox288/ledger-of-heroes/issues/647)
**Created:** 2025-12-15
**Status:** Ready for implementation

## Overview

Add class counter tracking (Rage, Ki Points, Sorcery Points, etc.) to the character API. Counters track limited-use features that reset on short/long rest.

## Existing Infrastructure

| Component | Status | Notes |
|-----------|--------|-------|
| `class_counters` table | ✅ Exists | 146 counter types with level progression |
| `CharacterFeature.uses_remaining/max_uses` | ✅ Columns exist | Need to ensure population |
| `HasLimitedUses` trait | ✅ Exists | use/reset methods |
| `FeatureUseService` | ✅ Exists | Needs enhancement for counters endpoint |
| `FeatureUseController` | ✅ Exists | Rename to `/counters` |
| `RestService` integration | ✅ Exists | Already resets features |

## Design Decisions

1. **Naming:** Use "counters" (aligns with `class_counters` table)
2. **Endpoint:** Rename `/feature-uses` → `/counters` (keep alias for deprecation)
3. **Slugs:** ID-agnostic format `{source}:{class}:{counter-name}` (e.g., `phb:barbarian:rage`)
4. **Sources:** Class, subclass, and feat counters all included

## API Contract

### Response Shape (in CharacterResource)

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

### Endpoints

```
GET  /api/v1/characters/{id}/counters
     → List all counters with uses

PATCH /api/v1/characters/{id}/counters/{slug}
     Body: { "spent": 2 }           // absolute value
     Body: { "action": "use" }      // decrement by 1
     Body: { "action": "restore" }  // increment by 1
     Body: { "action": "reset" }    // reset to max
```

---

## Implementation Tasks

### Phase 1: Audit & Foundation

- [ ] **1.1** Audit `FeatureUseService::initializeUsesForFeature()` call path
  - Trace from `CharacterFeatureService` when features are granted
  - Identify why `max_uses` is not being populated

- [ ] **1.2** Add feat counter support to `ClassCounter` lookup
  - `getCounterValueForFeature()` currently only checks `class_id`
  - Add support for `feat_id` when feature is a Feat

### Phase 2: Service Layer

- [ ] **2.1** Add `getCountersForCharacter(Character): Collection` to `FeatureUseService`
  - Return: `[{slug, name, current, max, reset_on, source, source_type, unlimited}]`
  - Generate slug as `{source}:{class-slug}:{counter-name-slug}`
  - Handle unlimited counters (`counter_value = -1`)

- [ ] **2.2** Add `getCounterBySlug(Character, string): ?array` method
  - For PATCH endpoint to find counter by slug

- [ ] **2.3** Add `updateCounterBySlug(Character, string, int|string): bool` method
  - Handle both absolute `spent` and `action` modes

### Phase 3: API Layer

- [ ] **3.1** Rename route `/feature-uses` → `/counters`
  - Update `routes/api.php`
  - Keep `/feature-uses` as deprecated alias
  - Rename controller methods for clarity

- [ ] **3.2** Add `counters` to `CharacterResource`
  - Add `'counters' => $this->getCounters()`
  - Eager load features with `max_uses` in controller

- [ ] **3.3** Create `CounterResource` for response formatting
  - Consistent shape across endpoints

- [ ] **3.4** Update PATCH endpoint for slug-based routing
  - `PATCH /characters/{id}/counters/{slug}`
  - Create `UpdateCounterRequest` form request

### Phase 4: Tests (TDD)

- [ ] **4.1** `tests/Unit/Services/FeatureUseServiceCountersTest.php`
  ```php
  - it returns empty for character without counters
  - it returns rage counter for barbarian
  - it returns ki points scaling with monk level
  - it returns feat counters (Lucky)
  - it generates correct slug format
  - it marks unlimited counters correctly (-1 → unlimited: true)
  ```

- [ ] **4.2** `tests/Feature/Api/CharacterCounterApiTest.php`
  ```php
  - it includes counters in character response
  - it lists counters via GET /counters
  - it uses counter via PATCH action=use
  - it restores counter via PATCH action=restore
  - it resets counter via PATCH action=reset
  - it sets absolute value via PATCH spent=N
  - it rejects use when no uses remaining
  - it rejects spent exceeding max
  ```

- [ ] **4.3** `tests/Feature/Api/CharacterCounterRestTest.php`
  ```php
  - it resets short_rest counters on short rest
  - it resets long_rest counters on long rest
  - it does not reset long_rest counters on short rest
  ```

- [ ] **4.4** `tests/Feature/Api/CharacterCounterMulticlassTest.php`
  ```php
  - it shows counters from all classes
  - it calculates max from class-specific level
  - it includes subclass counters
  ```

### Phase 5: Integration

- [ ] **5.1** Ensure `max_uses` is initialized on feature grant
  - Find call site in `CharacterFeatureService`
  - Verify `initializeUsesForFeature()` is called

- [ ] **5.2** Add counters to rest result response
  - `RestService::shortRest()` → include `counters_reset: [names]`
  - `RestService::longRest()` → include `counters_reset: [names]`

### Phase 6: Quality & Documentation

- [ ] **6.1** Run quality gates
  ```bash
  docker compose exec php ./vendor/bin/pint
  docker compose exec php ./vendor/bin/pest --testsuite=Unit-DB
  docker compose exec php ./vendor/bin/pest --testsuite=Feature-DB
  ```

- [ ] **6.2** Update Controller PHPDoc for OpenAPI
  - Document new endpoints with `@queryParam`, `@response`

- [ ] **6.3** Update `CHANGELOG.md`

- [ ] **6.4** Create frontend handoff
  - Note endpoint rename: `/feature-uses` → `/counters`
  - Document full API contract

---

## Acceptance Criteria

From issue #647:

- [ ] Character response includes `counters` array (empty if no counters)
- [ ] Each counter has: slug, name, current, max, reset_on, source, unlimited
- [ ] PATCH endpoint to update current value (spent or action)
- [ ] Counters reset appropriately on short/long rest
- [ ] Multiclass characters get counters from all classes
- [ ] Counter max values scale with class level (not total level)
- [ ] Feat-based counters included (Lucky, etc.)

---

## Related Issues

- Blocks: #632 (Frontend class resource counters)
- Related: #606 (DM Screen resource tracker)
- Related: #607 (Party stats endpoint)
