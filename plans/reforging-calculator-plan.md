# Reforging Calculator Implementation Plan

## Overview

Add an advanced reforging calculator to the MoP Gem Calculator that optimizes gear reforges to hit stat caps while maximizing weighted stat value.

## MoP Reforging Rules

### Core Mechanics
- Can reforge **40%** of one secondary stat into another secondary stat
- Cannot reforge a stat into itself
- Cannot reforge a stat that already exists on the item
- Only **secondary stats** can be reforged:
  - Hit, Expertise, Crit, Haste, Mastery, Spirit, Dodge, Parry, PvP Power, PvP Resilience

### Reforgeable Stats (Secondary Only)
| Stat | Reforgeable |
|------|-------------|
| Strength | ❌ Primary |
| Agility | ❌ Primary |
| Intellect | ❌ Primary |
| Stamina | ❌ Primary |
| Hit | ✅ |
| Expertise | ✅ |
| Crit | ✅ |
| Haste | ✅ |
| Mastery | ✅ |
| Spirit | ✅ |
| Dodge | ✅ |
| Parry | ✅ |
| PvP Power | ✅ |
| PvP Resilience | ✅ |

---

## Stat Caps

### Hit Rating
| Role | Cap % | Rating Required |
|------|-------|-----------------|
| Melee DPS | 7.50% | 2550 |
| Ranged Physical | 7.50% | 2550 |
| Caster DPS (1H+OH or Staff) | 15.00% | 5100 |
| Caster DPS (Dual Wield) | 17.00% | 5780 |
| Healer | 0% | 0 (not needed) |
| Tank | 7.50% | 2550 |

### Expertise Rating
| Role | Soft Cap (Dodge) | Hard Cap (Parry) |
|------|------------------|------------------|
| Melee DPS | 7.50% / 2550 | 15.00% / 5100 |
| Tank | 7.50% / 2550 | 15.00% / 5100 |
| Ranged/Caster | N/A | N/A |

### Rating Conversions (Level 90)
- **340 rating = 1%** for Hit, Expertise, Crit, Haste, Mastery
- **425 rating = 1%** for Dodge, Parry
- **Spirit** converts to Hit for certain specs at varying rates

---

## Racial Bonuses

### Hit Bonuses
| Race | Bonus |
|------|-------|
| Draenei | +1% Spell Hit (Heroic Presence) |

### Expertise Bonuses
| Race | Weapon Type | Bonus |
|------|-------------|-------|
| Human | Swords, Maces | +1% (340 rating) |
| Dwarf | Maces | +1% (340 rating) |
| Gnome | Daggers, 1H Swords | +1% (340 rating) |
| Orc | Axes, Fist Weapons | +1% (340 rating) |
| Troll | Bows, Thrown | +1% (340 rating) |
| Tauren | (None) | - |
| Blood Elf | (None) | - |
| Goblin | (None) | - |
| Worgen | (None) | - |
| Night Elf | (None) | - |
| Undead | (None) | - |
| Pandaren | (None) | - |

---

## Spirit-to-Hit Conversion

Certain specs convert Spirit to Hit rating at a 1:1 ratio via talents/passives:

| Spec | Passive Name | Conversion |
|------|--------------|------------|
| Balance Druid | Balance of Power | 100% Spirit → Hit |
| Elemental Shaman | Elemental Precision | 100% Spirit → Hit |
| Shadow Priest | Spiritual Precision | 100% Spirit → Hit |
| Mistweaver Monk | Eminence (when DPSing) | 100% Spirit → Hit |

**Important**: These specs should count Spirit toward their Hit cap, but may still want additional Spirit for mana regen purposes.

---

## Data Structures

### New Constants

```javascript
const REFORGE_RATE = 0.40; // 40% of stat value

const SECONDARY_STATS = [
  'hit', 'expertise', 'crit', 'haste', 'mastery', 
  'spirit', 'dodge', 'parry', 'pvppower', 'pvpresilience'
];

const STAT_CAPS = {
  // Role-based caps: { stat: rating_required }
  melee_dps:    { hit: 2550, expertise: 2550 },
  ranged_dps:   { hit: 2550 },
  caster_dps:   { hit: 5100 },
  healer:       {},
  tank:         { hit: 2550, expertise: 5100 }
};

const RACES = {
  human:    { faction: 'alliance', expertise_weapons: ['sword', 'mace'], expertise_bonus: 340 },
  dwarf:    { faction: 'alliance', expertise_weapons: ['mace'], expertise_bonus: 340 },
  gnome:    { faction: 'alliance', expertise_weapons: ['dagger', '1h_sword'], expertise_bonus: 340 },
  draenei:  { faction: 'alliance', spell_hit_bonus: 340 }, // 1% = 340 rating
  night_elf:{ faction: 'alliance' },
  worgen:   { faction: 'alliance' },
  orc:      { faction: 'horde', expertise_weapons: ['axe', 'fist'], expertise_bonus: 340 },
  troll:    { faction: 'horde', expertise_weapons: ['bow', 'thrown'], expertise_bonus: 340 },
  undead:   { faction: 'horde' },
  tauren:   { faction: 'horde' },
  blood_elf:{ faction: 'horde' },
  goblin:   { faction: 'horde' },
  pandaren: { faction: 'neutral' }
};

const SPIRIT_TO_HIT_SPECS = [
  'balance', 'elemental', 'shadow', 'mistweaver'
];
```

