# Implementation Plan — Centralized, Reusable Timezone Architecture (`rawadhai_module/utils/timezone_utils/`)

Design and implement a single source of truth for timezone resolution, IANA validation, date boundary filtering, and API serialization across the RawadhAI backend.

## User Review Required

> [!IMPORTANT]
> **Priority-Based Request Timezone Resolution**
> Timezone resolution will strictly follow this priority order across all APIs, filters, serializers, and background tasks:
> 1. **Explicit Request Override**: `X-Timezone` HTTP Header OR `?timezone=` Query Parameter (if valid IANA name provided in `request`).
> 2. **Camera / Project Site Location Timezone**: Configured IANA timezone for the camera/project site (e.g. `Asia/Riyadh` for Saudi projects).
> 3. **UTC Fallback**: Pure `UTC` fallback if no timezone is available. Server/container/machine local time (`settings.TIME_ZONE`) will **NEVER** be used.

## Proposed Changes

### Central Utility Package

#### [NEW] [rawadhai_module/utils/timezone_utils/](file:///home/aneesh/Documents/python/docker/rawadhv2/rawadhai_backend/rawadhai_module/utils/timezone_utils/)
Create a dedicated python package containing:
- `__init__.py`: Public API exports (`get_request_timezone`, `get_project_tz`, `get_camera_tz`, `validate_iana_timezone`, `utc_to_local`, `local_to_utc`, `format_datetime`, `local_date_to_utc_range`, `get_project_local_date`, `get_project_local_datetime`).
- `resolver.py`: Request header (`X-Timezone`) and query parameter (`?timezone=`) resolution combined with entity inheritance (`Camera` -> `Project` -> `Company`).
- `validator.py`: IANA timezone name validation using Python `zoneinfo.ZoneInfo`.
- `converter.py`: Safe conversions between UTC and local timezones, formatting helpers, and naive datetime localization.
- `boundary.py`: Local calendar date to UTC start/end range calculations for database queries.

#### [MODIFY] [rawadhai_module/timezone_utils.py](file:///home/aneesh/Documents/python/docker/rawadhv2/rawadhai_backend/rawadhai_module/timezone_utils.py)
- Convert into a backward-compatibility module re-exporting all public functions from `rawadhai_module.utils.timezone_utils`.

#### [MOVE & UPDATE] [rawadhai_module/utils/timezone_utils/timezone_handling_guide.md](file:///home/aneesh/Documents/python/docker/rawadhv2/rawadhai_backend/rawadhai_module/utils/timezone_utils/timezone_handling_guide.md)
- Move existing `rawadhai_module/docs/timezone_handling_guide.md` to `rawadhai_module/utils/timezone_utils/timezone_handling_guide.md`.
- Rewrite and expand the guide using junior-developer-friendly point-by-point explanations and real-world analogies.

---

### Serializer Integration

#### [MODIFY] [cameraImageserializer.py](file:///home/aneesh/Documents/python/docker/rawadhv2/rawadhai_backend/rawadhai_module/api/serializer/cameraImageserializer.py)
- Update `CameraImageListSerializer` to resolve effective timezone via `get_request_timezone(request, camera=obj.camera, project=obj.project)`.
- Format `capture_date`, `capture_date_time`, `capture_date_date`, `created`, `created_time`, `created_date`, `modified`, `modified_time`, `modified_date` using the effective timezone.
- Fix historical field bugs in `created_time` and `capture_date_date`.

#### [MODIFY] [cameraserializer.py](file:///home/aneesh/Documents/python/docker/rawadhv2/rawadhai_backend/rawadhai_module/api/serializer/cameraserializer.py)
- Update `CameraListSerializer` / `CameraSerializer` to format datetime fields in effective site timezone.

#### [MODIFY] [projectserializer.py](file:///home/aneesh/Documents/python/docker/rawadhv2/rawadhai_backend/rawadhai_module/api/serializer/projectserializer.py) & [TimelapseVideosSerializer.py](file:///home/aneesh/Documents/python/docker/rawadhv2/rawadhai_backend/rawadhai_module/api/serializer/TimelapseVideosSerializer.py)
- Update project and timelapse serializers to format start/end/created/modified dates in effective site/request timezone.

---

### Filter & View Integration

#### [MODIFY] [camerasimagelistfilter.py](file:///home/aneesh/Documents/python/docker/rawadhv2/rawadhai_backend/rawadhai_module/api/filters/camerasimagelistfilter.py)
- Update filter methods to resolve request timezone (`X-Timezone` / `?timezone=` / `project.timezone`) before computing UTC range boundaries for database filtering.

---

### Practice Materials & Tests

#### [NEW] [practice/test_centralized_timezone_utils.ipynb](file:///home/aneesh/Documents/python/docker/rawadhv2/rawadhai_backend/practice/test_centralized_timezone_utils.ipynb)
- Create a comprehensive practice Jupyter notebook demonstrating all 14 requirements (resolution priority, IANA validation, header/query override, date range filtering, UTC conversion, API formatting).

#### [MODIFY] [rawadhai_module/tests/test_timezone_handling.py](file:///home/aneesh/Documents/python/docker/rawadhv2/rawadhai_backend/rawadhai_module/tests/test_timezone_handling.py)
- Add comprehensive unit tests covering header overrides, query parameter overrides, invalid timezone handling, UTC conversion, local date range filtering, and server independence.

---

## Verification Plan

### Automated Tests
- Run Django unit tests inside Docker container:
  ```bash
  docker compose exec rawadh_backend python manage.py test rawadhai_module.tests.test_timezone_handling
  ```

### Manual Verification
- Test sample request:
  `GET /api/v1/project/2/camera/CAM2/images/?daylight_with_current_live=true&page_size=180&page=1`
  - Default: Returns Saudi site time (`10:09:31`).
  - With Header `X-Timezone: Asia/Kolkata`: Returns India time (`12:39:31`).
  - With Query `?timezone=America/New_York`: Returns New York time (`03:09:31`).
