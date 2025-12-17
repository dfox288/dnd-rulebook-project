# API Schema Validator Design

**Date:** 2025-12-17
**Status:** Approved
**Issue:** [#748](https://github.com/dfox288/ledger-of-heroes/issues/748)

## Problem

Frontend reported that `EntityChoiceResource` doesn't appear in the OpenAPI docs, but backend claimed it exists. Investigation revealed the docs are accurate - the API simply doesn't expose `choices` on entity resources yet.

This highlighted a gap: we have no automated way to verify that API responses match their documented schemas. Drift can occur when:
- Fields are added to Resources but not documented properly
- Documentation promises fields that aren't actually returned
- Refactoring removes fields without updating docs

## Solution

An Artisan command that validates actual API responses against the OpenAPI spec, detecting schema drift.

## Command Interface

```bash
php artisan api:validate-schema [options]

Options:
  --entity=ENTITY     Entity type to validate (races, classes, backgrounds, feats)
  --slug=SLUG         Specific record slug to test
  --wizard            Test curated wizard endpoint set (default)
  --all               Test all entities and records
  --stop-on-first     Stop at first drift detected
  --json              Output as JSON instead of human-readable
```

**Justfile recipe:**
```just
api-validate *ARGS:
    docker compose exec php php artisan api:validate-schema {{ARGS}}
```

## Validation Logic

### What We Check
- **Missing fields**: Schema documents a field, but response doesn't include it
- **Extra fields**: Response includes a field not in the schema

### What We Don't Check
- Type mismatches (string vs int, etc.)
- Business logic correctness
- Missing endpoints

### Field Comparison

```
Documented schema: {id, slug, name, choices, modifiers}
Actual response:   {id, slug, name, modifiers, tags}

Result:
  Missing: choices
  Extra:   tags
```

### Edge Cases

**Conditional fields (`whenLoaded`):**
- Scramble marks these as nullable
- Nullable fields are optional - don't flag as "missing" if absent

**`$ref` schemas:**
- Resolve references before comparison
- Cache resolved schemas

**Pagination:**
- Validate `data` array items against schema
- Skip `links`/`meta` (standard Laravel pagination)

## Default Test Set (`--wizard`)

Curated records that exercise wizard flows:

| Entity | Slugs | Why |
|--------|-------|-----|
| Races | `phb:half-elf`, `phb:human`, `phb:dragonborn` | Choices, subraces, variants |
| Classes | `phb:fighter`, `phb:wizard`, `phb:cleric` | Martial, caster, domain spells |
| Backgrounds | `phb:acolyte`, `phb:criminal` | Languages, proficiencies, equipment |
| Feats | `phb:lucky`, `phb:magic-initiate` | Simple, spell choices |

## Output Format

### Human-readable (default)

```
API Schema Validation
=====================
Testing wizard endpoint set (12 endpoints)

✅ GET /v1/races
   Tested: 3 records

✅ GET /v1/races/phb:human
   Fields: 15 documented, 15 returned

❌ GET /v1/races/phb:half-elf
   Missing from response:
     - choices
   Extra in response:
     (none)

❌ GET /v1/feats/phb:magic-initiate
   Missing from response:
     - choices
     - spell_choices
   Extra in response:
     (none)

✅ GET /v1/classes/phb:fighter
   Fields: 22 documented, 22 returned

─────────────────────────────────
Summary: 10 passed, 2 failed
Exit code: 1
```

### JSON (`--json`)

```json
{
  "passed": false,
  "summary": {"total": 12, "passed": 10, "failed": 2},
  "results": [
    {"endpoint": "GET /v1/races/phb:half-elf", "passed": false, "missing": ["choices"], "extra": []},
    ...
  ]
}
```

## Exit Codes

- `0` - All endpoints match their schemas
- `1` - Drift detected

## Implementation Structure

```
app/Console/Commands/
  ValidateApiSchemaCommand.php    # Main command

app/Services/ApiSchemaValidator/
  SchemaValidator.php             # Core comparison logic
  OpenApiParser.php               # Extracts schemas from spec
  ValidationResult.php            # DTO for endpoint results
  FieldDiff.php                   # DTO for missing/extra fields
```

## Data Source

Uses real database data - no factories or test data generation. This validates against production-like data.

## Authentication

Not required. Entity lookup endpoints are public.

## Future Enhancements

- CI integration (run on PR)
- Type mismatch detection (opt-in)
- Coverage report (which endpoints lack validation)
- Auto-generate issues for detected drift
