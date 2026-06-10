---
name: map-layer-temporal-filters
description: Add or extend interval/temporal filters for map layers in AIOZ World Deck.
---

# Map Layer Temporal Filters

## When to use
- User wants interval/date filters for map layers.
- Need decide which map layers should expose temporal filtering.
- Need wire map-layer filter UI to backend query params or client-side date filtering.
- Need align semantic meaning of layer with what interval means to user on map.

## Core rule
Implement temporal filtering only for layers where time window has clear user meaning on map.

Good fit:
- event/history layers
- warnings/alerts with issue window
- incident feeds
- article/event streams

Bad fit:
- static infrastructure
- current snapshot layers where interval would imply history that UI does not show
- layers belonging to excluded semantic groups

## Required workflow
1. Confirm scope by map-layer group before coding.
   - Ask which group(s) are in scope when user says exclude another group.
   - Do not assume `news` vs `live_news`; verify actual map-layer key in code.
2. Inspect existing layer state and registry.
   - `src/constants/mapLayers.ts`
   - `src/store/map-layer/index.ts`
   - `src/store/map-filter-layer-registry.ts`
3. Inspect existing filter UI and hook plumbing.
   - Filter UI lives under `src/components/widget/MapLayersFilter/`
   - Map hooks live under `src/hooks/query/maps/`
4. Determine temporal transport pattern per endpoint.
   - Some APIs use `since` / `until`
   - Some APIs use `start_date` / `end_date`
   - Some layers need client-side post-filtering because API has no time params
   - If API spec/codegen was regenerated during task, re-open generated query types before finalizing. Do not trust earlier assumptions about param names or cardinality; subtle changes like adding `until` beside `since` are easy to miss.
5. Add default interval in `src/store/map-layer/index.ts` only for in-scope layers.
6. Add or register filter UI.
   - Reuse shared interval component when layer only needs interval.
   - For mixed filters, render interval above existing type chips.
7. Translate `interval` out of Redux query state before API call.
   - Never send raw UI-only `interval` field to SDK endpoint.
   - Convert to backend query params in map hook.
8. After editing `.ts` / `.tsx`, run `pnpm type-check`.
9. Separate new errors from pre-existing repo errors before reporting status.

## Implementation patterns

### Shared interval utilities
Keep date math in `src/utils/intervalFilter.ts`.
Useful helpers:
- `SinceInterval` type — `'3d' | '7d' | '30d' | '90d' | '180d' | '1y' | 'all'`
- `getDateRangeFromInterval(interval: Nullish<SinceInterval>)` returning `[start, end]`
- client-side `isDateWithinInterval()` for feeds without backend time params

**Type discipline for interval**: Always type the `interval` field as `SinceInterval` (or the UI export `LayerInterval`) in the intersection type, never `string`. The old pattern of `interval?: string` with an `as SinceInterval` cast is dead — the type system should enforce the domain directly.

**`?? 'all'` default is dead code**: `getDateRangeFromInterval` already handles `null`/`undefined` natively (returns `[null, null]`), same as passing `'all'`. The `?? 'all'` fallback is a no-op that adds visual noise. Drop it:

```ts
// ✅ correct
const [start, end] = getDateRangeFromInterval(interval);

// ❌ noisy — ?? 'all' is redundant
const [start, end] = getDateRangeFromInterval(
  (interval as SinceInterval | undefined) ?? 'all',
);
```

Prefer a single destructured call over inline tuple indexing. If user asks to simplify temporal plumbing, remove one-off `getIntervalStartDate` / `getIntervalEndDate` helpers and use:
```ts
const [start, end] = getDateRangeFromInterval(interval);
// then start?.toISOString(), end?.toISOString()
```
This keeps all interval math in one source of truth, avoids helper drift during API-spec regeneration, and eliminates the redundant second function call.

**End-boundary semantics**: `getDateRangeFromInterval` uses `dayjs().add(1, 'day').startOf('day')` for the end bound (not `startOf('day')`). This means a `'30d'` interval includes the **entire current day** up to midnight tonight, not just up to midnight last night. Confirm this is the intended behavior with the user before assuming. If it changes, it affects cache-key stability because the derived `end_date`/`until` value shifts at midnight.

### Interval-only filter UI
If layer only needs interval:
- create tiny wrapper component under `src/components/widget/MapLayersFilter/`
- read layer query from Redux
- write `interval` back directly (no toggle-ternary)
- register component in `src/store/map-filter-layer-registry.ts`

**`handleSetFilterInterval` — just set `interval` directly, no toggle.**

The old toggle-ternary pattern:
```ts
dispatch(setLayerQuery({ layer, query: { ...queryLayer, interval: queryLayer?.interval === interval ? undefined : interval } }));
```
was meant to let users toggle-off the same interval. But it breaks the UX contract: when a user clicks an interval, they expect to see data for that interval. The toggle-off is handled by the "All" button in `LayerIntervalFilters` (which calls `onChange(undefined)`). Keep `handleSetFilterInterval` simple:

```ts
const handleSetFilterInterval = (interval: Nullish<LayerInterval>) => {
  dispatch(
    setLayerQuery({
      layer: 'my_layer',
      query: { ...queryLayer, interval },
    }),
  );
};
```

Also simplify the `value` prop — no need for the `as { interval?: LayerInterval }` cast on `queryLayer` when the hook already typed the layer query correctly. Just pass `queryLayer?.interval` directly.

### Mixed filter UI
If layer already has chips/toggles:
- keep current chip logic
- add interval UI above chips
- preserve existing query keys like `event_types` or `types`
- if shared chip component supports `onAllChange`, wire it for warning/type groups so partial selections show an `All` reset back to default query values
- for semantic warning-level chips, do not use one shared active color. Extend `IMapLayerFilter` with per-option active styling (for example `activeStyle?: CSSProperties`) and feed `createBadgeStyleFromColor(WIDGET_THEME_REGISTRY.*)` per option so `Advisory` / `Watch` / `Warning` / `Flash` can carry distinct meaning

