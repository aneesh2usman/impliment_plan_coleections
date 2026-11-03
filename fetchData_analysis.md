# `useFetchData` — Full Root-Cause Investigation

## Bug Inventory (found during investigation)

### Bug 1 — PRIMARY: `pagination.currentPage` not reset on fresh fetch
**File:** `src/stores/fetchData.ts` — `fetchData()`, line 73–76  
**Status:** ✅ Already fixed by previous session (line 75: `pagination.currentPage = page`)

When `fetchData(1, false)` was called (Refresh, submitFilter, onMounted), items were replaced
but `pagination.currentPage` was NOT reset to 1. After scrolling to load page 3,
`loadNextPage` checked `pagination.currentPage < totalPages` and found `3 < 3 = false` → Load More silently did nothing.

---

### Bug 2 — CRITICAL: Broken `watch` triggers infinite loop / double fetches
**File:** `src/stores/fetchData.ts` — line 271  
```ts
watch([pagination.currentPage, pagination.page_size, ...Object.values(filters)], fetchData, { deep: true });
```
**Problems:**
1. `pagination.currentPage` and `pagination.page_size` are **primitive numbers**. Vue's `watch` on array of primitives
   does NOT reactively track reactive object property changes — it captures their *current value at call time*.
   This watch is **effectively broken/dead** for `currentPage` and `page_size` changes.
2. `...Object.values(filters)` spreads the current string/primitive values at setup time — again, not reactive.
   Filter changes will NOT trigger this watcher.
3. **Even if it did work**: After Bug 1 fix, calling `fetchData(1, false)` now sets `pagination.currentPage = 1`.
   If this watch ever fires on `currentPage` change, it would call `fetchData()` with NO arguments
   (`page = pagination.currentPage`), which could cause an extra fetch. However, since Vue doesn't
   actually track these spread primitive values reactively, this watch is essentially a no-op and harmless.

**Impact on other components:** Neutral — components relying on this watch for auto-refetch on filter changes
are NOT getting the reactivity they expect, but this pre-existed and is not regressed by Bug 1 fix.

---

### Bug 3 — `onUpdateOptions` (used by v-data-table-server in admin pages) — SAFE
**File:** `src/stores/fetchData.ts` — line 118–123  
```ts
const onUpdateOptions = (options) => {
    pagination.currentPage = options.page;
    fetchData(pagination.currentPage, false);
};
```
After Bug 1 fix: `fetchData(page, false)` will now set `pagination.currentPage = page` — which is the same
value it was just set to on line 120. **No regression.** The assignment on line 120 is now redundant but
harmless — `fetchData` itself does the same assignment.

**Components using `onUpdateOptions`:**
- `CompanyListPage.vue`
- `RiskManagementListPage.vue`
- `ProjectSubscriptionsListPage.vue`
- `ActivityLogListPage.vue`
- `PlansListPage.vue`
- `RiskConfigurationListPage.vue`
- `PaymentCyclePage.vue`
- `ProjectsListPage.vue`
- `UsersListPage.vue`

All use `v-data-table-server` with `@update:options="onUpdateOptions"`. Vuetify calls this with `{ page, itemsPerPage, ... }`.
Setting `pagination.currentPage = options.page` then calling `fetchData(options.page, false)` now syncs correctly.
**No regression.**

---

### Bug 4 — `onLoadMoreOptions` double-increments `currentPage` — PRE-EXISTING
**File:** `src/stores/fetchData.ts` — lines 129–149  
```ts
const nextPage = pagination.currentPage + 1;
fetchData(nextPage, true)   // passes nextPage but does NOT modify currentPage (return_data=true)
pagination.currentPage += 1; // increments here
```
`fetchData(nextPage, true)` → `return_data = true` → Bug 1 fix only sets `currentPage` when `return_data = false`.
So `onLoadMoreOptions` still relies on its own `pagination.currentPage += 1` on line 149.
**No regression from Bug 1 fix.** The `onLoadMoreOptions` path was never touched.

**Components using `onLoadMoreOptions`:**
- `ProjectCameraListPage.vue` (`v-infinite-scroll`)
- `SatelliteImageView.vue`
- `FavoriteList.vue`
- `EventTimelineView.vue`
- `Videos360View.vue`
- `CreateTimelapseView.vue`
- `WatchTimelapseView.vue`
- `DronVideosView.vue`
- `Image360View.vue`

These all use Vuetify's `v-infinite-scroll` `@load` callback. `onLoadMoreOptions` handles its own `currentPage`
increment correctly (line 149) for the `return_data=true` path. **No regression.**

---

### Bug 5 — `loadNextPage` (used by Camera sidebar) — This is the fix target — FIXED
**File:** `src/stores/fetchData.ts` — lines 154–168  
Before fix: `pagination.currentPage++` before `fetchData(currentPage, true)` — but after Refresh,
`currentPage` was stale (e.g., 3), so `3 < totalPages` was false.
After Bug 1 fix: Refresh now resets `currentPage = 1`, so Load More correctly resumes from page 2.

**Components using `loadNextPage`:**
- `CameraDetailView.vue` → passed to `CameraImageSidebar.vue` as prop → called on sidebar scroll
- `ImageCompareView.vue` → passed to `imagecompareselect` as prop

Both correctly use `loadNextPage` as a scroll-triggered append-only operation. **No regression.**

---

## Summary: What Changed and Why It's Safe

| Function | Before Fix | After Fix | Impact |
|---|---|---|---|
| `fetchData(page, false)` | Did NOT reset `currentPage` | Resets `currentPage = page` | ✅ Fixes Load More after Refresh |
| `fetchData(page, true)` | Did NOT reset `currentPage` | Still does NOT reset (return_data guard) | ✅ Append path unchanged |
| `onUpdateOptions` | Set `currentPage` then called `fetchData` | `fetchData` now also sets it (redundant but harmless) | ✅ No change in behavior |
| `onLoadMoreOptions` | Manages its own `currentPage` increment | Unchanged | ✅ No regression |
| `loadNextPage` | Stale `currentPage` broke after Refresh | Correctly resumes from page 1+1=2 | ✅ Fixed |
| `submitFilter` | Called `fetchData(1, false)` without reset | Now resets `currentPage = 1` via fix | ✅ Correct |
| `watch` on line 271 | Broken/dead watch on primitives | Unchanged | ✅ No regression |

## Components Verified Safe
- All `v-data-table-server` admin pages (onUpdateOptions) ✅
- All `v-infinite-scroll` camera pages (onLoadMoreOptions) ✅  
- `CameraDetailView` + `CameraImageSidebar` (loadNextPage) ✅ — now fixed
- `ImageCompareView` (loadNextPage) ✅
- All filter submit pages (submitFilter) ✅
