# One frame through pocketvio

Status: Session 37 (2026-11-10, both phones). This page follows one camera frame from the sensor to the pose on the screen. For each step it gives what the step does, where it lives, the decision record behind it, and what it costs on the phone. The numbers are Protocol 45 (below) unless another protocol is named; its S25 Ultra column was added in Session 37, together with a trace of the app's threads during live runs.

```
 t = 0          exposure starts (first row)                     camera HAL, ultrawide, 640×480 @ 30 fps
 t ≈ 15.5 ms    frame time: centre row, mid-exposure             ADR-0007 (exposure 20 ms, readout 11.1 ms)
 t ≈ 60 ms      AImage arrives (AHardwareBuffer)                 camera thread ─┐
                  zero-copy import ─► pyramid ─► FAST ─► KLT ►─  Vulkan, 0.63 ms of GPU time
                  readbacks, backward KLT, track manager         CPU
 t ≈ 65 ms      FrameTracks queued (≤ 4)                         front-end 4.6 ms median live
 t ≤ 70 ms      filter thread (5 ms poll) takes the frame        IMU already past the frame time
                  propagate ─► clone ─► updates ─► marginalise   MSCKF, 1.2 ms of work, 3.7–5.4 ms live
 t ≈ 75 ms      pose of time 15.5 ms published to the UI         ~60 ms behind the scene

 in parallel:   IMU events, 475 Hz, received 3.9 ms after the sample, one per delivery
```

## 1. Exposure and the frame's time

The ultrawide (camera id 2 on both phones) runs at 640×480 and 30 fps with OIS and EIS off and the focus locked (CLAUDE.md constraints 6–7). The HAL timestamp is the start of exposure of the first row on `CLOCK_BOOTTIME`, the same clock as the IMU, so no clock mapping is needed (Protocol 3). The frame's time for the filter is the centre row at mid-exposure: start + exposure/2 + readout/2 + the profile's `t_d` (ADR-0007). On the stairs loop the exposure is 20 ms (11.8–20 ms) and the readout 11.09 ms, which puts the frame time 15.5 ms after the start. The exposure itself is set from the turn rate when the blur budget is on (ADR-0024: 4 px of motion blur at most, the gain makes up the light).

## 2. The IMU stream, in parallel

`ASensorEventQueue` delivers the uncalibrated gyroscope and accelerometer at 475 Hz on the S24 Ultra (471.7 Hz on the S25 Ultra), one event per delivery, each 3.9 ms after its sample time (median; p99 6.2 ms gyro, 9.3 ms accel; stairs log). The filter thread wakes every 5 ms, pairs gyro and accelerometer samples onto the gyro's timestamps (ADR-0007 §2), applies the profile's calibration at the boundary (accelerometer scale, misalignment and bias, ADR-0019; accelerometer-to-gyro rotation, ADR-0026; gyro scale, ADR-0027) and appends them to the filter's buffer. Propagation happens later, when a frame is processed (section 6). Because the image arrives about 44 ms after its frame time and the IMU about 4 ms after each sample, the IMU is always past the frame time when the frame is ready: the camera, not the IMU, sets the latency.

## 3. Import and the GPU front-end

The `AImageReader` callback runs on the camera thread (`android/app/src/main/cpp/vio.cpp`, `onImage`). The frame's `AHardwareBuffer` is imported into Vulkan without a copy (ADR-0010) and the Y plane copied into pyramid level 0 on the GPU. One command buffer then records the pyramid (levels 1–3 from the full-resolution level 0), FAST-9 with non-maximum suppression, and the forward pyramidal KLT (21×21, 3 levels) of the 150 live tracks from the previous frame (ADR-0011). With the gyro search start on (ADR-0015 §6, opt-in), each track's search starts at its gyro-predicted position. The backward KLT runs in its own submission after the forward tracks are read back.

| GPU stage, 640×480 (GPU timestamps), median / p99 | S24 Ultra, Adreno 750 | S25 Ultra, Adreno 830 |
|---|---|---|
| pyramid, levels 1–3 | 0.031 / 0.032 ms | 0.027 / 0.028 ms |
| FAST-9 + NMS (1899 corners median) | 0.122 / 0.139 ms | 0.102 / 0.115 ms |
| KLT forward, 150 tracks | 0.237 / 0.275 ms | 0.199 / 0.221 ms |
| KLT backward | 0.233 / 0.271 ms | 0.198 / 0.220 ms |
| **total GPU time** | **0.626 / 0.709 ms** | **0.528 / 0.571 ms** |

