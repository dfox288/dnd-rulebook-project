# Racial Trait Limited-Use Tracking

**Issue:** #717
**Date:** 2025-12-16
**Status:** Design Complete

## Summary

Add limited-use tracking for racial traits like Breath Weapon and Drow Magic. This involves making the counter system fully polymorphic and extending it to support racial traits and innate spellcasting.

## Scope

### In Scope
- Direct traits with uses (Breath Weapon - 1/short rest)
- Innate spellcasting traits (Drow Magic - spells with 1/long rest)
- Targeted text parsing for usage patterns (expandable later)
- Full polymorphic refactor of `class_counters` → `entity_counters`

### Out of Scope
- Comprehensive NLP parsing of all possible usage phrasings
- Item charges (can be added later with same pattern)

## Key Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Counter storage | Polymorphic `entity_counters` table | Cleaner architecture, extensible |
| Column naming | `reference_type`/`reference_id` | Match existing `entity_*` tables |
| Table rename | `class_counters` → `entity_counters` | Accurate naming for polymorphic table |
| Innate spell tracking | On `character_spells` pivot | Keep spell tracking with spells |
| Text parsing | Targeted patterns with room to grow | Pragmatic - handle known cases first |

## Database Schema Changes

### 1. Rename and restructure `class_counters` → `entity_counters`

```php
// Migration: rename + make polymorphic
Schema::rename('class_counters', 'entity_counters');

Schema::table('entity_counters', function (Blueprint $table) {
    // Add polymorphic columns
    $table->string('reference_type', 255)->after('id');
    $table->unsignedBigInteger('reference_id')->after('reference_type');

    // Index for polymorphic queries
    $table->index(['reference_type', 'reference_id']);
});

// Migrate existing data:
// - class_id NOT NULL → reference_type='App\Models\CharacterClass', reference_id=class_id
// - feat_id NOT NULL → reference_type='App\Models\Feat', reference_id=feat_id

// Drop old FK columns after data migration
Schema::table('entity_counters', function (Blueprint $table) {
    $table->dropForeign(['class_id']);
    $table->dropForeign(['feat_id']);
    $table->dropColumn(['class_id', 'feat_id']);
});
```

### 2. Add columns to `character_spells` (for innate spellcasting)

```php
Schema::table('character_spells', function (Blueprint $table) {
    $table->smallInteger('max_uses')->nullable()->after('preparation_status');
    $table->smallInteger('uses_remaining')->nullable()->after('max_uses');
    $table->string('resets_on', 20)->nullable()->after('uses_remaining');
});
```

## Model Changes

### `EntityCounter` (renamed from `ClassCounter`)

```php
class EntityCounter extends BaseModel
{
    protected $table = 'entity_counters';

    protected $fillable = [
        'reference_type',
        'reference_id',
        'level',
        'counter_name',
        'counter_value',
        'reset_timing',
    ];

    public function reference(): MorphTo
    {
        return $this->morphTo();
    }
}
```

### Update relationship in `CharacterClass`, `Feat`, `CharacterTrait`

```php
public function counters(): MorphMany
{
    return $this->morphMany(EntityCounter::class, 'reference');
}
```

### Update `CharacterSpell`

```php
// Add to $fillable
'max_uses', 'uses_remaining', 'resets_on'

// Add to $casts
'max_uses' => 'integer',
'uses_remaining' => 'integer',
'resets_on' => ResetTiming::class,

// Add trait
use HasLimitedUses;
```

## Parser Changes

### New trait: `ParsesUsageLimits`

```php
trait ParsesUsageLimits
{
    /**
     * Extract usage limits from trait description text.
     *
     * NOTE: Targeted patterns only. May need expansion for edge cases.
     */
    protected function parseUsageLimits(string $text): array
    {
        $maxUses = null;
        $resetsOn = null;

        // Pattern: "you can't use it again until you complete a short or long rest"
        if (preg_match('/can\'t use it again until you (?:complete|finish) a (short|long|short or long) rest/i', $text)) {
            $maxUses = 1;
            $resetsOn = str_contains(strtolower($text), 'short or long') || str_contains(strtolower($text), 'short rest')
                ? ResetTiming::SHORT_REST
                : ResetTiming::LONG_REST;
        }

        // Pattern: "once per day" or "once per long rest"
        if (preg_match('/once per (day|long rest)/i', $text, $matches)) {
            $maxUses = 1;
            $resetsOn = ResetTiming::LONG_REST;
        }

        // Pattern: "X times per long/short rest"
        if (preg_match('/(\w+) times? (?:per|before|between) (?:a )?(short|long) rest/i', $text, $matches)) {
            $maxUses = $this->wordToNumber($matches[1]);
            $resetsOn = strtolower($matches[2]) === 'short'
                ? ResetTiming::SHORT_REST
                : ResetTiming::LONG_REST;
        }

        // Pattern: "regain use when you finish a long rest"
        if (preg_match('/regain (?:the )?use when you finish a (short|long) rest/i', $text, $matches)) {
            $maxUses = 1;
            $resetsOn = strtolower($matches[1]) === 'short'
                ? ResetTiming::SHORT_REST
                : ResetTiming::LONG_REST;
        }

        return [
            'max_uses' => $maxUses,
            'resets_on' => $resetsOn,
        ];
    }
}
```

