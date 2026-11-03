# Frontend Development Guide: Multi-Timezone Architecture & API Integration

## Executive Summary

The RawadhAI frontend application (`rawadhai_frontend`) already features a complete **Timezone Domain Module** located under `src/views/timezone/`. 

This document summarizes **already implemented frontend components** and specifies the remaining **API interceptor integration step** required to connect the frontend to the backend's new 4-tier priority timezone resolution engine.

---

## 1. Status of Implemented Frontend Features

| Feature / Component | File Location | Status | Summary |
| :--- | :--- | :--- | :--- |
| **Timezone Pinia Store** | `src/views/timezone/stores/timezoneStore.ts` | **Implemented ✅** | Manages `activeProjectTimezone`, `userTimezone`, and `userDisplayMode` ('site' vs 'user'). |
| **App Bar Header Popover** | `src/components/toolbar/MainAppbar.vue` | **Implemented ✅** | `<ProjectTimezonePopover />` is mounted in the header displaying project live clock and viewer local time. |
| **Display Mode Toggle** | `src/views/timezone/components/TimezoneToggle.vue` | **Implemented ✅** | Switches display mode between `site` (Site Timezone) and `user` (User Timezone). |
| **Project Timezone Selector** | `src/components/projects/EditProjectsPopup.vue` | **Implemented ✅** | `<ProjectTimezoneSelector />` allows admins to update project timezone with confirmation modal. |
| **Project Warning Modal** | `src/views/timezone/components/TimezoneWarningModal.vue` | **Implemented ✅** | Warns admin before submitting project timezone changes. |
| **Camera Timezone Display** | `src/components/cameras/EditCamerasPopup.vue` | **Implemented ✅** | `<CameraTimezoneDisplay />` renders camera timezone as **locked & inherited** from parent project. |

---

## 2. 4-Tier Backend Priority Chain & Frontend Alignment

The backend resolves timezones in this strict order:

```text
1. Valid X-Timezone HTTP header (sent when userDisplayMode === 'user')
       ↓ (if omitted or mode === 'site')
2. Valid ?timezone= query parameter
       ↓ (if omitted)
3. Project/Location site timezone (default from DB)
       ↓ (if missing)
4. UTC fallback
```

### Display Mode Behavior:
- **`userDisplayMode === 'site'` (Default)**:
  - Do **NOT** send `X-Timezone` header or `?timezone=` parameter.
  - Backend automatically formats all datetimes and applies date filters using the project's site timezone (`Asia/Riyadh`).
- **`userDisplayMode === 'user'`**:
  - Automatically attach `X-Timezone: <userTimezone>` (e.g. `Asia/Kolkata`) header via Axios interceptor.
  - Backend formats output timestamps directly in `Asia/Kolkata` for that request.

---

## 3. Immediate Action Item: Inject `X-Timezone` Header in Axios Interceptor

To connect the existing `timezoneStore` with outgoing API calls, update `src/stores/api.ts`:

### Update in `src/stores/api.ts`:

```typescript
import { useTimezoneStore } from '@/views/timezone/stores/timezoneStore';

// 1. AuthApi Request Interceptor
AuthApi.interceptors.request.use(
  config => {
    try {
      const customizeTheme = useCustomizeThemeStore();
      if (customizeTheme && customizeTheme.localCode) {
        config.headers['Accept-Language'] = customizeTheme.localCode;
      }

      // Inject X-Timezone header when user requested viewer timezone override
      const timezoneStore = useTimezoneStore();
      if (timezoneStore && timezoneStore.userDisplayMode === 'user' && timezoneStore.userTimezone) {
        config.headers['X-Timezone'] = timezoneStore.userTimezone;
      }
    } catch (e) {
      console.warn("Failed to set headers:", e);
    }
    return config;
  },
  error => Promise.reject(error)
);

// 2. NoAuthApi Request Interceptor
NoAuthApi.interceptors.request.use(
  config => {
    try {
      const customizeTheme = useCustomizeThemeStore();
      if (customizeTheme && customizeTheme.localCode) {
        config.headers['Accept-Language'] = customizeTheme.localCode;
      }

      const timezoneStore = useTimezoneStore();
      if (timezoneStore && timezoneStore.userDisplayMode === 'user' && timezoneStore.userTimezone) {
        config.headers['X-Timezone'] = timezoneStore.userTimezone;
      }
    } catch (e) {
      console.warn("Failed to set headers:", e);
    }
    return config;
  },
  error => Promise.reject(error)
);
```

---

## 4. Date Filtering & Calendar Picker Behavior

When the user selects a date on the calendar picker:
- Pass the date as `YYYY-MM-DD` string in query parameters:
  `GET /api/v1/project/2/camera/CAM2/images/?date=2026-09-12`
- Do **NOT** manipulate or convert the date string on the frontend. The backend's `TimezoneDateFilter` translates `2026-09-12` into exact UTC query boundaries based on the effective request timezone automatically.

---

## 5. Serialized API Response Field Usage

The backend API returns pre-formatted timezone-aware fields:

```json
{
  "id": 766,
  "name": "20260912100931033-schedule.jpg",
  "capture_date": "12-Sep-2026 12:39:31",
  "capture_date_time": "12:39",
  "capture_date_date": "2026-09-12",
  "created": "12-Sep-2026 12:45:03"
}
```

- **`capture_date`**: Use for tooltips and full timestamp text (`"12-Sep-2026 12:39:31"`).
- **`capture_date_time`**: Use for image card time overlays (`"12:39"`).
- **`capture_date_date`**: Use for timeline date headers (`"2026-09-12"`).

---

## 6. Verification Checklist

- [x] `useTimezoneStore` Pinia store managing site vs user mode.
- [x] `<ProjectTimezonePopover />` rendered in `MainAppbar.vue`.
- [x] `<ProjectTimezoneSelector />` rendered in `EditProjectsPopup.vue` with warning modal.
- [x] `<CameraTimezoneDisplay />` rendered in `EditCamerasPopup.vue` (locked & read-only).
- [ ] **Pending**: Add `X-Timezone` header injection in `src/stores/api.ts` request interceptors when `userDisplayMode === 'user'`.
- [ ] **Pending**: Test switching toggle to "User Time" and verify headers sent on API network requests.
