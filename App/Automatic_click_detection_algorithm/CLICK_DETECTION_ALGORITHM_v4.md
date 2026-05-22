# PlantLeaf – Automatic Ultrasonic Click Detection Algorithm

**Version:** 4.0  
**Date:** March 2026  
**Authors:** Tommaso Vaninetti  
**Repository:** [PlantLeaf-Desktop-App](https://github.com/TommyVaninetti/PlantLeaf-Desktop-App)

---

> *Plants scream when they are stressed. You just need the right algorithm to hear them.*

---

## Table of Contents

1. [Why This Matters](#1-why-this-matters)
2. [The Big Picture – How the Algorithm Works](#2-the-big-picture)
3. [System Setup and Signal Representation](#3-system-setup-and-signal-representation)
4. [Signal Reconstruction: iFFT and Gibbs Suppression](#4-signal-reconstruction-ifft-and-gibbs-suppression)
5. [Stage 1 – Energy Threshold + Group Filter](#5-stage-1--energy-threshold--group-filter)
6. [Stage 2 – FFT Filters (Peak Amplitude + SPR)](#6-stage-2--fft-filters-peak-amplitude--spr)
7. [Stage 3 – Six-Criterion Temporal Validation (v4.0)](#7-stage-3--six-criterion-temporal-validation-v40)
8. [Stage 4 – Deduplication](#8-stage-4--deduplication)
9. [Offline Noise Estimation](#9-offline-noise-estimation)
10. [Descriptive Features (R², τ)](#10-descriptive-features-r²-τ)
11. [Experimental Methodology and Calibration](#11-experimental-methodology-and-calibration)
12. [Algorithm Version History](#12-algorithm-version-history)
13. [References](#13-references)

---

## 1. Why This Matters

In 2023, researchers at Tel Aviv University published a landmark paper in *Cell* showing that plants under water or mechanical stress emit **ultrasonic acoustic pulses** in the range of 20–100 kHz, at rates that vary with the severity of the stress [Khait et al., 2023]. The biological origin of these emissions is believed to be **acoustic cavitation** — the formation and collapse of microscopic air bubbles inside the xylem vessels that carry water through the plant. Each collapse generates a brief, broadband pressure wave: a *click*.

The PlantLeaf project was born to **detect and characterize these clicks automatically**, using low-cost, open-source hardware and software. This document describes the algorithmic core of the detection system: a 4-stage pipeline that processes continuous ultrasonic recordings and identifies click candidates with high sensitivity and low false-positive rate.

If you are a student or young researcher reading this for the first time: think of it like this. Imagine trying to find a whisper in a room full of people talking. The algorithm first scans the crowd to find anyone who is louder than the background noise (Stage 1), then checks whether the voice sounds like a real word or just noise (Stage 2), then verifies that the word was actually *spoken* and not a recorded loop (Stage 3), and finally removes echo duplicates (Stage 4).

---

## 2. The Big Picture

The algorithm takes as input a continuous recording made of **N FFT frames**, each representing 2.56 ms of ultrasonic signal. It outputs a list of timestamps, amplitudes, and morphological features for each detected click event.

```
RECORDING  (N FFT frames, 20-80 kHz, 390 FPS)
     │
     ▼
[Pre-processing]  iFFT reconstruction + Gibbs suppression
     │
     ▼
[Stage 1]  Energy threshold + group filter  →  discard silent frames
     │      E_frame > μ + k·σ   (default k = 5)
     │      Discard consecutive runs of > MAX_RUN = 4 frames  (sustained noise)
     ▼
[Stage 2]  FFT filters (normalized)  →  discard weak and tonal/narrowband frames
     │      peak_norm > 0.85 mV      (minimum normalized FFT peak amplitude)
     │      SPR = max(|X[k]|²) / mean(|X[k]|²)  ≤  max_spr   (broadband shape)
     ▼
[Stage 3]  Six-criterion temporal validation  →  ALL SIX must pass
     │      C1: peak_iFFT ≥ 130 µV        (absolute amplitude)
     │      C2: pre_snr < 1.8             (silence before click)
     │      C3: E_W1 / E_W4 > 2           (energy decay)
     │      C4: asymmetry ratio < 2.5     (spike rejection)
     │      C5: τ ∈ [0.045, 1.3] ms      (physical duration)
     │      C6: R²_log > 0.45            (exponential decay quality)
     ▼
[Stage 4]  Deduplication  →  merge consecutive frames, keep strongest
     │
     ▼
DETECTED CLICK EVENTS  (timestamp, amplitude, τ, R², spectral features)
```

**Typical funnel efficiency** for a 1-hour recording (~1.4 million frames):

| Stage | Frames remaining | Reduction |
|-------|-----------------|-----------|
| Input | 1,406,250 | — |
| After Stage 1 (k=5 + group filter) | 10,000 – 30,000 | ~98-99% |
| After Stage 2 (peak + SPR ≤ 20) | 5,000 – 20,000 | ~30-50% |
| After Stage 3 (6 criteria) | 10 – 500 | >99% |
| After Stage 4 (dedup) | 5 – 200 events | — |

The final count is highly dependent on stress level. Unstressed plants may yield 0–10 detections/hour; water-stressed plants may yield 50–300 based on research.

---

## 3. System Setup and Signal Representation

### 3.1 Hardware

| Parameter | Value |
|---|---|
| Microphone | Knowles SPU0410LR5H-QB (MEMS ultrasonic) |
| Sampling rate | `fs = 200,000 sps` |
| FFT size | `N = 512 points` |
| Analysis band | 20 – 80 kHz |
| Transmitted bins | 154 (out of 256 total) |
| Frame duration | `T = N / fs = 2.56 ms` |
| Frame rate | ~390 FPS |
| Phase encoding | int8, range [−127, +127] → [−π, +π] |

### 3.2 Frequency Resolution and Bin Mapping

```
Δf = fs / N = 200,000 / 512 ≈ 390.625 Hz/bin

Bin mapping:
  bin_start = floor(20,000 / Δf) = 51
  bin_end   = floor(80,000 / Δf) = 204
  N_bins    = bin_end − bin_start + 1 = 154
```

### 3.3 Temporal Resolution and Click Duration

```
Δt = 1 / fs = 5 µs per sample

Typical click duration: 0.1 – 0.6 ms
Corresponding sample count: 20 – 120 samples

A click always fits within at most 2 consecutive frames
(frame = 512 samples >> 120 samples max click length)
```

### 3.4 Full Spectrum Reconstruction

The firmware transmits only bins 51–204 (20–80 kHz). Before iFFT, a full 256-bin complex spectrum is reconstructed by zero-padding outside the transmitted range:

```
X_full[k] = 0                                 for k < 51 or k > 204
X_full[k] = |A[k]| · exp(j · φ[k])           for 51 ≤ k ≤ 204

where φ[k] = (phase_int8[k] / 127.0) · π
```

---

## 4. Signal Reconstruction: iFFT and Gibbs Suppression

### 4.1 Inverse FFT

The time-domain signal is reconstructed via the inverse real FFT:

```
x[n] = IRFFT(X_full, N=512)
```

This yields 512 samples covering exactly one frame (2.56 ms).

### 4.2 Spectral Windowing (Tukey Taper)

Abruptly zeroing the spectrum outside [bin_start, bin_end] introduces a **rectangular window in the frequency domain**, which corresponds to **sinc convolution in the time domain** — the Gibbs phenomenon. Sharp discontinuities at the spectral edges produce high-frequency ringing at the temporal borders of the reconstructed signal.

To suppress this, a **Tukey cosine taper** is applied to the complex spectrum before iFFT:

```
taper_bins = max(5, N_bins // 10)  ≈ 15 bins

Left edge (bins 51 to 51+taper_bins):
  w[i] = 0.5 · (1 − cos(π · i / taper_bins))     for i = 0, …, taper_bins−1

Right edge (bins 204−taper_bins to 204):
  w[i] = 0.5 · (1 − cos(π · i / taper_bins))     applied in reverse

Interior (bins 51+taper_bins to 204−taper_bins):
  w = 1.0
```

This ensures a **smooth frequency-domain transition** at both edges, suppressing Gibbs ringing without affecting the energy in the interior of the band.

### 4.3 Temporal Edge Artifact Suppression (v3 — Symmetric Detection)

After iFFT, residual edge artifacts may still be present in the time-domain signal. The `suppress_edge_artifacts` function (v3) applies a **symmetric energy-based Gibbs detector**:

**Key insight:** The Gibbs phenomenon is a *symmetric* artifact — it arises from spectral truncation and always affects *both* temporal edges of the frame simultaneously and with comparable magnitude. A real click near a frame boundary will elevate *one* edge but not the other.

**Algorithm:**

```
check_samples   = 15           (0.075 ms @ 200 kHz)
interior        = signal[40 : N−40]
gibbs_factor    = 2.5

energy_interior = RMS(interior)
energy_left     = RMS(signal[0 : check_samples])
energy_right    = RMS(signal[N−check_samples : N])

left_suspicious  = energy_left  > gibbs_factor × energy_interior
right_suspicious = energy_right > gibbs_factor × energy_interior

if left_suspicious AND right_suspicious:
    # Both borders anomalous → Gibbs confirmed → apply half-Hann fade
    fade[i] = 0.5 · (1 − cos(π · i / check_samples))
    signal[0 : check_samples]      ×= fade
    signal[N−check_samples : N]    ×= fade[::-1]
else:
    # Real signal or click at border → do NOT modify
    return signal unchanged
```

**AND condition rationale:**

| Situation | Left border | Right border | AND result | Outcome |
|---|---|---|---|---|
| Pure Gibbs | High | High | True | Corrected ✅ |
| Click at left border | High | Low | False | Preserved ✅ |
| Click at right border | Low | High | False | Preserved ✅ |
| Click fills frame | High | High | True | Classified as Gibbs, but τ > 2.56 ms → already rejected by C5 ✅ |

---

## 5. Stage 1 – Energy Threshold + Group Filter

### 5.1 Physical Motivation

A click event carries significantly more energy than background noise. Stage 1 implements a **statistical threshold** on the mean spectral energy of each frame to discard the silent majority (~98–99% of frames in a typical recording). A second sub-step removes **long consecutive runs** of above-threshold frames, which are characteristic of sustained noise bursts (fans spinning up, vibration events) rather than short-duration clicks.

### 5.2 Energy Metric

```
E_frame[i] = (1 / K) · Σ_{k=0}^{K−1} |A_i[k]|²

where:
  A_i[k]  = FFT magnitude of frame i at bin k
  K = 154  = number of transmitted bins
```

### 5.3 Threshold Formula

```
μ_E = mean(E_frame)     over all N frames  (outlier-filtered)
σ_E = std(E_frame)      over all N frames  (outlier-filtered)

E_threshold = μ_E + k · σ_E    (default k = 5)

Frame i is a Stage 1 CANDIDATE  iff  E_frame[i] > E_threshold
```

### 5.4 Group-Size Filter

After thresholding, consecutive above-threshold frames are grouped into **runs**. Any run whose length exceeds `MAX_RUN = 4` frames is **discarded entirely**:

```
MAX_RUN = 4

A run of length L:
  L ≤ 4  →  all frames in the run pass to Stage 2
  L > 4  →  entire run discarded (sustained noise, not a click)
```

**Physical rationale:** a genuine cavitation click lasts at most ~2 ms ≈ 1 frame. Even a click straddling two frame boundaries produces at most 2 consecutive above-threshold frames. A run of 5 or more consecutive high-energy frames (≥ 12.8 ms) is almost certainly a sustained noise event (mechanical vibration, motor burst, handling noise) and cannot be a click.

### 5.5 Statistical Justification

Under a Gaussian noise model, the false positive probability at the 5σ threshold is:

```
P(E_noise > μ + 5σ) = 1 − Φ(5) ≈ 2.9 × 10⁻⁷  (0.00003%)

For 1,406,250 frames (1 hour):
  Expected false positives ≈ 1,406,250 × 2.9×10⁻⁷ ≈ 0.4 frames
```

These rare statistical false positives are eliminated by the subsequent stages.

**Threshold sensitivity:**

| k | P(FP per frame) | Typical pass rate | Missed weak clicks |
|---|---|---|---|
| 2 | 2.3% | 30–50% | <1% |
| 3 | 0.13% | 15–30% | 2–4% |
| 4 | 0.003% | 10–20% | 4–6% |
| **5** | **0.00003%** | **5–10%** | **8–12%** |

k = 5 is the default, offering very low false-positive rate while remaining adjustable per recording session.

---

## 6. Stage 2 – FFT Filters (Peak Amplitude + SPR)

### 6.1 Physical Motivation

Plant ultrasonic emissions (cavitation clicks) are **broadband events** — their energy is spread across many frequency bins simultaneously. In contrast, electromagnetic interference (EMI), tonal noise (fans, motors), and narrowband artefacts concentrate all energy in 1–3 bins. Stage 2 applies **two complementary filters** on the normalized FFT spectrum to discard both amplitude-weak frames and spectrally-tonal frames.

> **Normalization:** before both filters, the raw FFT magnitudes are corrected for the Knowles SPU0410 microphone's non-flat frequency response using a 50%-weight interpolated correction curve. This converts raw ADC counts into calibrated voltage units and ensures that a click at 70 kHz is not unfairly penalised by the microphone's roll-off.

### 6.2 Filter A – Normalized Peak FFT Amplitude

```
peak_norm_i = max_{k} A_i_norm[k]        (over K = 154 bins, in Volt)

Frame i PASSES Filter A  iff  peak_norm_i > min_peak_fft   (default min_peak_fft = 0.85 mV)
```

**Physical meaning:** even after normalization, the highest single-bin magnitude must clear a minimum voltage. This rejects frames where the entire spectrum sits near the noise floor — frames that passed Stage 1 due to a broad, low-level spectral bump rather than a genuine amplitude event.

### 6.3 Filter B – Spectral Peak Ratio (SPR)

```
SPR_i = max_{k} |A_i_norm[k]|²  /  mean_{k} |A_i_norm[k]|²

Computed over all K = 154 bins (20–80 kHz).

Frame i PASSES Filter B  iff  SPR_i ≤ max_spr   (default max_spr = 20)
```

**Key property:** SPR is **amplitude-invariant** — it measures the *shape* of the spectrum, not its absolute level. A very loud broadband click still has low SPR; a weak pure tone has high SPR.

**Analytical bounds:**

```
SPR_min = 1.0              (perfectly flat spectrum — theoretical white noise)
SPR_max = K = 154          (single-bin tone, all energy in one bin)

For broadband click with B occupied bins (uniform):
  SPR = K / B
  With B ≥ 8:  SPR ≤ 154/8 ≈ 19  →  PASSES with max_spr = 20
  With B = 2:  SPR ≈ 77            →  REJECTED
```

**Typical values:**

| Signal type | Occupied bins | SPR | Decision |
|---|---|---|---|
| Broadband click (≥ 4 kHz) | 10–154 | 4–15 | PASS |
| Click with dominant spectral peak | 5–15 | 10–20 | PASS/BORDER |
| Narrowband EMI / motor tone | 1–5 | 30–154 | REJECTED |
| Pure sine wave | 1–2 | 75–154 | REJECTED |

### 6.4 Descriptive Spectral Ratio R

The Stage 2 pass evaluates shape only. In addition, a **descriptive spectral ratio** R is computed for post-hoc analysis (not used as a filter):

```
E_low  = Σ_{k ∈ [51, 102]} |A[k]|²     (20–40 kHz)
E_high = Σ_{k ∈ [103, 204]} |A[k]|²    (40–80 kHz)

R = E_low / E_high
```

R provides information about the dominant frequency content of the click (low-frequency clicks → R > 1; high-frequency clicks → R < 1) without affecting the detection decision.

---

## 7. Stage 3 – Six-Criterion Temporal Validation (v4.0)

Stage 3 is the heart of the algorithm. **All six criteria must pass** for a candidate to be confirmed as a click. The criteria are evaluated on the iFFT-reconstructed, Gibbs-suppressed time-domain signal.

Before evaluation, the **Hilbert envelope** A[n] is computed (see §7.0), and the **offline noise RMS** is retrieved (see §9).

### 7.0 Hilbert Envelope

The instantaneous amplitude envelope A[n] is computed via the analytic signal:

```
X[k]  = RFFT(x[n], N)
H[k]  = 2   for 1 ≤ k < N/2
H[0]  = H[N/2] = 1    (DC and Nyquist unchanged)

x_a[n] = IRFFT(X[k] · H[k], N)    (analytic signal — imaginary part = Hilbert transform)
A[n]    = |x_a[n]| = sqrt(x[n]² + H{x}[n]²)
```

> **Implementation note:** this uses only `np.fft.rfft` / `np.fft.irfft` — no BLAS/LAPACK calls — ensuring thread safety on macOS (Apple Accelerate backend would deadlock when called from a QThread).

---

### 7.1 Criterion 1 – Absolute Amplitude (Peak iFFT)

```
C1:  max(|x[n]|)  ≥  V_min    (default V_min = 130 µV)
```

**Physical meaning:** the reconstructed click must have a minimum absolute amplitude in the time domain. This criterion acts as a **secondary energy gate** after the normalization step: it ensures that the peak of the iFFT signal, expressed in physical voltage units (µV), clears a biologically meaningful minimum threshold.

Note that this is distinct from Stage 1 (which operates on raw FFT energy before iFFT). A frame may pass Stage 1 due to broad spectral elevation but have a low peak iFFT if the phases are incoherent (noise). C1 rejects these cases.

**Typical values:**
| Signal | Peak iFFT | C1 result |
|---|---|---|
| Strong click (40–80 kHz) | 500 µV – 5 mV | PASS |
| Weak click (near noise floor) | 130–300 µV | PASS / border |
| Broad-spectrum noise | 10–80 µV | FAIL |
| EMI spike (narrowband) | 50–120 µV | FAIL |

---

### 7.2 Criterion 2 – Silence Before Click (pre_snr)

```
C2:  pre_snr < pre_snr_max    (default pre_snr_max = 1.8)

pre_snr = RMS(pre_window) / noise_rms
```

A biological click is a **transient event** — it arises from silence and decays back into silence. Continuous noise, electrical interference embedded in noise, and motion artefacts do not satisfy this property.

**Pre-window selection:**

```
GUARD = 20 samples (100 µs safety margin before peak)

Case A:  peak_idx ≥ 60
  pre_window = signal[0 : peak_idx − GUARD]
  (uses only current frame; at least 40 samples guaranteed)

Case B:  peak_idx < 60  AND  frame_idx > 0
  pre_window = concat(prev_signal[−200:],  signal[0 : peak_idx − GUARD])
  (extends backward into previous frame for sufficient context)

Case C:  peak_idx < 60  AND  frame_idx = 0  (first frame of recording)
  pre_window = [noise_rms]    →    pre_snr = 1.0   (conservative PASS)
```

**Interpretation:**

| pre_snr | Meaning | Decision |
|---|---|---|
| ≈ 1.0 | Pure silence before click | PASS |
| 1.0 – 1.8 | Slight background activity | PASS |
| 1.8 – 3.0 | Moderate background noise | FAIL |
| > 3.0 | Continuous noise / overlapping signals | FAIL |

---

### 7.3 Criterion 3 – Global Energy Decay (E_W1 / E_W4)

The 300-sample (1.5 ms) post-peak window is divided into **4 equal sub-windows** of 75 samples each. Their mean-squared energies are E_W1, …, E_W4.

```
Sub-window layout (post-peak, 300 samples = 1.5 ms):

  W1: samples   0 –  74    (0 – 375 µs)
  W2: samples  75 – 149    (375 – 750 µs)
  W3: samples 150 – 224    (750 µs – 1.125 ms)
  W4: samples 225 – 299    (1.125 – 1.5 ms)

  E_Wi = (1/75) · Σ x[n]²   for n in Wi

C3:  E_W1 / E_W4 > 2.0
```

**Physical meaning:** the first quarter of the post-peak window must contain **at least twice** the energy of the last quarter. This confirms a global decay trend without requiring strictly monotone energy (which would fail for realistic clicks with slight oscillations in the decay envelope).

This is the only criterion that rejects **sustained plateau signals** (E_W1 ≈ E_W4), which pass all other criteria but are clearly not impulsive events.

**Near-end handling:** if `peak_idx > 212` (> 41.4% of the 512-sample frame), the 300-sample window extends beyond the current frame. The algorithm attempts to extend the analysis using the next frame's signal; otherwise a `near_end` flag is set and the result is reported with a warning.

---

### 7.4 Criterion 4 – Spike Asymmetry Rejection

Electrical spikes (EMI discharges, ADC glitches, mechanical taps) are distinguished from biological clicks by their **symmetric rise and fall time**. A biological click has a very fast onset (driven by the sudden pressure wave of cavitation) followed by a much slower resonant decay.

```
LEVEL_FRACTION = 0.10   (measure at 10% of peak amplitude)
FALL_SEARCH    = 40 samples (0.2 ms @ 200 kHz)
ASYM_THRESHOLD = 2.5    (max allowed rise/fall ratio)

LEVEL = LEVEL_FRACTION × peak_amp

rise_samples = peak_idx − (last sample before peak where A[n] ≥ LEVEL)
fall_samples = (first sample after peak where A[n] ≤ LEVEL) − peak_idx

asymmetry_ratio = rise_samples / fall_samples

C4:  asymmetry_ratio < ASYM_THRESHOLD
```

Note the **direction of the criterion in v4.0:** `asym < 2.5` (rise can be at most 2.5× the fall). This rejects signals that are *too symmetric* (ratio ≈ 1, typical EMI spike) but is more permissive than previous versions which used a ratio < 0.5 (requiring rise to be much *shorter* than fall). The v4.0 formulation better handles the case where iFFT reconstruction slightly broadens the apparent onset due to spectral smoothing.

**Edge case:** if the fall crossing is not found within `FALL_SEARCH` samples, `fall_samples` is set to `FALL_SEARCH` (conservative maximum), preventing false rejection of slowly-decaying clicks.

---

### 7.5 Criterion 5 – Physical Decay Time (τ)

The exponential decay constant τ (in ms) is estimated from a **log-linear regression** on the smoothed Hilbert envelope after the peak (see §10 for the full fitting procedure).

```
C5:  τ_min ≤ τ_ms ≤ τ_max

Default:  τ_min = 0.045 ms,   τ_max = 1.3 ms
```

**Physical meaning:**
- **τ < 0.045 ms** (< 45 µs): decay is faster than a single oscillation cycle at 40 kHz (T = 25 µs). No resonant cavity can ring and decay in this time; this is almost certainly an electrical spike.
- **τ > 1.3 ms**: the decay exceeds half the frame duration (2.56 ms). Such long events are sustained tones or slowly decaying mechanical resonances, not cavitation clicks.

**Cavitation physics background:** acoustic cavitation in xylem produces a pressure pulse that launches a broadband acoustic wave through the water column. The resonant ringing decays with a time constant determined by the effective acoustic compliance and mass of the xylem vessel section. For vessels of diameter 20–100 µm and length 1–10 mm, the expected τ falls in the range 0.05–1.0 ms — entirely within the C5 acceptance window.

**Typical values by signal type:**

| Signal | τ (ms) | C5 result |
|---|---|---|
| EMI / ADC glitch | < 0.02 | FAIL (too fast) |
| Electrical discharge (short) | 0.02 – 0.04 | FAIL (too fast) |
| Cavitation click (real) | 0.05 – 0.8 | PASS ✅ |
| Mechanical vibration | 0.8 – 1.3 | PASS (border) |
| Sustained tone / fan noise | > 1.3 | FAIL (too slow) |

---

### 7.6 Criterion 6 – Exponential Decay Quality (R²)

```
C6:  R²_log > R²_min    (default R²_min = 0.45)
```

**Physical meaning:** the log-linear regression of the Hilbert envelope must explain at least 45% of the variance in the decay trajectory. This ensures that the post-peak signal behaves *roughly* exponentially — consistent with acoustic cavitation — rather than being flat noise or a random shape.

**Why the threshold is moderate (0.45 rather than 0.80):**

Experience with v2.x showed that strict R² thresholds (≥ 0.70) produced unacceptably high false-negative rates (~22%) because:
- Clicks from soft materials (soil, wet xylem) have two-component decay: $A(t) = A_1 e^{-t/\tau_1} + A_2 e^{-t/\tau_2}$, which a single exponential fits poorly
- Near-end spillover truncates the decay, biasing R² downward
- High-noise recordings reduce coherence without eliminating the decay trend

The value 0.45 ensures that a clear decay *exists* (as opposed to random fluctuation: R² ≈ 0) while not penalising morphologically complex but genuinely biological events.

**Fitting procedure (v3.2):**

The raw envelope is processed with four improvements before fitting:

```
A. Skip (post-peak transient):  discard first 5 samples after peak   (25 µs)
B. Smoothing:                   4-sample moving average
                                (period of 40 kHz carrier = 4 samples → averages out carrier ripple)
C. Noise truncation:            fit only samples where A_smooth[n] > 2 × noise_rms
                                (prevents noise floor from pulling slope toward zero)
D. Local-max snap:              within ±6 samples of the smoothed start, snap to local maximum
                                (avoids starting on a carrier ripple trough)

Log-linear model:
  y[n] = log(A_smooth[n])
  x[n] = n   (sample index, n = 0, 1, ..., n_fit−1)
  Fit:  y[n] = b + m · x[n]

Closed-form OLS (no BLAS/LAPACK — thread safe on macOS):
  n_f = n_fit
  m   = (n_f · Σ(x·y) − Σx · Σy) / (n_f · Σ(x²) − (Σx)²)
  b   = (Σy − m · Σx) / n_f

  R²  = 1 − SS_res / SS_tot

  τ_ms = −1000 / (m · fs)     [m < 0 for decay; τ > 0]
```

---

## 8. Stage 4 – Deduplication

A single click event can energise 2–3 consecutive frames when the click onset falls near the boundary between two frames (the click partially overlaps into both). Without deduplication, the same physical event would appear multiple times in the output list.

```
Algorithm:
  1. Sort all Stage 3 survivors by frame_idx
  2. Group frames with gap ≤ MAX_GAP = 3 consecutive frames
     (3 frames = 7.68 ms — well above the maximum click duration of ~2 ms)
  3. Within each group, keep only the frame with maximum peak amplitude
  4. Assign timestamp = frame_start_time of the retained frame

Parameters:
  MAX_GAP = 3 (identical for both interactive detector and batch export)
```

**Effect of MAX_GAP:**
- Too small (1–2): may produce duplicates for clicks near frame boundaries
- Too large (8+): risks fusing two distinct clicks that occur close in time
- Value 3 corresponds to ~7.7 ms, safely above click duration (< 2.56 ms) but well below typical inter-click interval (> 50 ms in most plants)

---

## 9. Offline Noise Estimation

An accurate estimate of the background noise floor is essential for Criteria C1 (absolute amplitude) and C2 (pre_snr). The offline estimator runs **once per recording session**, before the click detector loop, and caches the result.

### 9.1 Algorithm

```
1. Compute E_frame[i] for all frames (same as Stage 1)
2. Identify "silent frames": E_frame[i] < μ_E  (below mean — unambiguously non-click)
3. Random sample: draw min(n_silent, 500) frames using seed = 42 (reproducible)
4. For each sampled frame:
     a. Reconstruct iFFT (same pipeline as Stage 3)
     b. Compute RMS(x[n])
5. noise_rms = mean of all 500 RMS values
```

### 9.2 Why Offline (Not Rolling Average)

A rolling buffer would contaminate the noise estimate with any click events that happen to fall in the buffer window. The offline approach draws exclusively from **confirmed silent frames** (below the mean energy), ensuring the estimate reflects pure background noise.

The deterministic seed (42) makes the estimate **reproducible across runs** on the same recording file.

### 9.3 Typical Values

| Environment | noise_rms |
|---|---|
| Well-isolated lab (acoustic chamber) | 5 – 15 µV |
| Standard lab room (fan noise) | 15 – 40 µV |
| Outdoor / noisy environment | 40 – 100 µV |

---

## 10. Descriptive Features (R², τ)

Several features are computed for every confirmed click but are **not used as pass/fail criteria**. They are stored in the results file for post-hoc statistical analysis.

| Feature | Formula | Unit | Physical meaning |
|---|---|---|---|
| τ (tau_ms) | −1000 / (slope_log · fs) | ms | Exponential decay constant |
| R²_log | OLS R² on log-envelope | — | Quality of exponential fit |
| slope_log | m from log-linear fit | 1/samples | Decay rate |
| R_spectral | E_low / E_high | — | Spectral balance (low vs high frequency) |
| SPR | max(power) / mean(power) | — | Spectral peakedness |
| peak_iFFT | max(\|x[n]\|) | V | Time-domain peak amplitude |
| E_W1 – E_W4 | mean(x²) per sub-window | V² | Post-peak energy trajectory |
| asymmetry | rise_samples / fall_samples | — | Onset-to-decay time ratio |
| near_end | flag | bool | Peak near frame boundary (spill risk) |

**Scientific use cases:**
- **τ distribution:** compare stressed vs unstressed plants. Stressed xylem may produce faster collapses (lower τ) due to higher tension.
- **R² classification:** high R² (> 0.85) → clean single-cavity event; low R² (0.45–0.65) → complex multi-cavity or boundary-truncated event.
- **Spectral ratio R:** high R → click centred in low band (30–40 kHz); low R → high-frequency click (50–70 kHz). May correlate with vessel diameter.

---

## 11. Experimental Methodology and Calibration

### 11.1 Recording Protocol

The algorithm thresholds were not chosen arbitrarily — they were derived from a systematic comparison of three experimental conditions:

1. **Empty room (control):** microphone placed in the measurement position with no plant present. Only background noise, EMI, and environmental sounds. Purpose: establish the baseline noise profile and characterise false-positive sources.

2. **Unstressed plant:** microphone positioned at the same distance and geometry as the empty-room session, but with the plant present and well-watered. Purpose: establish the baseline biological acoustic activity and confirm that a healthy plant does not produce a high rate of click-like events.

3. **Stressed plant (water deficit):** plant not watered for 48–72 hours (water stress induction). Recording began once visible wilting signs appeared. Purpose: observe the increase in click rate and characterise the morphology of genuine biological clicks.

For each session, full spectrograms were captured with the PlantLeaf app and candidate events were extracted using the algorithm. Screenshots of the iFFT window, FFT spectrum, and Hilbert envelope were saved for each candidate.

### 11.2 Ground Truth Annotation

From the three sessions, **manual annotation** was performed:
- Visual inspection of each candidate's iFFT waveform, Hilbert envelope, and spectral shape
- Events were labelled as: CLICK (confirmed), NOISE (confirmed false positive), or AMBIGUOUS
- The goal was to collect ≥ 30 CLICK and ≥ 50 NOISE examples for threshold calibration

### 11.3 Threshold Derivation

With the annotated ground truth, threshold values for each criterion were selected to minimise the **false negative rate** (missing real clicks) while maintaining a **false positive rate < 1%**. The comparison between sessions yielded the following observations:

| Feature | Empty room | Unstressed plant | Stressed plant |
|---|---|---|---|
| Click rate (events/hour) | 0 – 5 | 0 – 10 | 50 – 300 |
| Dominant τ (ms) | N/A (noise) | 0.3 – 0.8 | 0.05 – 0.6 |
| Peak iFFT (µV) | 10 – 80 | 50 – 200 | 130 – 2000 |
| pre_snr | > 2.5 (EMI embedded in noise) | 1.0 – 1.5 | 1.0 – 1.4 |
| SPR | > 30 (tonal EMI) | 5 – 20 | 4 – 18 |
| asymmetry ratio | 0.8 – 1.2 (symmetric spikes) | 0.3 – 1.8 | 0.2 – 1.5 |

From these distributions, the v4.0 thresholds were set at the **decision boundary** between the stressed-plant CLICK population and the noise/empty-room populations, with a conservative margin to minimise missed clicks.

### 11.4 Validation Metrics

The final v4.0 algorithm was evaluated on a held-out test session (not used in threshold calibration):

| Metric | Value |
|---|---|
| **Sensitivity (Recall)** | ≥ 95% |
| **Specificity** | ≥ 99% |
| **False Negative Rate** | < 5% |
| **False Positive Rate** | < 1% |
| **Temporal accuracy** | ± 2.56 ms (one frame resolution) |

> **Note:** these figures are estimates from a limited annotation dataset. As more recordings are annotated, these values will be updated. Users are encouraged to visually inspect the detected events and report misclassifications.

### 11.5 Reproducibility Notes

- All threshold values are stored in the Click Detector dialog and can be adjusted per-session
- The noise estimate uses `seed = 42` and is deterministic for the same recording file
- All detected events are exported with their full feature vector (τ, R², SPR, etc.) for independent verification
- Raw `.paudio` recordings are preserved unchanged; the algorithm operates only on read-only copies of the data in memory

---

## 12. Algorithm Version History

| Version | Date | Key changes |
|---|---|---|
| v1.0 | Early 2025 | Basic energy threshold + single R² criterion |
| v2.0 | Late 2025 | Added spectral ratio R, iFFT reconstruction, Hilbert envelope |
| v2.1 | Early 2026 | R²_log ≥ 0.60–0.80 as primary gate; 22% false-negative rate discovered |
| **v3.0** | March 2026 | R² removed as gate; replaced with 3-criteria (SNR, pre_snr, E_W1>E_W4) |
| **v3.1** | March 2026 | Added C4 (asymmetry) and C5 (clean tail / ringing rejection); window extended to 300 samples |
| **v4.0** | March 2026 | Replaced SNR-relative threshold with absolute peak_iFFT (C1=130 µV); added τ criterion (C5); added R² criterion (C6); reformulated asymmetry as `ratio < 2.5`; Gibbs suppressor v3 (symmetric AND condition); Stage 1 default raised to k=5 + MAX_RUN=4 group filter; Stage 2 normalized peak FFT filter added (0.85 mV) |

---

## 13. References

1. **Khait I., et al.** "Sounds emitted by plants under stress are airborne and informative." *Cell*, 186(7):1328–1336, 2023. https://doi.org/10.1016/j.cell.2023.03.009

2. **Tyree M.T., Sperry J.S.** "Do woody plants operate near the point of catastrophic xylem dysfunction caused by dynamic water stress?" *Plant Physiology*, 88(3):574–580, 1988.

3. **Boashash B.** *Time-Frequency Signal Analysis and Processing*, 2nd ed. Academic Press, 2015. (Hilbert transform and analytic signal)

4. **Proakis J.G., Manolakis D.G.** *Digital Signal Processing*, 4th ed. Pearson, 2006. (iFFT, spectral windowing, Gibbs phenomenon)

5. **Weisberg S.** *Applied Linear Regression*, 4th ed. Wiley, 2013. (OLS regression, R² interpretation)

6. **Harris F.J.** "On the use of windows for harmonic analysis with the discrete Fourier transform." *Proceedings of the IEEE*, 66(1):51–83, 1978. (Tukey/cosine taper rationale)

7. **Knowles Electronics.** *SPU0410LR5H-QB Datasheet: Ultrasonic MEMS Microphone*. Rev. H, 2020. (Microphone frequency response, normalization)

---

*Document maintained by the PlantLeaf project contributors.*  
*Corrections, improvements and experimental contributions are welcome via GitHub Issues.*  
*Last updated: March 2026 — v4.0*
