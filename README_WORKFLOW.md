# Soulvegr RPG VTT — Quick Reference

## The Big Picture

You have three problems. This system solves all of them.

### Problem 1: "Where does game data live?"
**Solution:** `data/soulvegr_data.json` — single source of truth

### Problem 2: "How do I get it into PocketBase?"
**Solution:** `scripts/pb_seed_data.js` — one command, everything imported

### Problem 3: "How do my UI components use it?"
**Solution:** `js/pocketbase_data_layer.js` — clean API for queries

---

## 30-Second Startup

```bash
# 1. Install PocketBase (one time)
brew install pocketbase
# OR download from https://github.com/pocketbase/pocketbase/releases

# 2. Run setup (handles everything)
bash setup.sh

# 3. Open browser
open soulvegr_sheet.html

# 4. Login
# Email: admin@example.com
# Password: changeme

# 5. Test: Create a vessel, add a talent, reload page
# Talent should still be there ✓
```

Done. Everything is working.

---

## Adding Game Content

You've written new rules. Now what?

### 1. Add to `data/soulvegr_data.json`

```json
{
  "talents": [
    {
      "name": "My New Talent",
      "description": "Does something cool",
      "bp_cost": 30,
      "epochs": ["Initiate", "Advanced"]
    }
  ]
}
```

### 2. Run seed script

```bash
node scripts/pb_seed_data.js
# ✓ talents: 1 created, 0 updated
```

### 3. Refresh browser

Done. It's in your VTT.

**No HTML changes needed. No recompiling. Just JSON + script.**

---

## Architecture Overview

```
Your PDF Rulebooks
        ↓
  Edit JSON file
        ↓
  Run seed script
        ↓
  PocketBase gets data
        ↓
  HTML queries data layer
        ↓
  Users see content
        ↓
  Changes save to PocketBase
        ↓
  Reload page, still there ✓
```

Each layer is independent. You can:
- Update JSON without touching code
- Replace PocketBase without changing queries
- Swap UI without affecting database

---

## File Structure

```
soulvegr-rpg/
├── soulvegr_sheet.html          # Character sheet UI
├── data/
│   └── soulvegr_data.json       # ALL game content (edit this)
├── scripts/
│   └── pb_seed_data.js          # Import data to PocketBase (run this)
├── js/
│   └── pocketbase_data_layer.js # Query interface (use in HTML)
├── pb_data/                      # PocketBase local storage (auto-created)
├── setup.sh                      # One-command setup
├── DATA_PIPELINE.md              # How it works (this file)
├── INTEGRATION_CHECKLIST.md      # Code changes needed
└── README_WORKFLOW.md            # Detailed examples
```

---

## Common Tasks

### Task: Add a new power to a player character

```javascript
// In console:
await pbData.addAbilityToCharacter(charId, powerId, 'powers');
// Character gets the power, saves to PB automatically
```

### Task: Search for all powers from a specific origin

```javascript
// In HTML or console:
const powers = await pbData.searchPowers('', 'Martial');
// Returns all powers with origin='Martial'
```

### Task: Update character data after they level up

```javascript
// In console:
D.level = 5;
D.bpTotal = 200;
await pbData.updateCharacter(charId, D);
// Saves to PB, persists across reloads
```

### Task: Populate all races from rulebook

```json
// In data/soulvegr_data.json:
{
  "races": [
    { "name": "Human", "base_speed": 30 },
    { "name": "Celestial", "base_speed": 25 },
    { "name": "Void-Touched", "base_speed": 35 }
  ]
}
```

Then: `node scripts/pb_seed_data.js`

---

## The Data Layer API

Use these functions in your HTML/JavaScript:

```javascript
// FETCHING
await pbData.getTalents()           // All talents
await pbData.getPerks()             // All perks
await pbData.getHindrances()        // All hindrances
await pbData.getPowers()            // All powers
await pbData.getGear()              // All gear
await pbData.getWeapons()           // Weapons only
await pbData.getArmor()             // Armor only
await pbData.getRaces()             // All races

// SEARCHING
await pbData.searchTalents('Iron')           // Find talents with 'Iron' in name
await pbData.searchPowers('Blast', 'Martial')// Powers with 'Blast' from Martial origin
await pbData.searchGear('Plasma')            // Gear with 'Plasma' in name

// RACE INFO
await pbData.getRaceByName('Human')          // Get human race object
await pbData.getRaceSpeed('Human')           // Get base speed (30 ft)

// CHARACTER OPERATIONS
await pbData.createCharacter(data)           // Create new character
await pbData.getCharacter(charId)            // Load character
await pbData.updateCharacter(charId, data)   // Save character
await pbData.addAbilityToCharacter(charId, abilityId, 'talents')  // Add talent
await pbData.removeAbilityFromCharacter(charId, abilityId, 'talents') // Remove

// CACHE
pbData._clearCache()                         // Force fresh data on next query
```

