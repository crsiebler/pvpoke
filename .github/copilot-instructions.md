# PvPoke Project Guidelines

PvPoke is a Pokemon GO PvP battle simulator and ranking tool. The codebase uses a non-framework approach with vanilla JavaScript and PHP, creating a custom MVC-like architecture.

## Architecture Overview

### Core Data Flow (Model-View-Controller Pattern)

1. **Model**: `GameMaster.js` loads `data/gamemaster.json` containing all Pokemon and move data
2. **View**: PHP files generate HTML and import JavaScript (e.g., `battle.php`, `rankings.php`)
3. **Controller**: Core classes like `Battle.js`, `Ranker.js`, `TeamRanker.js` process data
4. **Interface**: Scripts in `/js/interface` initialize UI and handle interactions via singleton `InterfaceMaster`

**Key architectural decisions:**
- Singleton pattern for `GameMaster` and `InterfaceMaster` ensures single data source
- No modern framework by design (started as quick project, grew organically)
- "Lasagna code" - structured layers but tightly coupled

### Critical Components

**Battle System** (`src/js/battle/`):
- `Battle.js` - Core battle simulator with `simulate()` method
- `Pokemon.js` - Pokemon instances with stats, moves, type effectiveness
- `Ranker.js` - Generates rankings via `rank()` method for all league/scenario combinations
- `TeamRanker.js` - Team composition analysis

**Data Management** (`src/data/`):
- `compile.php` - Combines JSON chunks into `gamemaster.json` (base + pokemon + moves + formats + cups)
- `parse.php`, `parseElite.php` etc. - Convert Pokemon GO game master files to site format
- `write.php` - Saves generated rankings JSON via AJAX

## Development Workflows

### Running Locally

```bash
# Using Docker (recommended)
docker-compose up
# Access at http://localhost (or PVPOKE_PORT env variable)

# Files are volume-mounted from src/ for live editing
```

### Generating Rankings

Rankings are CPU-intensive and generated locally:

1. Visit `ranker.php` in browser
2. Open browser console for output
3. Simulations run for every league/category, save JSON to `/data/rankings/`
4. For overall rankings: visit `rankersandbox.php`, click Simulate per league
5. Results save to `/data/overall/`

**Ranking algorithm** (`Ranker.js`):
- Simulates battles between all eligible Pokemon pairs
- Battle rating formula: `(healthRating + damageRating) * 500`
- Iterative weighting by opponent strength (7 iterations for custom cups)
- Soft caps at 700+, harsh penalties below 300 to identify volatile Pokemon

### Modifying GameMaster

Edit JSON chunks in `src/data/gamemaster/`:
- `base.json` - Core settings, scenarios, tags
- `pokemon.json` - All Pokemon entries
- `moves.json` - All move definitions  
- `cups/*.json` - Individual cup definitions
- `formats.json` - League formats

Run `compile.php` to rebuild `gamemaster.json` and `gamemaster.min.json`

## Code Conventions

### Object Creation Patterns

**Singleton Pattern** (GameMaster, InterfaceMaster):
```javascript
var GameMaster = (function () {
    var instance;
    function createInstance() {
        var object = new Object();
        // methods here
        return object;
    }
    return {
        getInstance: function() {
            if (!instance) instance = createInstance();
            return instance;
        }
    };
})();
```

**Class Pattern** (Battle, Pokemon):
```javascript
function Battle() {
    var self = this;
    // private vars
    this.publicMethod = function() { /* ... */ }
}
```

### Pokemon Search System

**Search string syntax** (`GameMaster.generatePokemonListFromSearchString`):
- `,` separates OR queries: `"fire,water"` → fire OR water types
- `&` combines AND conditions: `"fire&flying"` → fire AND flying
- `!` negates: `"!legendary"` → exclude legendaries
- `@` searches moves: `"@legacy"`, `"@ice"`, `"@1weather"` (fast only), `"@2blast"` (charged only)
- `+` family search: `"+bulbasaur"` → all Bulbasaur line
- Special: `gen1`, `xl`, `4*`/`hundo`, `meta`, `10k` (cost), `5km` (buddy distance)

