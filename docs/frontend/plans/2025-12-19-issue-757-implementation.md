# Enhanced Equipment Display Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Enhance ItemDetailModal to display weapon, armor, and magic item details from backend CharacterEquipmentResource.

**Architecture:** Use inline equipment data for combat stats (no extra fetch), fetch `/items/{slug}` only for description. Graceful fallback when fields are missing.

**Tech Stack:** Vue 3, TypeScript, Vitest, NuxtUI

---

## Task 1: Add Equipment Item Type Interface

**Files:**
- Modify: `app/components/character/inventory/ItemDetailModal.vue:18-35`

**Step 1: Write test for damage type from equipment data**

Add to `tests/components/character/inventory/ItemDetailModal.test.ts`:

```typescript
describe('equipment inline data', () => {
  it('displays damage type from equipment item data', async () => {
    const weaponWithDamageType: CharacterEquipment = {
      ...mockWeapon,
      item: {
        ...mockWeapon.item,
        damage_type: 'Slashing'
      }
    }

    const wrapper = await mountSuspended(ItemDetailModal, {
      props: { open: true, item: weaponWithDamageType }
    })
    await flushPromises()

    expect(wrapper.text()).toContain('Slashing')
  })
})
```

**Step 2: Run test to verify it fails**

Run: `just test-file tests/components/character/inventory/ItemDetailModal.test.ts`
Expected: FAIL - damage_type not displayed

**Step 3: Update FullItemData interface and add equipmentItemData computed**

In `ItemDetailModal.vue`, update the interface at line ~18:

```typescript
// Equipment item data (inline from /characters/{id}/equipment response)
interface EquipmentItemData {
  name?: string
  slug?: string
  weight?: string
  item_type?: string
  armor_class?: number
  damage_dice?: string
  requires_attunement?: boolean
  // NEW: Weapon fields
  damage_type?: string
  versatile_damage?: string
  properties?: Array<{ id: number, code: string, name: string, description?: string }>
  range?: { normal: number, long: number } | null
  // NEW: Armor fields
  armor_type?: 'light' | 'medium' | 'heavy' | null
  max_dex_bonus?: number | null
  stealth_disadvantage?: boolean | null
  strength_requirement?: number | null
  // NEW: Magic item fields
  is_magic?: boolean
  rarity?: string | null
  magic_bonus?: number | null
  // NEW: Charge capacity
  charges_max?: number | null
  recharge_formula?: string | null
  recharge_timing?: string | null
}
```

Add computed to use equipment item data:

```typescript
// Equipment item data (inline from equipment response)
const equipmentItemData = computed(() => props.item?.item as EquipmentItemData | null)
```

Update damageType computed to prefer equipment data:

```typescript
const damageType = computed(() => {
  // Prefer inline equipment data
  if (equipmentItemData.value?.damage_type) {
    return equipmentItemData.value.damage_type
  }
  // Fallback to fetched data
  return fullItemData.value?.damage_type?.name ?? null
})
```

**Step 4: Run test to verify it passes**

Run: `just test-file tests/components/character/inventory/ItemDetailModal.test.ts`
Expected: PASS

**Step 5: Commit**

```bash
git add app/components/character/inventory/ItemDetailModal.vue tests/components/character/inventory/ItemDetailModal.test.ts
git commit -m "feat(inventory): Add equipment inline data interface for damage type (#757)"
```

---

## Task 2: Add Versatile Damage Display

**Files:**
- Modify: `app/components/character/inventory/ItemDetailModal.vue`
- Modify: `tests/components/character/inventory/ItemDetailModal.test.ts`

**Step 1: Write test for versatile damage**

```typescript
it('displays versatile damage when present', async () => {
  const versatileWeapon: CharacterEquipment = {
    ...mockWeapon,
    item: {
      ...mockWeapon.item,
      damage_dice: '1d8',
      versatile_damage: '1d10'
    }
  }

  const wrapper = await mountSuspended(ItemDetailModal, {
    props: { open: true, item: versatileWeapon }
  })
  await flushPromises()

  expect(wrapper.text()).toContain('Versatile')
  expect(wrapper.text()).toContain('1d10')
})
```

**Step 2: Run test to verify it fails**

Run: `just test-file tests/components/character/inventory/ItemDetailModal.test.ts`
Expected: FAIL

**Step 3: Add versatile damage computed and template**

Add computed:

