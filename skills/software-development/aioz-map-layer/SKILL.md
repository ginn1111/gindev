---
name: aioz-map-layer
description: "AIOZ World Deck map layer operations: add, migrate/rename, and add type filters. All files, registry entries, and patterns."
version: 1.0.0
author: Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [aioz, map-layer, map, registry, filter, migration]
---

# AIOZ World Deck — Map Layer Operations

Use this when building, renaming, or adding filters to a map layer in the AIOZ World Deck project.

---

## Project conventions

- Layer keys: **snake_case** (`military_tracker`, `emerg_squawks`, `vessels_ais`)
- Layer IDs in Maplibre: **kebab-case** (`military-tracker-layer`, `emerg-squawks-clusters`)
- Icon sprite IDs: **kebab-case** prefixed `icon-` (`icon-military-tracker`, `icon-emerg-squawks`)
- SVG icon exports: prefix `svgLayer` + PascalName (`svgLayerMilitaryTracker`)
- Filter files: `<LayerName>Filters.tsx` (camelCase domain, PascalCase component)
- All text sizes use `src/styles/theme-fonts.css` utilities — no `text-[Npx]`

### File locations

| Concern | Path |
|---------|------|
| Layer keys / categories | `src/constants/mapLayers.ts` |
| Layer appearance config | `src/constants/mapLayerConfig.ts` |
| Layer component registry | `src/store/map-layer-registry.ts` |
| Layer filter component registry | `src/store/map-filter-layer-registry.ts` |
| Store initial query state | `src/store/map-layer/index.ts` |
| Store query type constants | `src/store/map-layer/constants.ts` |
| Interactive layer IDs | `src/store/map/index.ts` (INTERACTIVE_LAYER_IDS) |
| Interactive layer selectors | `src/store/map-layer/selector.ts` |
| SVG layer icons | `src/assets/icons/LayerIcons.ts` |
| Icon sprite loader | `src/hooks/useMapImageLoader.ts` |
| Component files | `src/components/widget/MaplibreViewer/<LayerName>Layer.tsx` |
| Filter component files | `src/components/widget/MapLayersFilter/<LayerName>Filters.tsx` |
| Layer hook (GeoJSON) | `src/hooks/query/maps/use<Domain>GeoJson.ts` |
| Layer counts | `src/hooks/query/useLayerCounts.ts` |
| Map layers query | `src/hooks/query/useMapLayers.ts` |
| Default preset layers | `src/features/layout-manager/model/defaultMapLayers.ts` |
| Hook query keys | `src/hooks/hook-keys.ts` |
| Docs | `docs/data/map-layers.md` |

---

## Operation 1: Add a new map layer

Touch ALL of these files:

1. **`src/constants/mapLayers.ts`** — add key to one of the `*_MAP_LAYERS` arrays (or `LEGACY_MAP_LAYERS` if placeholder)
2. **`src/constants/mapLayerConfig.ts`** — add `layer_key: { cluster: bool, color: '#hex' }`
3. **`src/store/map-layer/index.ts`** — add `layer_key: { limit: -1, ...filterParams }` to `initialState.layerQuery`
4. **`src/store/map-layer-registry.ts`** — add `const layer_key = lazy(() => import(...))` + entry in `LAYER_COMPONENT_MAP`
5. **`src/store/map/index.ts`** — add `'layer-key-layer'` and `'layer-key-clusters'` to `INTERACTIVE_LAYER_IDS`
6. **`src/store/map-layer/selector.ts`** — add `activeLayers.layer_key && INTERACTIVE_LAYER_IDS_MAP['layer-key-layer']` and cluster variant
7. **`src/components/widget/MaplibreViewer/<LayerName>Layer.tsx`** — create the component
8. **`src/hooks/query/maps/use<Domain>GeoJson.ts`** — create the GeoJSON hook
9. **`src/assets/icons/LayerIcons.ts`** — add `svgLayer<Name>` export
10. **`src/hooks/useMapImageLoader.ts`** — add `['icon-layer-name', svgLayer<Name>]` sprite entry
11. **`src/hooks/query/useLayerCounts.ts`** — add count entry
12. **`src/hooks/query/useMapLayers.ts`** — add `layer_key: true` (or false)
13. **`src/features/layout-manager/model/defaultMapLayers.ts`** — add to relevant preset
14. **`docs/data/map-layers.md`** — add row to table

### Layer component skeleton

```tsx
'use client';

import { memo } from 'react';
import { Layer, Popup, Source } from 'react-map-gl/maplibre';

import { CLUSTER_BUBBLE_DEFAULTS, getMapClusterSourceProps } from '@/constants/mapCluster';
import { getClusterColor } from '@/constants/mapLayerConfig';
import { use<Domain>GeoJSON } from '@/hooks/query/maps/use<Domain>GeoJson';
import { useAppDispatch, useAppSelector } from '@/store/hooks';
import { INTERACTIVE_LAYER_IDS_MAP, setSelectedEntity } from '@/store/map';
import { getActiveEntitySelector, getActiveLayerById, getLayerQuerySelector } from '@/store/map-layer';
import { LiquidBadge, POPUP_INTERACTIVE_CLASS, PopupCard, PopupContent, PopupField, PopupHeader } from './MapPopupCard';

const <LayerName>Layer = () => {
  const dispatch = useAppDispatch();
  const isActive = useAppSelector((state) => getActiveLayerById(state, '<layer_key>'));
  const layerQuery = useAppSelector((state) => getLayerQuerySelector(state, '<layer_key>'));
  // ...use<Domain>GeoJSON, Source/Layer/Popup...
};

export default memo(<LayerName>Layer);
```

