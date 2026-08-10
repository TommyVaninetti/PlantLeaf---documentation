# PlantLeaf – Automatic Ultrasonic Click Detection Algorithm

**Version:** 5.1
**Date:** June 2026
**Authors:** Tommaso Vaninetti
**Repository:** [PlantLeaf-Desktop-App](https://github.com/TommyVaninetti/PlantLeaf-Desktop-App)

---

> *Plants scream when they are stressed. You just need the right algorithm to hear them.*

---

## Table of Contents

1. [Overview and Motivation for v5](#1-overview-and-motivation-for-v5)
2. [v4 → v5 Comparison Summary](#2-v4--v5-comparison-summary)
3. [System Setup and Signal Representation](#3-system-setup-and-signal-representation)
4. [Adaptive Noise Estimator (NEW in v5)](#4-adaptive-noise-estimator-new-in-v5)
5. [Stage 1 – Adaptive Energy Threshold + Run-Length Filter](#5-stage-1-–-adaptive-energy-threshold--run-length-filter)
6. [Stage 2 – FFT Parameters and Hard Gates](#6-stage-2-–-fft-parameters-and-hard-gates)
7. [Pre-processing Pipeline for Stage 3 Candidates](#7-pre-processing-pipeline-for-stage-3-candidates)
8. [Stage 3 – Multi-Criterion Temporal Validation (v5)](#8-stage-3-–-multi-criterion-temporal-validation-v5)
9. [Stage 4 – Deduplication](#9-stage-4--deduplication)
10. [Fit Pipeline: τ and R²](#10-fit-pipeline-τ-and-r)
11. [Feature Summary Table](#11-feature-summary-table)
12. [SVM Classifier — Training Protocol and Results](#12-svm-classifier--training-protocol-and-results)
13. [Algorithm Version History](#13-algorithm-version-history)
14. [References](#14-references)

---

## 1. Overview and Motivation for v5

Version 4.0 was designed and validated primarily in **controlled indoor environments**. Outdoor field recordings introduced new challenges that motivated a substantial revision:

- **Variable noise floor**: wind, traffic, insects, and mechanical vibration produce a noise floor that changes on timescales of seconds, making the static `noise_rms` estimate of v4 unreliable.
- **Impulsive outdoor noise**: transient events (claps, footsteps, mechanical impacts) produce FFT patterns superficially similar to clicks — broadband, brief, high-energy — and pass multiple v4 criteria.
- **Hardware coupling artefacts**: without rigid mounting, PCB vibrations couple mechanically into the MEMS microphone, producing low-frequency broadband bursts concentrated near the 20 kHz analysis band edge.
- **Tonal interference**: some outdoor environments (and the MCU itself) produce narrowband tones at fixed frequencies (e.g. 40 kHz, 80 kHz) that pass energy thresholds.
- **Hardware developments**: with future evolutions in the hardware follows the need of parameters without fixed energy thresholds.

v5 addresses all of these with: an adaptive noise estimator, a more robust run-length filter in Stage 1, a fully redesigned set of Stage 3 features, and an improved decay fit pipeline. All hard thresholds from v4 are replaced with **dimensionless, scale-invariant features** suitable for an SVM classifier.

> **Principal Design:** no single experimentally chosen threshold. All parameters are either physically motivated constants or features fed to an SVM. The noise floor and its standard deviation are the two fundamental quantities from which most features are derived, ensuring invariance to hardware gain and environmental amplitude changes. The SVM is trained with hundreds of events coming from over 160 hours of recordings.

---

## 2. v4 → v5 Comparison Summary

| Aspect | v4.0 | v5.0 |
|---|---|---|
| Noise floor estimate | Global offline, static per session | Adaptive sliding window (minimum-statistics) |
| Stage 1 threshold | `μ + k·σ` (static k=5) | `k · Ê_floor` (adaptive floor, k=1.5) |
| Stage 1 run filter | MAX_RUN = 4 frames | MAX_RUN = 3 frames |
| C1 – Amplitude | Absolute 130 µV | Peak SNR = `max(A[n]) / noise_floor` (dimensionless) |
| C2 – Pre-silence | Fixed 20-sample guard | Dynamic: pre_window ends where `A[n] < noise_floor + std_noise` going backward from peak |
| C3 – Energy decay | `E_W1 / E_W4 > 2` (fixed windows) | **Removed** — replaced by τ from improved fit |
| C4 – Asymmetry | Rise/fall sample ratio < 2.5 | Rise time (ms) + fall time (ms) as separate features + asymmetry integral |
| C5 – Tau range | Fixed [0.045, 1.3] ms | τ as SVM feature (range used as soft guidance, not hard gate) |
| C6 – R² | Fixed > 0.45 | R² as SVM feature |
| New features | — | post_SNR, ZCR_pre, ZCR_click, ZCR_post, kurtosis, spectral centroid shift, fit_coverage |
| Fit window selection | Fixed 300-sample window + fixed 5-sample skip | Dynamic: decay_start via slope criterion, decay_end via noise threshold |
| Fit smoothing | Moving average W=4, mode='same' | Gaussian σ=2 samples, mode='valid' |
| Classifier | AND logic on 6 hard thresholds | SVM on full feature vector |

---

## 3. System Setup and Signal Representation

Unchanged from v4. Key parameters:

| Parameter | Value |
|---|---|
| Microphone | Knowles SPU0410LR5H-QB |
| Sampling rate | `fs = 200,000 sps` |
| FFT size | `N = 512 points` |
| Frame duration | `T = 2.56 ms` |
| Frame rate | ~390 FPS |
| Analysis band | 20–80 kHz (bins 51–204, 154 bins) |
| Phase encoding | int8, range [−127, +127] → [−π, +π] |

---

## 4. Adaptive Noise Estimator (NEW in v5)

### 4.1 Motivation

The v4 `noise_rms` was computed once per session from randomly sampled silent frames. This worked indoors but fails outdoors where the noise floor drifts over seconds (wind gusts, passing vehicles, changing humidity). v5 replaces this with **two parallel sliding-window minimum-statistics estimators** — one operating in the FFT energy domain (for Stage 1) and one operating in the iFFT/Hilbert envelope domain (for Stage 3 features). Both share the same burst protection gate and the same window length W.

> **Why two separate estimators?** Stage 1 operates on FFT frame energy (a scalar in V²/bin). Stage 3 features require `noise_floor` and `std_noise` in physical voltage units of the Hilbert envelope (µV). These two quantities are related but not trivially convertible — the iFFT reconstruction, Tukey taper, and Hilbert transform all affect the amplitude distribution in ways that depend on the specific signal chain. Computing each estimator directly in its own domain avoids any calibration coefficient and ensures accuracy regardless of hardware changes.

---

### 4.2 Shared Definitions

**Frame energy (FFT domain):**

```
E_i = (1/K) · Σ_{k=0}^{K-1} |A_i[k]|²

K = 154  (number of transmitted bins, 20–80 kHz)
Units: V²  (normalized FFT magnitudes)
```

**Per-frame envelope statistics (iFFT domain):**

For every frame i (including silent frames), reconstruct the iFFT and compute the Hilbert envelope A[n], then extract two scalars:

```
env_mean_i = mean(A[n])    over 512 samples
env_std_i  = std(A[n])     over 512 samples
Units: V  (physical voltage of reconstructed signal)
```

> **Implementation note (offline PC):** computing iFFT + Hilbert for every frame adds negligible overhead on a PC. For future STM32 firmware with hardware analog bandpass filters (Phase 2/3 hardware), the time-domain signal is directly available from the ADC — no iFFT reconstruction is needed and the envelope statistics can be computed directly from ADC samples.

---

### 4.3 Window Length

Both buffers use the same window length:

```
W = 750 frames  (~1.92 s at 390 FPS)
```

**Motivation:**
- Long enough to avoid burst noise dominating the minimum: a burst of 30–50 ms corresponds to ~20 frames — negligible in a window of 750.
- Short enough to track real floor changes: wind gusts, passing vehicles, and environmental transitions occur on timescales of 1–5 s. W = 750 frames ≈ 1.92 s tracks these adequately.

---

### 4.4 Burst Protection (shared gate)

Both buffers share the same burst protection condition, evaluated once per frame using the FFT energy:

```
if E_i > α · Ê_floor(i-1):
    → frame is energetic (burst or click candidate)
    → do NOT update Buffer 1 (E_i excluded from minimum)
    → do NOT update Buffer 2 (env_mean_i, env_std_i excluded)
else:
    → frame is silent
    → update both buffers normally
```

**α = 4** (suggested starting value — verify experimentally on outdoor recordings).

**Physical meaning:** a burst frame has energy more than four times the current floor estimate. Using the same gate for both buffers ensures that a click candidate never contaminates either noise estimate simultaneously.

---

### 4.5 Buffer 1 — FFT Domain (for Stage 1)

```
CIRCULAR BUFFER: B1[], length W = 750
Updated only for non-burst frames: B1[i mod W] = E_i
```

**Why not a pure minimum?**

A single anomalously low frame (microphone dropout, ADC glitch, momentary silence
between two noise events) can pull `min(B1)` far below the true noise floor,
causing the Stage 1 threshold to drop and producing a burst of false positives.
Stahl et al. (2000) proposed using the q-th quantile of the buffer as a more
robust alternative. However, Rangachari & Loizou (2006) showed experimentally
that a fixed global percentile is fragile when the noise distribution changes —
the correct quantile depends on the noise type, which varies outdoors.

**Adopted solution: median of local minima (M sub-windows)**

Divide B1[] into M = 10 non-overlapping sub-windows of W/M = 75 frames each.
Compute the minimum of each sub-window, then take the median of those M minima:

```
Sub-windows: S_1, S_2, ..., S_10   (75 frames each, ~192 ms each)

m_j = min_{i ∈ S_j} B1[i]         for j = 1, ..., 10

Ê_floor(i) = β · median(m_1, ..., m_10)
```

**Why this is robust:** a single anomalously low frame affects at most one of
the 10 sub-window minima. The median of 10 values is insensitive to a single
outlier — it would need 5 out of 10 sub-windows to be contaminated to shift
the median, which is physically implausible for isolated glitches.

**Why β is still needed:** each local minimum still underestimates the true
sub-window mean (the minimum of N samples from a distribution is always below
the mean). β = 1.3 corrects this bias, consistent with Martin (2001).

```
β = 1.3  (slightly smaller than 1.5 Martin 2001 correction factor)

RAM:  B1[] buffer:          750 × 4 bytes = 3.0 kB
      local minima array:    10 × 4 bytes = 0.04 kB
      Total Buffer 1:                     = 3.04 kB

STM32 cost: sub-window minimum updated once per frame (O(1));
            sort of 10 floats runs once every 75 frames (~192 ms) → O(10 log 10) ≈ 33 ops.
            Negligible on STM32F411 at 100 MHz.
```

---

### 4.6 Buffer 2 — iFFT/Hilbert Envelope Domain (for Stage 3)

```
TWO CIRCULAR BUFFERS: B2_mean[], B2_std[], each length W = 750
Updated only for non-burst frames:
  B2_mean[i mod W] = env_mean_i
  B2_std[i mod W]  = env_std_i
```

**noise_floor estimation — same median-of-local-minima method as Buffer 1:**

```
Divide B2_mean[] into M = 10 sub-windows of 75 frames each.

n_j = min_{i ∈ S_j} B2_mean[i]    for j = 1, ..., 10

noise_floor(i) = β · median(n_1, ..., n_10)
```

Same robustness argument as Buffer 1: a single low-energy glitch frame
affects at most one sub-window minimum and cannot shift the median.
β = 1.3 same motivation.

**std_noise estimation:**

```
std_noise(i) = mean_{j ∈ B2_std[]} { B2_std[j] }
```

Why mean and not median-of-minima for std_noise?
noise_floor requires the lowest recent background level → minimum-based.
std_noise represents the *typical variability* of the noise amplitude,
not its floor. The mean of per-frame stds from confirmed silent frames
gives the best estimate of this typical variability. A minimum-based
approach would give the most stationary frame's std, which would
systematically underestimate the real variability.

```
RAM:  B2_mean[] buffer:          750 × 4 bytes = 3.0 kB
      B2_std[] buffer:           750 × 4 bytes = 3.0 kB
      local minima array (mean): 10  × 4 bytes = 0.04 kB
      Total Buffer 2:                          = 6.04 kB
```

---

### 4.7 Initialization (first W frames)

Before the buffer is full, the minimum is unstable (it reflects only a partial window). Strategy:

```
For the first W/2 = 375 frames:
  Accept all frames into both buffers without burst protection.
  Rationale: not enough history to estimate Ê_floor reliably,
  so burst protection cannot be applied correctly.

From frame W/2 onward:
  Apply burst protection normally.

Until buffer is full (i < W):
  Compute min/mean over available entries only.
```

---

### 4.8 Complete Per-Frame Update Sequence

```
EVERY FRAME i:
│
├─ [1] Compute E_i from FFT magnitudes
├─ [2] Compute iFFT → Hilbert envelope A[n] → env_mean_i, env_std_i
│
├─ [3] Burst check:
│       if E_i > α · Ê_floor(i-1):
│           → skip buffer updates  (energetic frame)
│       else:
│           B1[i mod W]      = E_i
│           B2_mean[i mod W] = env_mean_i
│           B2_std[i mod W]  = env_std_i
│
├─ [4] Update estimates:
│       m_j = min of each of 10 sub-windows of B1
│       Ê_floor(i)     = β · median(m_1,...,m_10)        → used in Stage 1
│       n_j = min of each of 10 sub-windows of B2_mean
│       noise_floor(i) = β · median(n_1,...,n_10)        → used in Stage 3
│       std_noise(i)   = mean(B2_std)                    → used in Stage 3
│
└─ [5] Stage 1 threshold check:
        if E_i > k · Ê_floor(i):
            → frame is Stage 1 CANDIDATE → proceed to Stage 2
        else:
            → discard frame
```

---

### 4.9 Constants Summary

| Constant | Value | Motivation |
|---|---|---|
| W | 750 frames (~1.92 s) | Tracks floor changes; stable against bursts |
| M | 10 sub-windows (75 frames each) | Median-of-minima robustness; single outlier cannot shift median |
| β | 1.3 | Martin (2001) correction — local min underestimates true floor |
| α | 4 (verify experimentally) | Burst exclusion: E_i > 4× current floor |
| k | 1.5 | Stage 1 threshold multiplier |

---

### 4.10 RAM Budget

| Buffer | Size | RAM |
|---|---|---|
| B1 (FFT energy) | 750 × float32 | 3.00 kB |
| B1 local minima | 10 × float32 | 0.04 kB |
| B2_mean (env mean) | 750 × float32 | 3.00 kB |
| B2_std (env std) | 750 × float32 | 3.00 kB |
| B2_mean local minima | 10 × float32 | 0.04 kB |
| **Total** | | **~9.1 kB** |

Well within the 128 kB SRAM of the STM32F411. On the PC, negligible.

---

## 5. Stage 1 – Adaptive Energy Threshold + Run-Length Filter

### 5.1 Threshold

```
Frame i is a Stage 1 CANDIDATE  iff  E_i > k · Ê_floor(i)
```

- **k** is the final threshold multiplier. v4 used k=5 on a static estimate and its standard deviation; v5 applies k=1.5 to the adaptive floor.
- The adaptive floor means the threshold automatically rises in noisy environments and falls in quiet ones — no manual recalibration needed per session.

### 5.2 Run-Length Filter

Consecutive above-threshold frames are grouped into runs. Runs longer than **MAX_RUN = 3** frames are discarded:

```
MAX_RUN = 3   (changed from 4 in v4)

Run of length L:
  L ≤ 3  →  passes to Stage 2
  L > 3  →  entire run discarded (sustained noise)
```

**Physical rationale:** a genuine cavitation click lasts at most ~0.5 ms ≈ 1–2 frames. A run of 4+ consecutive above-threshold frames (≥ 10.24 ms) is sustained noise. MAX_RUN=3 is slightly tighter than v4's 4 to better reject outdoor impulsive bursts that can span 3–4 frames.

---

## 6. Stage 2 – Hard Gates

Stage 2 reads the microphone-normalized FFT spectrum and computes three spectral features — SPR, R_spectral, and FPE — which are passed to Stage 3 for SVM classification. Their physical meaning and interpretation are documented alongside the other Stage 3 features in §8.9, where all 16 SVM inputs are described together.

In addition, Stage 2 applies two **hard rejection gates** that discard candidates before the SVM is invoked. SPR is used both as an SVM feature (§8.9) and as the basis for Gate 2; the full SPR description is in §8.9.

### 6.1 Hard Rejection Gates (NEW in v5)

Two hard gates are applied before Stage 3. Candidates failing either gate are permanently rejected and do not reach the SVM.

---

**Gate 1 — Minimum Fit Quality (R² < 0.10)**

```
if R² < 0.10  →  reject  (stage_blocked = "Stage2_R2")
```

The exponential fit (§10) is computed before this gate. When R² falls below 0.10 the fit has completely failed: the decay segment is either too short, too noisy, or not monotone at all. In this case τ, decay_start, decay_end, and all features derived from the decay window (fall_time_ms, post_SNR, ZCR_post, asymmetry_integral) are unreliable or undefined.

Passing a candidate with R²<0.10 to the SVM would mean invoking the model on garbage inputs — the SVM was trained on candidates where the fit succeeded, and cannot generalize to this pathological case. The gate is therefore a **validity check**, not a classification decision.

> **Why 0.10 and not a higher value?** The threshold is intentionally lenient. R²=0.10 only rejects complete fit failures. Candidates with moderate fit quality (R²=0.3–0.6, common for clicks with secondary reflections) are passed to the SVM, which was specifically trained to handle them.

---

**Gate 2 — Out-of-Distribution SPR (SPR ≥ 100)**

```
if SPR ≥ 100  →  reject  (stage_blocked = "Stage2_SPR")
```

SPR ≥ 100 indicates an extremely tonal signal — a narrowband oscillator or EMI tone that is completely unlike anything in the training set. Inspection of the full labeled dataset confirmed that **no training sample (click or noise) had SPR ≥ 100**. Applying the SVM to an input outside its training distribution is unsafe: the model's decision boundary was not calibrated for this region of feature space, and the output probability is uninterpretable.

This gate is an **out-of-distribution (OOD) safeguard**, not a learned classification boundary. It is intentionally set far above all observed training values to avoid discarding borderline cases that a better-trained SVM could handle correctly.

> **Relationship to the SVM feature SPR:** SPR values within the training distribution (observed range ~1–30) are still passed to the SVM as a continuous feature. The hard gate only activates at the extreme tail never seen during training.

---

## 7. Pre-processing Pipeline for Stage 3 Candidates

Every frame that survives Stage 1 and Stage 2 undergoes the following reconstruction pipeline before feature extraction.

```
FFT frame (154 bins, magnitudes + int8 phases)
    │
    ▼
[STEP 1] Full spectrum reconstruction
         Zero-pad to 256 bins; apply Tukey taper to COMPLEX spectrum
         (not magnitudes only — see FFT_PHASE_TECHNICAL_SPECIFICATION.md)
    │
    ▼
[STEP 2] iFFT
         x[n] = IRFFT(X_full, N=512)   →  512 samples, 2.56 ms
    │
    ▼
[STEP 3] Gibbs temporal suppression (symmetric AND condition)
         energy_left  = RMS(x[0:15])
         energy_right = RMS(x[497:512])
         energy_int   = RMS(x[40:472])
         if BOTH borders > 2.5 × energy_int:
             apply half-Hann fade on both borders
         else:
             signal unchanged
    │
    ▼
[STEP 4] Hilbert envelope (RAW)
         A[n] = |IRFFT(RFFT(x) · H)|    H doubles positive freqs
         Used for: peak detection, pre_window, ZCR, asymmetry,
                   kurtosis, decay_start/end search
    │
    ▼
[STEP 5] Peak detection
         peak_idx = argmax(A[n])
         peak_amp = A[peak_idx]
```

### 7.1 Stitched Click Context (frame-grid independence)

A cavitation click lasts ≤ ~0.5 ms but it can be borderline, meaning that it happens near the end of the frame. Computing features on a single frame therefore **silently truncates** every time-domain feature of a click that straddles a frame boundary — the decay window index can exceed 511 while the raw-signal array is only 512 long, and NumPy slicing returns a short array instead of raising. This makes `ZCR_post`, `kurtosis`, `centroid_shift_hz` and `asymmetry_integral` frame-alignment dependent.
v5 removes this by resolving and measuring every candidate on a **stitched context** rather than a single frame:

```
build_click_context(prev_sig, curr_sig, next_sig):
    signal   = [ prev | curr | next ]        (up to 1536 samples)
    envelope = Hilbert(signal)               (continuous across the joins)
    origin   = index of curr's first sample
    seams    = sample indices of the frame joins

resolve_click(ctx, noise_floor, std_noise):
    frame_peak = argmax(envelope over the current frame)   # Stage-1 excursion
    onset      = last sample below LEVEL, scanning back from frame_peak
    peak       = argmax(envelope over [onset : onset + 512])  # the TRUE peak
    decay_start, decay_end = decay window from `peak` on the stitched envelope
```

All feature functions then receive the stitched `signal`/`envelope` and indices into them, so a decay that crosses a frame boundary addresses real samples on both sides. The envelope is the Hilbert transform of the *stitched* signal (not per-frame envelopes concatenated), so it is continuous; each frame keeps its own Tukey taper, so the joins carry a mild seam whose positions are tracked and drawn honestly rather than hidden.

**Event identity.** The peak's absolute sample and its owning frame define a model-independent identity used by Stage 4:

```
peak_abs            = frame_idx · 512 + (peak − origin)
canonical_frame_idx = peak_abs // 512
```

The two Stage 1 candidates a straddling click produces (one in the frame where it rises, one in the frame where the tail is detected) scan back to the **same onset** and re-maximise to the **same peak**, so they carry an integer-identical `peak_abs`. This is what makes them collapse exactly in Stage 4.

---

## 8. Stage 3 – Multi-Criterion Temporal Validation (v5)

In v5, Stage 3 produces a **feature vector** fed to an SVM classifier rather than applying hard AND-logic thresholds. The features are grouped below by category.

### 8.1 Peak SNR  *(replaces C1 absolute amplitude)*

```
peak_SNR = peak_amp / noise_floor
```

Dimensionless. Invariant to hardware gain — if an amplifier is added, both `peak_amp` and `noise_floor` scale equally, preserving `peak_SNR`. Replaces the absolute 130 µV threshold of v4.

---

### 8.2 Pre-window and pre_SNR  *(replaces C2)*

**Pre-window definition:**

The pre-window boundary is found dynamically by scanning backward from
`peak_idx` on the raw Hilbert envelope:

```
pre_boundary = last n (going backward from peak_idx) where:
    A[n] < noise_floor + std_noise
```

The pre-window is then the **P = 100 samples immediately before `pre_boundary`**
(or all available samples if fewer than P exist):

```
pre_window = A[pre_boundary - P : pre_boundary]    P = 100 samples (0.5 ms)
```

This is **symmetric with the post-window** (§8.3): both capture the immediate
silence adjacent to the click event, not the entire pre/post-click silence
region. Averaging over the entire pre-click silence would give pre_SNR ≈ 1.0
by construction (the noise estimator is trained on those same silent frames),
reducing discriminative power. The 100-sample localized window is sensitive
to noise or oscillations occurring immediately before the click.

If the click emerges near the start of its frame, the pre-window reaches into the
previous frame automatically: the pre-window is sliced from the stitched context
(§7.1), so P samples are always available regardless of click position. (The
standalone `_build_pre_window` helper that used to do this per-frame is retained
only for reference and is no longer on the live path.)

**pre_SNR:**

```
pre_SNR = RMS(pre_window) / noise_floor
```

A genuine click is preceded by silence → pre_SNR ≈ 1.0.
Embedded noise or a sustained event → pre_SNR > 1.5–2.0.

---

### 8.3 Post-window and post_SNR  *(new)*

**Post-window definition:**

Scan forward from `decay_end` (see §10) for a fixed number of samples:

```
post_window = A[decay_end : decay_end + P]    P = 100 samples (0.5 ms)
```

`decay_end` and the post-window are indices into the stitched context (§7.1), so
a decay that ends in the next frame is measured on real samples rather than
truncated at the 512-sample boundary.

**post_SNR:**

```
post_SNR = RMS(post_window) / noise_floor
```

A genuine click returns to silence → post_SNR ≈ 1.0. A sustained event → post_SNR > 1. Complementary to pre_SNR: pre_SNR checks the approach, post_SNR checks the return.

---

### 8.4 Rise time and Fall time  *(replaces C4 asymmetry ratio)*

**Level definition:**

```
LEVEL = noise_floor + std_noise
```

This defines "the signal has emerged from the noise floor" in a statistically grounded way, invariant to hardware gain and environment.

**Rise time:**

```
rise_time_ms = (peak_idx - last n before peak_idx where A[n] < LEVEL) / fs × 1000
```

**Fall time:**

```
fall_time_ms = (first n after peak_idx where A[n] < LEVEL) / fs × 1000
```

For the fall: scan forward on the raw envelope, using the same logic as the pre-window backward scan. The fall time endpoint is consistent with `decay_end` in the fit pipeline.

**Physical meaning:**
- `rise_time_ms < 0.025 ms` (25 µs): physically impossible for cavitation → likely electrical spike. The SVM will learn this; it is not a hard cut.
- `rise_time_ms > 0.3 ms`: slow mechanical vibration onset, not an impulsive event.
- Both are fed as raw features to the SVM, not used as hard thresholds.

---

### 8.5 Asymmetry Integral  *(replaces C4 asymmetry ratio)*

The asymmetry integral measures the **shape** of the Hilbert envelope around the peak — whether the signal decays slower than it rises, as expected for a cavitation click.

**Window:**

```
W = number of samples from peak_idx to decay_end
    (same endpoint as the fit — consistent with LEVEL-based definition)

left_side  = A[peak_idx - W : peak_idx]
right_side = A[peak_idx : peak_idx + W]
left_flipped = left_side[::-1]    (reversed so both sides start at peak)

asymmetry_integral = sum(right_side - left_flipped) / (W × peak_amp)
```

**Interpretation:**
- **Positive** → decay slower than rise → consistent with cavitation click ✓
- **≈ 0** → symmetric rise and fall → consistent with EMI spike or oscillator ✗
- **Negative** → rise slower than fall → rare, unlikely for genuine click ✗

---

### 8.6 Zero Crossing Rate (ZCR)  *(new)*

ZCR measures oscillation frequency in a robust way, using a **hysteresis band** to avoid false crossings from noise micro-fluctuations.

**Hysteresis band:**

```
band = [-std_noise, +std_noise]
```

A crossing is counted only when the signal **completely crosses** the band:

```
crossing if:  x[n-1] < -std_noise  AND  x[n] > +std_noise   (upward)
          or: x[n-1] > +std_noise  AND  x[n] < -std_noise   (downward)
```

Computed on the raw iFFT signal `x[n]` (not the envelope).

**Three ZCR features:**

```
ZCR_pre:    in pre_window           → low for genuine click (silence before)
ZCR_click:  in [peak_idx - W_click : peak_idx + W_click]
            W_click based on τ from fit  → measures click oscillation frequency
ZCR_post:   in [peak_idx : decay_end]   → decreases during decay for genuine click
```

All normalized:

```
ZCR_norm = n_crossings / window_duration_ms    [crossings per ms]
```

---

### 8.7 Kurtosis  *(new)*

Kurtosis measures the impulsivity of the signal — how much of the energy is concentrated in rare extreme samples vs. spread over the whole window.

```
K = mean((x - mean(x))^4) / std(x)^4

K_excess = K - 3    (excess kurtosis; Gaussian noise → 0)
```

**Window:** from the first `A[n] ≥ noise_floor + std_noise` before the peak, to `decay_end`. Captures the full event without diluting with surrounding silence.

**Typical values:**

| Signal | K_excess |
|---|---|
| Gaussian noise | ≈ 0 |
| Sustained vibration | 2 – 7 |
| Genuine cavitation click | 15 – 50 |
| EMI spike (very sharp) | > 100 |

Note: both genuine clicks and EMI spikes can have high kurtosis. Kurtosis alone is not sufficient — it works in combination with rise_time (spikes have physically impossible rise times) and asymmetry_integral (spikes are symmetric).

---

### 8.8 Spectral Centroid Shift  *(new)*

The spectral centroid SC is the amplitude-weighted mean frequency of the normalized FFT spectrum:

```
SC = Σ(f[k] × |A_norm[k]|²) / Σ(|A_norm[k]|²)    over k ∈ [51, 204]
```

For a genuine click, higher frequencies decay faster than lower frequencies due to frequency-dependent attenuation. This produces a downward shift in SC during the decay.

**Two-window computation:**

```
SC_early: compute SC on the iFFT signal windowed to the first third of [peak_idx : decay_end]
SC_late:  compute SC on the iFFT signal windowed to the last third of [peak_idx : decay_end]

centroid_shift = SC_early - SC_late    [Hz]
```

**Interpretation:**
- `centroid_shift > 2–5 kHz` → high frequencies decaying first → consistent with genuine click ✓
- `centroid_shift ≈ 0` → stationary spectral shape → consistent with sustained noise or oscillator ✗

---

### 8.9 FFT Spectral Features (computed in Stage 2, passed to SVM)

These three features are computed directly from the normalized FFT magnitude spectrum of the candidate frame. Because they require only the frequency-domain data already available at Stage 2, they are computed there and carried forward into the SVM feature vector — but their role is classification, not gating.

> **Known limitation (deferred):** SPR, R_spectral and FPE_hz are still computed from the *whole 2.56 ms frame FFT*, not from the click's onset→decay_end region. For a click occupying a small slice of its frame, they describe mostly noise, and for a borderline click the frame holds only half the click's energy. Unlike the time-domain features (§7.1), these were **not** migrated to the stitched-context region in this revision — that is a separate, deliberately deferred change, so that only one feature-distribution shift lands per SVM retrain.

---

**Spectral Peak Ratio (SPR)**

```
SPR = max_k |A_norm[k]|²  /  mean_k |A_norm[k]|²
```

Measures how tonal (concentrated) the spectrum is, independently of absolute amplitude. A genuine click excites a broad range of frequencies → low SPR. A narrowband oscillator or EMI tone concentrates energy in one bin → high SPR. The v4 hard SPR threshold is replaced here by a soft SVM input; only the extreme OOD tail (SPR ≥ 100) is still hard-gated in Stage 2 (§6.1).

**Typical ranges observed in training data:** clicks ≈ 1–8, noise ≈ 1–30.

---

**Spectral Energy Ratio (R_spectral)**

```
E_low  = Σ_{k ∈ [51, 102]}  |A[k]|²     (20–40 kHz half-band)
E_high = Σ_{k ∈ [103, 204]} |A[k]|²     (40–80 kHz half-band)

R_spectral = E_low / E_high
```

Describes the dominant frequency content of the event: R > 1 → energy concentrated in the lower half of the analysis band; R < 1 → energy concentrated in the upper half. Plant cavitation clicks can fall on either side depending on xylem conduit geometry and the embolism event. The SVM learns the click distribution in this feature without a hard threshold.

---

**Frequency of Peak Energy (FPE)**

```
FPE_hz = f[ argmax_k |A_norm[k]|² ]
```

The frequency bin carrying the maximum spectral power, converted to Hz. Provides the SVM with information about where in the 20–80 kHz band the click is brightest. In the feature importance analysis (§12.9), FPE ranked 3rd — clicks tend to cluster in the 20–40 kHz range while many noise events peak higher.

---

**Fit-derived features: τ, R², fit_coverage**

| Feature | Formula | SVM? |
|---|---|---|
| `tau_ms` | From OLS fit pipeline (§10) | ✓ |
| `R2` | From OLS fit pipeline (§10) | ✓ |
| `fit_coverage` | `n_fit / (decay_end − decay_start)` | ✗ (diagnostic only — see §12.5) |

---

## 9. Stage 4 – Deduplication

Deduplication now groups by the click's **absolute peak sample** (`peak_abs`, §7.1), not by frame-index proximity. Because both Stage 1 candidates of a straddling click resolve to an integer-identical `peak_abs`, they land in the same group by construction — identity is deterministic and independent of the SVM.

```
PEAK_MATCH_SAMPLES = 8 samples (~40 µs)   # a jitter margin, NOT a time window

1. Sort Stage 3 survivors by peak_abs
2. Start a new group when the peak_abs gap exceeds PEAK_MATCH_SAMPLES
3. Within each group keep the CANONICAL representative — the candidate whose
   own frame owns the peak (frame_idx == canonical_frame_idx), so its context
   has the peak centred and therefore the cleanest envelope. Fall back to
   highest svm_probability, then earliest frame.
4. Assign timestamp = frame_start_time of the retained (canonical) frame
```

**Behaviour change vs. the old frame-gap logic.** The previous rule merged any detections within 3 frames (~7.7 ms) and broke ties by `svm_probability`. That merged two genuinely distinct clicks less than 7.7 ms apart, and let the retained frame move with every retrained model. Peak-sample grouping keeps distinct clicks separate and fixes the retained frame to the one that physically owns the peak. Expect detection counts to shift accordingly.

`DEDUP_WINDOW_FRAMES` (= 3) is retained in the code only because `evaluate_candidates.py` imports it; it no longer drives Stage 4.

---

## 10. Fit Pipeline: τ and R²

### 10.1 Overview

The fit estimates the **exponential decay constant τ** and the **goodness-of-fit R²** from the Hilbert envelope of the reconstructed click. The physical model is:

```
A(t) = A₀ · exp(-t / τ)
```

Natural_Log-linearizing:

```
ln(A(t)) = ln(A₀) - t/τ    →    y = b + m·x
```

where `m = -1/τ_samples` (always negative for a decay) and `τ_ms = -1000 / (m · fs)`.

### 10.2 Step-by-Step Pipeline

```
INPUT: raw Hilbert envelope A[n], peak_idx, noise_floor, std_noise

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
STEP A — Find decay_start on RAW envelope
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Starting from peak_idx, scan forward.
Compute sliding local slope with window W=4:

    slope_local[n] = (A[n + W - 1] - A[n]) / (W - 1)
    threshold      = 0.01 × peak_amp

Find first n such that:
    slope_local[n]   < -threshold
    slope_local[n+1] < -threshold
    slope_local[n+2] < -threshold
    (3 consecutive windows of W=4 all showing descent)

decay_start = that n
Hard cap: decay_start ≤ peak_idx + 20 samples (100 µs = τ_min)
Fallback: decay_start = peak_idx + 5 if not found within cap

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
STEP B — Find decay_end on RAW envelope
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Scan forward from decay_start.
Find first n such that:
    A[n+1], A[n+2], A[n+3], A[n+4]   all < noise_floor + α × std_noise
    (Y = 4 consecutive samples, α = 1.5)

decay_end = that n
If not found within current frame: extend into next frame.
If still not found: decay_end = end of available data.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
STEP C — Gaussian smoothing (for fit only)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Apply Gaussian kernel with σ = 2 samples (10 µs) to
A[decay_start : decay_end]:

    kernel: w[k] = exp(-k² / (2 × 2²)), normalized to sum = 1
    convolution with mode='valid'
    → fit_window  (slightly shorter due to valid mode)

Motivation: σ = 10 µs is an order of magnitude below the
shortest click (100 µs), suppressing noise without distorting
the decay shape. Gaussian chosen over moving average because
it has no sidelobes in frequency domain — no spurious ripple
introduced into the log-envelope.
mode='valid' avoids zero-padding artefacts at borders.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
STEP D — OLS log-linear fit
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Guard: if len(fit_window) < 10 → τ = -1, R² = 0, skip.

log_env = log(max(fit_window, 1e-9))
n_array = [0, 1, ..., n_fit - 1]

Closed-form OLS (thread-safe, no BLAS/LAPACK):
    n_pts  = n_fit
    sum_x  = Σ n
    sum_y  = Σ log_env
    sum_xx = Σ n²
    sum_xy = Σ n · log_env
    denom  = n_pts · sum_xx - sum_x²

    slope m     = (n_pts · sum_xy - sum_x · sum_y) / denom
    intercept b = (sum_y - m · sum_x) / n_pts

if m < 0:
    τ_ms = -1000 / (m × fs)
else:
    τ_ms = -1    (no decay — sentinel)

R² computed in log space:
    ŷ[n]   = m · n + b
    SS_res = Σ (log_env - ŷ)²
    SS_tot = Σ (log_env - mean(log_env))²
    R²     = 1 - SS_res / SS_tot

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
OUTPUT
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
τ_ms           — decay time constant
R²             — goodness of exponential fit (log space)
fit_coverage   — len(fit_window) / (decay_end - decay_start)
                 diagnostic: low value → short or truncated fit
decay_start    — used also for asymmetry integral window W
decay_end      — used also for fall_time, ZCR_post, post_SNR
```

### 10.3 Why OLS on the Full Fit Window (not monotone segment)

The second local peak visible in some click recordings (a reflection or secondary cavitation event) slightly degrades R². However:

- Restricting the fit to the longest monotone segment introduces a new, fragile segmentation step.
- The R² degradation from the secondary peak is moderate (typically 0.65–0.75 instead of 0.85+) and **informative** — the SVM can learn that moderate R² combined with other strong features still indicates a genuine click.
- `fit_coverage` provides a complementary diagnostic: a fit over the full window with moderate R² is more trustworthy than a fit over a tiny monotone segment with high R².

---

## 11. Feature Summary Table

16 of the 17 computed features are fed to the SVM. `fit_coverage` is computed but excluded from inference (see §12.5).

| # | Feature | Domain | Physical meaning | SVM? |
|---|---|---|---|---|
| 1 | `peak_SNR` | iFFT | Amplitude relative to noise floor | ✓ |
| 2 | `pre_SNR` | iFFT | Silence before click | ✓ |
| 3 | `post_SNR` | iFFT | Return to silence after click | ✓ |
| 4 | `rise_time_ms` | iFFT envelope | Onset speed | ✓ |
| 5 | `fall_time_ms` | iFFT envelope | Decay duration at noise level | ✓ |
| 6 | `asymmetry_integral` | iFFT envelope | Rise/fall shape asymmetry | ✓ |
| 7 | `ZCR_pre` | iFFT raw | Oscillation rate before click | ✓ |
| 8 | `ZCR_click` | iFFT raw | Oscillation rate during click | ✓ |
| 9 | `ZCR_post` | iFFT raw | Oscillation rate during decay | ✓ |
| 10 | `kurtosis` | iFFT raw | Impulsivity of event | ✓ |
| 11 | `centroid_shift_hz` | FFT | Spectral evolution during decay | ✓ |
| 12 | `τ_ms` | iFFT envelope fit | Exponential decay constant | ✓ |
| 13 | `R²` | iFFT envelope fit | Exponential fit quality | ✓ |
| 14 | `fit_coverage` | iFFT envelope fit | Fraction of decay used in fit | ✗ (diagnostic only) |
| 15 | `SPR` | FFT | Spectral broadband shape | ✓ |
| 16 | `R_spectral` | FFT | Low vs high frequency balance | ✓ |
| 17 | `FPE_hz` | FFT | Dominant frequency | ✓ |

**Removed from v4:**
- `E_W1 / E_W4` ratio (fixed 300-sample window energy decay) → replaced by τ from improved fit
- `asymmetry_ratio` (rise/fall sample count) → replaced by rise_time_ms, fall_time_ms, asymmetry_integral
- Absolute amplitude threshold 130 µV → replaced by peak_SNR

---

## 12. SVM Classifier — Training Protocol and Results

### 12.1 Design Philosophy: High-Recall Bias

The SVM is optimized for **recall** (sensitivity), not F1 or accuracy. The rationale follows from the cost asymmetry of the two error types in plant stress research:

- **False Negative (missed click):** a genuine acoustic emission is lost. In experiments that may run for hours with only a few dozen real events, losing even one click represents a significant fraction of the evidence. Repeated false negatives can mask or distort the plant's stress response.
- **False Positive (false alarm):** an additional noise event appears in the output. A human reviewer can quickly dismiss it; automated downstream analysis can tolerate a controlled FP rate.

Given this asymmetry, the recall target is set at **≥ 0.90**, and the decision threshold is lowered from the SVM default (0.50) to whatever value on the ROC curve first achieves that recall. At the same time, precision is monitored to ensure the FP rate does not become unmanageably high.

---

### 12.2 Dataset Composition

The training dataset was assembled from manually labeled Stage 1 candidates collected across multiple recording sessions and plant species:

| | Count |
|---|---|
| Total labeled rows | 285 |
| Confirmed clicks (label=1) | 91 |
| Noise events (label=0) | 194 |
| Recording sessions | 38 |

**Species and recording conditions in the dataset:**

| Category | Species / Environment | Stimulus type |
|---|---|---|
| Stressed plants | *Aloe vera* | Mechanical (needle), water stress |
| Stressed plants | *Ferrocactus* (cactus) | Mechanical (needle), water stress |
| Stressed plants | *Calanchoe* | Water stress |
| Stressed plants | Carnivorous plant | Mechanical |
| Noise-only | Outdoor balcony | Wind, traffic, bird, construction |
| Noise-only | Indoor laboratory room | HVAC, electronic, ambient |
| Noise-only | Empty room, no plant | Baseline MEMS noise |

Recording conditions span controlled indoor environments and uncontrolled outdoor exposures, deliberately chosen to make the noise class diverse and representative of real deployment scenarios.

**Set B (held-out test set):** one session of *Aloe vera* + water stress (`Aloe_acqua50ml_misurazione1_11032026_09`, 16 clicks + 10 noise), withheld entirely from training and hyperparameter search.

---

### 12.3 Hard-Negative Mining (Noise Pre-Filtering Before Training)

Before the SVM is trained, Stage 2 hard gates (§6.1) are applied to the **noise samples only**:

```
Noise samples with R² < 0.10  or  SPR ≥ 100  →  removed from training set
194 noise samples  →  192 noise samples (2 removed)
Click samples: unchanged (all 91 retained)
```

**Rationale:** trivially rejectable noise samples (catastrophic fit failures, extreme tonal signals) would be correctly classified by almost any model. Including them in training inflates apparent specificity without teaching the SVM anything about the hard cases — noise signals that are broadband, impulsive, and superficially similar to genuine clicks in feature space. By removing the trivial negatives, the SVM is forced to learn the discrimination boundary in the region that actually matters.

This is a form of **hard-negative mining**: rather than curating individual difficult examples, the Stage 2 gates act as an automatic filter that concentrates training on the non-trivial portion of the noise distribution.

---

### 12.4 Session-Level Cross-Validation (Preventing Data Leakage)

Cross-validation uses **StratifiedGroupKFold** with groups defined at the session level (one group = one `.paudio` recording file):

- Each fold is guaranteed to have no session appearing in both training and validation splits.
- **Why this matters:** a single recording session can contribute dozens of Stage 1 candidates. Without session-level grouping, the same physical recording could have some candidates in training and others in validation. The SVM could then learn recording-specific acoustic signatures (microphone position, plant height, background noise level) rather than generalizable click morphology — artificially inflating CV metrics.
- 5-fold cross-validation; stratification ensures each fold preserves the click/noise class ratio.

---

### 12.5 `fit_coverage` Exclusion

`fit_coverage` (fraction of the decay window successfully used in the OLS fit) is computed alongside the other features but **excluded from the SVM feature vector**:

```
Active SVM features: 16 of 17  (fit_coverage excluded)
```

The reason is potential artificial discrimination: a genuine click tends to have a clean, long decay that the fit pipeline can fit well (high coverage), while a noise burst may produce a shorter or noisier decay (lower coverage). Including fit_coverage would give the SVM a shortcut that may not generalize — coverage depends on the specific decay window selection algorithm and the noise floor estimate, both of which could change with hardware or environmental conditions. The other 16 features are more directly tied to the physical properties of the event.

---

### 12.6 Kernel and Hyperparameter Search

Two kernel families were evaluated in preliminary experiments: linear and RBF. The RBF kernel was selected because the click/noise boundary in the 16-dimensional feature space is non-linear (demonstrated empirically).

**Grid search (RBF kernel):**

| Hyperparameter | Search range | Best value |
|---|---|---|
| C (regularization) | {0.1, 1, 10, 50, 100} | **50** |
| γ (RBF bandwidth) | {0.001, 0.01, 0.1, 1} | **0.01** |

- Primary scoring metric: **recall** (not F1, not accuracy)
- 20 combinations × 5 folds = 100 fits
- Best CV recall at default threshold: 0.728

---

### 12.7 Decision Threshold Optimization

The SVM's default decision threshold (0.50) gives poor recall (0.467 at best CV params). The optimal threshold is found from the out-of-fold ROC curve:

```
Threshold = lowest value achieving recall ≥ 0.90 on cross-validated predictions
Result: threshold = 0.220
```

At this threshold the model aggressively accepts borderline candidates. The precision drop (0.660 → 0.540) is acceptable given the cost asymmetry described in §12.1.

---

### 12.8 Results

**Cross-validated performance on Set A (StratifiedGroupKFold, 5 folds):**

| Metric | Default threshold = 0.50 | Operational threshold = 0.220 |
|---|---|---|
| Recall | 0.467 | **0.907** |
| Precision | 0.660 | 0.540 |
| Specificity | 0.901 | 0.681 |
| F1 | 0.547 | 0.677 |
| AUC-ROC | 0.835 | 0.835 |

**Held-out test set (Set B — unseen session, *Aloe vera* + water):**

| Metric | Value |
|---|---|
| Recall | **0.875** (14/16 clicks detected) |
| Precision | 0.824 |
| Specificity | 0.700 |
| F1 | 0.848 |
| AUC-ROC | **0.925** |

The Set B recall (0.875) is slightly below the CV target (0.907). This is expected: Set B represents a genuinely different session and the threshold was optimized on Set A. The AUC (0.925) confirms that the underlying discrimination is strong; the small gap is threshold positioning, not model failure.
Future developments will aim to a more precise selection of clicks and noises in Set A (note that precision is low because of hard-mining and that the current dataset, because of this reason, might have misclassified some noises; proof that the current SVM achieves good precision can be accounted in Dataset B, composed of all noise passing stage 1 and 2). An even stronger annotation method to divide clicks and noise confirmed as such will be soon thoght and developed. If you have any suggestion, don't hesitate to reach out!

---

### 12.9 Feature Importance

Permutation importance computed on Set A (n_repeats=15, metric=recall drop when feature is shuffled):

| Rank | Feature | Δ Recall | Interpretation |
|---|---|---|---|
| 1 | `fall_time_ms` | +0.119 ± 0.028 | Primary decay duration; most discriminative single feature |
| 2 | `peak_SNR` | +0.104 ± 0.021 | Amplitude above noise; second-most important |
| 3 | `FPE_hz` | +0.065 ± 0.032 | Dominant frequency; clicks cluster in 20–40 kHz range |
| 4 | `post_SNR` | +0.064 ± 0.028 | Return to silence after event |
| 5 | `pre_SNR` | +0.061 ± 0.027 | Silence before event |
| 6 | `tau_ms` | +0.056 ± 0.020 | Decay time constant |
| 7 | `rise_time_ms` | +0.047 ± 0.020 | Onset speed |
| 8 | `kurtosis` | +0.044 ± 0.034 | Impulsivity |
| 9 | `ZCR_post` | +0.032 ± 0.023 | Oscillation during decay |
| 10 | `asymmetry_integral` | +0.025 ± 0.020 | Rise/fall shape |
| 11 | `ZCR_pre` | +0.023 ± 0.022 | Oscillation before click |
| 12 | `ZCR_click` | +0.017 ± 0.013 | Oscillation during click |
| 13 | `R_spectral` | +0.015 ± 0.018 | Spectral balance |
| 14 | `R2` | +0.010 ± 0.021 | Fit quality (soft version of the hard gate) |
| 15 | `SPR` | +0.006 ± 0.019 | Spectral tonality (soft version of the hard gate) |
| 16 | `centroid_shift_hz` | −0.002 ± 0.016 | No measurable contribution at current dataset size |

> **Note on `centroid_shift_hz`:** the near-zero importance does not mean spectral centroid shift is physically irrelevant. It may lack discriminative power at the current dataset size (n=75 clicks) or may be correlated with other features already captured. 

> **Note on `R2` and `SPR`:** both appear at the bottom of the importance ranking, consistent with the hard gate design — the hard gates already remove the extreme cases where R² and SPR would have been most predictive. What remains in the training set is the moderate range where both features still carry some signal but are no longer decisive alone.

---

### 12.10 Deployment

The trained model is saved as a `.pkl` file containing:

```python
{
    'pipeline':  sklearn.Pipeline,   # steps: imputer → scaler → SVM
    'threshold': 0.220,              # operational decision threshold
    'kernel':    'rbf',
    'features':  [...],              # ordered list of 16 feature names
    'all_results': {...}             # full CV log
}
```

Inference:

```python
import joblib, numpy as np
model = joblib.load('plantleaf_svm_v5_nofitcoverage.pkl')
pipe  = model['pipeline']
thr   = model['threshold']   # 0.220
# X: (n, 16) array, columns in model['features'] order
proba = pipe.predict_proba(X)[:, 1]
pred  = (proba >= thr).astype(int)   # 1 = click, 0 = noise
```

---

## 13. Algorithm Version History

| Version | Date | Key changes |
|---|---|---|
| v1.0 | August 2025 | Basic energy threshold + single R² criterion |
| v2.0 | November 2025 | Added R, iFFT reconstruction, Hilbert envelope |
| v2.1 | December 2026 | R²_log ≥ 0.60–0.80 as primary gate; 22% FNR discovered |
| v3.0 | Februery 2026 | R² removed as gate; 3-criteria (SNR, pre_snr, E_W1>E_W4) |
| v3.1 | March 2026 | Added asymmetry (C4), τ range (C5); window extended to 300 samples |
| v4.0 | March 2026 | Absolute peak_iFFT (C1=130µV); τ criterion; R² criterion; asymmetry reformulated; Gibbs suppressor v3; Stage 1 k=5 + MAX_RUN=4; Stage 2 normalized peak filter |
| **v5.0** | **May–June 2026** | **Adaptive noise estimator; Stage 1 adaptive threshold + MAX_RUN=3; Stage 2 hard gates (R²<0.10, SPR≥100); all other thresholds replaced with SVM features; improved fit pipeline (dynamic window, Gaussian smoothing, slope-based decay_start); new features: post_SNR, ZCR×3, kurtosis, centroid_shift, rise/fall time, asymmetry_integral; E_W1/E_W4 removed; RBF-SVM trained on 285 labeled events from 38 sessions (16 features, threshold=0.220, CV recall=0.907, Set B AUC=0.925)** |
| v5.1 | July 2026 | Frame-grid-independent features: every time-domain feature is now resolved and measured on a stitched prev\|curr\|next context (§7.1), fixing the silent truncation of `ZCR_post`/`kurtosis`/`centroid_shift_hz`/`asymmetry_integral` for boundary-straddling clicks. Stage 4 deduplicates by absolute peak sample (`peak_abs`) instead of frame-index gap (§9); one screenshot per physical click, peak-centred and frame-independent. Spectral features (SPR/R_spectral/FPE_hz) still frame-based — Region-FFT migration deferred. **Changes feature values → SVM retrain required.** |

---

## 14. References

1. **Khait I., et al.** "Sounds emitted by plants under stress are airborne and informative." *Cell*, 186(7):1328–1336, 2023.
2. **Martin R.** "Noise power spectral density estimation based on optimal smoothing and minimum statistics." *IEEE Trans. Speech Audio Process.*, 9(5):504–512, 2001. *(Minimum-statistics noise floor estimator)*
3. **Cohen I., Berdugo B.** "Noise estimation by minima controlled recursive averaging." *IEEE Signal Process. Lett.*, 9(1):12–15, 2002. *(Adaptive noise tracking)*
4. **Vaseghi S.V.** *Advanced Digital Signal Processing and Noise Reduction*, 4th ed. Wiley, 2008.
5. **Tyree M.T., Sperry J.S.** "Do woody plants operate near the point of catastrophic xylem dysfunction?" *Plant Physiology*, 88(3):574–580, 1988.
6. **Boashash B.** *Time-Frequency Signal Analysis and Processing*, 2nd ed. Academic Press, 2015.
7. **Harris F.J.** "On the use of windows for harmonic analysis with the DFT." *Proceedings of the IEEE*, 66(1):51–83, 1978.
8. **Knowles Electronics.** *SPU0410LR5H-QB Datasheet*, Rev. H, 2020.
9. **STMicroelectronics.** *STM32F411CEU6 Datasheet*, 2023.

---

*Document maintained by the PlantLeaf project contributors.*
*Last updated: June 2026 — v5.1*