```typescript
const versatileDamage = computed(() => {
  return equipmentItemData.value?.versatile_damage ?? fullItemData.value?.versatile_damage ?? null
})
```

Add to template stats grid (after damage row):

```vue
<div v-if="versatileDamage">
  <span class="font-semibold text-gray-900 dark:text-gray-100">Versatile:</span>
  <span class="ml-1 text-gray-600 dark:text-gray-400">{{ versatileDamage }}</span>
</div>
```

**Step 4: Run test to verify it passes**

Run: `just test-file tests/components/character/inventory/ItemDetailModal.test.ts`
Expected: PASS

**Step 5: Commit**

```bash
git add -A && git commit -m "feat(inventory): Add versatile damage display (#757)"
```

---

## Task 3: Add Range Display for Ranged Weapons

**Files:**
- Modify: `app/components/character/inventory/ItemDetailModal.vue`
- Modify: `tests/components/character/inventory/ItemDetailModal.test.ts`

**Step 1: Write test for range display**

```typescript
it('displays range for ranged weapons', async () => {
  const rangedWeapon: CharacterEquipment = {
    ...mockWeapon,
    item: {
      name: 'Longbow',
      item_type: 'Ranged Weapon',
      damage_dice: '1d8',
      range: { normal: 150, long: 600 }
    }
  }

  const wrapper = await mountSuspended(ItemDetailModal, {
    props: { open: true, item: rangedWeapon }
  })
  await flushPromises()

  expect(wrapper.text()).toContain('150/600 ft')
})
```

**Step 2: Run test to verify it fails**

Run: `just test-file tests/components/character/inventory/ItemDetailModal.test.ts`
Expected: FAIL

**Step 3: Update range computed to use equipment data**

Replace existing range computed:

```typescript
const range = computed(() => {
  // Prefer inline equipment data (new format: object)
  const equipRange = equipmentItemData.value?.range
  if (equipRange) {
    if (equipRange.long) return `${equipRange.normal}/${equipRange.long} ft`
    return `${equipRange.normal} ft`
  }
  // Fallback to fetched data (old format: separate fields)
  const normal = fullItemData.value?.range_normal
  const long = fullItemData.value?.range_long
  if (!normal) return null
  if (long) return `${normal}/${long} ft`
  return `${normal} ft`
})
```

**Step 4: Run test to verify it passes**

Run: `just test-file tests/components/character/inventory/ItemDetailModal.test.ts`
Expected: PASS

**Step 5: Commit**

```bash
git add -A && git commit -m "feat(inventory): Add range display from equipment data (#757)"
```

---

## Task 4: Add Armor Type and DEX Cap Display

**Files:**
- Modify: `app/components/character/inventory/ItemDetailModal.vue`
- Modify: `tests/components/character/inventory/ItemDetailModal.test.ts`

**Step 1: Write tests for armor display**

```typescript
it('displays armor type badge', async () => {
  const armor: CharacterEquipment = {
    id: 10,
    item: {
      name: 'Chain Mail',
      item_type: 'Heavy Armor',
      armor_class: 16,
      armor_type: 'heavy',
      max_dex_bonus: 0,
      stealth_disadvantage: true,
      strength_requirement: 13
    },
    item_slug: 'phb:chain-mail',
    is_dangling: 'false',
    custom_name: null,
    custom_description: null,
    quantity: 1,
    equipped: true,
    location: 'armor'
  }

  const wrapper = await mountSuspended(ItemDetailModal, {
    props: { open: true, item: armor }
  })
  await flushPromises()

  expect(wrapper.text()).toContain('Heavy Armor')
})

it('displays DEX bonus cap for medium armor', async () => {
  const mediumArmor: CharacterEquipment = {
    id: 11,
    item: {
      name: 'Breastplate',
      item_type: 'Medium Armor',
      armor_class: 14,
      armor_type: 'medium',
      max_dex_bonus: 2
    },
    item_slug: 'phb:breastplate',
    is_dangling: 'false',
    custom_name: null,
    custom_description: null,
    quantity: 1,
    equipped: false,
    location: 'backpack'
  }

  const wrapper = await mountSuspended(ItemDetailModal, {
    props: { open: true, item: mediumArmor }
  })
  await flushPromises()

  expect(wrapper.text()).toContain('DEX Bonus')
  expect(wrapper.text()).toContain('+2 max')
})

it('displays DEX bonus none for heavy armor', async () => {
  const heavyArmor: CharacterEquipment = {
    id: 12,
    item: {
      name: 'Plate',
      item_type: 'Heavy Armor',
      armor_class: 18,
      armor_type: 'heavy',
      max_dex_bonus: 0
    },
    item_slug: 'phb:plate',
    is_dangling: 'false',
    custom_name: null,
    custom_description: null,
    quantity: 1,
    equipped: false,
    location: 'backpack'
  }

  const wrapper = await mountSuspended(ItemDetailModal, {
    props: { open: true, item: heavyArmor }
  })
  await flushPromises()

  expect(wrapper.text()).toContain('DEX Bonus')
  expect(wrapper.text()).toContain('None')
})
```

