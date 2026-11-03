# Implementation Plan — Strict Site/Location Timezone API Response Formatting

Eliminate all server timezone fallbacks in API responses for camera and project resources. All date and time fields in API responses will be converted directly from stored UTC to the configured camera/project site location IANA timezone.

## User Review Required

> [!IMPORTANT]
> **Zero Server Timezone Dependency**
> The server's default timezone (`settings.TIME_ZONE = 'Asia/Kolkata'`) will no longer influence serialized API response fields for cameras, camera images, projects, timelapses, or drone/360 videos. All datetime fields will be localized strictly to the site location's configured IANA timezone (e.g. `Asia/Riyadh` for Saudi projects).

## Proposed Changes

### Core Timezone Utilities

#### [MODIFY] [timezone_utils.py](file:///home/aneesh/Documents/python/docker/rawadhv2/rawadhai_backend/rawadhai_module/timezone_utils.py)
- Add `get_entity_site_tz(instance)` helper to resolve the effective location `ZoneInfo` for any entity (`CameraImage`, `Camera`, `Project`, `CameraTimelapse`, `TimelapseConfig`, `VideoDron`, `Video360`).
- Add `format_datetime_for_site(dt, tz_or_instance, fmt=None)` helper to convert UTC datetimes directly to the site location timezone and format them without using Django/server timezone defaults.

---

### Camera & Image Serializers

#### [MODIFY] [cameraImageserializer.py](file:///home/aneesh/Documents/python/docker/rawadhv2/rawadhai_backend/rawadhai_module/api/serializer/cameraImageserializer.py)
- Update `CameraImageListSerializer` to format `capture_date`, `created`, `modified`, `capture_date_time`, `capture_date_date`, `created_time`, `created_date`, `modified_time`, and `modified_date` using the camera/project site location timezone.
- Fix historical bug in `get_created_time` (was using `modified` instead of `created`) and `get_capture_date_date` (was using `created` instead of `capture_date`).

#### [MODIFY] [cameraserializer.py](file:///home/aneesh/Documents/python/docker/rawadhv2/rawadhai_backend/rawadhai_module/api/serializer/cameraserializer.py)
- Update `CameraSerializer` and `CameraListSerializer` to format camera datetime fields (`last_captured`, `last_cronrun`, `created`, `modified`) in the site location timezone.

---

### Project & Media Serializers

#### [MODIFY] [projectserializer.py](file:///home/aneesh/Documents/python/docker/rawadhv2/rawadhai_backend/rawadhai_module/api/serializer/projectserializer.py)
- Update `ProjectListSerializer` / `ProjectSerializer` to format project date/time fields (`start_date`, `end_date`, `created`, `modified`) using the project's configured site timezone.

#### [MODIFY] [TimelapseVideosSerializer.py](file:///home/aneesh/Documents/python/docker/rawadhv2/rawadhai_backend/rawadhai_module/api/serializer/TimelapseVideosSerializer.py)
- Update `CameraTimelapseSerializer` and `TimelapseConfigSerializer` to format `created`, `modified`, `start_date`, and `end_date` using the project/camera site location timezone.

#### [MODIFY] [dronv2serializer.py](file:///home/aneesh/Documents/python/docker/rawadhv2/rawadhai_backend/rawadhai_module/api/serializer/dronv2serializer.py) & [video360V2serializer.py](file:///home/aneesh/Documents/python/docker/rawadhv2/rawadhai_backend/rawadhai_module/api/serializer/video360V2serializer.py)
- Ensure drone and 360 video serializers format `created` and `modified` in the project site location timezone.

---

### Documentation & Practice Materials

#### [NEW] [docs/site_timezone_api_serialization.md](file:///home/aneesh/Documents/python/docker/rawadhv2/rawadhai_backend/rawadhai_module/docs/site_timezone_api_serialization.md)
- Create markdown documentation detailing the site location timezone serialization rules and API examples.

#### [NEW] [practice/test_site_timezone_api_response.ipynb](file:///home/aneesh/Documents/python/docker/rawadhv2/rawadhai_backend/practice/test_site_timezone_api_response.ipynb)
- Create Jupyter practice notebook verifying timezone conversion from UTC to site timezone for API responses.

---

## Verification Plan

### Automated Tests
- Run Django test suite in Docker:
  ```bash
  docker compose exec rawadh_backend python manage.py test rawadhai_module.tests.test_timezone_handling
  ```

### Manual Verification
- Execute API request using `docker compose exec rawadh_backend python manage.py shell`:
  Verify endpoint `GET /api/v1/project/2/camera/CAM2/images/?daylight_with_current_live=true&page_size=180&page=1` output returns:
  `capture_date`: `"12-Sep-2026 10:09:31"` (Saudi site time matching `20260912100931033-schedule.jpg`).