Results are cached in `searchStringCache` object for performance.

### Move Selection & Rankings

**Forced movesets** (`moveSelectMode = "force"`):
- Rankings use pre-computed best movesets from existing ranking data
- Override system in `Ranker.overrideMoveset()` for cup-specific sets
- Pokemon have `weightModifier` to adjust importance in matchup calculations

**Auto movesets** (`moveSelectMode = "auto"`):
- Determines moves through simulation (one scenario only)
- Rarely used, primarily for initial move discovery

## Data Structures

### Pokemon Entry Format
```json
{
  "speciesId": "pokemon_name",
  "speciesName": "Display Name", 
  "dex": 1,
  "baseStats": {"atk": 118, "def": 111, "hp": 128},
  "types": ["grass", "poison"],
  "fastMoves": ["VINE_WHIP", "TACKLE"],
  "chargedMoves": ["SLUDGE_BOMB", "SEED_BOMB"],
  "defaultIVs": {"cp1500": [22, 0, 14, 15], "cp2500": [...]},
  "tags": ["starter", "shadoweligible"],
  "released": true
}
```

### Cup/Format System
- **Cups** (`cups/*.json`): Define include/exclude filters, tier rules, level caps
- **Formats** (`formats.json`): Combine cup + CP limit for UI navigation
- Filters use `filterType`: "type", "tag", "id", "move", "dex", "cost", "evolution"

### Ranking Scenarios
```javascript
// From base.json - defines battle conditions
{
  "slug": "leads",
  "shields": [1, 1],
  "energy": [0, 0]  // Energy is in turns of advantage
}
```

## Performance Considerations

- **Pokemon caching**: `GameMaster.getAllPokemon(battle)` reuses Pokemon objects per CP
- **Search caching**: `searchStringCache` avoids re-parsing identical queries  
- **Flush cache** when gamemaster changes: `GameMaster.flushAllPokemonCache()`
- Rankings generate thousands of battles - expect minutes for full leagues

## Common Pitfalls

1. **Singleton state**: GameMaster/InterfaceMaster changes persist across page interactions
2. **Shadow Pokemon**: Have `_shadow` suffix, apply 1.2x ATK / 0.833x DEF multipliers automatically
3. **Form changes**: Some Pokemon (Aegislash, Meloetta) change forms mid-battle
4. **Legacy moves**: Tag with `legacy: true`, marked with † in UI. Elite TMs use `elite: true` with *
5. **Level caps**: Some Pokemon (legendaries) default to level 40, check `levelCap` property
6. **STAB bonus**: Same Type Attack Bonus is 1.2x, applied in Pokemon calculations

## PHP Modules

Reusable UI components in `modules/`:
- `pokeselect.php`, `pokemultiselect.php` - Pokemon selection widgets
- `leagueselect.php`, `cupselect.php`, `formatselect.php` - Filter dropdowns
- `rankingdetails.php` - Detailed Pokemon stats overlay
- Included via `require` in main PHP pages

## Testing & Debugging

- Browser console for ranking generation output
- `Battle.setDebug(true)` logs turn-by-turn actions
- `ranker.php` and `rankersandbox.php` for experimenting with ranking algorithms
- Copy `Ranker.js` or `RankerOverall.js` to test new algorithms without affecting production

## Project Philosophy

**What fits:**
- Bug fixes, performance improvements, algorithm refinements
- Data entry for Pokemon/moves
- Minor refactoring, additions to existing tools
- New features using existing interfaces

**Out of scope:**
- Third-party frameworks/libraries
- Platform overhauls
- User accounts
- Non-responsive features (mobile-first project)

**Development cycle**: Two-week sprints, critical issues handled immediately

Keep changes simple, maintainable, and mobile-friendly. When in doubt, check existing patterns in `/js/battle` and `/js/interface` directories.