---

## Operation 2: Safe-migrate / rename a layer

Replace an old layer key with a new one. **Two-phase:** add new, then deprecate old.

### Phase 1 — Add new key

Same files as Operation 1, but:
- The new component/hook is a copy of the old one with all internal refs renamed
- The old key stays in its category for now
- `pnpm type-check` after every file

### Phase 2 — Deprecate old key

1. Move old key from its category to `LEGACY_MAP_LAYERS` in `src/constants/mapLayers.ts`
2. Wire old key to `PLACEHOLDER` in `src/store/map-layer-registry.ts`
3. Remove old key from `initialState.layerQuery` in `src/store/map-layer/index.ts`
4. Remove old key from `src/store/map-layer/selector.ts` (interactive layer entries)
5. Remove old key from `src/store/map/index.ts` (INTERACTIVE_LAYER_IDS entries)
6. Remove old key from `src/constants/mapLayerConfig.ts`
7. Remove old key from `src/hooks/query/useLayerCounts.ts`
8. Remove old key from `src/hooks/query/useMapLayers.ts`

### Phase 3 — Clean up dead files

- Delete old component `.tsx`
- Delete old GeoJSON hook `.ts`
- Delete old service files if no other consumers
- Delete old standalone hook (e.g., `useEmergencySquawk.ts`)
- Remove orphaned query keys from `src/hooks/hook-keys.ts`
- Update `docs/data/map-layers.md`
- Run `pnpm type-check` and `pnpm build` to verify

---

## Operation 3: Add a type filter to a layer

Follow `docs/how-to-impl-map-layer-filter.md`. Exact files:

1. **`src/store/map-layer/constants.ts`** — add default types array (e.g., `MILITARY_TRACKER_DEFAULT_TYPES`)
2. **`src/store/map-layer/index.ts`** — reference the constant in `initialState.layerQuery[layer]` as `types: DEFAULT_TYPES`
3. **`src/components/widget/MapLayersFilter/<LayerName>Filters.tsx`** — create filter component
4. **`src/store/map-filter-layer-registry.ts`** — add lazy import + registry entry

### Filter component template

```tsx
import { Get<Domain>Data } from '@/__generated__/api';
import { IMapLayerFilter } from '@/constants/mapLayerFilter';
import { useAppDispatch, useAppSelector } from '@/store/hooks';
import { getLayerQuerySelector, setLayerQuery } from '@/store/map-layer';
import { DEFAULT_TYPES } from '@/store/map-layer/constants';
import { ExtractQuery } from '@/types';
import LayerFilterGrid from './LayerFilterGrid';

const COMMON_VALUE = { limit: -1 };
type FilterParams = ExtractQuery<Get<Domain>Data>;

const OPTIONS: IMapLayerFilter<FilterParams>[] = [
  { key: 'type_a', label: 'Type A', value: { ...COMMON_VALUE, types: ['type_a'] } },
  { key: 'type_b', label: 'Type B', value: { ...COMMON_VALUE, types: ['type_b'] } },
];

const isMatchOption = (query: Nullish<FilterParams>, params: FilterParams) =>
  query?.types?.some((t) => params.types?.includes(t));

const <LayerName>Filters = () => {
  const queryLayer = useAppSelector<Nullish<FilterParams>>((state) =>
    getLayerQuerySelector(state, '<layer_key>'),
  );
  const dispatch = useAppDispatch();

  const handleSetFilter = (opt: FilterParams, isActive: boolean) => {
    const newQueryLayer = { ...queryLayer, types: [...(queryLayer?.types || [])] };
    if (opt.types) {
      if (isActive) {
        newQueryLayer.types = newQueryLayer.types.filter((t) => !opt.types?.includes(t));
      } else {
        newQueryLayer.types.push(...opt.types);
      }
    }
    if (newQueryLayer.types.length === 0) newQueryLayer.types = DEFAULT_TYPES;
    dispatch(setLayerQuery({ layer: '<layer_key>', query: newQueryLayer }));
  };

  return <LayerFilterGrid options={OPTIONS} queryLayer={queryLayer} isMatchOption={isMatchOption} onToggle={handleSetFilter} />;
};

export default <LayerName>Filters;
```

### Hook must read types from store, not hardcode

```ts
// ✅ Good — types come from store layerQuery, fall back to a default constant
const internalQuery = useMemo(() => ({
  ...query,
  types: query?.types ?? DEFAULT_TYPES,
}), [query]);

// ❌ Bad — types hardcoded in hook
const internalQuery = useMemo(() => ({
  ...query,
  types: ['some_type'],
}), [query]);
```

---

## Verification

After any layer operation:

```bash
pnpm type-check    # Must pass with 0 new errors
pnpm build         # Must succeed (if feasible)
```

Check in browser:
- Layer appears in `MapLayersFilter` under the correct category
- Toggling layer on/off shows/hides markers
- Clusters render and are clickable
- Popup shows correct data on click
- Filters (if added) update the map on toggle