**Step 2: Run tests to verify they fail**

Run: `just test-file tests/components/character/inventory/ItemDetailModal.test.ts`
Expected: FAIL

**Step 3: Add armor computed properties**

```typescript
const armorType = computed(() => equipmentItemData.value?.armor_type ?? null)

const maxDexBonus = computed(() => {
  const bonus = equipmentItemData.value?.max_dex_bonus
  // null means unlimited (light armor), don't display
  if (bonus === null || bonus === undefined) return null
  if (bonus === 0) return 'None'
  return `+${bonus} max`
})
```

**Step 4: Add DEX bonus to template**

Add after existing armor restrictions section:

```vue
<div v-if="maxDexBonus">
  <span class="font-semibold text-gray-900 dark:text-gray-100">DEX Bonus:</span>
  <span class="ml-1 text-gray-600 dark:text-gray-400">{{ maxDexBonus }}</span>
</div>
```

**Step 5: Run tests to verify they pass**

Run: `just test-file tests/components/character/inventory/ItemDetailModal.test.ts`
Expected: PASS

**Step 6: Commit**

```bash
git add -A && git commit -m "feat(inventory): Add armor type and DEX cap display (#757)"
```

---

## Task 5: Add Magic Bonus Badge

**Files:**
- Modify: `app/components/character/inventory/ItemDetailModal.vue`
- Modify: `tests/components/character/inventory/ItemDetailModal.test.ts`

**Step 1: Write test for magic bonus badge**

```typescript
it('displays magic bonus badge for magic weapons', async () => {
  const magicWeapon: CharacterEquipment = {
    ...mockWeapon,
    item: {
      ...mockWeapon.item,
      is_magic: true,
      magic_bonus: 1,
      rarity: 'uncommon'
    }
  }

  const wrapper = await mountSuspended(ItemDetailModal, {
    props: { open: true, item: magicWeapon }
  })
  await flushPromises()

  expect(wrapper.text()).toContain('+1')
})

it('displays +2 magic bonus correctly', async () => {
  const magicWeapon: CharacterEquipment = {
    ...mockWeapon,
    item: {
      ...mockWeapon.item,
      is_magic: true,
      magic_bonus: 2,
      rarity: 'rare'
    }
  }

  const wrapper = await mountSuspended(ItemDetailModal, {
    props: { open: true, item: magicWeapon }
  })
  await flushPromises()

  expect(wrapper.text()).toContain('+2')
})
```

**Step 2: Run tests to verify they fail**

Run: `just test-file tests/components/character/inventory/ItemDetailModal.test.ts`
Expected: FAIL

**Step 3: Add magic bonus computed and badge**

Add computed:

```typescript
const isMagic = computed(() => equipmentItemData.value?.is_magic ?? false)
const magicBonus = computed(() => equipmentItemData.value?.magic_bonus ?? null)
```

Add badge in template (after existing badges):

```vue
<UBadge
  v-if="magicBonus"
  color="spell"
  variant="subtle"
  size="md"
>
  +{{ magicBonus }}
</UBadge>
```

**Step 4: Run tests to verify they pass**

Run: `just test-file tests/components/character/inventory/ItemDetailModal.test.ts`
Expected: PASS

**Step 5: Commit**

```bash
git add -A && git commit -m "feat(inventory): Add magic bonus badge display (#757)"
```

---

## Task 6: Add Static Charge Display (Fallback)

**Files:**
- Modify: `app/components/character/inventory/ItemDetailModal.vue`
- Modify: `tests/components/character/inventory/ItemDetailModal.test.ts`

**Step 1: Write test for static charge display**

