# Case study: why a phone spun in the hand made the estimator run away, and what finally fixed it

*A self-contained account of one problem in this project, written for a reader who has not seen the rest of it. Every term is defined where it first appears; the project's own documents are cited at the end for anyone who wants the raw numbers.*

---

## 1. What the system is

**pocketvio** is a program that runs on an Android phone and continuously estimates where the phone is and how it is oriented, using only two things the phone already has: its camera and its inertial measurement unit. It needs no GPS, no map, no external markers. The technical name for this is **visual-inertial odometry** (VIO).

The **inertial measurement unit** (IMU) is a chip containing two sensors sampled about 475 times per second on the phones used here:

- a **gyroscope** ("gyro"), which measures how fast the phone is turning about each of its three axes (angular rate, in radians per second);
- an **accelerometer**, which measures the force per unit mass acting on the chip (the *specific force*), in metres per second squared. At rest it reads the pull of gravity, about 9.81 m/s² pointing up; in motion it reads gravity plus the phone's own acceleration.

If you integrate the gyro you get the orientation; if you rotate the accelerometer reading into the world, subtract gravity, and integrate twice, you get the position. Doing only that is called **dead reckoning**, and it drifts: every small error in the orientation lets a bit of gravity leak into the horizontal, and that leak is integrated into velocity and then position. One degree of orientation error about a horizontal axis leaks 9.81 × sin(1°) ≈ 0.17 m/s² of fake acceleration, which becomes 200 metres of position error after 50 seconds. This number is the key to the whole story: **the estimator's sense of "which way is down" (its *tilt*) must be right to a small fraction of a degree, all the time.**

The camera is what keeps the tilt and the position honest. The program tracks a few hundred small image patches ("features") from frame to frame at 30 frames per second. When the camera *moves*, each feature's apparent position shifts by an amount that depends on how far away it is; this shift is called **parallax**. From the parallax of a feature seen from several camera positions the program can compute the feature's 3-D position (**triangulation**), and once a feature's 3-D position is known, every later view of it is a measurement of where the camera is and how it is oriented. The gyro and accelerometer tell the estimator what happened between frames; the camera tells it where it ended up. Each corrects the other's weaknesses.

The estimator is an **extended Kalman filter** of a particular design called the **MSCKF** (multi-state constraint Kalman filter). Three facts about Kalman filters matter for this story:

1. The filter keeps a **state** (orientation, position, velocity, and two slowly varying sensor offsets called the gyro bias and the accelerometer bias) **and a covariance matrix**, which is the filter's own statement of how uncertain each part of the state is and how the uncertainties are correlated.
2. When a measurement arrives, the filter corrects the state in proportion to how uncertain it believes itself to be. A measurement of velocity can correct the *orientation* only if the covariance says that the velocity error and the orientation error are correlated, which they are if the filter has been honest about the orientation uncertainty while it integrated gravity through it.
3. Between measurements the filter grows its covariance according to a **process-noise model**: a statement of how much error the sensors inject per second. For the IMU this model comes from a standard laboratory procedure, the **Allan variance**, measured here on each phone lying still on a floor for four to six hours.

An MSCKF additionally keeps a sliding window of the last few camera poses (here eleven, called **clones**), which is what lets it use a feature seen in several frames as one measurement. It also carries, in this project, the camera-to-IMU alignment and the camera-to-IMU time offset as estimated quantities.

Two more pieces of this particular filter will come up:

- A **zero-velocity update**: when the IMU and the image both say the phone is still, the filter is told "your velocity is zero", a measurement that stops drift at rest.
- The **hover update** (added earlier in this project for a different problem): on frames where no feature can be triangulated, the frame-to-frame motion of the features, with the rotation measured by the gyro removed, is multiplied by the typical distance of the scene seen recently (the *scene depth prior*) and used as a measurement of the camera's velocity.

Throughout this document, "run-away" means what it sounds like: the estimated velocity and position leave reality and grow without bound, typically to hundreds of metres within a minute, while the phone is in fact within arm's reach of where it started.

---

## 2. The experiment that exposed the problem

