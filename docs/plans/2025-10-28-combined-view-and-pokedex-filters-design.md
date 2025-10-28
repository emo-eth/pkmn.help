# Combined Offense + Defense View and Pokedex Filters Design

**Date:** 2025-10-28
**Status:** Approved for Implementation

## Overview

This design adds two major features to pkmn.help:
1. A combined offense + defense view in the type calculator
2. Advanced filtering capabilities in the Pokedex (type filters and form variant toggles)

## Requirements Summary

### Combined View
- New screen showing both offense and defense matchups simultaneously
- Defense section on top, offense section on bottom (both offset to the right)
- Shared type selector that applies to both sections
- Include all existing controls (abilities, special moves, tera types)

### Pokedex Filters
- Type filtering with AND logic (all selected types must match)
- Form variant toggles to exclude special Pokemon forms
- All toggles default to ON (inclusive)
- Filters persist in sessionStorage

## Architecture Decisions

**Approach:** Hybrid
- New `ScreenCombined` component for combined view (clean separation for complex layout)
- Extend existing `ScreenPokedex` inline with filter UI (additive feature)

**Rationale:**
- Combined view is complex enough to warrant its own component
- Pokedex filters are additive and fit naturally into existing screen
- Balances component size with feature cohesion

## Detailed Design

### 1. Combined Offense + Defense Screen

#### Component: `ScreenCombined.tsx`

**Layout Structure:**
```
┌─────────────────────────────────────┐
│ Generation Selector (shared)        │
├─────────────────────────────────────┤
│ Type Selector (shared)              │
├─────────────────────────────────────┤
│ Controls (abilities, moves, tera)   │
├─────────────────────────────────────┤
│                                     │
│ ━━ Defense ━━━━━━━━━━━━━━━━━━━━━━ │
│   Matchups component (defense mode) │
│                                     │
├─────────────────────────────────────┤
│                                     │
│ ━━ Offense ━━━━━━━━━━━━━━━━━━━━━━ │
│   Matchups component (offense mode) │
│                                     │
└─────────────────────────────────────┘
```

**State Management:**
- Single type selection state shared between offense and defense calculations
- Separate state for offense-specific options (special moves, offensive abilities)
- Separate state for defense-specific options (defensive abilities, tera type)
- Persist all state in sessionStorage under `combined.*` keys

**Component Reuse:**
- `MultiTypeSelector` for type selection
- `Matchups` component (used twice: once for offense, once for defense)
- Existing `partitionMatchups` and `matchupFor` logic from data-matchups.ts
- `CheckboxGroup` for abilities and special moves
- `Select` component for tera type and ability selectors

**Navigation:**
- Add "Combined" as third tab in main navigation
- Position between or after "Offense" and "Defense" tabs
- Update routing in App.tsx to handle new route

**SessionStorage Keys:**
```typescript
combined.types: Type[]
combined.offense.specialMoves: string[]
combined.offense.abilities: string[]
combined.defense.ability: Ability
combined.defense.teraType: Type
```

#### Implementation Notes

The component will calculate matchups for both modes simultaneously:
1. Get selected types from shared state
2. Calculate defensive matchups using defense-specific modifiers
3. Calculate offensive matchups using offense-specific modifiers
4. Display both results in separate sections

### 2. Pokedex Type Filters

#### Updates to `ScreenPokedex.tsx`

**New Filter Section (positioned between search bar and grid):**

```
┌─────────────────────────────────────┐
│ Search: [________________] [Sort ▼] │
├─────────────────────────────────────┤
│ Filters:                            │
│                                     │
│ Types: [fire] [water] [grass] ...   │
│                                     │
│ ☑ Include Mega Evolutions           │
│ ☑ Include Regional Forms            │
│ ☑ Include Primal Reversions         │
│ ☑ Include Eternamax                 │
│ ☑ Include Terastal/Stellar          │
│ ☑ Include Alternative Forms         │
├─────────────────────────────────────┤
│ [Pokemon Grid...]                   │
└─────────────────────────────────────┘
```

