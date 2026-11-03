# Video Processing Optimization — Implementation Plan

## What We Are Solving

Think of the current video/timelapse generation like a factory that tries to do **everything at once** with no limits. When a large timelapse job is started, it:

- Downloads **every image** into RAM simultaneously using 8 parallel threads
- Feeds all those images **into memory at once** to FFmpeg (`b''.join(open(img,'rb').read() for img in image_paths)` — this loads the entire dataset into a single Python bytes object in RAM)
- Runs FFmpeg with **no CPU thread limit** — it automatically uses all available cores
- Allows **unlimited video jobs to run at the same time**, competing for the same CPU and RAM
- Has **no dedicated queue** for heavy video work — it shares workers with normal API tasks

This is why the server crashes. The fix is to add walls, speed governors, and limits — not to rebuild the factory.

---

## What We Will Change (and What We Will NOT Change)

> [!IMPORTANT]
> We will make the **smallest safe changes** to fix stability. We will not rewrite working business logic, routing, email, progress tracking, or S3 upload mechanics.

---

## Proposed Changes

### 1. `timelapse_videos_with_progress.py` — FFmpeg & Image Download Fixes

#### Critical Bug: Entire image dataset loaded into RAM
**Current code (line 269):**
```python
input_data = b''.join(open(img, 'rb').read() for img in image_paths)
ffmpeg.run(output, input=input_data, ...)
```
This is the primary crash cause. For 500 images at 2MB each = **1 GB+ in one Python variable**.

**Fix:** Use FFmpeg's native `concat` demuxer with a text file listing all images on disk instead of piping bytes through Python. FFmpeg reads each file itself, never loading all into memory simultaneously.

#### Critical Fix: Image download parallelism
**Current:** 8 parallel S3 download threads.
**Fix:** Reduce to 3 parallel threads. This is still fast enough but drastically reduces RAM spike during the download phase.

#### Critical Fix: FFmpeg CPU throttling
**Current (non-HIGH performance path):** `'threads': 2` — but only for non-HIGH performance. The default `performance` parameter in `video_process_project_files` is `VideoPerformanceLevel.HIGH.value`, meaning the CPU limit is **never applied** for scheduled/default timelapse jobs.
**Fix:** Always cap FFmpeg threads. For scheduled/low-performance jobs, cap at 1–2 threads. Add `ultrafast` or `superfast` preset to further reduce CPU peak. Add a small `nice` level via subprocess if needed.

#### Fix: NamedTemporaryFile for video output
**Current:** Video is created into a `NamedTemporaryFile` opened without `delete=False`, then read back for upload while the file is still "open". On some Linux configs this can cause issues.
**Fix:** Use `tempfile.mkstemp()` with explicit cleanup in `finally`, ensuring the file exists for the upload then is definitely deleted.

#### Fix: Music temp file cleanup on failure
The `TimelapseMusicHandler.cleanup()` is only called on the **success path** (line 367). If an exception happens, the temp audio file is orphaned.
**Fix:** Move `music_handler.cleanup()` into the `finally` block.

---

### 2. `tasks.py` — Concurrency & Queue Routing

#### Critical Fix: Unlimited parallel video jobs
**Current:** `process_camera_timelapse` spawns one `video_process_project_files.delay()` per camera — all running simultaneously on the same worker pool with no limit.
**Fix:** Route video tasks to a **dedicated Celery queue** named `video_processing`. This queue will be consumed by a separate worker started with `--concurrency=1`, so only **one heavy video job runs at a time**. All other queues (API tasks, translations, scheduled image fetching) are unaffected.

#### Fix: Cache lock is cleared before video finishes
**Current (lines 264–266):**
```python
finally:
    cache.delete(cache_key)
```
The cache key is deleted **as soon as** `video_process_project_files.delay()` is called, not when the video job finishes. This means duplicate jobs can be queued immediately.
**Fix:** Don't use the cache lock here to gate the video task dispatch — since the video runs asynchronously in a worker, the lock serves no real deduplication purpose at the task dispatch level. Instead, document this clearly and rely on the queue concurrency limit (1 at a time) as the real protection. Alternatively, set the cache key with a long TTL and only clear it from inside `video_process_project_files` itself on completion/failure.

#### Fix: Task routing
Add `CELERY_TASK_ROUTES` for video tasks in `settings.py` so they go to the `video_processing` queue automatically.

---

### 3. `settings.py` — New Settings

Add:
```python
TIMELAPSE_VIDEO_GENERATE = {
    ...existing...,
    "VIDEO_MAX_FFMPEG_THREADS": config("VIDEO_MAX_FFMPEG_THREADS", cast=int, default=2),
    "VIDEO_S3_DOWNLOAD_WORKERS": config("VIDEO_S3_DOWNLOAD_WORKERS", cast=int, default=3),
}
```
Add task route for video tasks:
```python
CELERY_TASK_ROUTES = {
    'translation_management.tasks.*': {'queue': 'translation'},
    'rawadhai_module.timelapse_videos_with_progress.video_process_project_files': {'queue': 'video_processing'},
}
```

