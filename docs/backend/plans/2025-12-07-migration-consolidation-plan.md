# Migration Consolidation Plan

**Date:** 2025-12-07
**Status:** Draft
**Scope:** Reduce 144 migrations to ~15-20 clean, well-organized migrations

## Current State

| Metric | Count |
|--------|-------|
| Total migrations | 144 |
| Create migrations | 53 |
| Alter migrations | 84 |
| Rename/drop migrations | 7 |

### High-Churn Tables (most migration activity)
- `items` - 11 migrations
- `proficiencies` - 10 migrations
- `spells` - 9 migrations
- `characters` - 9 migrations
- `races` - 7 migrations

## Consolidation Strategy

Since no backwards compatibility is needed, we will:
1. Delete all 144 existing migrations
2. Create ~15 fresh migrations with final schema state
3. Ensure MySQL/SQLite compatibility from the start

## Proposed Migration Structure

### Group 1: Foundation (3 migrations)

```
0001_01_01_000001_create_users_table.php        # Laravel default (keep as-is)
0001_01_01_000002_create_cache_table.php        # Laravel default (keep as-is)
0001_01_01_000003_create_jobs_table.php         # Laravel default (keep as-is)
```

### Group 2: Lookup Tables (1 migration)

```
2025_01_01_000001_create_lookup_tables.php
```

Creates all static lookup/reference tables:
- `sources`
- `spell_schools`
- `conditions`
- `languages`
- `proficiency_types`
- `senses`
- `skills`
- `damage_types` (via lookup_tables pattern)

### Group 3: Core Entity Tables (5 migrations)

```
2025_01_01_000002_create_spells_table.php
2025_01_01_000003_create_items_table.php
2025_01_01_000004_create_races_backgrounds_feats_tables.php
2025_01_01_000005_create_classes_table.php
2025_01_01_000006_create_monsters_table.php
```

Each includes all columns from final schema state (no alterations needed later).

### Group 4: Relationship Tables (3 migrations)

```
2025_01_01_000007_create_entity_polymorphic_tables.php
2025_01_01_000008_create_class_related_tables.php
2025_01_01_000009_create_item_monster_related_tables.php
```

**entity_polymorphic_tables** includes all `entity_*` tables:
- `entity_sources`
- `entity_spells`
- `entity_conditions`
- `entity_languages`
- `entity_items`
- `entity_proficiencies`
- `entity_modifiers`
- `entity_prerequisites`
- `entity_senses`
- `entity_saving_throws`
- `entity_traits`
- `entity_data_tables`

### Group 5: Character System (2 migrations)

```
2025_01_01_000010_create_characters_table.php
2025_01_01_000011_create_character_related_tables.php
```

**characters_table** - main character table with all current columns

**character_related_tables** - all character pivot/child tables:
- `character_classes` (slug-based from start)
- `character_spells` (slug-based)
- `character_equipment` (slug-based)
- `character_languages` (slug-based)
- `character_proficiencies` (slug-based)
- `character_conditions` (slug-based)
- `character_features`
- `character_notes`
- `character_spell_slots`
- `multiclass_spell_slots`
- `feature_selections` (slug-based)

### Group 6: Support Tables (1 migration)

```
2025_01_01_000012_create_support_tables.php
```

- `tags`
- `taggables`
- `media`
- `personal_access_tokens`
- `optional_features`
- `class_optional_features`

### Group 7: MySQL-Specific (1 migration)

```
2025_01_01_000013_add_mysql_fulltext_indexes.php
```

Conditional FULLTEXT indexes (MySQL only, skip for SQLite).

## MySQL/SQLite Compatibility Rules

### 1. Avoid ENUMs - Use String Columns

**Instead of:**
```php
$table->enum('source', ['class', 'race', 'background']);
```

**Use:**
```php
$table->string('source', 50)->default('class');
```

### 2. Conditional FULLTEXT Indexes

```php
public function up(): void
{
    if (DB::getDriverName() !== 'mysql') {
        return;
    }
    DB::statement('ALTER TABLE spells ADD FULLTEXT INDEX spells_search (name, description)');
}
```

### 3. Avoid ->change() Operations

Since we're starting fresh, no column modifications needed. All columns defined correctly from the start.

### 4. Foreign Key Patterns

Use standard Laravel patterns that work in both:
```php
$table->foreignId('user_id')->constrained()->cascadeOnDelete();
```

### 5. No Unsigned Integer Tricks

SQLite handles integers differently. Use Laravel's standard types:
```php
$table->unsignedBigInteger('id');  // Works in both
$table->integer('quantity');        // Works in both
```

## Implementation Steps

### Phase 1: Preparation

1. [ ] Export current table schemas using `SHOW CREATE TABLE` (MySQL)
2. [ ] Create a backup branch with current migrations
3. [ ] Document all column types, constraints, indexes from current state

### Phase 2: Fresh Start

1. [ ] Create new migrations directory structure
2. [ ] Write Group 1-7 migrations (all 13-15 files)
3. [ ] Review each table definition against current models

### Phase 3: Validation

1. [ ] Drop database and run fresh migrations (MySQL)
2. [ ] Run full test suite with SQLite
3. [ ] Verify all models work correctly
4. [ ] Run import:all to verify importers work

### Phase 4: Cleanup

1. [ ] Remove old migrations (144 files)
2. [ ] Update any migration-related documentation
3. [ ] Commit consolidated migrations

## Tables by Migration File

### 2025_01_01_000001_create_lookup_tables.php