All functions return promises. Use `async/await` or `.then()`.

---

## Workflow Examples

### Example 1: Player selects a new talent

```javascript
// User clicks "Add Talent" button
async function onAddTalentClick() {
  // Get all available talents
  const talents = await pbData.getTalents();
  
  // Show in modal
  renderTalentModal(talents);
}

// User selects "Iron Will"
async function onTalentSelected(talentId) {
  // Add to character
  await pbData.addAbilityToCharacter(charId, talentId, 'talents');
  
  // Update UI
  renderAbilities();
  
  // Auto-saves to PocketBase
}

// Player refreshes page
// Iron Will is still there ✓
```

### Example 2: GM creates NPC

```javascript
// Get random powers and talents
const talents = await pbData.getTalents();
const powers = await pbData.getPowers();

// Build NPC
const npc = {
  name: "Captain Void",
  talents: [talents[Math.random() * talents.length]],
  powers: [powers[Math.random() * powers.length]]
};

// Save
await pbData.createCharacter(npc);
```

### Example 3: Adding new combat actions

```javascript
// Get all weapons
const weapons = await pbData.getWeapons();

// For each equipped weapon, calculate attack bonus
weapons.filter(w => w.equipped).forEach(weapon => {
  const skillRank = D.skills[weapon.attack_skill] || 0;
  const attrMod = getMod(D.attrs.force);
  const bonus = skillRank + attrMod;
  
  // Create attack entry
  const attack = {
    name: weapon.name,
    bonus: bonus,
    damage: weapon.damage,
    range: weapon.range
  };
  
  // Display in combat tab
  renderCombatAction(attack);
});
```

---

## Troubleshooting Checklist

**VTT won't start?**
- [ ] PocketBase running? `curl http://127.0.0.1:8090/api/health`
- [ ] HTML file exists? `soulvegr_sheet.html`
- [ ] No console errors? Check browser dev tools (F12)

**Can't login?**
- [ ] Seed script ran? `node scripts/pb_seed_data.js --verify`
- [ ] Correct credentials? admin@example.com / changeme
- [ ] PocketBase has users collection? Check admin panel http://127.0.0.1:8090/_/

**Character won't save?**
- [ ] PocketBase running?
- [ ] Characters collection exists?
- [ ] Check console errors (F12)

**New content not appearing?**
- [ ] Added to `data/soulvegr_data.json`?
- [ ] Ran seed script? `node scripts/pb_seed_data.js`
- [ ] Refreshed browser? (Not cached)
- [ ] Clear cache? `pbData._clearCache()`

**Data layer returns empty?**
- [ ] Seed script succeeded? (check for errors)
- [ ] `--verify` flag passes? `node scripts/pb_seed_data.js --verify`
- [ ] Fallback registry available? (Should show if PB fails)

---

## Next Features to Build

Once data pipeline is solid:

1. **Combat Tracker**
   - Real-time HP/EP tracking
   - Turn order with initiative rolls
   - Condition management

2. **Encounter Builder**
   - Load creatures/NPCs
   - Auto-calculate difficulty
   - Loot generation

3. **Campaign Manager**
   - Session logging
   - NPC relationship tracking
   - Divine House allegiance manager

4. **PDF Export**
   - Character sheet to PDF
   - Campaign summary reports

5. **Multi-player Sync**
   - GM sees all party sheets live
   - Players see shared campaign data

All use same pattern:
- New collection in `pb_schema.json`
- Data in `data/soulvegr_data.json`
- Query via `pbData.*()` in UI
- Done

---

## Key Takeaway

**You're no longer stuck because you have a system.**

- Change game rules? Edit JSON + run script
- Add new features? Query the data layer
- Debug issues? Check each layer independently
- Scale content? Just add to JSON
- Switch platforms? Same data, different UI

The pipeline is the engine. Everything else is just driving it.

---

## Support

- **Data pipeline broken?** Check `DATA_PIPELINE.md`
- **Integration questions?** See `INTEGRATION_CHECKLIST.md`
- **PocketBase issues?** https://pocketbase.io/docs/
- **Stuck?** Clear cache, restart PocketBase, verify seed ran

Good luck, and happy building! 🚀