---

### 4. Docker Compose Files — Dedicated Video Worker

Add a new `rawadh_celery_video` service to the dev and production/staging compose files:
```yaml
rawadh_celery_video:
  image: ...  # same image as other workers
  command: celery -A rawadhai worker -l info -Q video_processing --concurrency=1
```
`--concurrency=1` is the key safety valve. Only one video job processes at a time on this worker.

> [!IMPORTANT]
> The existing `rawadh_celery` worker that handles the `celery` and `translation` queues will **not** be changed. This means normal operations are never blocked by video processing.

---

## Files to Change

| File | Type | What Changes |
|---|---|---|
| [timelapse_videos_with_progress.py](file:///home/aneesh/Documents/python/docker/rawadhv2/rawadhai_backend/rawadhai_module/timelapse_videos_with_progress.py) | MODIFY | Fix memory loading bug, reduce download threads, cap FFmpeg threads, fix cleanup |
| [tasks.py](file:///home/aneesh/Documents/python/docker/rawadhv2/rawadhai_backend/rawadhai_module/tasks.py) | MODIFY | Fix cache lock logic, add queue routing |
| [settings.py](file:///home/aneesh/Documents/python/docker/rawadhv2/rawadhai_backend/rawadhai/settings.py) | MODIFY | Add new config values, add task route |
| [docker-compose-dev.yml](file:///home/aneesh/Documents/python/docker/rawadhv2/rawadhai_backend/docker-compose-dev.yml) | MODIFY | Add video worker service |
| [docker-compose-celery-prod.yml](file:///home/aneesh/Documents/python/docker/rawadhv2/rawadhai_backend/docker-compose-celery-prod.yml) | MODIFY | Add video worker service |
| [docker-compose-celery-staging.yml](file:///home/aneesh/Documents/python/docker/rawadhv2/rawadhai_backend/docker-compose-celery-staging.yml) | MODIFY | Add video worker service |
| [docker-compose-celery-staging-aws.yml](file:///home/aneesh/Documents/python/docker/rawadhv2/rawadhai_backend/docker-compose-celery-staging-aws.yml) | MODIFY | Add video worker service |
| `rawadhai_module/docs/video_processing_optimization.md` | NEW | Required markdown documentation |
| `practice/rawadhai_module/video_processing_optimization.ipynb` | NEW | Required practice notebook |

---

## Key Technical Decisions

### Why concat demuxer instead of pipe?
The `concat` demuxer makes FFmpeg open each image file on its own — it never needs more than ~1 image worth of RAM at a time. This is how FFmpeg was **designed** to process large numbers of images. The current pipe approach was chosen for simplicity but is catastrophically memory-hungry.

### Why concurrency=1 for video worker?
One video job alone will use 100–200% CPU for minutes. Two jobs simultaneously = 200–400% = server dead. `--concurrency=1` is a hard, simple guarantee. The queue itself becomes the line — jobs wait their turn.

### Why not reduce video quality?
The user explicitly asked not to reduce quality. The `vcodec=libx264` with `pix_fmt=yuv420p` is kept. We only add `preset=slower` (which actually produces **better** quality at lower CPU/memory than `faster`) and thread caps.

### Why keep `superfast`/`ultrafast` as an option?
For scheduled/default timelapse jobs (which already use `LOW` performance level), `superfast` preset reduces CPU load significantly with a minor quality impact on intermediate frames. The final video quality is still excellent. This is the right tradeoff for scheduled background jobs.

---

## Open Questions

> [!IMPORTANT]
> **Do you want to keep one video job at a time (concurrency=1), or allow 2 simultaneous video jobs?**
> Concurrency=1 is the safest. If you have a powerful server, 2 could work. My recommendation is 1 for now.

> [!IMPORTANT]
> **Should the video worker run on the same server as the main worker, or a separate server?**
> The plan currently puts both on the same Docker host. If the server is small (2–4 CPU cores, 4–8 GB RAM), this is fine with concurrency=1. Let me know if they should be on different machines.

---

## Verification Plan

### Docker Check
- Restart Docker services and confirm `rawadh_celery_video` worker starts and connects to Redis.
- Confirm `rawadh_celery` worker does NOT pick up `video_processing` queue tasks.

### Manual Test
- Trigger a timelapse generation for a large project (many cameras, long date range).
- Monitor RAM usage inside the container with `docker stats` — should stay stable, not spike to OOM.
- Monitor CPU — should stay moderate (not pin all cores) for the duration of encoding.
- Confirm the video is still generated and uploaded to S3 correctly.
- Confirm temp files are cleaned up after the job finishes.

### Failure Test
- Kill the video worker mid-job — confirm temp files are cleaned up by the `finally` block.
- Confirm the next job starts cleanly without duplicate processing.