Both columns are Protocol 45: the same 3504 frames of the stairs log, the same arm64 binary. Protocol 8's figures for the same four stages were 0.57 ms (S24 Ultra) and 0.47 ms (S25 Ultra) on a static scene; 150 tracks in motion cost about a tenth more. The GPU is not the cost of a frame: the host side is (next section).

Every kernel has a scalar CPU reference in the same module and a test that compares them (constraint 3; `core/tests/test_vk_kernels.cpp`). Protocol 45 extends that check from three frames to a whole log: the GPU front-end on the Adreno 750 gives the same tracks as the CPU reference on the 3504 frames of the stairs log to within 0.014 px, until frame 154, where one track is kept by one and dropped by the other (a threshold decision moved by a hundredth of a pixel); after that the two track sets carry different identifiers. Section 6 reading 4 gives what that does to the result. The Adreno 830's tracks on the same log are byte-identical to the Adreno 750's (reading 8).

## 4. The track manager, on the CPU

`GpuFrontEnd::finishFrame` reads back the corners and the forward tracks, runs the backward KLT round trip, and updates the track manager (ADR-0012): forward-backward check at 0.5 px, fundamental-matrix RANSAC (normalised 8-point, Sampson distance 1 px) on undistorted ideal coordinates, border handling, and replenishment from the bucketed corners back to 150 tracks. On the stairs log 99.3 % of the tracks survive the KLT, 98.0 % the backward check and 96.7 % RANSAC per frame (medians); a random observation belongs to a track 68 frames long (median). The app also downloads a 160×120 thumbnail for the run screen. The result, `FrameTracks` (frame time + one `(id, u, v)` per track), goes into a queue of at most four frames for the filter thread.

| Front-end on the camera thread | S24 Ultra | S25 Ultra |
|---|---|---|
| live, whole frame, median / p99 (last 600 frames of the run) | 4.6 / 11.2 ms (first stairs walk); 6.8 / 15.5 ms with the screen recorder on (stairs loop, Protocol 43) | 4.6–5.2 / 11.1–11.8 ms (three room loops, Protocol 47); 5.7 / 12.2 ms with the screen recorder on (stairs loop, Protocol 46) |
| GPU replay of the stairs log (Protocol 45, staging upload instead of the import) | 6.40 / 10.85 ms: upload + first submission 3.09, corner readback 0.35, backward KLT round trip 1.01, track manager 1.79 (medians) | 3.55 / 5.69 ms: upload + first submission 1.39, corner readback 0.24, backward KLT round trip 0.57, track manager 1.32 |

The live figure includes the zero-copy import and the thumbnail and excludes the upload; the replay figure includes the staging upload. Session 8 measured the track manager at 0.11 ms on a static scene; on the stairs it is 1.79 ms median in the replay (open item 3).

## 5. Hand-off

The filter thread (`VioRunner::run`) pops a frame once the IMU has reached its time. When a frame arrives while the queue holds four, the oldest queued frame is dropped and counted (none on the stairs loop: 3501 in, 2 skipped by the filter, 0 dropped). The filter runs on its own thread; nothing in the update reads a clock or another thread's state, so a log replays bit-identically (constraint 2).

## 6. The filter update

`Msckf::processFrame` (`core/src/estimator/msckf.cpp`, ADR-0013). The state is the IMU state `[q_WI, p_WI, v_WI, b_g, b_a]` (15 error dimensions), the camera–IMU extrinsics (6, ADR-0017) and time offset (1, ADR-0016), and up to 11 IMU-pose clones (6 each): 88 dimensions in a full window. Per frame, in order:

| Stage | What it does | ADR | Median ms (share) |
|---|---|---|---|
| propagate | preintegrated IMU increments to the frame time (Forster et al., bias Jacobians), covariance with the rate-proportional gyro noise and first-estimate Jacobians | 0008, 0023 | 0.016 (1 %) |
| clone + observations | re-acquisition check, a new IMU-pose clone, the frame's tracks attached to their features | 0021, 0013 | 0.069 (6 %) |
| stillness | zero-velocity update and the zero-flow pose anchor when the scene and the IMU are still | 0018 | 0.001 (mean 0.021) |
| low-parallax regime | whether this stretch is parallax-starved (rotation-only) | 0022 | 0.005 |
| **feature residuals** | for every track that ended or touches a leaving clone: inverse-depth Gauss-Newton triangulation, parallax and depth checks, reprojection residuals and Jacobians, projection onto the left nullspace of the point, χ² gate | 0013 | **0.343 (28 %)** |
| **update** | stacked rows (188 median, 512 p99) compressed by QR to at most 88, Kalman gain through an LDLT of S, Joseph-form covariance, state correction with the invariant reset | 0013 | **0.766 (63 %)**: QR 0.205, gain 0.257, covariance 0.185, correction 0.101 |
| hover + depth | the flow-bounded hover velocity where a frame gave no feature rows; the window of triangulated depths | 0020 | 0.008 |
| marginalise | the oldest clone leaves (the anchor clone is kept) | 0013, 0018 | 0.010 |
| **total** | | | **1.219 ms median, 2.41 p99** |