```typescript
it('displays static charge info when charges_max present', async () => {
  const wand: CharacterEquipment = {
    id: 20,
    item: {
      name: 'Wand of Magic Missiles',
      item_type: 'Wand',
      is_magic: true,
      rarity: 'uncommon',
      charges_max: 7,
      recharge_formula: '1d6+1',
      recharge_timing: 'dawn'
    },
    item_slug: 'dmg:wand-of-magic-missiles',
    is_dangling: 'false',
    custom_name: null,
    custom_description: null,
    quantity: 1,
    equipped: false,
    location: 'backpack'
  }

  const wrapper = await mountSuspended(ItemDetailModal, {
    props: { open: true, item: wand }
  })
  await flushPromises()

  expect(wrapper.text()).toContain('7')
  expect(wrapper.text()).toContain('1d6+1')
  expect(wrapper.text()).toContain('dawn')
})
```

**Step 2: Run test to verify it fails**

Run: `just test-file tests/components/character/inventory/ItemDetailModal.test.ts`
Expected: FAIL

**Step 3: Add charge display computed properties**

```typescript
const chargesMax = computed(() => equipmentItemData.value?.charges_max ?? null)
const rechargeFormula = computed(() => equipmentItemData.value?.recharge_formula ?? null)
const rechargeTiming = computed(() => equipmentItemData.value?.recharge_timing ?? null)

const hasChargeInfo = computed(() => chargesMax.value !== null)
```

**Step 4: Add charge section to template**

Add new section before description:

```vue
<!-- Charges Section -->
<div
  v-if="hasChargeInfo"
  class="bg-spell-50 dark:bg-spell-900/20 rounded-lg p-3"
>
  <div class="flex items-center justify-between">
    <div>
      <span class="font-semibold text-gray-900 dark:text-gray-100">Max Charges:</span>
      <span class="ml-1 text-gray-600 dark:text-gray-400">{{ chargesMax }}</span>
    </div>
  </div>
  <div
    v-if="rechargeFormula || rechargeTiming"
    class="text-sm text-gray-500 dark:text-gray-400 mt-1"
  >
    Recharges {{ rechargeFormula }}<span v-if="rechargeTiming"> at {{ rechargeTiming }}</span>
  </div>
</div>
```

**Step 5: Run test to verify it passes**

Run: `just test-file tests/components/character/inventory/ItemDetailModal.test.ts`
Expected: PASS

**Step 6: Commit**

```bash
git add -A && git commit -m "feat(inventory): Add static charge display for magic items (#757)"
```

---

## Task 7: Add Interactive Charge Tracking (Future-Ready)

**Files:**
- Modify: `app/components/character/inventory/ItemDetailModal.vue`
- Modify: `tests/components/character/inventory/ItemDetailModal.test.ts`

**Step 1: Write test for interactive charges**

```typescript
it('displays interactive charge controls when charges.current available', async () => {
  const wandWithCharges: CharacterEquipment = {
    id: 21,
    item: {
      name: 'Wand of Magic Missiles',
      item_type: 'Wand',
      is_magic: true
    },
    item_slug: 'dmg:wand-of-magic-missiles',
    is_dangling: 'false',
    custom_name: null,
    custom_description: null,
    quantity: 1,
    equipped: false,
    location: 'backpack',
    // Top-level charges object (backend Phase 3)
    charges: {
      current: 5,
      max: 7,
      recharge_formula: '1d6+1',
      recharge_timing: 'dawn'
    }
  } as CharacterEquipment

  const wrapper = await mountSuspended(ItemDetailModal, {
    props: { open: true, item: wandWithCharges }
  })
  await flushPromises()

  expect(wrapper.text()).toContain('5')
  expect(wrapper.text()).toContain('7')
  // Check for +/- buttons
  expect(wrapper.find('[data-testid="charge-decrement"]').exists()).toBe(true)
  expect(wrapper.find('[data-testid="charge-increment"]').exists()).toBe(true)
})

it('hides charge controls when charges.current is null', async () => {
  const wandStaticOnly: CharacterEquipment = {
    id: 22,
    item: {
      name: 'Wand of Magic Missiles',
      item_type: 'Wand',
      is_magic: true,
      charges_max: 7,
      recharge_formula: '1d6+1',
      recharge_timing: 'dawn'
    },
    item_slug: 'dmg:wand-of-magic-missiles',
    is_dangling: 'false',
    custom_name: null,
    custom_description: null,
    quantity: 1,
    equipped: false,
    location: 'backpack'
  }

  const wrapper = await mountSuspended(ItemDetailModal, {
    props: { open: true, item: wandStaticOnly }
  })
  await flushPromises()

  expect(wrapper.find('[data-testid="charge-decrement"]').exists()).toBe(false)
  expect(wrapper.find('[data-testid="charge-increment"]').exists()).toBe(false)
})
```