#### Type Filter Logic

**Component:** Reuse existing `MultiTypeSelector`

**Behavior:**
- Multiple types can be selected
- AND logic: Pokemon must have ALL selected types
- Examples:
  - Select "fire" → shows all Pokemon with fire type (fire, fire/flying, fire/water, etc.)
  - Select "fire" + "flying" → shows only fire/flying dual types
  - Select no types → shows all Pokemon (no filter applied)

**Implementation:**
```typescript
function matchesTypeFilter(pokemon: Pokemon, selectedTypes: Type[]): boolean {
  if (selectedTypes.length === 0) return true; // No filter
  return selectedTypes.every(type => pokemon.types.includes(type));
}
```

#### Form Variant Filter Logic

**Toggles (all default to ON/checked):**

1. **Include Mega Evolutions** (48 Pokemon)
   - Filter: `pokemon.name.includes('-mega')`

2. **Include Regional Forms** (57 Pokemon)
   - Filter: `pokemon.name.includes('-alola') || pokemon.name.includes('-galar') || pokemon.name.includes('-hisui') || pokemon.name.includes('-paldea')`

3. **Include Primal Reversions** (2 Pokemon)
   - Filter: `pokemon.name.includes('-primal')`

4. **Include Eternamax** (1 Pokemon)
   - Filter: `pokemon.name === 'eternatus-eternamax'`

5. **Include Terastal/Stellar** (2 Pokemon)
   - Filter: `pokemon.name.includes('-terastal') || pokemon.name.includes('-stellar')`

6. **Include Alternative Forms** (remaining forms)
   - Filter: Exclude all other hyphenated forms not caught by above categories
   - Examples: origin, therian, crowned, ash, blade, school, etc.

**Implementation Strategy:**

```typescript
interface PokedexFilters {
  types: Type[];
  includeMega: boolean;
  includeRegional: boolean;
  includePrimal: boolean;
  includeEternamax: boolean;
  includeTerastal: boolean;
  includeAlternative: boolean;
}

function isRegionalForm(name: string): boolean {
  return name.includes('-alola') || name.includes('-galar') ||
         name.includes('-hisui') || name.includes('-paldea');
}

function isAlternativeForm(name: string): boolean {
  // Has hyphen but not caught by other categories
  return name.includes('-') &&
         !name.includes('-mega') &&
         !isRegionalForm(name) &&
         !name.includes('-primal') &&
         name !== 'eternatus-eternamax' &&
         !name.includes('-terastal') &&
         !name.includes('-stellar');
}

function matchesFormFilter(pokemon: Pokemon, filters: PokedexFilters): boolean {
  const name = pokemon.name;

  // Check each form category
  if (name.includes('-mega') && !filters.includeMega) return false;
  if (isRegionalForm(name) && !filters.includeRegional) return false;
  if (name.includes('-primal') && !filters.includePrimal) return false;
  if (name === 'eternatus-eternamax' && !filters.includeEternamax) return false;
  if ((name.includes('-terastal') || name.includes('-stellar')) && !filters.includeTerastal) return false;
  if (isAlternativeForm(name) && !filters.includeAlternative) return false;

  return true;
}
```

#### Filter Application Order

```typescript
function filterPokemon(
  allPokemon: Pokemon[],
  filters: PokedexFilters,
  searchQuery: string,
  sortOrder: string
): Pokemon[] {
  let filtered = allPokemon;

  // 1. Apply form variant filters
  filtered = filtered.filter(p => matchesFormFilter(p, filters));

  // 2. Apply type filters
  filtered = filtered.filter(p => matchesTypeFilter(p, filters.types));

  // 3. Apply search query (existing logic with matchSorter)
  if (searchQuery) {
    filtered = applySearchQuery(filtered, searchQuery);
  }

  // 4. Apply sorting (existing logic)
  filtered = applySorting(filtered, sortOrder);

  return filtered;
}
```

#### State Management

