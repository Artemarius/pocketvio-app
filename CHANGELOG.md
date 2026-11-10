# Changelog

All notable changes. Every result quoted here names its protocol; the protocols are in `docs/` and the full results page is [docs/results.md](docs/results.md).

## [Unreleased]

## [1.0.0] - 2026-11-10

The first release: real-time visual-inertial odometry on the phone, with an MSCKF in C++ and a Vulkan compute front-end, live on the Galaxy S24 Ultra and S25 Ultra. It is rc2 with one app fix. The estimator is unchanged since rc1 (frozen for 1.0.0), the 27 reference replays are byte-identical, and the sample log below replays to its reference trajectory byte for byte. What the release does is listed under [1.0.0-rc1](#100-rc1---2026-11-09), and what rc2 added under [1.0.0-rc2](#100-rc2---2026-11-10).

### Fixed
- A run that cannot be saved (its folder not writable) is reported on the run screen and in the log; it no longer closes the app at Stop.

### Measured (the headline numbers; every protocol is in [docs/results.md](docs/results.md))
- Stairs loop, two floors up and down: 0.29 m from the AprilTags after 43 m and a 7 m climb on the S24 Ultra (0.68 %, Protocol 43); 0.22 m after 44 m on the S25 Ultra (0.49 %, Protocol 46).
- Against ARCore on the same spot (S24 Ultra, 24 runs, Protocol 37): ARCore wins the walked loops (0.004 m against 0.116 m) by relocalising onto its start; pocketvio wins the fast hand spins (0.048 m / 1.1° against 0.459 m / 93°); ARCore makes single-frame jumps in 10 of 12 runs, pocketvio in none.
- Ten loops on the S25 Ultra: 0.055 m median return to origin (Protocol 34). Ten minutes at rest: 1.0 cm (S24 Ultra), 1.3 cm (S25 Ultra) (Protocols 14, 17). EuRoC MH_01: ATE 0.235 m (Protocol 35).
- Per frame: the filter's arithmetic takes 1.2 ms on either phone; the front-end kernels take 0.63 ms (Adreno 750) and 0.53 ms (Adreno 830) of GPU time (Protocol 45).

### Known limitations
- The filter is overconfident: position NEES 4–7× the expected value on EuRoC; on the stairs, a vertical σ of 1.7 cm against a 17 cm error.
- Odometry only: no map reuse, no loop closure, no relocalisation. The error grows with the path: 0.49–0.68 % of it on the two stairs loops.
- Static initialisation only. Monocular. The rolling shutter is modelled in the low-parallax rows only.
- Shipped profiles for the Galaxy S24 Ultra and S25 Ultra. Any other phone gets a generated profile that the wizard calibrates, but the wizard's IMU step does not resolve the accelerometer-to-gyroscope rotation. On the S25 Ultra treated as unknown, the room loops end 0.163 m from the tags against 0.073 m with the shipped profile (median of three, Protocol 47). Not yet run on a third phone.
- On the S25 Ultra the app's threads run on the efficient cores at about 1.4 GHz, and the filter's live p99 reaches 19–41 ms against the 33 ms frame interval; no frame is dropped (Protocol 45).

### Assets
- `pocketvio-1.0.0-arm64.apk`: the app (hsr flavor; arm64, Android 12+, Vulkan 1.3), release-signed. It installs over rc2. Over rc1 or a locally built debug APK it does not: uninstall that first, after copying off the runs you want to keep (`Android/data/dev.artemarius.pocketvio/files/runs`).
- `pocketvio-1.0.0-linux-x86_64.tar.gz`: `pocketvio_replay`, `pocketvio_timeoffset`, `pocketvio_calib_imu`, `pocketvio_calib_camera` and `pocketvio_profile` (Release, Ubuntu 24.04, static libstdc++), with the device profiles and the printable targets.
- `pocketvio-sample-stairs.tar.gz`: the S24 Ultra stairs loop's log, without the frames between the AprilTag windows, plus its tracks and the reference result. The replay reproduces the reference trajectory byte for byte, and the tag evaluation gives 0.293 m (instructions in its `README.txt`).
- `stairs_1x.mp4`, `stairs_4x.mp4`: the desktop render of the same run.

## [1.0.0-rc2] - 2026-11-10

The second release candidate: the APK is signed with the release key, a phone without a shipped profile gets one generated from its own camera and sensor data, and the S25 Ultra has its stairs loop, its pipeline timings and its on-phone calibration. The estimator is unchanged since rc1 (frozen for 1.0.0); the 27 reference replays are byte-identical.

### Added
- **A profile for a phone without a shipped one** (ADR-0032). It is generated from the phone's own camera2 characteristics and sensor list:
  - the back camera without OIS and with the widest field of view that offers 640×480;
  - the camera's own intrinsics as a start;
  - the camera-to-IMU rotation from `SENSOR_ORIENTATION`;
  - a conservative default IMU noise.

  The wizard calibrates it as before, and its camera step now also measures the rolling-shutter readout. The home screen says where the profile comes from. `pocketvio_profile` does the same on the desktop from a log's `sensor.txt`. Protocol 44: with the wizard's keys, the two phones treated as unknown end the 27 reference logs as close to the start as with their shipped profiles. Protocol 47, on the S25 Ultra: the phone generates the same profile and makes the same wizard fits as the desktop, but the wizard's IMU step does not resolve the accelerometer-to-gyroscope rotation (it stays out by ADR-0026's gate), and without it the room loops end about twice as far from the tags (0.163 against 0.073 m median of three; the shipped profile closer on all four loops). Not yet run on a third phone.
