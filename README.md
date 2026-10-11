# Camera — GyroLock V28.6 (gyro/flow fusion fix)

This revision removes the experimental additive sparse-optical-flow correction from the per-frame render path. The V28.5 flow estimator integrated global block-match displacement and then added that result on top of the gyro-derived correction. Since both estimates can represent the same camera rotation, this could apply motion compensation twice and create visible jelly/bounce.

The 240-point gyro trajectory, current stabilization gain, Vulkan render path, 30 FPS output timestamp normalization, audio remux, and portrait rotation handling are retained. This change is intended to eliminate conflicting correction; it is not a claim that rolling-shutter wobble is fully corrected. The optical-flow class remains in the source tree but is not invoked during rendering.

## Build

Push the contents of this project folder to the root of a GitHub repository and run **Actions → Build GyroLock Vulkan APK → Run workflow**. The workflow installs the Android SDK/NDK and builds a debug APK. This environment did not run a full Android SDK build, so a successful APK build has not been verified here.

Output target remains 30 FPS with the original audio remuxed without re-encoding. Actual stabilization quality depends on gyro alignment, camera readout timing, and device codec behavior.

## Motion Photo experimental mode

This branch adds a continuous low-resolution CameraX video buffer, full-quality still capture, a 1.5-second post-shutter capture window, gyro stabilization, and JPEG+MP4 Motion Photo packaging with Google Camera XMP metadata. Still capture uses CameraX `CAPTURE_MODE_MAXIMIZE_QUALITY`; this does not guarantee hardware HDR because the device camera may not expose an HDR capture mode. Motion Photo gallery support varies by gallery app and OEM firmware. The rolling buffer is trimmed to approximately the last three seconds before stabilization. This is an experimental device-test build; verify playback and metadata on the target Oppo device.

## Patch 2.9.1 — Motion Photo buffer/orientation/flash
- Keeps the rolling low-resolution CameraX recording running from camera startup and computes the post-shutter wait from the shutter timestamp (not the delayed JPEG callback).
- Preserves portrait/landscape orientation metadata when trimming the MP4 buffer, preventing portrait clips from being displayed sideways.
- Adds a blinking torch mode (roughly 260 ms per phase) rather than leaving the LED continuously lit.
- Slightly strengthens and smooths the gyro correction curve for short Motion Photo clips. Real-device testing is still required; software gyro availability varies by device.


### Fix 2.9.3 experimental
- Shutter is gated until at least 1.6 seconds of camera video buffer is available.
- Flash control now uses CameraX still-photo flash, which fires once at the still exposure instead of blinking continuously.
- The stabilizer applies a short 1.20x-to-1.12x easing crop at the beginning of the video only; the JPEG still remains unchanged.
- MP4 trim uses MediaMetadataRetriever for source rotation and no longer creates the muxer twice.


## Fix 2.9.4 — Motion Photo interoperability and flash timing
- Writes both current Motion Photo Format 1.0 `Camera:MotionPhoto` + `GContainer:Directory` item metadata and legacy Google Camera `GCamera:MicroVideoOffset` fields. The primary item length includes the inserted XMP APP1 segment; the video item is appended at EOF.
- Uses the recommended `...MP.jpg` naming pattern so readers that filter by filename can recognize the container.
- Publishes a companion MP4 in Movies/Camera because TikTok and many social apps import ordinary video files rather than extracting embedded motion data from a JPEG Motion Photo. A standard Motion Photo cannot force every third-party app to treat its JPEG as a video.
- With flash enabled, a short torch focus-assist precedes the capture, then CameraX's one-shot flash capture is used; the clip is shortened after the shutter so the still-frame marker is toward the end. Without flash, the marker remains centered.
- This is a source-level change; no successful Android build or device-level interoperability test is claimed until GitHub Actions and the Oppo A54 test confirm it.


## Fix 2.9.5 — Restore legacy still-frame marker and flash-aware HDR
- Retains current Motion Photo Format 1.0 `Camera:MotionPhoto` / `GContainer:Directory` metadata and restores the legacy `GCamera:MicroVideo`, `GCamera:MicroVideoOffset`, and `GCamera:MicroVideoPresentationTimestampUs` fields for readers that rely on the older marker convention.
- Corrects the primary `GContainer:Item:Length` value to `0` as defined by Motion Photo Format 1.0, while retaining the actual appended MP4 length in the MotionPhoto item and legacy offset.
- Requests the rear camera's HDR scene mode only while flash is off; when flash is enabled, scene HDR is disabled so the still flash capture can operate normally. This is a hardware scene-mode request, not guaranteed multi-frame computational HDR; the camera may ignore it or not expose HDR support.
- No guarantee that TikTok or every social app will parse an embedded Motion Photo JPEG. The companion MP4 remains available for normal video uploads.
- Source changes have not been device-tested; verify GitHub Actions build and test on the Oppo A54.
