# MoP Classic Gem Strategy Calculator

A single-file web app (`index.html`) for optimizing gem choices in World of Warcraft: Mists of Pandaria Classic.

## Features

### 1. Addon Import
- Paste JSON export from the in-game gear addon
- Auto-detects class, spec, and professions (JC, Engineering, Blacksmithing)
- Populates gear planner with all 16 equipment slots and current gems
- Fetches socket colors and socket bonuses from the Wowhead MoP Classic API

### 2. Stat Weight Configuration
- 34 spec presets with default stat weights (auto-selected from import)
- Fully editable weights for: Strength, Agility, Intellect, Stamina, Hit, Expertise, Crit, Haste, Mastery, Spirit, Dodge, Parry, PvP Power, PvP Resilience

### 3. Gem Recommender
- Ranks all available gems per socket color by weighted score
- Covers all gem types: Red, Yellow, Blue, Orange, Green, Purple, Meta, Serpent's Eye (JC), Cogwheel (Engineering)

### 4. Socket Bonus Evaluator
- Input any gear piece's sockets and bonus
- Calculates whether matching socket colors for the bonus beats stacking your best stat
- Shows point values for both strategies with a clear verdict

### 5. Full Gear Planner
- 16 gear slots auto-populated from addon import
- Per-piece optimization: flags suboptimal gems and suggests replacements
- Match vs. Stack decision per piece
- Summary panel with total stats (current vs. optimized) and a gem swap shopping list
- JC optimization: identifies the best 2 sockets for Serpent's Eye upgrades

## Tech Stack
- Single `index.html` file (HTML + CSS + JS, no dependencies)
- Dark WoW-themed UI with gem color accents
- Wowhead MoP Classic tooltip API for resolving item sockets and gem data
- All calculations run client-side with pure JS math

## Wowhead API
- Endpoint: `https://nether.wowhead.com/mop-classic/tooltip/item/{id}?locale=0`
- Returns item name, socket colors, and socket bonuses from tooltip HTML
- Used during import to auto-populate gear socket data

## Gem Database
Embedded database covering ~80+ gem cuts with item IDs:
- **Red** (Primordial Ruby): 5 cuts — primary stats (+160)
- **Yellow** (Sun's Radiance): 5 cuts — secondary stats (+320)
- **Blue** (River's Heart): 4 cuts — Hit, Spirit, Stamina, PvP Power
- **Orange** (Vermilion Onyx): ~20 hybrid cuts (Red+Yellow)
- **Green** (Wild Jade): ~18 hybrid cuts (Yellow+Blue)
- **Purple** (Imperial Amethyst): ~14 hybrid cuts (Red+Blue)
- **Meta** (Primal Diamond): 14 standard + 4 legendary
- **Serpent's Eye**: 12 JC-only epic gems
- **Cogwheel**: 8 Engineering-only gems (+600 secondary)
- **Sha-Touched**: 3 legendary quest gems (+500 primary)

## Addon Export Format
The tool accepts JSON exports in this structure:
```json
{
  "class": "paladin",
  "spec": "holy",
  "professions": [{"name": "Engineering", "level": 600}, ...],
  "gear": {
    "items": [
      {"id": 95922, "gems": [95344, 76699]},
      ...
    ]
  }
}
```
Gear slot order: Head, Neck, Shoulders, Back, Chest, Wrist, Hands, Waist, Legs, Feet, Ring 1, Ring 2, Trinket 1, Trinket 2, Main Hand, Off Hand.