**SessionStorage Keys:**
```typescript
pokedex.filters.types: Type[]
pokedex.filters.includeMega: boolean (default: true)
pokedex.filters.includeRegional: boolean (default: true)
pokedex.filters.includePrimal: boolean (default: true)
pokedex.filters.includeEternamax: boolean (default: true)
pokedex.filters.includeTerastal: boolean (default: true)
pokedex.filters.includeAlternative: boolean (default: true)
```

**Hook Usage:**
```typescript
const [filterTypes, setFilterTypes] = useSessionStorage<Type[]>('pokedex.filters.types', []);
const [includeMega, setIncludeMega] = useSessionStorage('pokedex.filters.includeMega', true);
// ... etc for other toggles
```

#### UI Components

**Type Filter:**
- Use existing `MultiTypeSelector` component
- Label: "Filter by Type:"
- No selection limit (allow all types to be selected)

**Form Toggles:**
- Use existing `CheckboxGroup` component
- Group label: "Include Special Forms:"
- All checkboxes default to checked
- Stack vertically or wrap to multiple columns based on screen width

### 3. Data Updates

**Current Status:**
- Dataset contains 1,198 Pokemon entries
- Comprehensive coverage through Gen 9
- All form variants present (Mega, Regional, Primal, Eternamax, Terastal, Alternative)

**Zygarde Forms:**
All existing Zygarde forms are in the dataset:
- zygarde-50 (50% Forme) - ID: 718
- zygarde-10-power-construct (10% Forme) - ID: 10118
- zygarde-complete (Complete Forme) - ID: 10120

**Future Addition:**
- Mega Zygarde from Pokémon Legends Z-A (not yet released)
- Will need to be added once game releases and official stats are available

**Conclusion:** No immediate data updates required.

## Technical Considerations

### Performance
- Filter operations run on every render with active filters
- With 1,198 Pokemon, filtering should be fast (<16ms for 60fps)
- Type checks are simple string operations
- Consider memoization if performance issues arise

### Accessibility
- All form toggles must be keyboard accessible
- Type selector maintains existing keyboard navigation
- Filter state changes should announce to screen readers
- Focus management when toggling filters

### Localization
- Form toggle labels need translation keys
- Type names already have translation support
- Filter section heading needs translation

### Mobile Responsiveness
- Filter panel should collapse/expand on mobile
- Type selector should wrap properly
- Form toggles should stack vertically on narrow screens

## Implementation Checklist

### Combined View
- [ ] Create `ScreenCombined.tsx` component
- [ ] Add shared type selector
- [ ] Add offense-specific controls (special moves, abilities)
- [ ] Add defense-specific controls (ability, tera type)
- [ ] Implement dual matchup calculations
- [ ] Add defense section with Matchups component
- [ ] Add offense section with Matchups component
- [ ] Implement sessionStorage persistence
- [ ] Add route to App.tsx
- [ ] Add navigation tab to UI
- [ ] Test with various type combinations

### Pokedex Filters
- [ ] Add type filter section to ScreenPokedex
- [ ] Add MultiTypeSelector for type filtering
- [ ] Implement type filter logic (AND logic)
- [ ] Add form variant toggle section
- [ ] Implement form variant filter functions
- [ ] Integrate filters into Pokemon list filtering
- [ ] Implement sessionStorage persistence for filters
- [ ] Test filter combinations
- [ ] Verify default states (all ON)
- [ ] Test mobile responsiveness

### Testing
- [ ] Combined view displays both sections correctly
- [ ] Shared type selector updates both offense and defense
- [ ] Type filter AND logic works correctly
- [ ] Form toggles correctly filter Pokemon
- [ ] All filters work together correctly
- [ ] SessionStorage persistence works
- [ ] Mobile layout is usable
- [ ] Keyboard navigation works
- [ ] No performance regressions

## Future Enhancements (Out of Scope)

- Add Mega Zygarde when Legends Z-A releases
- Gigantamax forms (if/when data becomes available)
- Filter by stats (HP, Attack, etc.)
- Filter by generation
- Filter by ability
- Save/load filter presets
- URL parameter support for sharing filtered views