**Step 2: Run tests to verify they fail**

Run: `just test-file tests/components/character/inventory/ItemDetailModal.test.ts`
Expected: FAIL

**Step 3: Add charges object support**

Add interface for top-level charges:

```typescript
interface ChargesData {
  current: number | null
  max: number
  recharge_formula: string | null
  recharge_timing: string | null
}
```

Add computed properties:

```typescript
// Top-level charges object (backend Phase 3)
const chargesObject = computed(() => (props.item as { charges?: ChargesData } | null)?.charges ?? null)

const hasInteractiveCharges = computed(() =>
  chargesObject.value !== null && chargesObject.value.current !== null
)

const currentCharges = computed(() => chargesObject.value?.current ?? null)
const maxChargesFromObject = computed(() => chargesObject.value?.max ?? null)

// Use charges object if available, otherwise fall back to item fields
const displayChargesMax = computed(() => maxChargesFromObject.value ?? chargesMax.value)
const displayRechargeFormula = computed(() => chargesObject.value?.recharge_formula ?? rechargeFormula.value)
const displayRechargeTiming = computed(() => chargesObject.value?.recharge_timing ?? rechargeTiming.value)
```

**Step 4: Update template for interactive charges**

Replace static charge section:

```vue
<!-- Charges Section -->
<div
  v-if="hasChargeInfo || hasInteractiveCharges"
  class="bg-spell-50 dark:bg-spell-900/20 rounded-lg p-3"
>
  <div class="flex items-center justify-between">
    <div class="flex items-center gap-2">
      <span class="font-semibold text-gray-900 dark:text-gray-100">Charges:</span>
      <!-- Interactive mode -->
      <template v-if="hasInteractiveCharges">
        <UButton
          data-testid="charge-decrement"
          size="xs"
          variant="ghost"
          icon="i-heroicons-minus"
          :disabled="currentCharges === 0"
          @click="$emit('decrement-charge')"
        />
        <span class="text-gray-600 dark:text-gray-400 min-w-[3rem] text-center">
          {{ currentCharges }} / {{ displayChargesMax }}
        </span>
        <UButton
          data-testid="charge-increment"
          size="xs"
          variant="ghost"
          icon="i-heroicons-plus"
          :disabled="currentCharges === displayChargesMax"
          @click="$emit('increment-charge')"
        />
      </template>
      <!-- Static mode -->
      <template v-else>
        <span class="text-gray-600 dark:text-gray-400">{{ displayChargesMax }} max</span>
      </template>
    </div>
  </div>
  <div
    v-if="displayRechargeFormula || displayRechargeTiming"
    class="text-sm text-gray-500 dark:text-gray-400 mt-1"
  >
    Recharges {{ displayRechargeFormula }}<span v-if="displayRechargeTiming"> at {{ displayRechargeTiming }}</span>
  </div>
</div>
```

Add emits:

```typescript
const emit = defineEmits<{
  'decrement-charge': []
  'increment-charge': []
}>()
```

**Step 5: Run tests to verify they pass**

Run: `just test-file tests/components/character/inventory/ItemDetailModal.test.ts`
Expected: PASS

**Step 6: Commit**

```bash
git add -A && git commit -m "feat(inventory): Add interactive charge tracking UI (#757)"
```

---

## Task 8: Add Properties from Equipment Data

**Files:**
- Modify: `app/components/character/inventory/ItemDetailModal.vue`
- Modify: `tests/components/character/inventory/ItemDetailModal.test.ts`

**Step 1: Write test for properties from equipment data**

