# Third-party components

Permissive licences only (CLAUDE.md rule 4). Every dependency is listed here before it is added.

| Component | Version | Licence | Used for | Notes |
|---|---|---|---|---|
| Eigen | 3.4.0 | MPL-2.0 (file-level) | linear algebra | header-only; no Eigen source is modified |
| GoogleTest | ≥ 1.14 | BSD-3-Clause | unit tests | test-only |
| Android NDK (sensor, camera2ndk, mediandk, vulkan) | r27 | Apache-2.0 / NOTICE | platform APIs | system libraries |
| Vulkan headers | 1.3 | Apache-2.0 / MIT | GPU compute | via NDK / Vulkan SDK |
| glslc (shaderc) | — | Apache-2.0 | shader compilation at build time | build tool only |
| EuRoC MAV dataset | — | CC BY 4.0 | evaluation | not redistributed; test fixture is synthetic in EuRoC layout |
| OpenCV (opencv-python) | ≥ 4.7 | Apache-2.0 | `tools/` only: target generation (`make_targets.py`), camera calibration, AprilTag 36h11 detection via the aruco module | never linked into `core/` or the app |
| Pillow | — | MIT-CMU (HPND) | `tools/make_targets.py`: PNG/PDF output with print resolution | `tools/` only |
| FFmpeg (`ffmpeg` executable) | any | LGPL-2.1+ / GPL-2+ depending on the build | `tools/render_run.py` only: H.264 and GIF encoding of rendered frames, run as a separate process | never linked, vendored or redistributed; the user's own install |
| **ARCore client library** (`com.google.ar:core`) | 1.57.0 | **proprietary — ARCore Additional Terms of Service** (not covered by this repository's MIT licence) | the `arcore` build flavor only: `mode=arcore`, the external comparison of ADR-0028 | fetched by Gradle, never vendored; the documented exception to the permissive-only rule (ADR-0028 §5); needs Google Play Services for AR on the phone |

Planned (not yet added): rapidyaml (MIT) for config parsing, if the in-tree flat-YAML subset parser (ADR-0008) ever becomes insufficient. The apriltag C library (BSD-2) is not needed: OpenCV's aruco module decodes the 36h11 family.

Explicitly **not** used, read, or ported: OpenVINS (GPL-3.0), VINS-Mono/Fusion (GPL-3.0), MSCKF_VIO (GPL-3.0), ORB-SLAM3 (GPL-3.0), Kimera (BSD, but not needed), Basalt (BSD core; not needed).
