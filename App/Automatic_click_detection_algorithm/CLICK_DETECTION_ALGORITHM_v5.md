# PlantLeaf – Automatic Ultrasonic Click Detection Algorithm

**Version:** 5.0 (draft)
**Date:** May 2026
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
5. [Stage 1 – Adaptive Energy Threshold + Run-Length Filter](#5-stage-1--adaptive-energy-threshold--run-length-filter)
6. [Stage 2 – FFT Filters (SPR + Normalized Peak)](#6-stage-2--fft-filters-spr--normalized-peak)
7. [Pre-processing Pipeline for Stage 3 Candidates](#7-pre-processing-pipeline-for-stage-3-candidates)
8. [Stage 3 – Multi-Criterion Temporal Validation (v5)](#8-stage-3--multi-criterion-temporal-validation-v5)
9. [Stage 4 – Deduplication](#9-stage-4--deduplication)
10. [Fit Pipeline: τ and R²](#10-fit-pipeline-τ-and-r)
11. [Feature Summary Table](#11-feature-summary-table)
12. [Algorithm Version History](#12-algorithm-version-history)
13. [References](#13-references)

---

## 1. Overview and Motivation for v5

Version 4.0 was designed and validated primarily in **controlled indoor environments**. Outdoor field recordings introduced new challenges that motivated a substantial revision:

- **Variable noise floor**: wind, traffic, insects, and mechanical vibration produce a noise floor that changes on timescales of seconds, making the static `noise_rms` estimate of v4 unreliable.
- **Impulsive outdoor noise**: transient events (claps, footsteps, mechanical impacts) produce FFT patterns superficially similar to clicks — broadband, brief, high-energy — and pass multiple v4 criteria.
- **Hardware coupling artefacts**: without rigid mounting, PCB vibrations couple mechanically into the MEMS microphone, producing low-frequency broadband bursts concentrated near the 20 kHz analysis band edge.
- **Tonal interference**: some outdoor environments (and the MCU itself) produce narrowband tones at fixed frequencies (e.g. 40 kHz, 80 kHz) that pass energy thresholds.

v5 addresses all of these with: an adaptive noise estimator, a more robust run-length filter in Stage 1, a fully redesigned set of Stage 3 features, and an improved decay fit pipeline. All hard thresholds from v4 are replaced with **dimensionless, scale-invariant features** suitable for an SVM classifier.

> **Principal Design:** no single experimentally chosen threshold. All parameters are either physically motivated constants or features fed to an SVM. The noise floor and its standard deviation are the two fundamental quantities from which most features are derived, ensuring invariance to hardware gain and environmental amplitude changes.

---

## 2. v4 → v5 Comparison Summary

| Aspect | v4.0 | v5.0 |
|---|---|---|
| Noise floor estimate | Global offline, static per session | Adaptive sliding window (minimum-statistics) |
| Stage 1 threshold | `μ + k·σ` (static k=5) | `k · Ê_floor` (adaptive floor, k to be set) |
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

The v4 `noise_rms` was computed once per session from randomly sampled silent frames. This worked indoors but fails outdoors where the noise floor drifts over seconds (wind gusts, passing vehicles, changing humidity). v5 replaces this with a **sliding-window minimum-statistics estimator** that tracks the noise floor locally in time.

### 4.2 Frame Energy

Each frame's energy is computed identically to Stage 1:

```
E_i = (1/K) · Σ_{k=0}^{K-1} |A_i[k]|²

K = 154  (number of transmitted bins)
```

### 4.3 Sliding Window Minimum

A circular buffer of the last **W frames** is maintained. The noise floor estimate for frame i is:

```
Ê_floor(i) = β · min_{j ∈ [i-W, i]} E_j
```

- **β** is a correction coefficient > 1, because the minimum underestimates the true mean noise floor. Martin (2001) suggests β ≈ 1.5 as a starting point; verify experimentally on your recordings.
- **W** must be long enough that the minimum is stable (avoiding burst noise setting a false low floor) but short enough to track real floor changes (wind steps):
  - Long enough: minimum stable → avoid bursts ≥ 30 ms → W ≥ ~20 frames
  - Short enough: floor changes like wind steps on timescales of ~1–2 s → W ≤ ~750 frames
  - **Chosen: W = 750 frames (~1.92 s)** → RAM ≈ 750 × 4 bytes ≈ 3 kB on STM32F411

### 4.4 Burst Protection

Impulsive outdoor noise (a clap, a car door) must not contaminate the minimum estimate. A frame is excluded from the minimum calculation if it is **energetic**:

```
if E_i > α · Ê_floor(i-1):
    discard E_i from minimum calculation  (do not update buffer)
```

- **α** controls the burst exclusion threshold. Suggested starting value: **α = 2**. Verify experimentally.

### 4.5 std_noise Estimation

The standard deviation of the noise is estimated from the same buffer of non-burst frames:

```
std_noise(i) ≈ std({ E_j : j ∈ [i-W, i], E_j ≤ α · Ê_floor })
```

`noise_floor` and `std_noise` together are the two fundamental quantities used in all downstream features.

### 4.6 STM32 Compatibility

The sliding window minimum and the circular buffer are entirely computable on the STM32F411 with integer arithmetic. The buffer of W=750 float32 values requires ~3 kB of RAM, well within the 128 kB SRAM of the STM32F411.

---

## 5. Stage 1 – Adaptive Energy Threshold + Run-Length Filter

### 5.1 Threshold

```
Frame i is a Stage 1 CANDIDATE  iff  E_i > k · Ê_floor(i)
```

- **k** is the final threshold multiplier. v4 used k=5 on a static estimate; v5 applies k to the adaptive floor.
- Suggested starting values: k ∈ {1.5, 2√3 ≈ 3.46}. Choose empirically on outdoor recordings.
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

## 6. Stage 2 – FFT Filters (SPR + Normalized Peak)

Unchanged from v4 in structure. Both filters operate on the microphone-normalized FFT spectrum.

### 6.1 Filter A – Normalized Peak Amplitude

```
peak_norm = max_k A_norm[k]    (over 154 bins)

Frame PASSES  iff  peak_norm > min_peak_fft
```

`min_peak_fft` is a session-adjustable parameter (default 0.85 mV in v4). In v5 this will eventually become a ratio to `noise_floor` for full scale invariance; for now it is kept as an absolute value for continuity.

### 6.2 Filter B – Spectral Peak Ratio (SPR)

```
SPR = max_k |A_norm[k]|²  /  mean_k |A_norm[k]|²

Frame PASSES  iff  SPR ≤ max_spr   (default 20)
```

SPR is amplitude-invariant — it rejects tonal/narrowband signals (EMI, oscillators at 40/80 kHz) regardless of their absolute amplitude.

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

**Pre-window definition (new):**

Instead of a fixed guard, the pre-window is found dynamically by scanning backward from `peak_idx` on the raw Hilbert envelope:

```
pre_window ends at the first n (going backward from peak_idx) where:
    A[n] < noise_floor + std_noise
```

Everything before that point is the pre-window. This defines "the signal has not yet emerged from the noise" without any arbitrary guard distance.

**pre_SNR:**

```
pre_SNR = RMS(pre_window) / noise_floor
```

A genuine click is preceded by silence → pre_SNR ≈ 1.0. Embedded noise or a sustained event → pre_SNR > 1.5–2.0.

---

### 8.3 Post-window and post_SNR  *(new)*

**Post-window definition:**

Scan forward from `decay_end` (see §10) for a fixed number of samples:

```
post_window = A[decay_end : decay_end + P]    P = 100 samples (0.5 ms)
```

If `decay_end` is near the frame boundary, extend into the next frame.

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

### 8.9 Features carried over from v4

| Feature | Formula | Notes |
|---|---|---|
| SPR | `max(power) / mean(power)` over 154 bins | Unchanged from v4 |
| R_spectral | `E_low / E_high` (20–40 kHz vs 40–80 kHz) | Descriptive, unchanged |
| FPE | `f[argmax(power)]` | Descriptive frequency location |
| τ | From improved fit pipeline (§10) | Now SVM feature, not hard threshold |
| R² | From improved fit pipeline (§10) | Now SVM feature, not hard threshold |
| fit_coverage | `n_fit / (decay_end - decay_start)` | Diagnostic: fraction of decay used in fit |

---

## 9. Stage 4 – Deduplication

Unchanged from v4.

```
MAX_GAP = 3 consecutive frames (~7.7 ms)

1. Sort Stage 3 survivors by frame_idx
2. Group frames with gap ≤ MAX_GAP
3. Within each group, keep frame with maximum peak_amp
4. Assign timestamp = frame_start_time of retained frame
```

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

All features below are fed to the SVM. No hard thresholds in Stage 3.

| # | Feature | Domain | Physical meaning |
|---|---|---|---|
| 1 | `peak_SNR` | iFFT | Amplitude relative to noise floor |
| 2 | `pre_SNR` | iFFT | Silence before click |
| 3 | `post_SNR` | iFFT | Return to silence after click |
| 4 | `rise_time_ms` | iFFT envelope | Onset speed |
| 5 | `fall_time_ms` | iFFT envelope | Decay duration at noise level |
| 6 | `asymmetry_integral` | iFFT envelope | Rise/fall shape asymmetry |
| 7 | `ZCR_pre` | iFFT raw | Oscillation rate before click |
| 8 | `ZCR_click` | iFFT raw | Oscillation rate during click |
| 9 | `ZCR_post` | iFFT raw | Oscillation rate during decay |
| 10 | `kurtosis` | iFFT raw | Impulsivity of event |
| 11 | `centroid_shift` | FFT | Spectral evolution during decay |
| 12 | `τ_ms` | iFFT envelope fit | Exponential decay constant |
| 13 | `R²` | iFFT envelope fit | Exponential fit quality |
| 14 | `fit_coverage` | iFFT envelope fit | Fraction of decay used in fit |
| 15 | `SPR` | FFT | Spectral broadband shape |
| 16 | `R_spectral` | FFT | Low vs high frequency balance |
| 17 | `FPE` | FFT | Dominant frequency |

**Removed from v4:**
- `E_W1 / E_W4` ratio (fixed 300-sample window energy decay) → replaced by τ from improved fit
- `asymmetry_ratio` (rise/fall sample count) → replaced by rise_time_ms, fall_time_ms, asymmetry_integral
- Absolute amplitude threshold 130 µV → replaced by peak_SNR

---

## 12. Algorithm Version History

| Version | Date | Key changes |
|---|---|---|
| v1.0 | August 2025 | Basic energy threshold + single R² criterion |
| v2.0 | November 2025 | Added R, iFFT reconstruction, Hilbert envelope |
| v2.1 | December 2026 | R²_log ≥ 0.60–0.80 as primary gate; 22% FNR discovered |
| v3.0 | Februery 2026 | R² removed as gate; 3-criteria (SNR, pre_snr, E_W1>E_W4) |
| v3.1 | March 2026 | Added asymmetry (C4), τ range (C5); window extended to 300 samples |
| v4.0 | March 2026 | Absolute peak_iFFT (C1=130µV); τ criterion; R² criterion; asymmetry reformulated; Gibbs suppressor v3; Stage 1 k=5 + MAX_RUN=4; Stage 2 normalized peak filter |
| **v5.0** | **May 2026** | **Adaptive noise estimator; Stage 1 adaptive threshold + MAX_RUN=3; all hard thresholds replaced with SVM features; improved fit pipeline (dynamic window, Gaussian smoothing, slope-based decay_start); new features: post_SNR, ZCR×3, kurtosis, centroid_shift, rise/fall time, asymmetry_integral; E_W1/E_W4 removed** |

---

## 13. References

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
*Last updated: May 2026 — v5.0 draft*