### Hook translation
Prefer `useMemo` for derived API params, not inline in `fetchRows`.

Call `getDateRangeFromInterval` once and destructure into variables rather than indexing the tuple inline twice. This avoids redundant computation and keeps intent clear:

```ts
const internalQuery = useMemo(() => {
  const { interval, ...rest } = (query ?? {}) as Query & { interval?: SinceInterval };
  const [startDate, endDate] = getDateRangeFromInterval(interval);
  return {
    ...rest,
    start_date: startDate?.toISOString(),
    end_date: endDate?.toISOString(),
  } satisfies Query;
}, [query]);

return useGeoJSONArtifacts<...>({
  queryKey: queryKeys.getLayerGeoJSON('layer_key', query), // raw query in key
  fetchRows: async () => {
    const { data } = await getXxx({ query: internalQuery }); // derived query at fetch
    // ...
  },
});
```

The same destructured pattern works when the translation is inside `fetchRows` (simpler hooks that don't need `useMemo`):

```ts
fetchRows: async () => {
  const { interval, ...rest } = (query ?? {}) as SomeQuery & { interval?: SinceInterval };
  const [startDate, endDate] = getDateRangeFromInterval(interval);
  const { data } = await getXxx({
    query: { ...rest, start_date: startDate?.toISOString(), end_date: endDate?.toISOString() },
  });
  // ...
},
```

For APIs that use `since`/`until` naming, destructure accordingly:

```ts
const [since, until] = getDateRangeFromInterval(interval);
// then use since?.toISOString(), until?.toISOString()
```

Avoid inlining the translation inside `fetchRows` when the derived params are also needed for cache-key stability — prefer `useMemo` for those cases. But when the hook is simple enough that `queryKey` uses the raw `query`, inlining inside `fetchRows` with destructured variables is acceptable and avoids useMemo overhead.

Patterns:
- `interval` -> `since`
- `interval` -> `since` + `until`
- `interval` -> `start_date` + `end_date`
- `interval` -> client-side `pubDate`/event-date filter after fetch

Concrete reminder from AIOZ map layers:
- `nuclear_tests` may require bounded RFC3339 window (`since` + `until`) after spec regeneration, not only `since`.
- `live_news` can drift in two different ways and both must be checked after API spec regeneration:
  1. query-shape drift — if `GetNewsData['query']` gains `start_date` / `end_date`, stop client-side `pubDate` filtering and send bounded backend params instead;
  2. layer-key drift — nearby files may still use stale `news` key even though actual map-layer key is `live_news`.
- For `live_news`, audit all three places together:
  - map hook query translation (`src/hooks/query/maps/useNewsGeoJson.ts`)
  - layer selectors / active checks in UI (`src/components/widget/MaplibreViewer/NewsLayer.tsx` or replacement layer component)
  - visual config / registry keys (`src/constants/mapLayerConfig.ts`, related registries)

Keep query key based on translated API query, not raw UI state, when interval changes affect fetch results.

## Pitfalls
- **User says "Try again" — fix, don't report.** If user says "Try again" or "do it" after you report session findings, they want the fix implemented immediately. Stop analyzing, start coding. This overrides analysis-first instincts.
- **Bulk-editing multiple similar hooks can corrupt files.** When rewriting the same interval-translation pattern across 10+ hook files, automated scripts (sed over multi-line patterns, Python regex replaces, terminal heredocs) can introduce line-number prefixes, broken indentation, or duplicate content. Prefer targeted `patch()` calls per file, or at minimum verify each output file parses after any mass edit. Use `pnpm type-check` with a targeted grep (not full output) to confirm all touched files are clean, because the repo may have a pre-existing error baseline that drowns new failures.
- **`live_news` query-key namespace changed.** The `useNewsGeoJson` hook went from `['getNews', query]` to `queryKeys.getLayerGeoJSON('live_news', query)`. If anything elsewhere invalidates the old `getNews` key, that cache invalidation silently stops working. Search the repo for `'getNews'` or `"getNews"` before merging.
- `news` may be stale naming in nearby files; active map-layer key can be `live_news`.
- User may ask to exclude layers from another group after docs/spec were drafted. Reconfirm scope before implementation.
- Do not broaden scope from one group to all semantic candidates unless user explicitly says so.
- If `pnpm type-check` fails, check whether failures come from touched files or unrelated pre-existing repo errors.
- Do not add synthetic map-layer keys to Redux state if not present in `MAP_LAYER_LIST`.
- **Bulk-editing multiple similar hooks**: When rewriting the same interval-translation pattern across 10+ hook files, automated edits can corrupt source (line-number prefixes, broken indentation, duplicate lines). Use targeted single-file patches instead of multi-file scripts. After any mass change, open each file and verify it parses.
- **Cache-key stability vs. end-boundary**: The `getDateRangeFromInterval` end bound uses `add(1, 'day').startOf('day')`. If this changes, `end_date`/`until` values in fetch requests shift at midnight. If the `queryKey` is correctly based on raw `layerQuery`, the cache key stays stable — only the *request payload* changes. But if any code accidentally puts derived dates into the query key, midnight churn will cause unnecessary refetches.

## Verification
- `pnpm type-check`
- Confirm new filter components are registered in `MAP_FILTER_LAYERS_REGISTRY`
- Confirm in-scope hooks translate `interval` before SDK call
- Confirm default interval exists only for intended layers

## References
- `references/aioz-world-deck-interval-patterns.md` — concrete AIOZ patterns, affected files, and query-param mapping examples.