```typescript
it('displays properties from equipment inline data', async () => {
  const weaponWithProps: CharacterEquipment = {
    ...mockWeapon,
    item: {
      ...mockWeapon.item,
      properties: [
        { id: 1, code: 'F', name: 'Finesse', description: 'Can use DEX for attack/damage' },
        { id: 2, code: 'L', name: 'Light', description: 'Good for dual wielding' }
      ]
    }
  }

  // Don't fetch - use inline data
  mockApiFetch.mockResolvedValue({ data: { ...mockFullItemData, properties: [] } })

  const wrapper = await mountSuspended(ItemDetailModal, {
    props: { open: true, item: weaponWithProps }
  })
  await flushPromises()

  expect(wrapper.text()).toContain('Finesse')
  expect(wrapper.text()).toContain('Light')
})
```

**Step 2: Run test to verify it fails**

Run: `just test-file tests/components/character/inventory/ItemDetailModal.test.ts`
Expected: FAIL (currently only uses fetched properties)

**Step 3: Update properties computed to prefer equipment data**

```typescript
const properties = computed(() => {
  // Prefer inline equipment data
  if (equipmentItemData.value?.properties?.length) {
    return equipmentItemData.value.properties
  }
  // Fallback to fetched data
  return fullItemData.value?.properties ?? []
})
```

**Step 4: Run test to verify it passes**

Run: `just test-file tests/components/character/inventory/ItemDetailModal.test.ts`
Expected: PASS

**Step 5: Commit**

```bash
git add -A && git commit -m "feat(inventory): Use properties from equipment inline data (#757)"
```

---

## Task 9: Final Integration Test and Cleanup

**Files:**
- Modify: `tests/components/character/inventory/ItemDetailModal.test.ts`

**Step 1: Write integration test with all fields**

```typescript
it('displays full magic weapon with all new fields', async () => {
  const fullMagicWeapon: CharacterEquipment = {
    id: 100,
    item: {
      name: 'Flametongue Longsword',
      item_type: 'Melee Weapon',
      damage_dice: '1d8',
      damage_type: 'Slashing',
      versatile_damage: '1d10',
      properties: [
        { id: 1, code: 'V', name: 'Versatile' }
      ],
      is_magic: true,
      magic_bonus: 2,
      rarity: 'rare',
      requires_attunement: true
    },
    item_slug: 'dmg:flametongue-longsword',
    is_dangling: 'false',
    custom_name: null,
    custom_description: null,
    quantity: 1,
    equipped: true,
    location: 'main_hand'
  }

  const wrapper = await mountSuspended(ItemDetailModal, {
    props: { open: true, item: fullMagicWeapon }
  })
  await flushPromises()

  // Verify all new fields display
  expect(wrapper.text()).toContain('Slashing')      // damage type
  expect(wrapper.text()).toContain('1d10')          // versatile
  expect(wrapper.text()).toContain('+2')            // magic bonus
  expect(wrapper.text()).toContain('Versatile')    // property
})
```

**Step 2: Run full test suite**

Run: `just test-file tests/components/character/inventory/ItemDetailModal.test.ts`
Expected: ALL PASS

**Step 3: Run inventory test suite**

Run: `just test-character`
Expected: ALL PASS

**Step 4: Run full test suite**

Run: `just test`
Expected: ALL PASS

**Step 5: Final commit**

```bash
git add -A && git commit -m "test(inventory): Add integration test for enhanced equipment display (#757)"
```

---

## Task 10: Create Feature Branch and PR

**Step 1: Ensure all changes are on feature branch**

```bash
git checkout -b feature/issue-757-equipment-display
git push -u origin feature/issue-757-equipment-display
```

**Step 2: Create PR**

```bash
gh pr create --title "feat(inventory): Enhanced equipment display (#757)" --body "## Summary
- Display weapon damage type, versatile damage, range from equipment data
- Display armor type and DEX bonus cap
- Display magic bonus badge (+1/+2/+3)
- Add charge tracking UI (interactive when backend ready, static fallback)

## Test Plan
- [x] Unit tests for all new fields
- [x] Integration test with full magic weapon
- [x] Full test suite passes

Closes #757"
```

---

## Summary

| Task | Description | Tests |
|------|-------------|-------|
| 1 | Equipment item type interface | 1 |
| 2 | Versatile damage display | 1 |
| 3 | Range display | 1 |
| 4 | Armor type and DEX cap | 3 |
| 5 | Magic bonus badge | 2 |
| 6 | Static charge display | 1 |
| 7 | Interactive charge tracking | 2 |
| 8 | Properties from equipment data | 1 |
| 9 | Integration test | 1 |
| 10 | Feature branch and PR | - |

**Total: 13 new tests**
