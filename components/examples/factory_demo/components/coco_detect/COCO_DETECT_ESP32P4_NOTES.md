# COCO Detect on ESP32-P4 — troubleshooting notes

This file documents the issues we hit while bringing the upstream
`coco_detect` component to ESP32-P4 (the same chip used on the
ESP32-P4-Eye board) and the matching ESPHome `yolo11_detection`
wrapper. It is meant as a checklist when "no detections" or random
crashes are observed.

## 1. Pick exactly one model variant

`coco_detect/Kconfig` defines four mutually exclusive YOLO11n variants:

| variant         | input  | typical use                              |
|-----------------|--------|------------------------------------------|
| `s8_v1`         | 224    | smallest, fastest, good baseline on P4   |
| `s8_v2`         | 224    | retrained, slightly better mAP           |
| `s8_v3`         | 224    | default in upstream factory_demo         |
| `320_s8_v3`     | 320    | best mAP, ~2× slower, needs more PSRAM   |

The Kconfig `choice "default model"` only takes effect if the matching
`FLASH_COCO_DETECT_YOLO11N_*` symbol is also enabled. Any mismatch
between the two ends with `dl::Model` reading a header that does not
match the embedded weights and the model either returns garbage scores
(filtered out by NMS → "no detections") or asserts at allocation time.

The included `sdkconfig.defaults` ships with
`CONFIG_FLASH_COCO_DETECT_YOLO11N_S8_V1=y` and
`CONFIG_FLASH_COCO_DETECT_YOLO11N_S8_V3=n`, but the Kconfig default for
`COCO_DETECT_YOLO11N_*` is still `S8_V3`. Without an explicit
`CONFIG_COCO_DETECT_YOLO11N_S8_V1=y` line in `sdkconfig.defaults`, the
choice drops back to a disabled variant and the build embeds nothing.
**Add an explicit `CONFIG_COCO_DETECT_YOLO11N_S8_V1=y` to
`sdkconfig.defaults`** (or whatever variant your firmware embeds).

## 2. PSRAM threshold for `param_copy`

`coco_detect.cpp` decides whether to copy parameters into PSRAM using
`heap_caps_get_total_size(MALLOC_CAP_SPIRAM) < 1024*1024*9`. On a
16 MB / 32 MB ESP32-P4-Eye this is fine, but if PSRAM init fails (bad
pin remap, wrong speed, XIP not enabled) the heap can report 0 and the
flag flips to `false`, which then forces the loader down the
flash-rodata path and crashes when the flash mapping is too small.

Confirm at runtime with:
```c
ESP_LOGI("psram", "psram=%u", heap_caps_get_total_size(MALLOC_CAP_SPIRAM));
```
You want ≥ 9 MB free for `param_copy=true` and clean YOLO11 inference.

## 3. Image preprocessor on P4

The constructor uses a P4-specific code path:
```cpp
#if CONFIG_IDF_TARGET_ESP32P4
m_image_preprocessor = new dl::image::ImagePreprocessor(m_model, {0,0,0}, {255,255,255});
#else
m_image_preprocessor = new dl::image::ImagePreprocessor(
    m_model, {0,0,0}, {255,255,255}, DL_IMAGE_CAP_RGB565_BIG_ENDIAN);
#endif
```
Make sure `CONFIG_IDF_TARGET_ESP32P4=y` is actually set in your build —
if you compile against a P4 SoC but the macro is missing the model is
fed RGB565 big-endian and you'll see only sporadic detections.

## 4. ESPHome `yolo11_detection` wrapper

The ESPHome wrapper used to set three conflicting build flags at once,
including a lowercase `s8_v3` typo:
```python
cg.add_build_flag("-DCONFIG_YOLO11_DETECT_S8_V1=1")
cg.add_build_flag("-DCONFIG_COCO_DETECT_YOLO11N_320_s8_v3=1")  # <- typo
cg.add_build_flag("-DCONFIG_YOLO11_DETECT_MODEL_TYPE=0")
```
That is what produced the "doesn't recognise anything / crashes" symptom
on P4. The fixed version is on
`youkorr/test2_esp_video_esphome@claude/fix-coco-detection-esp32-DKktc`
and exposes a `model_variant:` option:

```yaml
yolo11_detection:
  id: yolo_detect
  camera_id: tab5_cam
  canvas_id: ai_vision_canvas
  model_variant: s8_v1     # s8_v1 | s8_v2 | s8_v3 | 320_s8_v3
  score_threshold: 0.35
  nms_threshold: 0.5
  detection_interval: 8
```
