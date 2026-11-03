# Generic Timezone Filtering & Single Source-of-Truth Consolidation Plan

Refactor date/datetime filtering across all backend modules to use a generic, reusable timezone-aware filter class (`TimezoneDateFilter` / `TimezoneDateTimeRangeFilter`) in `rawadhai_module/utils/timezone_utils/filters.py`. Consolidate all timezone utilities into `rawadhai_module/utils/timezone_utils/` as the single source of truth, removing legacy `rawadhai_module/timezone_utils.py` and module-specific filter methods.

## User Review Required

> [!IMPORTANT]
> - **Zero Filter Boilerplate**: Individual filter classes (e.g. `SatelliteimageFilter`, `Image360Filter`, `CameraImageListFilter`, `DronFilter`, `Video360Filter`) will no longer implement custom `filter_date` or `filter_capture_date` methods. Instead, they will declare reusable timezone fields (e.g. `date = TimezoneDateFilter(field_name='created')`).
> - **Single Source of Truth**: All imports from `rawadhai_module.timezone_utils` will be migrated to `rawadhai_module.utils.timezone_utils`. The root file `rawadhai_module/timezone_utils.py` will be removed/deprecated to prevent competing timezone implementations.

## Open Questions

None. The requirements are clear.

---

## Proposed Changes

### 1. Centralized Timezone Filter Module (`rawadhai_module/utils/timezone_utils/`)

#### [NEW] [filters.py](file:///home/aneesh/Documents/python/docker/rawadhv2/rawadhai_backend/rawadhai_module/utils/timezone_utils/filters.py)
Create `TimezoneDateFilter` and `TimezoneDateTimeRangeFilter` subclasses of `django_filters.Filter` / `django_filters.CharFilter`.
- `TimezoneDateFilter`: Automatically converts single or comma-separated local date strings (`"2026-09-12"`) into UTC start/end range boundaries using `local_date_to_utc_range(val, tz)` and filters the queryset via `__gte` and `__lt` at database level.
- Resolves effective timezone once per request context without extra DB queries.

#### [MODIFY] [__init__.py](file:///home/aneesh/Documents/python/docker/rawadhv2/rawadhai_backend/rawadhai_module/utils/timezone_utils/__init__.py)
Export `TimezoneDateFilter` and `TimezoneDateTimeRangeFilter` from `rawadhai_module.utils.timezone_utils`.

#### [DELETE] [timezone_utils.py](file:///home/aneesh/Documents/python/docker/rawadhv2/rawadhai_backend/rawadhai_module/timezone_utils.py)
Remove competing top-level file after updating all import statements across the codebase.

---

### 2. Module Filters Refactoring

#### [MODIFY] [filters.py](file:///home/aneesh/Documents/python/docker/rawadhv2/rawadhai_backend/satellite_image/filters.py)
Replace custom `filter_date` method in `SatelliteimageFilter` with `date = TimezoneDateFilter(field_name='created', label='date')`.

#### [MODIFY] [filters.py](file:///home/aneesh/Documents/python/docker/rawadhv2/rawadhai_backend/image360/filters.py)
Replace custom `filter_capture_date` method in `Image360Filter` with `capture_date = TimezoneDateFilter(field_name='capture_date', label='capture_date')`.

#### [MODIFY] [camerasimagelistfilter.py](file:///home/aneesh/Documents/python/docker/rawadhv2/rawadhai_backend/rawadhai_module/api/filters/camerasimagelistfilter.py)
Replace custom `filter_capture_date_date_exact` and `multi_filter_capture_date_date_exact` with `TimezoneDateFilter(field_name='capture_date')`.

#### [MODIFY] [dronFilter.py](file:///home/aneesh/Documents/python/docker/rawadhv2/rawadhai_backend/rawadhai_module/api/filters/dronFilter.py)
Add `date = TimezoneDateFilter(field_name='created', label='date')` to `DronFilter`.

#### [MODIFY] [video360Filter.py](file:///home/aneesh/Documents/python/docker/rawadhv2/rawadhai_backend/rawadhai_module/api/filters/video360Filter.py)
Add `date = TimezoneDateFilter(field_name='created', label='date')` to `Video360Filter`.

---

### 3. Update Import Call Sites

#### [MODIFY] [common.py](file:///home/aneesh/Documents/python/docker/rawadhv2/rawadhai_backend/rawadhai_module/common.py)
Update import from `rawadhai_module.utils.timezone_utils`.

#### [MODIFY] [camera_images.py](file:///home/aneesh/Documents/python/docker/rawadhv2/rawadhai_backend/rawadhai_module/camera_images.py)
Update import from `rawadhai_module.utils.timezone_utils`.

#### [MODIFY] [model_common.py](file:///home/aneesh/Documents/python/docker/rawadhv2/rawadhai_backend/rawadhai_module/api/utils/model_common.py)
Update import from `rawadhai_module.utils.timezone_utils`.

#### [MODIFY] [migrate_camera_image_timezones.py](file:///home/aneesh/Documents/python/docker/rawadhv2/rawadhai_backend/rawadhai_module/management/commands/migrate_camera_image_timezones.py)
Update import from `rawadhai_module.utils.timezone_utils`.

#### [MODIFY] [test_timezone_handling.py](file:///home/aneesh/Documents/python/docker/rawadhv2/rawadhai_backend/rawadhai_module/tests/test_timezone_handling.py)
Update imports and add unit tests verifying `TimezoneDateFilter` on Satellite, 360, Camera Image, Drone, and 360 Video filters.

---

### 4. Practice Notebook & Documentation

#### [MODIFY] [test_centralized_timezone_utils.ipynb](file:///home/aneesh/Documents/python/docker/rawadhv2/rawadhai_backend/practice/test_centralized_timezone_utils.ipynb)
Update notebook with generic `TimezoneDateFilter` usage examples.

#### [MODIFY] [timezone_handling_guide.md](file:///home/aneesh/Documents/python/docker/rawadhv2/rawadhai_backend/rawadhai_module/utils/timezone_utils/timezone_handling_guide.md)
Update documentation guide for junior developers explaining generic `TimezoneDateFilter`.

---

## Verification Plan

### Automated Tests
- Run unit test suite in Docker container:
  ```bash
  docker compose -f docker-compose-dev.yml exec rawadh_backend python manage.py test rawadhai_module.tests.test_timezone_handling
  ```
- Run full app tests across `satellite_image`, `image360`, and `rawadhai_module` to verify zero regressions:
  ```bash
  docker compose -f docker-compose-dev.yml exec rawadh_backend python manage.py test satellite_image image360 rawadhai_module
  ```

### Manual Verification
- Verify `TimezoneDateFilter` converts local date filters into single-interval database range queries without running Python loops or extra DB queries.