### Extended Gear Data

Need to capture base item stats from Wowhead API:

```javascript
// Extended gear object
{
  slot: 'Chest',
  itemId: 95922,
  itemName: 'Robes of the Haunted Forest',
  // Existing gem data...
  gems: [...],
  sockets: [...],
  bonus: {...},
  // NEW: Base stats for reforging
  baseStats: {
    intellect: 1543,
    stamina: 2315,
    crit: 918,
    haste: 1052
  },
  // NEW: Current reforge state
  reforge: {
    from: null,     // 'haste'
    to: null,       // 'hit'
    amount: 0       // 420
  }
}
```

---

## Algorithm Design

### Optimization Problem

Given:
- Set of gear items with secondary stats
- Stat weight values
- Stat caps (Hit, Expertise)
- Racial/spec bonuses

Find:
- Optimal reforge for each item
- Minimizes wasted stats over cap
- Maximizes weighted value of remaining stats

### Algorithm: Greedy with Cap Priority

```
1. Calculate current total stats (including gems, enchants, racials)
2. Calculate how far from each cap we are
3. Sort items by "reforge efficiency" for reaching caps
4. Phase 1 - Reach Caps:
   For each uncapped stat (Hit, Expertise):
     While under cap AND items available:
       Find best item to reforge INTO this stat
       Apply reforge
       Update totals
5. Phase 2 - Optimize Remaining:
   For each item not reforged:
     Find lowest-weight stat on item
     Reforge to highest-weight stat not on item
6. Phase 3 - Reduce Overcap:
   If over any cap:
     Find items contributing to overcap
     Check if different reforge reduces waste
```

### Reforge Efficiency Score

For reaching a cap, score each potential reforge:

```javascript
function reforgeEfficiency(item, targetStat, weights, currentTotals, caps) {
  // Can't reforge if target stat already on item
  if (item.baseStats[targetStat]) return -Infinity;
  
  // Find best stat to reforge FROM (lowest weight, exists on item)
  let bestSource = null;
  let bestSourceWeight = Infinity;
  
  for (const stat of SECONDARY_STATS) {
    if (!item.baseStats[stat]) continue;
    if (stat === targetStat) continue;
    if (weights[stat] < bestSourceWeight) {
      bestSourceWeight = weights[stat];
      bestSource = stat;
    }
  }
  
  if (!bestSource) return -Infinity;
  
  const reforgeAmount = Math.floor(item.baseStats[bestSource] * REFORGE_RATE);
  const neededForCap = caps[targetStat] - currentTotals[targetStat];
  
  // Score: how much we gain toward cap vs how much weighted value we lose
  const effectiveGain = Math.min(reforgeAmount, neededForCap);
  const valueLost = reforgeAmount * bestSourceWeight;
  const valueGained = effectiveGain * weights[targetStat];
  
  return valueGained - valueLost;
}
```

---

## UI Design

### New Tab: "Reforging"