The project tests the estimator with a fixed protocol: the phone starts lying on a taped spot on a desk, with a printed sheet of **AprilTags** (square black-and-white markers whose size is known, 70 mm here) in the camera's view. The walker picks the phone up, does something, and puts it back on the spot. Because the tags are at a known place, the camera's position relative to them at the start and at the end can be computed independently of the estimator to a millimetre or two, which gives a ground truth for how far off the estimator is at the end.

On 23 October 2026 (the project's Session 17) the "something" was a **hand spin**: the phone lifted to chest height and turned about each of its three axes for a few seconds each, at rates from a gentle 0.7 rad/s (about a quarter turn per second) up to 8.7 rad/s (almost one and a half turns per second), then set back down. Four runs were recorded. Every one of them diverged:

| run | peak turn rate | where the estimator thought the phone was at the end |
|---|---|---|
| spin1 | 8.7 rad/s | 1007 m away, moving at 46 m/s |
| spin2 | 5.0 rad/s | 162 m away |
| spin3 | 3.9 rad/s | 866 m away, still accelerating a minute after being set down |
| spin4 | 3.1 rad/s | 147 m away |

Hand-held loops walked around the room on the same day closed to within 4–38 cm. A spin is not an exotic motion for a phone, and a pen-shaped device (the motivation for this project) spends much of its life being turned in a hand. So the failure mattered, and the next four working sessions were spent on it.

---

## 3. Why a spin is hard for visual-inertial odometry

When a phone turns in a hand, the camera moves very little (a few centimetres, on the lever arm of the wrist) and turns a lot. Three consequences follow, and they are worth stating plainly because every later finding rests on them.

**No parallax, so no triangulation.** A feature's apparent motion in the image is the sum of a part due to the camera's rotation, which does not depend on the feature's distance, and a part due to the camera's translation, which does. With almost no translation, every feature moves exactly as a feature infinitely far away would, and its distance cannot be computed. The filter's triangulation test correctly rejects it (in spin4, 41 230 rejected against 3 805 accepted). Those features still carry information about *rotation*, but the standard MSCKF has no way to use a feature whose distance it does not know, so the information is thrown away.

**Tilt is unobservable during the spin.** The accelerometer reads gravity plus the hand's accelerations, and during a spin the hand's accelerations are comparable to gravity, so the accelerometer alone cannot say which way is down. The camera could, through triangulated features, but there are none. The orientation is therefore carried by the gyro alone for the duration of the spin, with whatever error the gyro accumulates.

**A gyro accumulates error in proportion to how far it has turned.** A gyro's reading is not exactly proportional to the true rate: each axis has a small **scale-factor** error (it reads, say, 0.2 % high), the axes are not perfectly orthogonal (**cross-axis** or **misalignment** error), and the response is slightly nonlinear at high rates. These errors are tiny per sample, but they add up with the total angle turned, not with time. A hand spin turns the phone through *thousands* of degrees (the fastest spin here: 1381°). At 0.2 % that is about 3° of accumulated orientation error, and a tilt of 3° leaks 0.5 m/s² of fake acceleration into the velocity. None of this appears in the Allan-variance noise model, which was measured on a phone that was not turning at all.

Put together: during the spin the filter's orientation drifts by a few degrees, nothing can observe it, and when the spin stops the filter integrates gravity through the wrong tilt at half a metre per second squared. The question is what the filter *believes* about its own orientation while this happens. That belief turned out to be the whole problem, but it took four sessions and several fixes to a different problem to see it.

---

## 4. The investigation, in order

### 4.1 First diagnosis (1 November, Session 18)

The recorded spins were replayed on a desktop with every internal quantity logged. Four things were established.

*The run-away starts after the spin, not during it.* While spin4 was being turned, the hover update held the estimated speed below 0.25 m/s. At 45.1 s its scene depth prior expired (it was defined as the typical distance of features triangulated in the last three seconds, and none had been for three seconds), the hover stopped, and from that frame the velocity grew *linearly* at 0.4 m/s² along one horizontal axis with no measurement at all. A constant horizontal acceleration out of a dead-reckoning filter is a tilt error; 0.4 m/s² is 2.3°, made during the spin and hidden by the hover's velocity bound.

*The filter could not come back.* When spin3 was set down and lay still for a minute with the tags in view, no zero-velocity update fired: the update had a guard that refused to zero the velocity if the filter's own estimated speed was above 0.5 m/s (a protection against zeroing a moving vehicle during a lull in the IMU), and the estimate was already at 4 m/s. The guard locked a still phone out of recovery. Spin4 did get zeroed once, at 9.7 m/s, and ran away again at 0.25 m/s² because its tilt had not been corrected.

*The rotation information was there and was discarded.* Fitting a pure rotation to consecutive frames of spin4 explained the tracked features to 0.06 pixel, with 150 of them agreeing. About a milliradian per frame of relative rotation was being measured by the camera and never reached the state.

*The gyro and the camera disagreed on the amount of rotation by about 1 to 1.75 % about the camera's optical axis, consistently across all four spins.* This looked like a gyro scale-factor error, and was logged as a candidate cause with the note that a replay with the gyro scaled by 1 % gave end positions of 1.2 km, 0.5 km and 0.3 km for scales of 0.99, 0.995 and 1.0: "once the state is unobservable the end position is chaotic". That sentence was more important than it looked at the time (see §4.7).

A reading of the literature on small-translation and pure-rotation VIO (seven recent papers and the work they build on) produced a ranked list of remedies, and the next three were built in that order.

### 4.2 Fix 1: re-acquisition at a certain rest (ADR-0021, Session 18)

When the IMU and the image *both* say the phone is still and the filter's velocity estimate is nonetheless above the guard, the filter no longer gates a measurement; it re-initialises itself in place: roll and pitch are reset from the accelerometer (which at rest reads pure gravity), the velocity is set to zero, the gyro bias is reset to the mean of the still window, the clone window is discarded, and the covariance of the reset quantities is rebuilt at their initial values. The position is deliberately *kept*: the error the filter has made is reported, not hidden.

Result on the recorded spins: three of the four re-acquired once each after being set down and then held still to within a few centimetres per second. The end positions, however, were still 137–619 m away, because the re-acquisition only fixes the next minute, not the last one.

In Session 19 a second trigger was added to the same rule (§5 of the ADR): a re-level also fires when, under certain stillness, the tilt implied by the accelerometer disagrees with the filter's by more than 0.5° for one second. This was needed because, after Fix 2 below, a run-away could come to rest *below* the speed guard with its tilt intact, and that tilt (2.1° on one live spin) then stayed through forty seconds of zero-velocity updates while the position slid sixteen metres. Note what this trigger is: a direct comparison of the filter's orientation against an independent reference. That idea returns in §4.7.

### 4.3 Fix 2: the held scene depth (ADR-0020 §8, Session 18)

The hover's depth prior had expired after three seconds without a triangulation, which is exactly what a spin produces. The scene does not change when the filter stops triangulating it, only the filter's knowledge of it ages. So the last known depth is now held, with its stated uncertainty growing with age, and the hover keeps bounding the velocity through the whole turn. Spin4 went from 137 m to 4.3 m; spin3 from 149 to 74 m; spin2 from 109 to 28 m; spin1 from 619 to 510 m. The walked loops did not move.

The remaining run-aways now happened on frames *with* a handful of triangulated features during fast turns, on which the hover sat out by design (it was meant to fill the gaps between feature updates, not to compete with them).

### 4.4 Fix 3: rotation from the zero-parallax features (ADR-0022, Sessions 18–19)

A feature that fails the parallax test is no longer discarded. It is parameterised by its bearing (the direction in which it was first seen) and its *inverse* distance, a form that stays well defined when the distance is unknown or infinite, and every later view of it becomes a measurement of the rotation between the camera poses that saw it, plus a weak bound on their translation from a prior on the scene depth. This is the standard remedy in the literature (Civera et al. 2008; Huai & Huang 2018), implemented from the papers.

It worked for the spins (the slow ones to 1.2–1.4 m, spin3 to 8.6 m) but its first version cost the walked loops a centimetre or two, which the project's acceptance rule ("loops not worse") does not allow. Two amendments found why: the rows had been firing at the pick-up of a loop, before the scene had ever been triangulated, with a guessed depth; and a feature that *had* been triangulated earlier should keep its own depth rather than the scene's median. With both, every acceptance bar was met and the rows became the default on 3 November.

### 4.5 Live confirmation, and what the fast spins still did (Session 19, S24 Ultra)

Three new spins were recorded live on the phone with everything above installed. A slow one (peak 2–3 rad/s) now closed to 0.45 m live, the first spin ever to do so. Two fast ones (peaks 18.6 and 10.2 rad/s) still ran away *during* the spin, to 280 m and 39 m live, and the desktop replay of the second showed the re-level of §4.2 working as designed: the filter re-acquired 2 s after the set-down and then held to 3 cm for fifty seconds. The spin itself remained the open problem, and the reading at the time was that the hover was being switched off by the few features that triangulate during a fast turn.

### 4.6 The hover was not switched off; it was outvoted (Session 20)

To test that reading, the hover's complete input was recorded on every frame (whether or not the update ran), and fitted offline. The reading was wrong. During the run-away stretches of the two fast spins, 60–70 % of the frames had no feature update at all; the hover measurement was formed on hundreds of them and *applied on none*, because the velocity it implied was above its upper bound of 0.15 m/s. That velocity was the run-away's own: the scene depth prior had grown with the drifting velocity (the few features triangulated during the run-away were placed at 25–64 m by a filter that believed it was moving fast), so flow times depth agreed with the error. The hover and the run-away were feeding each other.

A diagnostic replay with the depth fixed at 1 m confirmed it (233 → 11 m on one spin, 98 → 7 m on the other), and the honest fix was to raise the bound to 0.3 m/s, just above the speed of a camera on the lever arm of a turning hand and well below walking. The fast spins went from 233 and 98 m to 14 and 6.4 m. One spin, spin1, still ran away (371 m), through frames that did carry a few features.

Three candidates for spin1 were measured and rejected: running the hover alongside weak feature updates (no gain), the hover on every frame (fixes spins, costs loops 6 cm and the EuRoC benchmark 7 cm), and freezing the depth while turning fast (fixes spin1, holds the desk's depth through a turn toward the room, costs loops). The session closed with spin1 as "open 1", and the suggested next form was a scene depth kept per viewing direction, so that it could not follow a run-away without freezing the desk.

### 4.7 Finding the actual cause (Session 21)

Session 21 started on that suggestion and on two other loose ends (the 1.7 % gyro scale candidate from §4.1, and a 3–4 % scale discrepancy against the tags that turned out to be an artefact of the tag measurement at long range and is not part of this story). All three were measured; the first two were dead ends, and the dead ends pointed at the cause.

**The per-direction depth was not the lever.** With the hover's input recorded for spin1, the depth prior during the stretch before the run-away was 1.4–1.9 m, in agreement with the earlier depth of the same viewing direction. A depth map keyed on direction would have produced the same prior and the same run-away. What the recording showed instead was this: between 43 and 47.7 s the phone was nearly still in the hand (gyro below 0.1 rad/s, no features), and the estimated velocity ran a *sawtooth* between 0.02 and 0.25 m/s, period about a third of a second: the hover pulled it to near zero, the next three or four frames were rejected by the hover's statistical gate, the velocity regrew at about 1 m/s², the gate's override fired, and so on. One metre per second squared of acceleration error on a phone that is barely moving is not a depth problem. It is 6° of tilt.

**The gyro scale was not wrong.** The replay was given a per-axis gyro scale factor and the spins were replayed at scales from 0.975 to 1.010 on the optical axis and at ±1 % on the other two. The filter was worst exactly *away* from unity: ±1 % on any axis multiplied the spin errors by ten to a hundred and broke the walked loops (one went from 9 cm to 518 m); a fine sweep put the minimum within 0.1–0.2 % of 1.000. The gyro was right to about 0.2 %. The 1.7 % the camera-against-gyro tool had read at 2–3 rad/s was the *camera's* error: a feature tracker following a patch smeared over 0.05 rad of arc during a 20 ms exposure lags the true position, and the lag reads as a smaller rotation. This mattered twice over: the candidate was dead, and the 0.2 % residual it left is precisely what §3 says accumulates to degrees over a spin.

**The tilt was measured directly, along the log, against its own sigma.** A small tool was written to compute, on any one-second window where the hand was quiet, the angle between the filter's "up" direction and the accelerometer's mean (the latter being pure gravity when the phone is nearly still). On spin1 it read 5.3° at 45 s, 6.2° at 47 s, 4.9° while the phone lay on the desk at 79–83 s, and 0.18° after the re-level fired at 84 s. The filter's own covariance at 45 s put the tilt uncertainty at **0.017°**. The estimator was three hundred times more confident than it had any right to be.

This is the fact everything else follows from. Recall point 2 of §1: a velocity measurement corrects the orientation only through the covariance's correlation between the two, and that correlation is built from the orientation's uncertainty. With the orientation declared certain to a hundredth of a degree, the hover's hundreds of velocity measurements could correct the velocity and nothing else, so the same tilt regrew the same error on the next frame, forever. The zero-velocity updates at rest did the same, which is why the re-level of §4.2 had to be bolted on as a separate rule: it was doing by hand what a consistent filter does on its own.

**The gyro bias was a symptom, not the cause.** The filter's gyro-bias state was seen moving by 0.0025 rad/s during the spins, thirty times its stated uncertainty, dragged by the rotation rows of §4.4 (which carry the tracker's blur lag). Freezing the bias at its initial value left spin1's tilt at 5.7°. Stiffening it made no difference, because its process noise was already negligible against the measurement pressure.

**The one-line test.** If the disease is an attitude covariance that does not grow enough during a spin, the crude cure is to make it grow more. The replay was given a knob to multiply the gyro's noise density, and the spins were replayed at ×10, ×30, ×100 and ×300:

| gyro noise density | spin1 | spin2 | spin3 | spin4 | spin5b | spin6 | spin7 | walked loop |
|---|---|---|---|---|---|---|---|---|
| ×1 (as measured at rest) | 371 m | 5.7 | 4.2 | 1.30 | 14.1 | 0.91 | 6.4 | 0.12 |
| ×10 | 0.36 | 1.7 | 0.12 | 0.20 | 0.53 | 0.05 | 1.4 | 0.08 |
| ×30 | 0.35 | 1.8 | 0.27 | 0.24 | 0.11 | 0.08 | 0.09 | 0.14 |
| ×100 | 1.00 | 1.6 | 0.43 | 0.45 | 0.11 | 0.13 | 0.08 | 0.43 |
| ×300 | 42 | 2.2 | 13.7 | 1.40 | 0.50 | 0.65 | 2.8 | 0.72 |

Every spin closed, none needed a re-acquisition, and spin1's tilt at 45 s fell from 5.3° to 0.5°. With a covariance that admitted the error, the measurements that had been there all along corrected it. A flat multiplier is the wrong shape, though: it inflates the orientation uncertainty as much at rest and on a walk as in a spin, and from ×100 the walked loop starts to pay.

**The right shape.** The Allan variance says how noisy the gyro is when it is *not turning*. The errors of §3 (scale, cross-axis, nonlinearity, and the jitter of the true sampling instant) are all proportional to the rate. So the filter's model of the gyro's noise was changed from a constant density *N* to an effective density `sqrt(N² + (κ·|ω|)²)`, where |ω| is the current turn rate and κ is a new per-device parameter: nothing extra at rest, twice the Allan density at a walk's 1 rad/s, six times it at a spin's 3 rad/s. This is an error-budget term, not a derivation: a constant scale error accumulates linearly with angle and a white-noise term as its square root, but over the twenty to forty seconds of a spin the filter cannot tell the two apart, and the white form keeps the filter's state vector as it is. κ was then measured the only way such a term can be, by the filter's own consistency across all the recorded data: the seven spins, eighteen walked loops on two phones, three loops with a sharp turn, a public drone dataset with ground truth, and a static recording.

| κ (per √Hz) | spins closing | S24 Ultra 8-loop median (bar: ≤ 0.095 m) | S25 Ultra 10-loop median (≤ 0.09) | drone dataset error (≤ 0.24) |
|---|---|---|---|---|
| 0 (before) | none of the fast ones | 0.0885 m | 0.081 m | 0.235 m |
| 1e-4 | all but spin1 (51 m) | | | |
| 1.5e-4 | all (spin1 1.1 m) | | | |
| **2e-4** | **all (0.02–1.8 m)** | **0.0880** | **0.052** | **0.234** |
| 2.5e-4 | all | 0.097 | 0.054 | 0.234 |
| 3e-4 | all | 0.102 | 0.060 | 0.234 |
| 5e-4 | all | 0.118 | 0.082 | 0.231 |

κ = 2e-4 is the smallest value that closes every spin and the largest that leaves the S24 Ultra's walked loops where they were; it improves the second phone's loops, the turn loops (0.41 → 0.10 m on the worst) and the drone dataset's accuracy and consistency, and leaves the static recording untouched. It is now in both phones' device profiles, and the replay of the spins with the profile value reproduces the swept experiment to the last bit. The tilt the filter carries into its rest, measured on every spin, is 0.01–0.09°.

---

### 4.8 Live confirmation on both phones (Session 22)

On the evening of 3 November 2026 the rebuilt program ran on both phones, two spins each, 90 s runs from and back to the tag spot (Protocol 26). The S24 Ultra's fast spin, at a 12 rad/s peak, ended 1.63 m from its start; the same phone's fast spin before the noise term (10.2 rad/s, §4.5) had ended 38.8 m away. Its slow spin closed to 0.21 m. The S25 Ultra, spinning for the first time, closed to 0.083 m at 9.4 rad/s and 0.061 m slow. None of the four re-acquired, and the tilt left at rest was 0.01–0.10° on every run. Live and desktop agree to within the error itself.

What the fastest spin still loses is not a tilt. For about ten seconds around its 12 rad/s peaks, with a 20 ms exposure, every frame is blurred by some 60 px and only one frame in four to eight carries a tracked feature; the velocity built up in those blind seconds (1.3 m/s) became 1.3 m of position before the tracks came back, and a position error, once made, has nothing to measure it against. That is a camera problem, exposure under fast rotation, and it is the next item.

## 5. What was actually learned

**1. A Kalman filter's covariance is not bookkeeping; it is the mechanism of correction.** Every fix before the last one treated a symptom: the velocity (hover bound), the recovery (re-level), the scene scale (held depth), the discarded rotation (the rows). They were each correct and each stays in the system. But they were needed in that form only because the filter's stated orientation uncertainty was a lie, and a lying covariance disables the ordinary correction path for every measurement. When the covariance was made honest, measurements that had been applied hundreds of times without effect started to work, and the recovery rule became a backstop that is rarely reached.

**2. Measure the state against an independent reference *along* the run, not only at the end.** The decisive measurement was a one-second-window comparison of the filter's orientation against the accelerometer, printed every two seconds along the log, next to the filter's own sigma. It took twenty lines of code. For three sessions the error was judged by the end position, which, once a state is unobservable, is chaotic: a 1 % change in an input moved the end by hundreds of metres with no order. The project had written that sentence in Session 18 and did not act on it until Session 21. End positions were right to say "it fails"; they were useless to say *why*.

**3. A noise model measured at rest is a model of the sensor at rest.** The Allan variance is the standard, correct way to characterise an IMU, and it was done carefully here (six hours of data, thermal settling, both phones). It cannot contain anything proportional to the rate, because the rate was zero. A phone in a hand is not a phone on a floor, and the difference is a factor of six in the gyro's effective noise at the rates a hand produces.

**4. Candidates must be killed by the filter, not by a side tool.** The camera-against-gyro fit was a sound idea and a correct implementation, and its 1.7 % was real, but it measured the tracker's motion blur, not the gyro. The filter itself, replayed with the candidate correction applied, was the better instrument: it was worst exactly where the tool said it should be best. The same filter-as-instrument sweep is what later put a number on κ.

**5. Honest negative results are the shortest path.** Three things were ruled out in Session 21 before the cause was found (per-direction depth, gyro scale key, bias stiffness), each by a direct measurement rather than an argument. Each exclusion narrowed the field; the last one (freezing the bias and watching the tilt stay) pointed straight at the covariance.

---

## 6. What remains

- The fix was measured on recorded data first and then live on both phones (§4.8): confirmed. What was left on the fastest spin was a position slide made during the seconds when the frames are too blurred to track; a shorter exposure under fast rotation (ADR-0024: exposure from a blur budget, gain holding the brightness) took that spin from 1.63 m to 0.455 m the same evening (Protocol 27), and the rest of it is the tracker's share of short, noisy frames.
- The rotation rows of §4.4 still carry the tracker's blur lag at 2–3 rad/s and pull the gyro bias; a rate-dependent noise on those rows is the open item.
- The accelerometer presumably has the same kind of rate- (or acceleration-) proportional residual; the drone dataset's long-standing need for a ten-times-larger accelerometer noise may be this. Untested.
- Above κ = 2e-4 two of the S24 Ultra's walked loops lose a few centimetres of horizontal accuracy during the walk while the other phone's loops gain; which turn does it, and why the phones differ, is not understood.

---

## 7. Glossary

- **Allan variance**: a statistical method that separates an inertial sensor's white noise from its slow bias wander by analysing a long static recording.
- **Clone**: one past camera pose kept in the filter's sliding window.
- **Covariance**: the filter's own estimate of the uncertainty of its state and of how the uncertainties of its parts are correlated.
- **Dead reckoning**: estimating position by integrating the IMU alone.
- **Gyro bias**: a slowly varying offset in the gyro's reading at zero rate; the filter estimates it.
- **Hover update**: this project's velocity measurement for frames without triangulated features, from the rotation-compensated image flow times the recent scene depth.
- **MSCKF**: multi-state constraint Kalman filter, the VIO estimator design used here.
- **Parallax**: the change in a feature's apparent direction when the camera moves; zero for a camera that only rotates.
- **Process noise**: the filter's model of how much uncertainty the sensors add per unit time between measurements.
- **Re-acquisition / re-level**: this project's rule that resets the orientation, velocity and window when the phone is certainly still but the filter's estimate is not.
- **Run-away**: the estimated velocity and position growing without bound while the phone is in fact nearby.
- **Scene depth prior**: the typical distance of recently triangulated features, used by the hover update to turn image flow into velocity.
- **Tilt**: the orientation error about the horizontal axes, that is, the error in the filter's sense of which way is down.
- **Triangulation**: computing a feature's 3-D position from its directions seen from two or more known camera positions.
- **Zero-velocity update**: a measurement of "velocity = 0" applied when both the IMU and the image say the device is still.

## 8. Where the details are

- The spin experiment: `docs/msckf.md`, Protocol 17. The first diagnosis and the literature: `docs/review-rotation-2026-11.md`.
- The fixes: `docs/decisions/ADR-0021-reacquisition-after-run-away.md`, `ADR-0020-hover-velocity.md` (§8 held depth, §9 speed bound), `ADR-0022-low-parallax-features.md`, and the one that found the cause, `ADR-0023-rate-proportional-gyro-noise.md`.
- The measurements, session by session: `docs/msckf.md`, Protocols 18–25. Protocol 24 is the hover's input recorded on every frame; Protocol 25 is the tilt along the log, the gyro-scale sweep and the κ sweep.
- The tools: `tools/tilt_at_rest.py` (the orientation against the accelerometer), `tools/gyro_vision_rotation.py` (the camera-against-gyro fit), `tools/apriltag_eval.py` (the tag truth), `tools/reference_sweep.sh` (the 27 reference recordings under a named filter variant).
- The recorded data is not in the repository (gigabytes of frames); the replay outputs of every experiment quoted are under `results/s16` to `results/s21`.