Those are the arithmetic's own times: the stairs log replayed on the S24 Ultra, back to back (the replay ran on the prime core at 3.34 GHz). Two things follow from the table. The update is two thirds of the cost and grows with the window, not with the number of tracks, because the QR compresses every frame's rows to the state's size; the IMU propagation, which runs at 475 Hz, is 1 %. And the filter is cheap enough that its time on the phone is set by the clock it is given, not by the arithmetic:

| The same 3490 frames, filter per frame, median / p99 | S24 Ultra (Snapdragon 8 Gen 3) | S25 Ultra (Snapdragon 8 Elite) |
|---|---|---|
| back to back (replay, 4 % of the time at 30 fps) | 1.22 / 2.42 ms (prime core, 3.34 GHz) | 1.24 / 2.16 ms (cpu6–7, 4.47 GHz) |
| paced at 30 fps from the shell, any core | 11.4 / 25.1 ms | 11.6 / 20.2 ms |
| paced, little cores (S24 Ultra 2.27 GHz max) | 17.2 / 28.7 ms | — |
| paced, mid cores (S24 Ultra 3.15 GHz max; S25 Ultra cpu0–5, 3.53 GHz max) | 11.8 / 24.7 ms | 11.6 / 20.1 ms |
| paced, prime cores (S24 Ultra 3.40 GHz; S25 Ultra cpu6–7, 4.47 GHz) | 6.1 / 12.1 ms | 5.7 / 9.4 ms |
| **live in the app (last 600 frames)** | **3.7 / 10.6 ms** (stairs loop), 5.4 / 10.4 ms (first stairs walk) | 4.8 / 14.3 ms (stairs loop, Protocol 46); 6.5–8.4 / 29.2–41.4 ms (three room loops, Protocol 47); 6.0 / 19.3 ms (a fourth loop after the system update) |
| earlier S25 Ultra live runs | | 4.2–8.0 ms median, p99 29–55 ms (Protocol 17); 4.2 ms, p99 30–32 ms (Protocol 26) |

Paced at 30 fps from the shell, every cluster stays at a low clock (median 0.61–0.96 GHz on the mid cores, 0.67 GHz on the prime core, sampled every 50 ms), and every stage of the table above slows by the same factor of 9–10. The app runs as the foreground app with the camera and the GPU busy, and lands in between. The frame budget is 33.3 ms; the live p99 on the S24 Ultra stays under a third of it. The S25 Ultra's live p99 reached 29–41 ms in the room loops of Session 37: its app threads run on the six-core cluster at about 1.4 GHz and never on the two fast cores, before and after a system update that changed the cpuset (Protocol 45 reading 9); no frame was dropped (the queue holds four).

## 7. On the screen

After the update the filter thread copies the state for the UI, appends the position to the live trajectory (the last 3000) and the frame's accepted triangulated points to the map (output only). The pose is the pose at the frame time, about 60 ms before it is shown. Nothing is extrapolated with the IMU to the display time; the run screen and the trajectory are a record, not a controller.

## Latency, end to end (S24 Ultra, stairs loop)

| | median | p99 |
|---|---|---|
| start of exposure → frame time (exposure/2 + readout/2) | 15.5 ms | |
| start of exposure → image in the app | 59.8 ms | 72.0 ms |
| frame time → image in the app | 44.4 ms | 56.7 ms |
| front-end on the camera thread | 4.6 ms (6.8 with the screen recorder) | 11.2 (15.5) ms |
| wait for the filter thread's poll | ≤ 5 ms | |
| filter | 3.7 ms | 10.6 ms |
| **frame time → pose available** | **≈ 55–60 ms** | |
| IMU sample → in the app | 3.9 ms | 6.2–9.3 ms |

The camera pipeline's own 44 ms after the frame time is the largest term and is outside the app's control. On the S25 Ultra the image arrives about 88 ms after the start of exposure (sensor_characterization.md), 28 ms later than on the S24 Ultra.

