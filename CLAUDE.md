# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Development
nvm use          # Use the correct Node version
yarn install     # Install dependencies
yarn dev         # Start Vite dev server

# Production
yarn build       # Build to /dist
yarn preview     # Preview production build locally
yarn gh          # Full pipeline: build → clean assets → deploy to GitHub Pages

# Deployment steps (individual)
yarn deploy               # Deploy /dist to GitHub Pages
yarn clean-dist-assets    # Remove icon/git artifacts from /dist before deploy
```

There is no test suite.

## Architecture

Vue 3 + Pinia SPA for calculating crafting requirements in the game Icarus. Deployed to GitHub Pages.

**Two entry points** via Vite multi-page setup:
- `index.html` / `src/index.js` → landing page (`/`)
- `icarus/index.html` / `src/pages/icarus/main.js` → calculator app (`/icarus/`)

**Data flow:**

1. On app mount, the Pinia store (`src/store/icarus.js`) fetches 4 JSON files from `public/icarus-game/Data/`:
   - `D_ItemTemplate.json`, `D_ItemsStatic.json`, `D_Itemable.json`, `D_ProcessorRecipes.json`
2. `src/utility/icarusData.js` processes and cross-links these into unified item/recipe objects with display names, icon paths, and crafting requirements.
3. The processed recipe data is stored in the Pinia store, keyed by item ID for O(1) lookups.
4. Users search items (fuzzy via Fuse.js, virtualized list via vue-virtual-scroller), add them to tabs, and the crafting calculator displays requirements in a merged tree view.

**State management (Pinia store `src/store/icarus.js`):**
- Multiple independent planning tabs, each with their own item list and quantities
- Settings (includeSubComponents, includeStationComponents, searchFuzzyMatch) persisted to localStorage
- All tab/item state persisted to localStorage

**Crafting tree UI (`CraftingCalculator.vue` + `CraftingTreeNode.vue`):**
- Interactive canvas viewport: scroll-to-zoom, drag nodes, pan, click to check/uncheck nodes.
- Up to 5 levels: root items (L1) → merged components (L2–L5). L3–L5 only when "Include sub-components" is on.
- `treeLevels` computed builds levels iteratively using `buildRawLevel()`, then **post-processes duplicates**: items that appear at multiple levels are moved to their deepest level with merged `parentIds[]` and summed quantities. This handles diamond dependencies (item needed both directly and as a sub-component).
- `allConnectorLines` computed: derives `{ key, x1, y1, x2, y2 }` from `nodePositions` using canvas-space coordinates. Builds a `levelKeyByItemId` lookup first so lines can connect across non-adjacent levels when items have been promoted to a deeper level.
- `nodePositions`: canvas-space `{ x, y }` per node (keyed `"root:id"`, `"l2:id"`, ...). Used by both drag logic and line calculation. `syncNodePositions()` places new nodes without resetting dragged ones.
- `checkedNodes`: toggle state per node key. `nodeCheckStates` computed derives `{ state: 'none'|'partial'|'full', checkedQty, totalQty }` — children auto-check when all parents checked; yellow "partial" when only some parents checked.
- `effectiveRecipeGetter` computed: returns a `(itemId) => recipe` function that respects `selectedVariants` (user-chosen recipe when multiple exist for the same output). Uses `recipeByOutput` map from the store.
- `recipeByOutput` (store): groups all recipes by their `outputItemId` field, enabling multi-recipe dropdown for items like Composite Paste (Gold vs Platinum).
- `CraftingTreeNode.vue` is a non-recursive display card. Shows `<n-select>` dropdown when `variants.length > 1`. Module-level constants `CARD_W`, `CARD_H_ROOT`, `CARD_H`, `GAP_H`, `GAP_V`, `CANVAS_PAD` control layout geometry.

**Key files:**
- `src/store/icarus.js` — central store, tab/item CRUD, recipe data loading
- `src/utility/icarusData.js` — data processing, `itemLabelMap` (display name overrides), `itemIgnoreMap` (items to exclude)
- `src/pages/icarus/components/craftingCalculator/CraftingCalculator.vue` — merged tree logic, SVG connector lines, crafting station calculation
- `src/pages/icarus/components/craftingCalculator/CraftingTreeNode.vue` — card component (icon + label + qty/controls), no recursive rendering
- `src/pages/icarus/components/craftingCalculator/ItemSearchView.vue` — fuzzy search + virtual list
- `src/constants/common.js` — `GAME_ASSETS_URL` constant (switches between local dev and GitHub-hosted production assets)
- `src/pages/icarus/Icarus.vue` — update this file when bumping game version, calendar date, update notes, and banner text

## Environment

- `.env.development`: `VITE_GAME_ASSETS_URL=/icarus-game` (local assets from `public/`)
- `.env.production`: `VITE_GAME_ASSETS_URL=https://raw.githubusercontent.com/Drumstix42/drumstix42.github.io/main/public/icarus-game`

Game data JSON files live in `public/icarus-game/Data/`. Item icons live in `public/icarus-game/ItemIcons/`. These are sourced from the UE4Export tool (see README for the extraction workflow).

## Updating for New Game Versions

1. Extract new JSON data files using UE4Export (`scripts/Ue4ExportFiles/`) and replace files in `public/icarus-game/Data/`
2. Copy updated item icons to `public/icarus-game/ItemIcons/`
3. Update `src/pages/icarus/Icarus.vue`: game version string, calendar date, update notes link, week number, and banner text
4. If item names changed, update `itemLabelMap` or `itemIgnoreMap` in `src/utility/icarusData.js`
