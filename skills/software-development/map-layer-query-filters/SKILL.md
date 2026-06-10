---
name: map-layer-query-filters
description: Implement and review map-layer filter plumbing in AIOZ World Deck, especially temporal/interval filters and layerQuery -> hook -> API param translation.
---

# Map Layer Query Filters

## When to use
- Adding or changing a map-layer filter in AIOZ World Deck
- Wiring `layerQuery` state into a map hook
- Implementing interval / temporal filters for map layers
- Reviewing cache-key behavior for map-layer queries
- Auditing stale layer-key aliases like `news` vs `live_news`

## Core rule
Keep **UI/state query shape** and **backend request shape** separate.

- `layerQuery` in Redux stores semantic UI state, e.g. `interval: '7d'`
- hook translates semantic state into backend params, e.g. `start_date`, `end_date`, `since`, `until`
- **queryKey must use raw `layerQuery` directly**, not a derived request object
- derived backend params belong in fetch request only

## Standard flow
1. Add or update default `layerQuery` state in `src/store/map-layer/index.ts`
2. Add or update per-layer filter UI in `src/components/widget/MapLayersFilter/`
3. Register filter component in `src/store/map-filter-layer-registry.ts`
4. In hook, destructure `interval` from `query`
5. Build backend request params from `getDateRangeFromInterval(interval)` inside fetch logic (or a local derived request object)
6. Keep `queryKey` based on raw `query` / `layerQuery`, not translated dates
7. Run `pnpm type-check`

## Temporal filter rules
Use `getDateRangeFromInterval()` as source of truth.

- start bound: `getDateRangeFromInterval(interval)[0]?.toISOString()`
- end bound: `getDateRangeFromInterval(interval)[1]?.toISOString()`
- do not re-introduce `getIntervalStartDate` / `getIntervalEndDate`

## Query-key pitfall
Do **not** do this:

```ts
queryKey: queryKeys.getLayerGeoJSON('water_alerts', {
  ...query,
  start_date,
  end_date,
})
```

Do this instead:

```ts
queryKey: queryKeys.getLayerGeoJSON('water_alerts', query)
```

and translate dates only in the request:

```ts
const { interval, ...rest } = (query ?? {}) as Query & { interval?: string };
const [start, end] = getDateRangeFromInterval(
  (interval as SinceInterval | undefined) ?? 'all',
);

const apiQuery = {
  ...rest,
  start_date: start?.toISOString(),
  end_date: end?.toISOString(),
};
```

Reason: cache key should follow source-of-truth `layerQuery`, not duplicate transform logic.

**Namespace change for `live_news`**: The `useNewsGeoJson` hook previously used `['getNews', query]` as its query key. After aligning with the layer key `live_news`, it now uses `queryKeys.getLayerGeoJSON('live_news', query)`. This changes the cache namespace. If anything elsewhere in the app invalidates or depends on the old `getNews` key, behavior will change. Audit before merging.

## Endpoint mapping checklist
After API regen, re-check generated query types before wiring params.

Typical mappings seen in this repo:
- `start_date` / `end_date` for `live_news`, `military_tracker`, `landslides`, severe-weather layers
- `since` / `until` for `nuclear_tests`, `net_outages`, `maritime_warnings`

Never assume old param names survived regen. Re-open generated `Get...Data['query']` types and align hook mapping.

## Layer-key pitfall
Check actual map-layer key before wiring selectors and config.

Known example:
- map layer key is `live_news`, not `news`

Audit these places together when fixing a stale alias:
- `getActiveLayerById(...)`
- `getLayerQuerySelector(...)`
- map layer visual config key
- hook query key namespace if layer-specific

## Review checklist
- raw `queryKey` uses `query` / `layerQuery` directly
- fetch request translates interval to backend params
- generated query type matches actual param names after API regen
- layer key matches registry/constants (`live_news` vs stale alias)
- interval defaults are set intentionally for each layer
- formatting remains readable after any bulk refactor

## Verification
- `pnpm type-check`
- if repo has unrelated red baseline, grep output for touched files and confirm no new errors

**Type-check baseline strategy**: When the repo has pre-existing type errors, run `pnpm type-check` and grep only for the specific files you changed. Example:
```bash
pnpm type-check 2>&1 | rg -n "useNewsGeoJson|useWaterAlertsGeoJson|useNuclearTestsGeoJson|..."
```
If the grep returns nothing, your changes introduced no new type errors — the full output failures are from unrelated pre-existing issues.

## References
- `references/aioz-world-deck-interval-filter-notes.md` — session notes, param mappings, pitfalls, and examples from interval-filter rollout