## Protocol 45 — The pipeline's stages timed on the phone without the camera (Session 36, S24 Ultra)

Setup. Galaxy S24 Ultra (SM-S928B, Android 17), on the USB cable, screen on, the app not running, battery 28 °C. Nothing used the camera: the input is the stairs loop of Protocol 43 (`run_20261109_154646`, recorded with the app's full-log switch on 2026-11-09), pushed to `/data/local/tmp`. Release builds for arm64 (NDK r27, `-O3`, `android-31`) at commit d73c10b:

- Filter stages: `pocketvio_replay --mode vio` on the stairs sample (the release asset's IMU, frame times and CPU tracks), with the profile and flags of the reference replay. A measurement-only build adds `steady_clock` laps between the stages of `processFrame` and inside `applyUpdate`, written per frame to a CSV, and an optional sleep in the replay loop that paces frames at 33.3 ms (`results/s36/pipeline/stage_patch.py`, `stages.patch`; compiled only with `-DPOCKETVIO_STAGE_TIMING`, never committed, because the estimator reads no clock). Five runs (`stages_phone.sh`): back to back, paced on any core, and paced under `taskset` on each cluster (cpus 0–1 little, 2–6 mid, 7 prime). Core clocks read from `scaling_cur_freq` every 50 ms during a back-to-back and a paced run of 1500 frames (`freq_phone.sh`).
- GPU front-end: `pocketvio_replay --mode track --gpu` (commit d73c10b; the app's `GpuFrontEnd` on the logged frames, a staging upload in place of the camera import) on the full stairs log, 3504 frames. Its tracks then go through the desktop filter replay and `tools/apriltag_eval.py`, as in Protocol 43.
- Latencies: the log's own receive times (`t_rx_ns` in `cam0/meta.csv` and `imu0/raw.csv`).

A profiler was tried first: `simpleperf` (user-space samples only on this phone; `cpu-clock:u`) puts 43 % of the replay's time in Eigen's matrix-product kernel but cannot attribute those samples to the calling stage (their call chains break with frame-pointer and with DWARF unwinding), so the stages were timed directly.

Readings:

1. **The filter's arithmetic is 1.2 ms per frame; its time on the phone is set by the CPU clock.** Back to back, 1.22 ms median and 2.42 p99, on the prime core at 3.34 GHz. Paced at 30 fps the clusters stay at 0.6–1.0 GHz and the same work takes 6–17 ms depending on the core; every stage slows by the same factor. The live app, 3.7–5.4 ms median, sits in between.
2. **The update is 63 % of the filter and the residuals 28 %**; propagation at 475 Hz is 1 %. The update's cost is set by the 88-dimensional state, because the QR brings the 188 median (512 p99) rows down to at most 88.
3. **The GPU front-end is 0.63 ms of GPU time per frame on the Adreno 750** over 3504 frames in motion (p99 0.71 ms). The front-end's frame on the CPU is 5–10 times that: readbacks, the backward round trip and the track manager.
4. **The GPU's tracks are the CPU's to 0.014 px until a threshold decision flips, and the result does not move beyond a loop's resolution.** On the GPU's tracks the stairs loop ends 0.295 m from the tags (0.270 horizontal, −0.119 vertical) against 0.293 m (0.238, −0.170) on the CPU's; the two end positions are 6.1 cm apart. The live run, which used the GPU, ended 2.1 cm from the replay on the GPU's tracks and 5.3 cm from the replay on the CPU's (Protocol 43 table 1): most of the live–desktop gap there is the front-end, not the filter. The vertical part of the end error moves by 5 cm with the tracks, so Protocol 43's floor-height reading compares the climbs and descents of one replay, not absolute heights to a few centimetres.
5. **Deterministic on each machine, not bit-identical across them.** All eight full filter runs on the phone produced the same trajectory file (md5 3f47e405…): the plain build, two under the profiler and five of the instrumented build, paced or not, on every cluster. Against the x86-64 reference the arm64 file differs at the rounding level: a 1 ns difference in the frame time on 42 of 3490 rows (through the online `t_d`) and a handful of values in the sixth significant digit; the end position is identical to the millimetre. The release's sample check (byte-identical) is an x86-64 statement. Session 14 found the arm64 replay identical to the last printed digit on EuRoC MH_01 (S25 Ultra); the stairs log, with its online time offset, is not.
6. **The camera sets the latency.** The image reaches the app 59.8 ms after the start of exposure (44.4 ms after the frame time); the IMU 3.9 ms after each sample.

**The S25 Ultra column (Session 37, 2026-11-10).** The S25 Ultra (SM-S938B, Android 17, build S938BXXSDDZJ1) on the cable, screen on, battery 26–27 °C. The arm64 binaries, the stairs log and the sample were copied from the S24 Ultra's `/data/local/tmp/pv` (md5 identical), so both columns are the same work. The SM8750 has two clusters, cpu0–5 (3.53 GHz) and cpu6–7 (4.47 GHz), so the pinned runs use those masks (`results/s37/p45_s25u.sh`; outputs and `stage_table.py` in `results/s37/pipeline/`). During three live runs a phone-side sampler (`results/s37/trace_app.sh`, started with `nohup`, so it survives the unplug) read every 100 ms both clusters' clocks and every app thread's last core and CPU time (`trace_table.py`).

7. **The same arithmetic on a faster clock is not faster.** Back to back the S25 Ultra runs the filter on cpu6–7 at 4.47 GHz in 1.24 ms median (S24 Ultra 1.22 ms at 3.34 GHz): its update is slower (0.86 against 0.77 ms; QR 0.21 / 0.21, gain 0.27 / 0.26, covariance 0.20 / 0.19, correction 0.13 / 0.10), its residuals faster (0.27 against 0.34). Paced, the clusters sit at 1.0–2.8 GHz and the filter takes 11.6 ms, 5.7 ms pinned to cpu6–7: the same picture as the S24 Ultra's.
8. **Deterministic across the two phones and their GPUs.** All six S25 Ultra filter runs give the S24 Ultra's trajectory file (md5 3f47e405…), and the GPU front-end's tracks on the Adreno 830 are byte-identical to the Adreno 750's (md5 121fb29a…) over 3504 frames. The pyramid and FAST are integer and bit-exact by design (ADR-0011); the KLT runs in `float` with a fixed reduction order, and that the two GPUs produce the same bits there is an observation on this log, not a guarantee. The Adreno 830 takes 0.53 ms of GPU time and 3.55 ms on the host per frame.
9. **The S25 Ultra runs the app on its efficient cores at about 1.4 GHz, with or without access to the fast ones.** In a traced room loop (`s25u_ship_loop2_trace`) no app thread ran on cpu6–7 (0 of about 1100 running samples per thread); the filter thread (15 % of a core), the front-end's `ImageReader` thread (13 %) and the UI (7 %) shared cpu0–5 at a median 1.36 GHz, while cpu6–7 idled at 1.02 GHz. The filter took 6.5 / 29.2 ms live. Back to back on cpu0–5 it takes 2.1 ms at up to 3.53 GHz; scaled to 1.36 GHz that is 5.4 ms, close to the live median. The cause was the phone's process state: `dumpsys activity` reported the visible app as `intermediate-top-activity` in scheduling group 2, which puts it in the `/foreground` cpuset (cpu0–5 on this phone), and the Settings app got the same. The S24 Ultra on Android 17 reports `top-activity`, group 3, `/top-app` (cpu0–7). After the phone's next system update (build S938BXXUDDZJB) the app is `top-activity` in `/top-app` on the S25 Ultra too, and the restriction is gone, but the placement is not: in a traced moving loop after the update (`s25u_ship_loop4_trace_upd`, the same route walked the other way round) the filter thread again ran only on cpu0–5 (0 of 1132 running samples on cpu6–7), at the same 1.36 GHz median (the process read `/top-app`, scheduling group 3, right after the run). Threads that use 13 % of a core are placed on the efficient cluster by the scheduler whether or not the fast one is allowed, so the cpuset was not what kept the filter slow; the clock the scheduler grants a light thread is. The filter took 6.0 / 19.3 ms in that loop against 6.5–8.4 / 29.2–41.4 before the update; one run does not separate the update from run-to-run spread. The earlier S25 Ultra p99 of 29–55 ms (Protocols 17, 26) is consistent with the same placement; no trace was taken then.

Open, after Protocol 45:
1. ~~The S25 Ultra~~: measured (readings 7–9). Still open: whether a performance hint for the filter thread (Android's `PerformanceHintManager` session with the frame budget as its target) moves it to a faster core or clock and brings the S25 Ultra's p99 down. An app change, to be measured before it is kept.
2. The app's own clocks: the live 3.7 ms lies between the back-to-back and the paced figures; a clock trace inside a live run would say how much of it is frequency and how much is sharing the cores with the camera and the UI.
3. The track manager's 1.79 ms in the replay against 0.11 ms on a static scene in Session 8: RANSAC on a moving scene (3 iterations median, 12 p99, against 2), or the replay's clocks while the CPU waits on the GPU.