| Table | Columns |
|-------|---------|
| sources | id, name, code, abbreviation, description, publisher, publication_year, url, is_official, timestamps |
| spell_schools | id, name, slug, description, timestamps |
| conditions | id, name, slug, description, timestamps |
| languages | id, name, slug, type, script, typical_speakers, description, timestamps |
| proficiency_types | id, name, slug, category, description, timestamps |
| senses | id, name, slug, description, timestamps |
| skills | id, name, slug, ability, description, timestamps |
| damage_types | id, name, slug, description, timestamps |

### 2025_01_01_000002_create_spells_table.php

| Table | Columns |
|-------|---------|
| spells | id, name, slug, full_slug, level, school_id (FK), casting_time, range, components, duration, description, higher_levels, ritual, concentration, source_page, timestamps |
| spell_effects | id, spell_id (FK), level, dice, effect_type, damage_type_id (FK), save_dc, save_ability, timestamps |

### 2025_01_01_000003_create_items_table.php

| Table | Columns |
|-------|---------|
| items | id, name, slug, full_slug, type, category, subcategory, cost, weight, description, detail, is_magic, rarity, attunement, charges_max, charges_used, recharge, ac, ac_bonus, strength_required, stealth_disadvantage, damage_dice, damage_type, versatile_dice, range_normal, range_long, weapon_category, timestamps |
| item_property | id, item_id (FK), property, timestamps |

### 2025_01_01_000004_create_races_backgrounds_feats_tables.php

| Table | Columns |
|-------|---------|
| races | id, name, slug, full_slug, parent_id (FK, nullable), size, speed, description, source_page, is_subrace_required, timestamps |
| backgrounds | id, name, slug, full_slug, description, feature_name, feature_description, source_page, timestamps |
| feats | id, name, slug, full_slug, description, prerequisite_text, source_page, resets_on, timestamps |

### 2025_01_01_000005_create_classes_table.php

| Table | Columns |
|-------|---------|
| classes | id, name, slug, full_slug, parent_id (FK, nullable), hit_die, description, source_page, archetype_level, spells_known_progression, cantrips_known_progression, slot_progression, base_slots, timestamps |
| class_features | id, class_id (FK), name, level, description, feature_type, choice_count, optional, sort_order, timestamps |
| class_spells | id, class_id (FK), spell_id (FK), timestamps |
| class_counters | id, class_id (FK), name, counter_type, counter_value, level, description, timestamps |

### 2025_01_01_000006_create_monsters_table.php

| Table | Columns |
|-------|---------|
| monsters | id, name, slug, full_slug, sort_name, size, type, subtype, alignment, ac, ac_type, hp, hp_formula, speed, str, dex, con, int, wis, cha, passive_perception, cr, xp, description, source_page, is_legendary, is_mythic, is_npc, legendary_actions_count, timestamps |
| monster_actions | id, monster_id (FK), name, type, description, attack_bonus, damage, damage_type, reach, range, uses, recharge, sort_order, timestamps |
| monster_legendary_actions | id, monster_id (FK), name, description, cost, timestamps |
| monster_traits | id, monster_id (FK), name, description, sort_order, timestamps |
| monster_types | id, name, slug, description, timestamps |
| monster_speeds | id, monster_id (FK), type, value, timestamps |

### 2025_01_01_000007_create_entity_polymorphic_tables.php

All `entity_*` tables with polymorphic `entity_type` and `entity_id` columns.

### 2025_01_01_000008_create_class_related_tables.php

| Table | Columns |
|-------|---------|
| optional_features | id, name, slug, full_slug, description, feature_type, level, source_page, resets_on, timestamps |
| class_optional_features | id, class_id (FK), optional_feature_id (FK), level, timestamps |
| proficiencies | id, name, slug, type, category, subcategory, proficiency_type_id (FK, nullable), quantity, choice_support, choice_grouping, timestamps |
| traits | id, name, slug, description, trait_type, timestamps |

### 2025_01_01_000009_create_item_monster_related_tables.php

Additional item/monster tables like damage resistances, immunities, etc.

### 2025_01_01_000010_create_characters_table.php

| Table | Columns |
|-------|---------|
| characters | id, user_id (FK), name, slug, public_id, race_slug, background_slug, alignment, level, experience_points, ability_score_method, strength, dexterity, constitution, intelligence, wisdom, charisma, max_hp, current_hp, temp_hp, armor_class_override, inspiration, death_save_successes, death_save_failures, notes, is_public, timestamps |

### 2025_01_01_000011_create_character_related_tables.php

All character_* tables with slug-based references (not ID-based).

### 2025_01_01_000012_create_support_tables.php

Tags, media, personal_access_tokens.

### 2025_01_01_000013_add_mysql_fulltext_indexes.php

Conditional MySQL FULLTEXT indexes.

## Rollback Strategy

Since this is a complete rewrite with no data migration:

1. Keep old migrations in a `database/migrations_archive/` folder temporarily
2. If issues arise, can restore from archive
3. After 2-3 successful dev cycles, delete archive

## Questions to Resolve

1. **ENUM columns**: Convert all to string? Or keep conditional logic?
   - Recommendation: Convert to string for simplicity

2. **Unused tables**: Any tables that can be dropped entirely?
   - `ability_score_bonuses`, `skill_proficiencies`, `monster_spellcasting` already dropped

3. **Index optimization**: Review which indexes are actually used?
   - Can audit with MySQL `EXPLAIN` after consolidation

## Timeline

This is a cleanup task - no new functionality. Estimated effort:
- Schema documentation: 1-2 hours
- Writing new migrations: 2-3 hours
- Testing and validation: 1-2 hours
- Total: ~5-7 hours of focused work

## Success Criteria

- [ ] All tests pass (Unit-Pure, Unit-DB, Feature-DB, Feature-Search)
- [ ] Import:all works correctly
- [ ] API endpoints return same data
- [ ] No SQLite compatibility errors in test suite
- [ ] Migration count reduced from 144 to ~15