- `pocketvio_replay --mode track --gpu` runs the Vulkan front-end on a logged run (a `POCKETVIO_VULKAN` build), with per-frame GPU and host timings.
- A saved run's `trajectory.csv` covers the whole run from the initialisation, with the time and the attitude of every frame (`#i,t_ns,t_s,x,y,z,qw,qx,qy,qz`); older four-column files still load in the viewer.
- Documentation: `docs/pipeline.md` (one frame end to end, with per-stage timings on both phones, Protocol 45) and one page per module under `docs/modules/`.

### Measured
- The S25 Ultra on the stairs loop (Protocol 46): 0.22 m from the tags after 44 m and a 7 m climb (0.49 %).
- Protocol 45 on both phones, without the camera: the filter's arithmetic is 1.2 ms per frame on either; the same filter output and the same GPU tracks on both phones, byte for byte; 0.63 ms (Adreno 750) and 0.53 ms (Adreno 830) of GPU time per frame over 3504 frames in motion.

### Fixed
- A run without the full log reported the previous run's logged-frame count in its status and `run.txt`.
- The developer panel's benches opened camera `"2"` instead of the profile's camera.

### Changed
- The APK is signed with the release key. It does not install over a debug-signed build (rc1's or a local one): uninstall that first, after copying off the runs you want to keep (`Android/data/dev.artemarius.pocketvio/files/runs`).
- Every capture opens the profile's `camera.id` instead of a fixed `"2"`. An adb `--es camera_id` still overrides it.
- `cam0/sensor.txt` gains `lens_facing`, `ois_modes`, `logical_multi_camera` and `yuv_640x480`.

### Known limitations (in addition to rc1's)
- The wizard's IMU step does not resolve the accelerometer-to-gyroscope rotation (Protocol 47); a generated profile lacks it.
- On the S25 Ultra the app's threads run on the efficient cores at about 1.4 GHz, and the filter's live p99 reaches 19–41 ms against the 33 ms frame interval; no frame is dropped (Protocol 45 reading 9).

### Assets
- `pocketvio-1.0.0-rc2-arm64.apk`: the app (hsr flavor; arm64, Android 12+, Vulkan 1.3), release-signed.
- `pocketvio-1.0.0-rc2-linux-x86_64.tar.gz`: `pocketvio_replay`, `pocketvio_timeoffset`, `pocketvio_calib_imu`, `pocketvio_calib_camera` and `pocketvio_profile` (Release, Ubuntu 24.04, static libstdc++), with the device profiles and the printable targets.

## [1.0.0-rc1] - 2026-11-09

The first release candidate: the complete pipeline, live on the Galaxy S24 Ultra and S25 Ultra, with the estimator frozen for 1.0.0. It is a pre-release because the APK is signed with the CI's debug key, the S25 Ultra's on-device calibration and ARCore comparison are still to run, and the showcase documents are not complete.

### What it does

- **Visual-inertial odometry on the phone, in real time.** An MSCKF with IMU-pose clones, preintegrated propagation, first-estimate Jacobians, nullspace projection and χ² gating (ADR-0013), with the camera–IMU time offset (ADR-0016) and extrinsics (ADR-0017) as filter states. It adds zero-velocity updates, a zero-flow pose anchor (ADR-0018), a flow-bounded hover velocity (ADR-0020), re-acquisition after a run-away (ADR-0021), low-parallax rows (ADR-0022) and a rate-proportional gyro noise (ADR-0023). On the S24 Ultra the filter takes 3.7 ms per frame at the median and the front-end 6.8 ms, of the 33 ms frame interval (Protocol 43).
- **The IMU first.** Raw `*_UNCALIBRATED` streams at 475 Hz (ADR-0004, Protocol 1); noise densities from 4–6.5 h Allan variance (Protocol 2). The accelerometer's scale, misalignment and bias (ADR-0019), the accelerometer-to-gyroscope rotation (ADR-0026) and the gyro scale (ADR-0027) are applied at the boundary.
- **A Vulkan compute front-end.** Zero-copy camera import from `AHardwareBuffer` (ADR-0010). Pyramid, FAST-9 and pyramidal KLT kernels take 0.38 ms (Adreno 750) and 0.26 ms (Adreno 830) of GPU time per frame and match their CPU references bit-exactly and to 0.000 px (ADR-0011, Protocol 7). The track manager does a forward–backward check, fundamental-matrix RANSAC and bucketed replenishment (ADR-0012). The gyro predicts each track's search start (ADR-0015), and the exposure comes from a blur budget under rotation (ADR-0024).
- **The app.** Home screen with the device check, run screen (camera with tracks, live 3D trajectory and map, a full-log switch), run viewer, runs list, a guided calibration wizard for the IMU and the camera (ADR-0031), and a developer panel (ADR-0030).
- **Re-calibration on the phone.** The six-position IMU fits and the checkerboard calibration in C++, without OpenCV, match Python/OpenCV (Protocols 38–39). The S24 Ultra's shipped camera profile comes from the wizard (Protocols 41–42). On a phone without a profile the wizard fills in only the calibration keys (see Known limitations).
- **Desktop tools.** `pocketvio_replay`: deterministic replay of EuRoC and of phone logs, with evaluation against ground truth. `pocketvio_timeoffset`, `pocketvio_calib_imu`, `pocketvio_calib_camera`. Python tools for Allan variance, the AprilTag evaluation (`tools/apriltag_eval.py`) and renders of a logged run (`tools/render_run.py`).

### Measured

- Stairs loop, two floors up and down, S24 Ultra: 0.29 m from the AprilTags after 43 m and a 7 m climb, 0.68 % of the path (Protocol 43).
- Against ARCore on the same spot (S24 Ultra, 24 runs): ARCore wins the walked loops, 0.004 m against 0.116 m, by relocalising. pocketvio wins the fast hand spins, 0.048 m / 1.1° against 0.459 m / 93°. ARCore makes single-frame jumps in 10 of 12 runs; pocketvio in none (Protocol 37).
- S25 Ultra, ten loops: 0.055 m median return to origin (Protocol 34). Ten minutes at rest: 1.0 cm (S24 Ultra), 1.3 cm (S25 Ultra) (Protocols 14, 17).
- EuRoC MH_01: ATE 0.235 m (Protocol 35).

### Known limitations

- The filter is overconfident: position NEES 4–7× the expected value on EuRoC; on the stairs, a vertical σ of 1.7 cm against a 17 cm error.
- Odometry only: no map reuse, no loop closure, no relocalisation. The error grows with the path, about 1 % on the walks measured.
- Static initialisation only. Monocular. The rolling shutter is modelled in the low-parallax rows only.
- Device profiles exist for the Galaxy S24 Ultra and S25 Ultra only. Any other phone runs on the S24 Ultra's profile, and its runs are labelled uncalibrated. A fallback built from the phone's own HAL intrinsics and default noise is not built yet.

### Assets

- `pocketvio-1.0.0-rc1-arm64.apk`: the app (hsr flavor; arm64, Android 12+, Vulkan 1.3). It is signed with the CI's debug key, so it does not install over a locally built debug or release APK: uninstall that first. The `arcore` comparison flavor is not distributed.
- `pocketvio-1.0.0-rc1-linux-x86_64.tar.gz`: `pocketvio_replay`, `pocketvio_timeoffset`, `pocketvio_calib_imu` and `pocketvio_calib_camera` (Release, Ubuntu 24.04, static libstdc++), with the device profiles and the printable targets.
- `pocketvio-sample-stairs.tar.gz`: the stairs loop's log, without the frames between the AprilTag windows, plus its tracks and the reference result. The replay reproduces the reference trajectory byte for byte, and the tag evaluation gives 0.293 m (instructions in its `README.txt`).
- `stairs_1x.mp4`, `stairs_4x.mp4`: the desktop render of the same run.