```
┌─────────────────────────────────────────────────────────────┐
│ [Setup & Import] [Gem Recommender] [Socket Evaluator]       │
│ [Gear Planner] [Reforging] ← NEW                            │
└─────────────────────────────────────────────────────────────┘

┌─ Character Info ────────────────────────────────────────────┐
│ Race: [Dropdown: Human, Dwarf, ...]                         │
│ Weapon Type: [Dropdown: Sword, Mace, Axe, ...]              │
│                                                             │
│ Hit Cap:       2550 / 5100 (varies by spec)                 │
│ Expertise Cap: 2550 / 5100 (varies by spec)                 │
└─────────────────────────────────────────────────────────────┘

┌─ Current Stats ─────────────────────────────────────────────┐
│ Hit:       1842 / 2550  [=======    ] 72.2%                 │
│ Expertise: 3102 / 2550  [==========+] 121.6% ⚠️ OVERCAP     │
│ Crit:      4521                                             │
│ Haste:     3892                                             │
│ Mastery:   2104                                             │
└─────────────────────────────────────────────────────────────┘

┌─ Optimized Reforges ────────────────────────────────────────┐
│ [Calculate Optimal Reforges]                                │
│                                                             │
│ Slot        Item                    Reforge                 │
│ ─────────── ─────────────────────── ─────────────────────── │
│ Head        Crown of Eternal Winter Haste → Hit (+412)      │
│ Shoulders   Mantle of the Haunted   (Keep current)          │
│ Chest       Robes of the Haunted    Crit → Hit (+367)       │
│ ...                                                         │
└─────────────────────────────────────────────────────────────┘

┌─ After Optimization ────────────────────────────────────────┐
│ Hit:       2544 / 2550  [==========] 99.8% ✓                │
│ Expertise: 2550 / 2550  [==========] 100.0% ✓               │
│ Crit:      4154 (-367)                                      │
│ Haste:     3480 (-412)                                      │
│ Mastery:   2104                                             │
│                                                             │
│ Weighted Value: 12,450 → 13,892 (+1,442 improvement)        │
└─────────────────────────────────────────────────────────────┘

┌─ Reforge Shopping List ─────────────────────────────────────┐
│ Visit the Reforger NPC and make these changes:              │
│                                                             │
│ 1. Crown of Eternal Winter                                  │
│    Reforge: Haste → Hit                                     │
│    Cost: ~50g                                               │
│                                                             │
│ 2. Robes of the Haunted Forest                              │
│    Reforge: Crit → Hit                                      │
│    Cost: ~50g                                               │
│                                                             │
│ Total Cost: ~150g                                           │
└─────────────────────────────────────────────────────────────┘
```

---

## Implementation Steps

### Phase 1: Data Foundation
1. Add `SECONDARY_STATS` constant
2. Add `STAT_CAPS` by role
3. Add `RACES` with bonuses
4. Add `SPIRIT_TO_HIT_SPECS` list
5. Update PRESETS to include role categorization

### Phase 2: Extend Import
6. Modify `parseItemTooltip()` to extract base stats
7. Update gear data structure with `baseStats` and `reforge` fields
8. Add race selection to Setup tab
9. Add weapon type selection (for expertise racial)

### Phase 3: Algorithm
10. Implement `calculateTotalStats()` - sum from gear, gems, reforges, racials
11. Implement `findOptimalReforges()` - main optimization function
12. Implement `reforgeEfficiency()` helper
13. Handle Spirit-to-Hit conversion for applicable specs

### Phase 4: UI
14. Add "Reforging" tab to navigation
15. Build character info section (race, weapon type)
16. Build current stats display with cap progress bars
17. Build reforge results table
18. Build shopping list output

### Phase 5: Integration
19. Share stat totals with Gear Planner tab
20. Update summary to show reforge impact
21. Add "Copy to Clipboard" for shopping list

---

## Technical Considerations

### Wowhead API for Base Stats

The current `parseItemTooltip()` extracts sockets and socket bonus. Need to extend it to extract base stats:

```javascript
function parseItemTooltip(data) {
  const html = data.tooltip || '';
  // ... existing socket parsing ...
  
  // NEW: Extract base stats
  const baseStats = {};
  const statRegex = /<!--stat(\d+)-->\+?([\d,]+)/g;
  // Or alternative: parse from readable text
  const readableStatRegex = /\+(\d+) ([\w\s]+?)(?:<|$)/g;
  let sm;
  while ((sm = readableStatRegex.exec(html)) !== null) {
    const key = STAT_FROM_TOOLTIP[sm[2].trim()];
    if (key) baseStats[key] = parseInt(sm[1].replace(',', ''));
  }
  
  return { name, sockets, bonus, quality, baseStats };
}
```

### Performance

The optimization algorithm should be fast since:
- Maximum 16 gear slots
- Each slot has at most ~4-5 secondary stats
- Greedy approach avoids combinatorial explosion

If needed, can add a "brute force" mode for truly optimal results (checking all permutations).

---

## Dependencies

- No new external dependencies needed
- All calculations remain client-side
- Same single-file architecture

---

## Testing Scenarios

1. **Melee DPS** (e.g., Arms Warrior)
   - Hit cap 7.5%, Expertise soft cap 7.5%
   - Human with sword should need less Expertise
   
2. **Caster DPS** (e.g., Shadow Priest)
   - Hit cap 15% but Spirit counts toward it
   - Should balance Spirit vs Hit rating
   
3. **Tank** (e.g., Protection Paladin)
   - Hit cap 7.5%, Expertise hard cap 15%
   - High Expertise requirement

4. **Healer** (e.g., Holy Priest)
   - No hit/expertise caps
   - Pure stat weight optimization

5. **Edge Cases**
   - Already at cap - don't over-reforge
   - Already over cap - suggest reforges away
   - No valid reforges available on an item