## Service Layer Changes

### `FeatureUseService` - Handle all feature types + spells

```php
public function getCountersForCharacter(Character $character): Collection
{
    $counters = collect();

    // 1. Feature-based counters (ClassFeature, Feat, CharacterTrait)
    $featureCounters = $character->features()
        ->whereNotNull('max_uses')
        ->with('feature')
        ->get()
        ->map(fn ($cf) => $this->mapFeatureToCounter($cf))
        ->filter();

    $counters = $counters->merge($featureCounters);

    // 2. Innate spell counters (racial/feat spells with limited uses)
    $spellCounters = $character->spells()
        ->whereNotNull('max_uses')
        ->with('spell')
        ->get()
        ->map(fn ($cs) => $this->mapSpellToCounter($cs))
        ->filter();

    $counters = $counters->merge($spellCounters);

    return $counters->values();
}

private function mapFeatureToCounter(CharacterFeature $cf): ?array
{
    $feature = $cf->feature;

    $resetsOn = match (true) {
        $feature instanceof ClassFeature => $feature->resets_on,
        $feature instanceof Feat => $feature->resets_on,
        $feature instanceof CharacterTrait => $feature->resets_on,
        default => null,
    };

    $source = match (true) {
        $feature instanceof ClassFeature => $feature->characterClass?->name ?? 'Class',
        $feature instanceof Feat => 'Feat',
        $feature instanceof CharacterTrait => 'Race',
        default => 'Unknown',
    };

    return [
        'id' => $cf->id,
        'type' => 'feature',
        'slug' => $cf->feature_slug,
        'name' => $feature->feature_name ?? $feature->name,
        'current' => $cf->uses_remaining,
        'max' => $cf->max_uses,
        'reset_on' => $resetsOn?->value,
        'source' => $source,
        'source_type' => $cf->source,
    ];
}
```

## Files to Modify

| File | Action |
|------|--------|
| `migrations/xxx_rename_class_counters_to_entity_counters.php` | Create |
| `migrations/xxx_add_uses_to_character_spells.php` | Create |
| `app/Models/EntityCounter.php` | Rename from ClassCounter |
| `app/Models/CharacterClass.php` | Update `counters()` relationship |
| `app/Models/Feat.php` | Update `counters()` relationship |
| `app/Models/CharacterTrait.php` | Add `counters()` + `resets_on` cast |
| `app/Models/CharacterSpell.php` | Add use tracking + `HasLimitedUses` |
| `app/Services/Parsers/Concerns/ParsesUsageLimits.php` | Create |
| `app/Services/Parsers/Concerns/ParsesTraits.php` | Use ParsesUsageLimits |
| `app/Services/Parsers/RaceXmlParser.php` | Use updated ParsesTraits |
| `app/Services/Importers/RaceImporter.php` | Create trait counters |
| `app/Services/FeatureUseService.php` | Handle all feature types + spells |
| `app/Services/CharacterFeatureService.php` | Initialize trait uses |
| `app/Services/Importers/ClassImporter.php` | Update to `reference_type/id` |
| `app/Services/Importers/FeatImporter.php` | Update to `reference_type/id` |
| `app/Console/Commands/AuditOptionalFeatureCountersCommand.php` | Update queries |
| `app/Services/ClassSubclassAudit/SubclassInventoryService.php` | Update queries |
| Tests (~5-7 files) | Create/Update |

## Acceptance Criteria

- [ ] Dragonborn Breath Weapon appears in character counters
- [ ] Drow innate spells appear in character counters
- [ ] Export/import round-trips racial trait counters correctly
- [ ] Rest mechanics reset racial trait counters appropriately
- [ ] Existing class feature counters still work (362 records)
- [ ] Existing feat counters still work (52 records)
- [ ] Counter API returns unified list from all sources

## Estimated Effort

~3-4 days including tests
