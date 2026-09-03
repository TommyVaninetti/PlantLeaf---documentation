# PlantLeaf – Automatic Ultrasonic Click Detection Algorithm

**Version:** 6.0
**Date:** September 2026
**Authors:** Tommaso Vaninetti
**Repository:** [PlantLeaf-Desktop-App](https://github.com/TommyVaninetti/PlantLeaf-Desktop-App)

---

> *Plants scream when they are stressed. You just need the right algorithm to hear them.*

---

## Table of Contents

1. [Overview and Motivation for v6](#1-overview-and-motivation-for-v6)
2. [v5 → v6 Comparison Summary](#2-v5--v6-comparison-summary)
3. [System Setup and Signal Representation](#3-system-setup-and-signal-representation)
4. [Adaptive Noise Estimator](#4-adaptive-noise-estimator)
5. [Stage 1 – Adaptive Energy Threshold + Local Peak Picking](#5-stage-1--adaptive-energy-threshold--local-peak-picking)
6. [Stage 2 – Gates on Measurable Features](#6-stage-2--gates-on-measurable-features)
7. [Pre-processing Pipeline for Stage 3 Candidates](#7-pre-processing-pipeline-for-stage-3-candidates)
8. [Feature Library](#8-feature-library)
9. [Stage 4 – Deduplication](#9-stage-4--deduplication)
10. [Fit Pipeline: τ and R²](#10-fit-pipeline-τ-and-r)
11. [Feature Summary Table](#11-feature-summary-table)
12. [SVM Classifier — Training Protocol and Results](#12-svm-classifier--training-protocol-and-results)
13. [Known Limitations and Open Questions](#13-known-limitations-and-open-questions)
14. [Algorithm Version History](#14-algorithm-version-history)
15. [References](#15-references)

---

## 1. Overview and Motivation for v6

Version 5 was built to survive **outdoor** noise. Version 6 comes from the opposite
discovery: once a large corpus was labelled exhaustively — every Stage 1 candidate in a
recording judged, not only the ones that looked interesting — it became measurable that
**v5's own filters were deleting real clicks, indoors, in the recordings where the
clicks actually are.**

Three measurements drove the version:

- **The Stage 1 run-length filter:** `MAX_RUN = 3` discarded **45–53 %** of
  above-threshold frames in mechanical-stimulus recordings and **0 %** in the empty
  room. A filter that removes noise should fire on the noise-only sessions too. This one
  fired exactly where the clicks were. Manual review of some files helped confirming the 
  hypothesis.
- **The Stage 2 fit gate:** Measured over 32 exhaustively-labelled recordings, rejecting
  candidates whose exponential decay fit failed cost **12.2 % of confirmed clicks**
  (23 of 189) and **32.3 % of ambiguous rows** — because **8–9 % of confirmed clicks
  have `fit_valid == 0`.** A failed fit is *evidence about* a candidate, not grounds for
  discarding it.
- **New Spectral Features:** Wanting to test new spectral features such as spectral_entropy
and spectral_novelty was initially the reason for creating a new algorithm.

None of this was visible in v5, and the reason is structural rather than an oversight:
**the leak a rejection filter causes cannot be measured from labels that the filter
itself selected.** v5's 285 labelled rows were drawn from the post-filter population, so
the clicks `MAX_RUN` deleted had never reached a CSV where anyone could label them, nor did
the ones discarded because of fit_valid or R2<0.10. 
The question "what fraction of confirmed clicks sit in runs longer than 3?" returns 0 by
construction and estimates nothing. Answering it required first removing the filter,
re-exporting, and re-labelling — which is what v6 is.

The corpus grew accordingly: from 285 labelled rows across 38 sessions to **402 861
exported rows**, of which **32 recordings (6 074 rows, 189 confirmed clicks, 99
ambiguous, 5 786 noise) are exhaustively labelled** — the subset every threshold in this
document is derived from.

> **Principal design, carried forward from v5:** no single experimentally chosen
> threshold. Every parameter is either a physically motivated constant or a feature fed
> to the classifier, and the noise floor and its standard deviation are the two
> quantities most features are normalised by, giving invariance to hardware gain and
> environmental amplitude.
>
> **The v6 corollary that gives it teeth:** *a gate whose click cost is unknown does not
> ship.* Every Stage 2 threshold below is quoted with the measured percentage of
> confirmed clicks, ambiguous rows and noise it removes. A Stage 2 rejection is **hard** —
> no probability, invisible to the classifier, absent from every later analysis — so a
> gate placed where the click distribution is merely thin costs recall irrecoverably.

### 1.1 A note on labels

v6 introduces a third label. Rows are `1` = click, `0` = noise, **`2` = ambiguous**, and
`''` = not yet judged. Ambiguous means *a reviewer looked and could not decide*. It is a
**decision, not a missing value**: it counts as labelling progress, never as a class, and
every consumer handles it explicitly. Left implicit it would reach the classifier as a
third class and silently invalidate every metric. The deployed model was trained with
`--ambiguous exclude`; the trainer also offers `click` and `noise` as a sensitivity
check, on the reasoning that if recall moves materially between the three, the ambiguous
set carries real signal and deserves a weighting scheme rather than a hard assignment.

---

## 2. v5 → v6 Comparison Summary

| Aspect | v5.1 | v6.0 |
|---|---|---|
| Noise floor (scalar) | Buffers 1 & 2, median-of-local-minima | **Unchanged** |
| Noise floor (per bin) | — | **Buffer 3**: per-bin noise PSD, rolling mean over 750 gated frames |
| Stage 1 rule | Threshold, then reject runs longer than `MAX_RUN = 3` | Threshold, then **select local maxima** over ±`R`, `R = 1`. Nothing is rejected for run length |
| Stage 1 candidate volume | baseline | **+32 %** overall (+119 % on one recording, +0 % on the empty room) |
| Stage 2 rule | Fit gate (`R² ≥ 0.10`, `τ > 0`) + `SPR < 100` | Gates on **measurable** features: `peak_SNR`, `n_seg`, `local_crest`, `harmonic_confinement`, plus two out-of-distribution bounds |
| Stage 2 click cost | **12.2 %** of confirmed clicks | **0.0 %**, measured |
| Stage 2 noise cut | 91.4 % | 83.6 % |
| Fit (τ, R²) role | **Gate** | **Feature** — `fit_valid`, `R2`, `tau_ms` go to the classifier |
| NaN convention | NaN must fail explicitly ("the gated quantity is broken") | NaN **passes** every gate ("not applicable") |
| Classifier input | 16 of 17 features | **7 features** |
| Scaler | `StandardScaler` | **`PowerTransformer` (Yeo–Johnson)** |
| Imputer | `SimpleImputer(mean)` | `SimpleImputer(**median**)` |
| Class weighting | none, hard-negatives only | **`balanced`**, chosen by cross-validation |
| Sentinel encoding | `tau_ms = −1`, `R2 = 0` fed to the model | **converted to NaN**, keyed on `fit_valid`, and imputed inside the Pipeline |
| Hard-negative mining | fit gate applied to label-0 rows before training | **off** — the training CSV *is* the Stage 2 survivor set |
| Grid scoring metric | `recall` | **`roc_auc`** |
| Decision threshold | 0.220 | **0.121** |
| Training corpus | 285 rows, 91 clicks, 38 sessions | **1 136 rows, 189 clicks, 30 sessions** |
| Cross-validated AUC-ROC | 0.835 | **0.929** |
| Held-out (Set B) AUC-ROC | 0.925 | **0.958** |
| Stage 4 dedup key | `peak_abs` (v5.1) | Unchanged |
| CSV schema | v5-era columns, no schema stamp | **57 columns**, `SCHEMA_VERSION = 'v6'` |
| Labels | `1` / `0` / `''` | `1` / `0` / **`2` ambiguous** / `''` |

> ⚠️ **The v5 row of the AUC comparison is not a like-for-like re-measurement.** 0.835 is
> the figure v5 reported on its own 285-row corpus. Re-running v5's full configuration on
> the v6 corpus gives **0.781** (§12.8). Both numbers are real; they measure different
> things, and §12.8 is the one to quote when comparing the two algorithms.

---

## 3. System Setup and Signal Representation

Unchanged from v5. Key parameters, verified against the constants block of
`src/core/click_pipeline_v5.py`:

| Parameter | Value |
|---|---|
| Microphone | Knowles SPU0410LR5H-QB |
| Sampling rate | `fs = 200,000 sps` |
| FFT size | `N = 512 points` |
| Frame duration | `T = 2.56 ms` |
| Frame rate | 390.625 FPS |
| Analysis band | 20–80 kHz (bins 51–204, 154 bins) |
| Bin width | `Δf = 390.625 Hz` |
| Phase encoding | int8, range [−127, +127] → [−π, +π] |
| Microphone normalisation | 50 % conservative, piecewise-linear from the datasheet response |

---

## 4. Adaptive Noise Estimator

### 4.1 Motivation and structure

The v5 estimator is unchanged in v6 and is reproduced here in full, because everything
downstream is normalised by it. It maintains **two parallel sliding-window
minimum-statistics estimators** sharing one burst-protection gate and one window length —
one in the FFT energy domain (for Stage 1), one in the iFFT/Hilbert envelope domain (for
the time-domain features). v6 adds a **third buffer**, per frequency bin (§4.9).

> **Why separate estimators per domain?** Stage 1 operates on FFT frame energy (a scalar
> in V²/bin). The time-domain features need `noise_floor` and `std_noise` in physical
> voltage units of the Hilbert envelope. These are related but not trivially
> convertible — the iFFT reconstruction, the Tukey taper and the Hilbert transform each
> affect the amplitude distribution in ways that depend on the signal chain. Estimating
> in each domain directly avoids any calibration coefficient and survives hardware
> changes.

### 4.2 Shared definitions

**Frame energy (FFT domain):**

```
E_i = (1/K) · Σ_{k=0}^{K-1} |A_i[k]|²

K = 154  (transmitted bins, 20–80 kHz)
Units: V²  (mic-normalised FFT magnitudes)
```

**Per-frame envelope statistics (iFFT domain),** computed for every frame including
silent ones:

```
env_mean_i = mean(A[n])    over 512 samples
env_std_i  = std(A[n])     over 512 samples
Units: V
```

### 4.3 Window length

```
W = 750 frames  (≈ 1.92 s at 390.625 FPS)
```

Long enough that a 30–50 frame burst adds under 7 % of the entries and barely shifts the
minimum; short enough to track the 1–5 s environmental transitions (wind gusts, passing
vehicles) the estimator exists for.

### 4.4 Burst protection (shared gate)

```
if E_i > α · Ê_floor(i-1):
    energetic frame  →  update NO buffer
else:
    silent frame     →  update all buffers
```

`α = 4.0`. The same gate for every buffer guarantees that a click candidate can never
contaminate one noise estimate while being excluded from another.

> ⚠️ The comment on `ALPHA` still reads *"verify experimentally on outdoor recordings"*,
> and that verification is only partly done — see §13.2.

### 4.5 Buffer 1 — FFT domain (Stage 1)

```
CIRCULAR BUFFER B1[], length W = 750, updated only on non-burst frames
```

**Why not a pure minimum?** A single anomalously low frame — a microphone dropout, an ADC
glitch, a momentary silence between two noise events — pulls `min(B1)` far below the true
floor, the Stage 1 threshold drops with it, and a burst of false positives follows. Stahl
et al. (2000) proposed the q-th quantile of the buffer as a more robust alternative;
Rangachari & Loizou (2006) then showed experimentally that a fixed global percentile is
fragile when the noise distribution changes, which outdoors it does.

**Adopted: median of local minima.** Divide `B1[]` into `M = 10` non-overlapping
sub-windows of 75 frames (≈ 192 ms) each:

```
m_j = min_{i ∈ S_j} B1[i]              j = 1 … 10

Ê_floor(i) = β · median(m_1, …, m_10)   β = 1.3
```

A single anomalous frame can affect at most one of the ten sub-window minima, and shifting
the median of ten values requires five of them to be contaminated — implausible for
isolated glitches. `β = 1.3` corrects the residual bias of a finite-buffer minimum
(Martin 2001); it is slightly below the usual 1.5 because the estimator takes a median of
minima rather than a single minimum.

### 4.6 Buffer 2 — Hilbert envelope domain

Two circular buffers of length `W`. `noise_floor` uses the identical median-of-local-minima
construction on `B2_mean[]`. `std_noise` does **not**:

```
std_noise(i) = mean( B2_std[] )
```

`noise_floor` wants the lowest recent background level, so it is minimum-based.
`std_noise` represents the *typical variability* of the noise amplitude — a minimum-based
estimate would return the most stationary frame's standard deviation and systematically
understate it.

### 4.7 Initialisation

For the first `W/2 = 375` frames all frames are accepted into the buffers with no burst
protection, because burst protection needs an `Ê_floor` estimate that does not yet exist.
From frame 375 the gate applies normally. Until the buffer is full, statistics are
computed over the available entries only.

### 4.8 Constants summary

| Constant | Value | Basis |
|---|---|---|
| `W_NOISE` | 750 frames (≈ 1.92 s) | Tracks floor changes; stable against bursts |
| `M_SUBWINDOWS` | 10 sub-windows of 75 frames | Median-of-minima robustness |
| `BETA` | 1.3 | Martin (2001) bias correction, adjusted for the median |
| `ALPHA` | 4.0 | Burst exclusion; partly unverified (§13.2) |
| `WARM_UP_FRAMES` | 375 | `W/2` |
| `K_STAGE1_DEFAULT` | 1.5 | Stage 1 threshold multiplier |

---

### 4.9 Buffer 3 — per-bin noise PSD (NEW in v6)

Every v6 spectral feature is defined on an **excess spectrum**,
`E[k] = max(0, P_frame[k] − P_noise[k])`, which requires a noise estimate **per frequency
bin** rather than the two scalars Buffers 1 and 2 provide. Buffer 3 supplies it, over the
same 154 transmitted bins and behind the same burst gate.

It stores **PSD in V²/Hz**, not raw magnitude. Under PSD scaling a stationary noise
estimate is invariant to segment length, so it can be subtracted from a region spectrum
of any duration with no correction factor; under amplitude scaling it falls as
`N^(−1/2)` and the correction would have to be recomputed per event and be exactly right.

#### 4.9.1 The estimator is a rolling mean, and the specification was wrong

The v6 proposal specified reusing Buffer 1's machinery per bin, with the *same* β = 1.3
and the same Martin (2001) justification. **That does not transfer, and it is wrong by a
factor of ~82.** Measured on synthetic noise with the correct statistics — a DFT bin of
Gaussian noise is complex Gaussian, so `|X[k]|` is Rayleigh and `|X[k]|²` is exponential,
i.e. χ²₂:

| estimator | estimate / true mean |
|---|---:|
| B1, scalar `E_i`, β = 1.3 | **1.067** ✅ |
| B3, per bin, β = 1.3 | **0.0122** ❌ (~82× low) |
| B3, per bin, rolling mean | **0.999** ✅ |

**Martin's β is a function of the effective degrees of freedom of the estimator's input,
not a universal constant.** Buffer 1's input `E_i` is already an average over 154 bins —
a high-DOF, low-variance quantity whose sub-window minimum sits just below its mean, which
is exactly the bias β ≈ 1.3 corrects. Buffer 3's input is a **raw periodogram bin**: χ²₂,
two degrees of freedom, where `E[min of W = 75] = μ/75`. That is not a bias to correct,
it is a different quantity; β would have to be ≈ 75, at which point it *is* the estimator
rather than a correction to one.

The adopted estimator is therefore

```
P_noise[k] = mean over the last W_NOISE = 750 burst-gated frames
```

implemented as a ring buffer plus running sum. It is also what the proposal's own opening
prose asked for — it correctly diagnoses that a single periodogram is "useless bin-by-bin"
and "needs averaging over many silent frames", then adopts machinery that takes minima.
And it **removes** a tunable constant instead of adding one. The minimum-statistics path
(`mode='min'`) is retained for comparison only, never as the default.

> **Narrowband interferers and the burst gate.** The gate is broadband — it tests total
> band energy against `α·Ê_floor` — so a *persistent* narrowband tone (a 40 kHz ranging
> sensor, a pest repeller) passes it and enters the mean. **That is correct, not a leak:**
> a stationary tone *is* part of the noise a region should be judged against, and
> subtracting it is exactly what `shape_novelty` needs in order not to flag it as novel.
> Only *intermittent* narrowband events would be a concern, and over a 750-frame window a
> single such event contributes at most 1/750 of the estimate.

#### 4.9.2 A structural difference from Buffer 1, stated rather than assumed

Buffer 1 stores all 750 raw values and recomputes all ten sub-window minima by slicing on
every frame, with sub-windows pinned to *array positions* — so once the circular buffer
wraps, the chronological seam falls inside one sub-window. Buffer 3's minimum-statistics
path is the textbook Martin (2001) form instead: a running minimum over the current
sub-window, rotated into a ring of ten slots every 75 accepted frames, so its ten minima
are the ten most recent *chronological* sub-windows. Buffer 3 uses the specified structure
because that is what the 154 × 11 memory budget describes and what a firmware port would
have to do — mirroring Buffer 1 would require 154 × 750 floats, 924 kB rather than 6.8 kB.
The divergence is quantified in the test suite rather than assumed away.

### 4.10 Stale-floor detection (observation only)

`STALE_FLOOR_FRAMES` is **not a new tunable — it is `SUBWINDOW_SIZE`**, the estimator's
own 75-frame granularity, which is ~12× the longest consecutive-burst run measured on real
stationary recordings (6 frames) and therefore cannot fire on ordinary data. When that many
consecutive frames are burst-gated, the floor is reported as possibly frozen. It changes no
behaviour.

The property it observes is inherited from v5 and is worth stating plainly: **a gated
frame never enters any buffer**, so if ambient noise steps up by more than `α = 4`, every
frame is classified as a burst, nothing is admitted, and the floor stays pinned at the old
value indefinitely. Buffers 1 and 2 have always behaved this way; Buffer 3 inherits it
through the shared gate.

---

## 5. Stage 1 – Adaptive Energy Threshold + Local Peak Picking

### 5.1 Threshold

```
Frame i is above threshold  if  E_i > k · Ê_floor(i)        k = 1.5
```

Unchanged from v5. The adaptive floor means the threshold rises in noisy environments and
falls in quiet ones with no per-session recalibration.

### 5.2 What replaced the run-length filter

v5 grouped consecutive above-threshold frames into runs and **discarded the entire run**
when it exceeded `MAX_RUN = 3` frames. v6 rejects nothing for run length. It selects
**local maxima of the frame-energy series**:

```
E_i >  E_j    for j ∈ [i−R, i−1]      strict left
E_i >= E_j    for j ∈ [i+1, i+R]      non-strict right

R = PEAK_REFRACTORY_R = 1  frame  (±2.56 ms)
```

Two details matter:

- The test is evaluated against the **full** energy series, not only the flagged frames. A
  sub-threshold neighbour is by definition below a flagged frame, so including it costs
  nothing and makes the peak test **independent of `k`**.
- The **asymmetry** (`>` left, `>=` right) resolves plateaus deterministically to their
  *first* frame, which guarantees exactly one candidate per plateau. With `>=` on both
  sides every frame of a plateau qualifies; with `>` on both sides none does; either way
  the rule stops partitioning the run.

### 5.3 Why the run-length rule had to go

The physical argument behind `MAX_RUN` — *a cavitation click cannot span four frames* —
is a statement about the **click**. The filter applied it to the **run**, which is a
property of the acoustic neighbourhood the click happened to land in. A click is not made
less impulsive by something else being audible 30 ms away.

Measured over the corpus, the share of above-threshold frames `MAX_RUN` discarded:

| recording kind | frames dropped by `MAX_RUN` |
|---|---|
| mechanical stimulus (where the clicks are) | **45 – 53 %** |
| water | 19 % |
| noise-only (empty room) | **0 %** |

A filter that removes noise would fire on the noise-only sessions too. This one fires
where the clicks are and nowhere else.

Run lengths are heavily tailed — 4, 5, 8, 13, 27, 41, 55 frames. A 55-frame run is 141 ms
of continuous above-threshold energy and is certainly not a click, so the long tail *is*
sustained noise and `MAX_RUN` was right to reject it. The defect is that it rejected it by
**deleting the whole run**, taking any genuine peak inside it along. v6 keeps the peaks of
a 55-frame run without keeping its 55 frames.

### 5.4 The measurement that set `R = 1`

The specification proposed `R = 2` and named the failure mode to watch for: peak-picking
can suppress a confirmed click that is not a local maximum within ±R because a louder
frame sits beside it. It prescribed dropping to `R = 1` if that happened. It happened.

**AC-1 — all 91 confirmed clicks then available, over 19 recordings:**

| | frame-level | **event-level** |
|---|---|---|
| `R = 2` | 84 / 91 | **87 / 91 — FAILS** |
| `R = 1` | 88 / 91 | **91 / 91 — PASSES** |

**Frame-level is the wrong test.** The two columns differ because *a confirmed click's
labelled `frame_idx` is not always the frame that owns its peak.* v5 exported every frame
of an accepted run, so a click straddling two frames produced two rows and both could be
labelled `1`; v6 emits only the peak. Three of the 91 are "lost" at frame level purely
because the label sits on the sibling frame — nothing is lost, the event is still exported
at the frame that actually owns the peak. This is also exactly why the schema keys label
migration on **`peak_abs`** rather than on `(file, frame_idx)`.

**All four genuine losses at `R = 2` share one mechanism, and it is not the one `MAX_RUN`
had.** Every one was an isolated `L = 1` run with a louder, unrelated event exactly two
frames (5.12 ms) away. The ±R window **reaches across run boundaries** — consecutive runs
are separated by at least one sub-threshold frame — so at `R = 2` a short run sitting one
or two frames from a taller neighbour is suppressed in its entirety, merging two distinct
events into one.

**`R = 1` is not merely luckier; it cannot empty a run, and this is provable.** Take the
first maximum `m` of any run `[a..b]`:

- *Left.* If `m == a`, then `E[a−1]` is sub-threshold and `E[a]` is not, so `E[m] > E[m−1]`.
  If `m > a`, then `E[m] > E[m−1]` because `m` is the *first* maximum.
- *Right.* `E[m] >= E[m+1]` holds because `m` is a maximum — or because `E[b+1]` is
  sub-threshold when `m == b`.

So every run yields at least one candidate, and `R = 1` cannot reach past the single gap
frame separating two runs, which is precisely why it resolves them as distinct events. The
test suite asserts both halves: that `R = 2` empties runs (2.4 % of them on a synthetic
corpus) and that `R = 1` empties none.

`R = 1` remains **provisional upward**: nothing here shows it is optimal, only that
`R = 2` costs confirmed clicks.

### 5.5 `local_crest`

```
local_crest(i) = E_i / median( E_j : j ∈ [i−C, i+C] \ {i−1, i, i+1} )

C = LOCAL_CREST_C = 10 frames  (±25.6 ms)
```

How far a frame stands above its own local background — the physics `MAX_RUN` was reaching
for, expressed without its pathologies. A click is a spike against a comparatively flat
background; sustained mechanical noise is flat and gives ≈ 1.

The three central frames are excluded from the background median so the event itself, and
any frame-boundary straddle of it, cannot inflate its own reference level.

It is deliberately **independent of `k` and of `Ê_floor`**. A run-based crest would inherit
both the threshold dependence (run length is a function of `k`) and the buffer-history
dependence (the burst gate makes runs grow or self-terminate) that make run length unusable
as a feature.

It returns **NaN**, not `−1`, when the background median is zero or non-finite. The
specification asked for a `−1` sentinel; a magic in-band number silently becomes data and
an imputer cannot tell it from a measurement, so the spec's *intent* — do not silently
substitute a value — is preserved by using NaN instead. A separate `local_crest_valid`
column is consequently redundant and was never added: it is `not isnan(local_crest)`.

`C = 10` is **provisional**. It is long enough to sample background either side of a 1–2
frame event and short enough that the adaptive floor has not meaningfully drifted, but it
has not been validated against labels (§13.4).

### 5.6 `k_ratio` and the retrospective threshold sweep

`k_ratio = E_i / Ê_floor(i)` is exported on every row, because `E_i > k·Ê_floor` is
*exactly* `k_ratio > k`. That turns "what happens if I raise the Stage 1 threshold?" into a
CSV filter instead of a full corpus re-pass:

| `k` | clicks kept | ambiguous kept | noise removed |
|---:|---:|---:|---:|
| **1.50** (current) | **100 %** | 100 % | 0 % |
| 1.60 | 95.2 % | 96.0 % | 33.3 % |
| 1.75 | 90.5 % | 85.9 % | 53.6 % |
| 2.00 | 85.2 % | 77.8 % | 66.1 % |
| 3.00 | 67.2 % | 51.5 % | 81.6 % |
| 3.46 | 62.4 % | 46.5 % | 84.9 % |

**Stage 2 strictly dominates raising `k`.** At `k = 3.0` the threshold removes 81.6 % of
noise and loses 32.8 % of clicks; the Stage 2 gates of §6 remove 83.6 % and lose none.

> **One caveat, stated precisely.** Filtering the CSV returns a **subset** of what Stage 1
> would emit at `k′`, not an exact match: raising `k` un-flags frames, which can expose a
> new peak previously suppressed by a neighbour that no longer qualifies. The sweep is a
> valid **lower bound** on surviving candidates, and that is what the test asserts.

`E_i` itself is not exported but is recoverable as `k_ratio × E_hat_floor`.

---

## 6. Stage 2 – Gates on Measurable Features

Stage 2 is a set of cheap hard rejections applied before the classifier is invoked. In v6
it was rebuilt from scratch.

**A Stage 2 rejection is hard.** The candidate gets no probability, is invisible to the
SVM, and is absent from every later analysis except as a `stage_blocked` string. That is
why every threshold below is justified by a *measured click cost* rather than by an
accuracy figure, and why the standing rule is that **a gate whose click cost is unknown
does not ship.**

All measurements come from the **exhaustively-labelled subset** — 32 recordings, 189
clicks, 99 ambiguous, 5 786 noise. A partly-labelled recording's "noise" is only the rows
someone chose to label, so any *% of noise removed* computed from it is inflated. The two
populations agree closely here (81.6 % vs 81.4 % for the same gate), which is itself the
evidence that the selection bias is small — but the exhaustive number is the one quoted.

### 6.1 What was removed, and what it cost

v5's Stage 2 rejected any candidate whose exponential decay fit had failed:
`fit_valid == 0`, NaN `R2`/`tau_ms`, `R2 < 0.10`, or `tau_ms <= 0`.

| Stage 2 rule | clicks lost | ambiguous lost | noise cut |
|---|---:|---:|---:|
| **old fit gate** | **12.2 %** (23 of 189) | **32.3 %** | 91.4 % |
| **new v6 gates** | **0.0 %** | **0.0 %** | **83.6 %** |

23 confirmed clicks were hard-rejected before Stage 3 and never reached the SVM, because
**8–9 % of confirmed clicks have `fit_valid == 0`.** Those rows had never reached a CSV in
the project's history — Stage 2 dropped them before export — which is why an
`EXPORT_UNFILTERED` mode had to be added before they could even be labelled. Once they
were, clicks turned out to be among them.

> **A failed decay fit is evidence about a candidate, not grounds for discarding it.**
> `fit_valid`, `R2` and `tau_ms` are therefore **features** in v6, and Stage 3 decides
> what they are worth (§10, §12.7).

### 6.2 The four discriminating gates

Each threshold sits strictly **outside** the labelled click distribution, and each is
quoted with the fraction of clicks / ambiguous / noise it removes:

| gate | extreme labelled click | clicks | ambiguous | noise |
|---|---|---:|---:|---:|
| `peak_SNR >= 4.5` | lowest click **4.640** | 0.0 % | 0.0 % | 76.8 % |
| `n_seg >= 10` | lowest click **10** | 0.0 % | 0.0 % | 45.2 % |
| `local_crest >= 1.2` | lowest click **1.265** | 0.0 % | 0.0 % | 33.6 % |
| `harmonic_confinement <= 1.6` | highest click **1.545** | 0.0 % | 0.0 % | 3.2 % indoor / 16.9 % corpus |
| **all four combined** | | **0.0 %** | **0.0 %** | **83.1 %** |

- **`peak_SNR ≥ 4.5`** — amplitude relative to the noise floor. 5.0 would reach 82.5 % of
  noise but costs 2.1 % of clicks; that is the aggressive tier (§6.5), not this one.
- **`n_seg ≥ 10`** — the onset→`decay_end` region length in samples. 12 already costs
  0.5 % of clicks.
- **`local_crest ≥ 1.2`** — prominence above the frame's own ±C background (§5.7).
- **`harmonic_confinement ≤ 1.6`** — energy confined to a 40/80 kHz harmonic pair (§8.11).
  The feature is bounded above by ≈ 3.36 by construction.

### 6.3 Two out-of-distribution bounds that are not discriminators

```
SPR      <  100      → Stage2_SPR
peak_SNR <= 1e4      → Stage2_nonphys
```

These are **not tuned against the click distribution at all.** They reject values that
cannot be measurements.

**The reconstruction outliers.** `peak_SNR` reaches **1.14 × 10⁴⁰** on real exported rows,
`k_ratio` 4.1 × 10⁸¹, `local_crest` 3.8 × 10⁸¹. A single one of these destroys a scaler.
There are **37 such rows in 402 861 (0.009 %)**, and their fingerprint identifies the
cause:

| | |
|---|---|
| `SPR == 154.0` **exactly** | **28 of 37** (`> 150` on 35) |
| `n_seg == 1536` (= 3 × 512, the whole stitched context) | **32 of 37** |
| `fit_valid == 1` | **all of them** — they fit fine |
| labelled clicks among them | **0** (17 noise, 20 unlabelled) |
| recordings involved | **26** — not one corrupt file |

`SPR = max/mean` over 154 bins, so `SPR = 154` *exactly* is its mathematical ceiling,
reachable only when **one bin holds all the power**. So: a frame whose transmitted
spectrum is essentially a single non-zero bin → its iFFT is a pure tone → it never decays
→ the decay window runs to the end of the whole stitched context → `peak_amp` is whatever
that bin held. This is an **upstream data-integrity problem, not a detection one**, and it
is still worth chasing at source; Stage 2 catching the symptom is not the same as
understanding it.

`SPR < 100` rejects all 37 — but it is the **only** thing that does, and only because this
particular pathology happens to be spectrally degenerate. **Every other v6 gate is a lower
bound, so a broadband blowup would pass all of them.** `peak_SNR <= 1e4` is the
independent net: **12.7× the highest labelled click (788.9)**, 0 clicks and 0 ambiguous
lost, firing on 35 corpus rows.

It is deliberately **not** tightened to ~10³ — that is only 1.3× the highest observed
click, and 208 positives cannot justify that ceiling. Three rows do survive with
`peak_SNR` 853–960 (`SPR` 3.7–7.5, `n_seg` 584–769, `fit_valid` 1); these look like
genuine long high-amplitude events rather than corruption — two are adjacent frames of one
event, both labelled noise — and they are nowhere near scaler-breaking.

### 6.4 The two combined figures, and why they differ

Both appear in this document and in the source, and they measure different things:

| scope | noise cut |
|---|---:|
| The four discriminating gates of §6.2 | **83.1 %** |
| The full `v6_conservative` rule, including the two OOD bounds of §6.3 | **83.6 %** |

Both are 0.0 % clicks and 0.0 % ambiguous.

### 6.5 Three selectable modes — v5 was kept, not deleted

| `stage2_mode` | clicks lost | ambiguous | noise cut | |
|---|---:|---:|---:|---|
| `v6_conservative` | **0.0 %** | 0.0 % | 83.6 % | **default** |
| `v6_aggressive` | 2.1 % | 2.0 % | 87.0 % | opt-in |
| `v5_fitgate` | 12.2 % | 32.3 % | 91.4 % | the original rule, preserved verbatim |

`v6_aggressive` raises the `peak_SNR` floor to 5.0. It exists for recordings whose
candidate rate makes human review impossible — measured up to **344 773 candidates/hour**
outdoors, 24 % of all frames. Its cost is real, is stated here, and is stamped into every
exported row so it can never be enabled invisibly.

`v5_fitgate` is kept so a v5 result stays reproducible without checking out an old commit —
for re-deriving a figure from a v5-era CSV, or asking what the old pipeline would have said
about a recording. It is preserved **bit-identically**, including its NaN handling, which
is the exact inverse of v6's (§6.6). It also never looked at `peak_SNR`. **The two paths
must not be merged.**

The mode is chosen in the export dialog and written into every row's `stage2_mode` column,
so an export is always attributable to the rule that produced it. An unknown mode raises
rather than silently defaulting.

### 6.6 NaN passes every gate

```python
def _lt(key, threshold):
    """True when the value is BELOW threshold. NaN and missing -> False (pass)."""
    ...
    return False if f != f else f < threshold      # f != f is the NaN test
```

**A row whose feature could not be measured cannot be judged by it.**
`harmonic_confinement` is NaN on ~23 % of rows *by design* (the second harmonic falls
outside the transmitted band — §8.11), and every v6 feature is NaN when `b3_frames == 0`.

This is the **opposite** of the retired fit gate's convention, and both are correct for
their own rule:

| | NaN means | NaN must |
|---|---|---|
| v5 fit gate | the quantity being gated on is **broken** | **fail** explicitly |
| v6 gates | the quantity is **not applicable** | **pass** |

The v5 side needs its explicit test because every comparison against NaN is `False` — so
without it, unfittable candidates would sail through into Stage 3 and be scored from
imputed values, which is precisely what the gate existed to prevent. Merging the two
conventions would break one of them.

### 6.7 Two features that behaved differently from expectation

- **`SPR` does not separate clicks from noise.** Medians **8.92 (click)** vs **9.22
  (noise)**, and it rejects only 0.4 % of candidates. It is not a discriminator and must
  not be treated as one — it was dropped from the classifier's feature set for exactly
  this reason (single-feature AUC 0.454). **But it must not be removed from Stage 2:** it
  is what catches 100 % of the single-bin iFFT blowups (§6.3), which is real work that the
  click-versus-noise framing completely hides.
- **`shape_novelty` is weak as a hard gate.** Click median 0.293 vs noise 0.159, ranges
  fully overlapping, 27–30 % NaN on noise. Potentially useful to a classifier; poor as a
  gate, and it is not used as one.

### 6.8 Verdict labels

Every candidate carries a `stage_blocked` string. The strings are part of the CSV
contract — the in-app export and the offline CLI must agree on the same recording.

| verdict | meaning |
|---|---|
| `''` | survived all four stages — a confirmed click |
| `Stage2_SNR` | `peak_SNR` below the lowest labelled click |
| `Stage2_nonphys` | `peak_SNR` non-physically large — a reconstruction artefact |
| `Stage2_nseg` | region too short to be an event |
| `Stage2_crest` | no local prominence over its own background |
| `Stage2_harm` | energy confined to a 40/80 kHz harmonic pair |
| `Stage2_SPR` | out-of-distribution tonality |
| `Stage2_R2` | **retired as a gate in v6.** Nothing emits it any more; the constant is kept because it appears in every pre-v6 CSV and the review dialog filters on it |
| `Stage3_SVM` | scored below the decision threshold |
| `Stage4_dedup` | duplicate of a higher-confidence detection |

---

## 7. Pre-processing Pipeline for Stage 3 Candidates

Unchanged from v5.1. Every frame that survives Stage 1 undergoes this reconstruction
before feature extraction.

```
FFT frame (154 bins, magnitudes + int8 phases)
    │
    ▼
[STEP 1] Full spectrum reconstruction
         Zero-pad to 256 bins; apply Tukey taper to the COMPLEX spectrum
         (not magnitudes only)
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
         A[n] = |IRFFT(RFFT(x) · H)|
    │
    ▼
[STEP 5] Peak detection
         peak_idx = argmax(A[n])
```

Whether Step 3 fired is exported per row as `gibbs_fired`, because a fired suppression
means a biased spectral subtraction for that frame.

### 7.1 Stitched click context (frame-grid independence)

A cavitation click lasts ≤ ~0.5 ms but can land near the end of a frame. Computing
features on a single frame therefore **silently truncates** every time-domain feature of a
click that straddles a frame boundary — a decay window index can exceed 511 while the
array is only 512 long, and NumPy slicing returns a short array rather than raising.

Every candidate is therefore resolved and measured on a **stitched context**:

```
build_click_context(prev_sig, curr_sig, next_sig):
    signal   = [ prev | curr | next ]        (up to 1536 samples)
    envelope = Hilbert(signal)               (continuous across the joins)
    origin   = index of curr's first sample
    seams    = sample indices of the frame joins

resolve_click(ctx, noise_floor, std_noise):
    frame_peak = argmax(envelope over the current frame)      # the Stage 1 excursion
    onset      = last sample below LEVEL, scanning back from frame_peak
    peak       = argmax(envelope over [onset : onset + 512])  # the TRUE peak
    decay_start, decay_end = decay window from `peak` on the stitched envelope
```

The envelope is the Hilbert transform of the *stitched* signal, not per-frame envelopes
concatenated, so it is continuous. Each frame keeps its own Tukey taper, so the joins carry
a mild seam whose positions are tracked and drawn honestly rather than hidden.

**Event identity.** The peak's absolute sample and its owning frame define a
model-independent identity:

```
peak_abs            = frame_idx · 512 + (peak − origin)
canonical_frame_idx = peak_abs // 512
```

The two Stage 1 candidates a straddling click can produce scan back to the **same onset**
and re-maximise to the **same peak**, so they carry an integer-identical `peak_abs`. This
is what makes them collapse exactly in Stage 4, and it is the key label migration is
performed on — `(file, frame_idx)` is not unique across schema versions, `peak_abs` is.

---

## 8. Feature Library

**The distinction the rest of this document depends on: *computed* ≠ *exported* ≠ *used by
the model*.** The pipeline computes about 26 feature-shaped quantities, writes **57 CSV
columns**, and the deployed classifier reads **7 of them**. `model['features']` is
authoritative at inference; schema membership is not model membership. A feature can be
exported for years without ever being used — and several are, deliberately, so that a
future retrain can evaluate them without a full corpus re-export.

### 8.1 – 8.9 The v5 feature set

Unchanged in definition from v5.1. Summarised here; the full derivations are in the v5
specification.

| feature | definition | domain |
|---|---|---|
| `peak_SNR` | `peak_amp / noise_floor` | envelope |
| `pre_SNR` | `RMS(pre_window) / noise_floor`, `P = 100` samples ending at the pre-boundary | envelope |
| `post_SNR` | `RMS(post_window) / noise_floor`, `P = 100` samples from `decay_end` | envelope |
| `rise_time_ms` | peak minus the last sample below `LEVEL` before it | envelope |
| `fall_time_ms` | first sample below `LEVEL` after the peak, minus the peak | envelope |
| `asymmetry_integral` | `Σ(right − reversed left) / (W · peak_amp)` over `W = peak → decay_end` | envelope |
| `ZCR_pre`, `ZCR_click`, `ZCR_post` | hysteresis-band crossings per ms, band `±std_noise` | raw iFFT |
| `kurtosis` | excess kurtosis over `[event_start : decay_end]` | raw iFFT |
| `centroid_shift_hz` | `SC_early − SC_late` over the first/last third of the decay | FFT |
| `SPR` | `max|A|² / mean|A|²` over the analysis band | frame FFT |
| `R_spectral` | `E_low(20–40 kHz) / E_high(40–80 kHz)` | frame FFT |
| `FPE_hz` | frequency of the maximum spectral power | frame FFT |
| `tau_ms`, `R2`, `fit_coverage` | from the decay fit (§10) | envelope fit |

with

```
LEVEL = noise_floor + std_noise
```

which defines "the signal has emerged from the noise floor" in a way invariant to hardware
gain and environment.

> **Kurtosis — a correction carried over from v5.1, repeated because it is still wrong in
> older material.** Measured on the labelled set, excess kurtosis medians are **−0.58**
> (noise) and **+0.45** (click); the maximum observed anywhere is 14.2. Earlier revisions
> quoted "genuine cavitation click 15–50 / EMI spike > 100". Those figures are inconsistent
> with the window actually used: excess kurtosis measures how much energy sits in *rare*
> extreme samples relative to the rest of the window, and a window cropped tightly to the
> event — which `[event_start : decay_end]` is, deliberately — is uniformly energetic and
> therefore scores near zero. Values in the tens require a window that is mostly baseline
> with an isolated spike. The formula and the implementation were always correct; only the
> reference table was wrong.

### 8.10 The v6 excess-spectrum family

All of these are defined on the **excess spectrum** of the onset→`decay_end` region:

```
E[m] = max(0, P_region[m] − P_noise[m])       on a fixed 12-band grid,
                                              5 kHz per band over 20–80 kHz
```

The band count `M = 12` is **fixed rather than variable**, so that events of different
duration stay comparable and the features do not re-encode region length — which
`fall_time_ms` already carries.

| feature | definition | rationale |
|---|---|---|
| `spectral_entropy` | `H = −Σ p log₂ p / log₂ M`, `p(m) = E[m]/ΣE` | spectral **spread**. `log₂` makes `N_eff = 2^(H·log₂M)`, the effective number of occupied bands, directly readable |
| `shape_novelty` | `1 − cos(P̂_region, P̂_noise)` | spectral **difference from ambient**. Takes `P_region`, **not** `E` — the L2 normalisation divides amplitude out entirely, so it sees shape only |
| `spectral_tilt` | `10·log₁₀(P̃_high/P̃_low) / (f_high − f_low)` dB/kHz, medians of the 20–50 and 50–80 kHz half-bands | spectral **slope** |
| `temporal_concentration` | `σ_t / T_region`, RMS duration of `A²[n]` about its centroid | energy **shape within** the region, not its length |
| `FPE_hz_region` | argmax of the excess spectrum on the padded native grid | dominant frequency of the *event*, not of the frame |
| `SPR_region` | `max/mean` of the region spectrum | the region counterpart of the frame `SPR` |
| `f_50_hz`, `IQR_f` | frequency quantiles of the cumulative excess energy, linearly interpolated | held back from the classifier — `f_50` is the median of the distribution `FPE` takes the mode of, and `IQR_f` collides with entropy |

Three design points are worth stating because each was nearly got wrong:

**`shape_novelty` is aimed at the hardest negatives specifically.** The candidates that
survive Stages 1 and 2 are, overwhelmingly, *amplitude excursions of the ambient noise*. A
louder burst of the same noise has the same spectral shape → novelty ≈ 0. A genuine click
has a different shape → novelty large. Every amplitude-based feature (`peak_SNR`,
`pre_SNR`, `post_SNR`, `kurtosis`) is blind to this by construction; this is the orthogonal
test.

**`spectral_entropy` must be computed on `E`, not on the raw spectrum.** The standard
definition uses `|X|²`, which is fine for acoustic-emission sensors running at high SNR
behind hardware thresholds — their raw spectrum *is* the event. At PlantLeaf SNR it is not:
raw-spectrum entropy gives `H → 1` for both a low-SNR click and a pure noise region, i.e.
it measures "how much noise is in this region", a badly conditioned proxy for `peak_SNR`.
The subtraction is not an embellishment; without it the feature does not work in this
regime.

**Distributional and location statistics live on different grids, deliberately.** Entropy,
novelty, tilt and the quantiles are computed on the 12-band grid; `FPE_hz_region` is
computed on the padded native spectrum. Zero-padding smooths noise fluctuation and drives
`H` upward, so a distributional feature computed on a padded grid changes value when the
user picks a different `n_fft` — and `n_fft` is a display choice. A location statistic is
the opposite case: padding cannot improve resolution but it genuinely refines the argmax
readout, and the 12-band grid would quantise `FPE` to 12 levels. **This looks like an
inconsistency and is not.**

Two implementation traps, both subtle enough to be worth recording:

- **(a) The taper is applied to the noise, squared, and must never be inverted.**
  `P_region` comes from the reconstructed signal, which already carries the analysis-band
  Tukey taper; Buffer 3 is built from transmitted magnitudes, which do not. So `P_noise` is
  multiplied by the taper **squared** — squared because these are PSDs and the taper acts
  on amplitude — before subtraction. Without it the subtraction over-subtracts at the band
  edges, where the region is attenuated and the noise estimate is not. **The taper reaches
  exactly 0 at bins 51/204 — not −6 dB, as two other specifications state** — so this must
  never be inverted by dividing `E[k]` by taper²: that divides by zero at the edges.
- **(b) 50 % microphone normalisation must be applied on both sides.** `P_noise` must be
  built from mic-corrected magnitudes, and the region comes from a signal reconstructed
  from the same normalised spectrum, so the frequency-dependent 0.55×–1.49× gain divides
  out of the subtraction. Feeding Buffer 3 raw magnitudes instead corrupts the whole family
  silently — measured at −5.2 … +3.5 dB.

`REGION_NFFT = 4096` is a **display grid, not a resolution**: Δf is fixed by `n_seg`, and
zero-padding only interpolates the DTFT. It is fixed rather than scaled with the segment so
that `FPE_hz_region`'s readout precision does not depend on the event's duration. The
distributional features are invariant to it — measured spread < 10⁻³ across `n_fft` 512 →
8192.

### 8.11 `harmonic_confinement`

Some recordings contain persistent artificial ultrasonic sources whose fundamental sits
near 40 kHz with a second harmonic near 80 kHz. A naive fix — *reject frames peaking near
40/80 kHz* — fails for a fundamental reason: Buffer 3 correctly absorbs a **stationary**
tone into `P_noise[k]`, so in a tone-contaminated session essentially every frame,
including genuine clicks, carries the tone in its raw spectrum. Testing the raw spectrum
would gate out real clicks precisely in the sessions where the interferer lives. The only
question that discriminates is: **does this frame's new (excess) energy sit at the harmonic
pair, or is it spread across the band?**

**Frame domain, and that is structural rather than a fallback:**

- The region window is defined by rise/fall logic that assumes the event returns to the
  noise floor. A sustained oscillation satisfies neither condition as intended — on the
  motivating frame `fall_time_ms` is 0.0150 ms because the envelope dipped below `LEVEL`
  for four samples mid-cycle, not because anything ended. The region is then a few samples
  wide and effectively arbitrary for exactly the noise class this feature targets.
- Region resolution (≈ 3.8 kHz) is coarser than the 1–2 kHz transducer linewidth the
  signature lives in; it would be smeared away before it could be tested.
- Buffer 3 is itself estimated at frame resolution over these 154 bins, so staying there
  makes `P_frame − P_noise` a same-grid, same-units subtraction with **no rebinning and no
  taper term** — avoiding the class of conversion factor that produced two previous scaling
  bugs in this project.

**Definition:**

```
E_frame[k] = max(0, P_frame[k] − P_noise[k])          k ∈ [51, 204]

f₁ = parabolic-interpolated argmax of E_frame over 36–44 kHz

A = [f₁ − 2 kHz, f₁ + 2 kHz]                              n_A bins
B = [2f₁ − 2 kHz, min(2f₁ + 2 kHz, 79 687.5 Hz)]          n_B bins, clipped

q_A = Σ_A E / Σ E      null_A = n_A / 154      r_A = q_A / null_A
q_B = Σ_B E / Σ E      null_B = n_B / 154      r_B = q_B / null_B

harmonic_confinement = log₂( min(r_A, r_B) )
```

`r = 1` is the expected value when the excess is spread uniformly — true both for a genuine
broadband click and for a flat ambient excursion. **`min`, not mean or sum**, is what
encodes "both bands simultaneously" without writing an AND-gate: a lone 40 kHz hum with
nothing at 80 kHz gives high `r_A` but `r_B ≈ 1`, so `min` stays near 1 and the feature
stays near 0. It rises only when both regions are confined together — the harmonic-pair
structure specifically, not a generic single tone, which is `spectral_entropy`'s job.
`log₂` makes the null exactly 0 regardless of frame-to-frame variation in `n_A`/`n_B` from
clipping.

Sub-bin parabolic interpolation of `f₁` is genuinely required here: this feature locates a
narrow tone against a 390.625 Hz grid and derives **two** bands from it, so a half-bin error
moves four band edges. It was written new for this purpose and deliberately **not**
retrofitted into `FPE_hz`, which is a plain argmax — doing so would change an existing
feature's values and break comparability with every past export.

**Three properties that must be known before reading any histogram of it:**

1. **Band B is non-empty only while `f₁ ≤ 40 843.75 Hz`.** A 41 or 42 kHz transducer
   therefore yields NaN, not a measurement — its second harmonic is entirely outside the
   transmitted range. The 36–44 kHz search window is honest about where to *look*, but the
   feature is only *defined* over roughly its lower half. This is physics, not a bug, and
   NaN is the correct output. **Measured: 70 of 339 candidates (20.6 %) on one real
   recording.**
2. **`min(r_A, r_B)` can be exactly 0**, because `E` is rectified and band B can hold
   exactly zero excess — giving `log₂(0) = −∞`. That is a real measurement, the strongest
   possible evidence against a harmonic pair, but `−∞` would poison any downstream scaler,
   so it is emitted as **NaN with `r_A`/`r_B` exported alongside**, keeping the case fully
   distinguishable from a genuinely undefined row. Measured at 7 of 339 rows (2.1 %).
3. **The feature is bounded above by ≈ 3.36 by construction**, not by the data: with all
   the excess inside the two bands, the best case splits it 2/3 : 1/3 and gives
   `min(r) = 154/15 = 10.27`. An even split scores ≈ 2.9.

> ⚠️ **The specification's own validation clause failed, and the direction is favourable —
> but the stated criterion was wrong and should not be reused.** It asked to verify that
> genuine clicks keep `r_A`, `r_B` near 1 at frame resolution. Measured on a tone-free
> recording (339 candidates): `hc` median **−2.96**, p90 **−0.77**, max **+0.52**, and
> **0 %** above +1. Candidates sit well **below** the null, not at it. The reason is
> structural: `r = 1` is the value for excess spread *uniformly*, but a real click is peaked
> at its own frequency — typically 45–55 kHz, which is in neither band A nor band B — so its
> `q_A`, `q_B` are small and `hc` is strongly negative. That *helps* separation (clicks
> negative, harmonic pairs positive), but "clicks keep `r` near 1" is not the right test.
> The right test is that clicks stay clearly below the tonal cluster.

**Validation status, and a finding that inverts the premise.** Per recording, the share of
rows with `hc > 1.6` and the located fundamental:

```
62.4 %   f₁ = 40.2 kHz   alocasia-soloianta-misurazione2
55.6 %   f₁ = 40.2 kHz   aloepezzostaccato_indorteca_misurazione4
22.5 %   f₁ = 40.2 kHz   solosuono_outdoortaverna_misurazione1   (the 344k/h recording)
 0.0 %   f₁ = 41.4 kHz   stimolomeccanico_cactus_misurazione1    (clean; f₁ is just noise argmax)
```

**The 40 kHz source is not an outdoor problem.** The worst-affected recordings are indoor
enclosure (`teca`) sessions. Something in that rig emits at 40.2 kHz, and finding it
physically is cheaper than filtering it. On a tone-free recording,
`corr(harmonic_confinement, spectral_entropy) = −0.14`, far below the 0.7 rule of thumb, so
it is not redundant with entropy there — but that measures null behaviour only, and the
discrimination claim still needs the tonal sessions (§13.6).

Future analysis will address these questions.

### 8.12 Quality and provenance columns

Emitted for every row and never used as features:

| column | meaning |
|---|---|
| `fit_valid` | 0/1 — distinguishes "no fit" from "a fit that was terrible" (§10) |
| `decay_len` | `decay_end − decay_start` |
| `n_seg` | region length in samples — recorded beside every v6 feature |
| `b3_frames` | frames in the Buffer 3 window; **0 ⇒ every v6 feature is NaN** |
| `gibbs_fired` | edge suppression tripped ⇒ biased subtraction that frame |
| `stage1_params` | `'v51_peakpick;k=1.50;R=1;C=10'` |
| `stage2_mode` | which Stage 2 rule produced the row |
| `k_ratio` | `E_i / Ê_floor(i)` — exactly what Stage 1 thresholds on |
| `run_id`, `run_length`, `run_crest`, `pos_in_run`, `would_pass_v5` | Stage 1 delta diagnostics |
| `note` | free text written during review |

`run_length` and `run_crest` are deliberately **outside** the feature vector: both are
threshold- and history-dependent, and the feature set is advertised as dimensionless and
scale-invariant. They are exported because the v5 → v6 delta analysis wants them and they
are unrecoverable after the fact. `would_pass_v5` is what makes that delta computable from
**one** export pass rather than by diffing two corpus runs.

Only exactly-derivable columns were dropped from the schema — `canonical_frame_idx` is
`peak_abs // 512`, `n_seg_valid` is `n_seg >= 45`. The rule applied, so that it stays
applicable later: *drop what is exactly derivable or file-constant; never drop a
measurement to save width.*

---

## 9. Stage 4 – Deduplication

Unchanged from v5.1.

```
PEAK_MATCH_SAMPLES = 8 samples (~40 µs)   # a jitter margin, NOT a time window

1. Sort Stage 3 survivors by peak_abs
2. Start a new group when the peak_abs gap exceeds PEAK_MATCH_SAMPLES
3. Within each group keep the CANONICAL representative — the candidate whose own
   frame owns the peak (frame_idx == canonical_frame_idx), so its context has the
   peak centred and therefore the cleanest envelope. Fall back to highest
   svm_probability, then earliest frame.
4. Assign timestamp = frame_start_time of the retained frame
```

Because both Stage 1 candidates of a straddling click resolve to an integer-identical
`peak_abs` (§7.1), they land in the same group by construction — identity is deterministic
and **independent of the model**. The pre-v5.1 rule merged any detections within 3 frames
(≈ 7.7 ms) and broke ties by `svm_probability`, which merged genuinely distinct clicks less
than 7.7 ms apart and let the retained frame move with every retrained model.

**Peak-picking and deduplication cannot fight each other.** Stage 1 resolves events at
`R = 1` frame separation; two peaks one frame apart are **512 samples** apart, 64× the
8-sample merge margin. The constraint is satisfied with room to spare, which closes an open
question the Stage 1 specification had raised. `DEDUP_WINDOW_FRAMES` is deprecated and
retained only because one offline script still imports it; it no longer drives Stage 4.

---

## 10. Fit Pipeline: τ and R²

### 10.1 The algorithm is unchanged; its role is inverted

The fit estimates the exponential decay constant τ and the goodness-of-fit R² from the
Hilbert envelope, by log-linearising `A(t) = A₀·exp(−t/τ)` and running a closed-form OLS
(no BLAS/LAPACK, for thread safety) over a dynamically located decay window:

```
STEP A  decay_start : first n where 3 consecutive W=4 local slopes are < −0.01·peak_amp
                      capped at peak_idx + 20 samples, fallback peak_idx + 5
STEP B  decay_end   : first n where 4 consecutive samples are < noise_floor + 1.5·std_noise
STEP C  smoothing   : Gaussian σ = 2 samples (10 µs), mode='valid'
STEP D  OLS on log(envelope), R² computed in log space
        τ_ms = −1000/(m·fs) if slope m < 0, else the −1 sentinel
```

**In v5 this was a gate. In v6 it is a feature source.** The change is the single largest
recall gain in the version (§6.1), and it moves the entire burden of interpreting a failed
fit onto the classifier — which is only safe if the classifier can *tell* that the fit
failed. That is what §10.2 is about.

### 10.2 Two sentinels that fail differently

When no decay can be fitted, the fit returns `tau_ms = −1.0` and `R2 = 0.0`. These are
**not measurements**, and they fail in opposite ways:

| sentinel | failure mode |
|---|---|
| `tau_ms = −1` | **Loud.** Real values are ~0.1–0.6 ms, so −1 sits far outside the range: a scaler's standard deviation is inflated and all genuine variation is compressed toward zero. |
| `R2 = 0.0` | **Silent, and worse.** Zero is *inside* the valid range [0, 1], so "the fit failed" and "the fit succeeded and was terrible" are indistinguishable in the data. |

`fit_valid` exists to disambiguate the second case. It does not merely protect the scaler;
it separates two physically different states that had been collapsed into one value. This
matters because `R2 = 0.0` is emitted from two different situations — a degenerate window
(a genuine failure) and a perfectly flat log-envelope on the normal path (`ss_tot ≈ 0`, not
a failure). A blanket "`R2 == 0` → missing" rewrite would conflate them; `fit_valid`
separates them correctly.

**On the v6 corpus, `fit_valid == 0` on 76–88 % of all candidates.** NaN in the decay
features is therefore the **norm, not an edge case**, and how it is encoded is a modelling
decision rather than a library default — see §12.7.

> **`fit_coverage` is not a sentinel.** Of 363 552 rows with `fit_valid == 0`, only 3 802
> have `fit_coverage == 0.0`. The other 359 750 carry a real measurement — `n_fit /
> decay_len`, saying how much of the decay window the fitter got through before giving up.
> Blanking it would destroy that, so it is passed through untouched.

### 10.3 Why OLS on the full fit window

The second local peak visible in some recordings (a reflection or a secondary cavitation
event) degrades R². Restricting the fit to the longest monotone segment would introduce a
new, fragile segmentation step; the degradation is moderate (typically 0.65–0.75 rather
than 0.85+) and is itself **informative** — a moderate R² combined with other strong
features is something a classifier can learn.

---

## 11. Feature Summary Table

**SVM** = read by the deployed model · **gate** = used as a Stage 2 gate ·
**CSV** = exported for analysis and future retrains only.

| # | Feature | Domain | Physical meaning | Status |
|---|---|---|---|---|
| 1 | `peak_SNR` | envelope | Amplitude relative to the noise floor | **SVM + gate** |
| 2 | `pre_SNR` | envelope | Silence before the click | **SVM** |
| 3 | `post_SNR` | envelope | Return to silence after it | **SVM** |
| 4 | `rise_time_ms` | envelope | Onset speed | **SVM** |
| 5 | `fall_time_ms` | envelope | Decay duration at noise level | **SVM** |
| 6 | `fit_valid` | fit | Did a decay fit converge at all | **SVM** |
| 7 | `R2` | fit | Exponential fit quality | **SVM** |
| 8 | `n_seg` | region | Region length in samples | **gate** |
| 9 | `local_crest` | Stage 1 | Prominence over the frame's own background | **gate** |
| 10 | `harmonic_confinement` | frame FFT | Energy confined to a 40/80 kHz pair | **gate** |
| 11 | `SPR` | frame FFT | Spectral tonality | **gate** (OOD only) |
| 12 | `tau_ms` | fit | Exponential decay constant | CSV |
| 13 | `fit_coverage` | fit | Fraction of the decay window fitted | CSV |
| 14 | `asymmetry_integral` | envelope | Rise/fall shape asymmetry | CSV |
| 15–17 | `ZCR_pre`, `ZCR_click`, `ZCR_post` | raw iFFT | Oscillation rate before / during / after | CSV |
| 18 | `kurtosis` | raw iFFT | Impulsivity | CSV |
| 19 | `centroid_shift_hz` | FFT | Spectral evolution during decay | CSV |
| 20 | `R_spectral` | frame FFT | Low vs high frequency balance | CSV |
| 21 | `FPE_hz` | frame FFT | Dominant frequency of the frame | CSV |
| 22 | `spectral_entropy` | region excess | Spectral spread | CSV |
| 23 | `shape_novelty` | region | Shape difference from ambient | CSV |
| 24 | `spectral_tilt` | region | Spectral slope | CSV |
| 25 | `temporal_concentration` | envelope | Energy shape within the region | CSV |
| 26 | `FPE_hz_region` | region excess | Dominant frequency of the event | CSV |
| 27 | `SPR_region` | region | Region tonality | CSV |
| 28–29 | `f_50_hz`, `IQR_f` | region excess | Spectral quantiles | CSV |
| 30–32 | `hc_f1_hz`, `hc_r_A`, `hc_r_B` | frame FFT | `harmonic_confinement`'s parts | CSV |

`hc_r_A` and `hc_r_B` are kept because `harmonic_confinement` is a `min()` and therefore
discards *which* band was flat — the distinction needed to separate a lone hum from a true
harmonic pair, and not recoverable after the fact. `hc_f1_hz` also disambiguates *why* a
NaN happened: NaN there means no excess at all; finite and above 40 843.75 Hz means band B
was clipped away.

### 11.1 What v6 removed from the classifier, and on what number

The deployed model reads 7 features where v5 read 16. Every removal has a measurement
behind it, not a preference:

| removed | measured basis |
|---|---|
| `SPR` | Single-feature AUC **0.454** on the population reaching Stage 3. Click/noise medians 8.92 vs 9.22 — it does not separate. Retained as an OOD **gate** (§6.3, §6.7) |
| `shape_novelty` | Single-feature AUC **0.493** — chance |
| `asymmetry_integral` | Single-feature AUC **0.575**; and in the v5 control run it falls from Set A rank **1** to Set B rank **16** (§12.10) |
| `tau_ms` | **52 % coverage** (undefined whenever the fit fails), Set A single-feature AUC 0.445 drifting to 0.617 on Set B — a +0.172 swing. Removing it cost **0.000** CV AUC (§12.8) |
| `FPE_hz` | AUC drifts **0.595 (Set A) → 0.475 (Set B)**, i.e. across chance to the wrong side; ranks **last** of 9 on held-out permutation importance. Consistent with the independently observed band-edge artefact (~20 % of rows park on the 20 kHz edge) |
| `fit_coverage`, `ZCR×3`, `kurtosis`, `centroid_shift_hz`, `R_spectral` | Not in any v6 candidate feature set; the feature-count ceiling (§12.8) does not permit carrying features that have not earned a slot |

All of them are still **computed and exported**, so a future retrain can reinstate any of
them without a corpus re-export.

---

## 12. SVM Classifier — Training Protocol and Results

### 12.1 Design philosophy: high-recall bias

Unchanged from v5. The two error types are not symmetric in cost:

- **False negative (missed click):** a genuine acoustic emission is lost. In experiments
  running for hours with only a few dozen real events, one lost click is a significant
  fraction of the evidence, and repeated losses can mask or distort the plant's response.
- **False positive:** an extra noise event appears in the output. Downstream analysis tolerates 
  a controlled rate.

The recall target is therefore **≥ 0.90**, and the decision threshold is moved off the
default 0.50 to the lowest value on the out-of-fold ROC curve achieving it.

### 12.2 Dataset

| set | rows | clicks | noise | sessions |
|---|---:|---:|---:|---:|
| Set A (training) | 905 | 158 | 747 | 28 |
| Set B (held out) | 231 | 31 | 200 | 2 |

The training CSV is the **Stage 2 (`v6_conservative`) survivor set** of the exhaustively
labelled corpus. That is why 5 786 labelled noise rows become 947 while all 189 confirmed
clicks survive: the gates remove 83.6 % of noise and 0 % of clicks. (83.6 % of 5 786 is
949, not 947 — the two counts were taken a few days apart and a handful of labels moved in
between; the agreement is to within 0.2 %.) Ambiguous rows are carried in the file and
excluded at train time (`--ambiguous exclude`).

Class balance is therefore **≈ 1 : 4.7** after Stage 2, against ≈ 1 : 30 before it. Stage 2
is doing a large share of the classifier's work, which is exactly the intent — Stage 3 only
ever sees the part of the noise distribution that is hard.

**Set B** is held out entirely from training and from hyperparameter search:
`stimolomeccanico_aloe_misurazione1_03032026_13.49` and
`stimolomeccanico_cactus_misurazione1_03032026_10.39mattina_final` — **one Aloe and one
cactus, chosen deliberately so that the held-out estimate is across species and not merely
across sessions.**

> ⚠️ **Set B holds 31 clicks.** Every Set B figure in this section carries a wide interval
> at that count. **The cross-validated numbers are the decision-grade ones; Set B is
> corroboration.**

### 12.3 Hard-negative mining was turned off — and that is the same idea, done better

v5 applied its Stage 2 gates to the **label-0 rows only** before training, on the reasoning
that trivially rejectable negatives teach the model nothing and inflate apparent
specificity. The reasoning is sound. The implementation, carried onto the v6 corpus, is
not: the filter is the fit gate, and applied to noise it keeps only *well-fitting* noise —
throwing away the hardest negatives and training the model against a class that is **not
the population Stage 2 actually passes.**

The effect is directly visible in the two runs' Set A composition:

| run | Set A rows | noise | sessions |
|---|---:|---:|---:|
| v5 configuration (noise pre-filter ON) | 463 | 305 | 24 |
| v6 (noise pre-filter OFF) | 905 | 747 | 28 |

The pre-filter discards 442 of 747 negatives — 59 % — and four entire sessions.

v6 gets the intended effect **structurally instead**: the training CSV *is* the Stage 2
survivor set (§12.2), so the negative class the model trains on is by construction exactly
the negative class it will meet at inference. No separate filter, no train/serve mismatch,
and the hard negatives are all still there.

### 12.4 Session-level cross-validation, and a failure mode it exposed

Cross-validation uses **`StratifiedGroupKFold` with one group per recording session**, so
no session appears in both training and validation. This matters because a single session
contributes dozens of candidates: without grouping, the model could learn
recording-specific acoustic signatures — microphone position, plant height, background
level — rather than click morphology, and inflate every CV metric.

**The failure v6 found.** On this corpus **13 of 28 Set A sessions contain no clicks at
all.** `StratifiedGroupKFold` is greedy — it keeps whole sessions together and balances as
it goes — so it can and does emit a validation fold made entirely of noise-only sessions.
On one configuration, fold 0 came out at 25 rows and 0 clicks.

That single fold is not a rounding error:

```
roc_auc_score returns NaN for a single-class y_true
    → np.average propagates it, so EVERY candidate's mean_test_score becomes NaN
    → sklearn (_search.py:1128) takes the all-NaN branch and ranks every candidate 1
    → best_index_ = argmin() returns 0
    → "best_params_" is whichever combination came FIRST IN GRID ORDER
```

It still prints like a normal answer. Only `best_score_ = nan` gives it away. Measured
cost on the affected file: the grid returned `C=0.1, gamma=scale` where a repaired search
picks `C=5, gamma=0.01` — out-of-fold AUC 0.9019 versus 0.9144.

**Only `roc_auc` is exposed to this.** `recall` (via `make_scorer(zero_division=0)`) and
`average_precision` both return 0.0 for such a fold — wrong, but finite, so the ranking
still happens. `roc_auc` is the v6 default, which is why this surfaced with v6 and not v5.

Two fixes, both in place:

1. The split seed is walked forward (`seed`, `seed+1`, …) until every validation fold holds
   at least `MIN_FOLD_POSITIVES = 5` clicks. The floor is 5 rather than 1 because an AUC
   over a single positive is the rank of one row yet carries the same 1/5 weight as a fold
   with 86 clicks. **The model seed never changes; only the partition does**, and the
   chosen split seed is recorded in the `.pkl`.
2. A non-finite `best_score_` after the grid is a **hard error**, not a warning. A model
   selected by grid position is not a selected model.

**The deployed model used `cv_split_seed = 43`** — seed 42 did not clear the floor. Its
five folds are 121/143/151/272/218 rows carrying 11/54/30/45/18 clicks, and
`best_score_ = 0.945` is finite, so the collapse above did not occur for this run.

### 12.5 Scaling: Yeo–Johnson, chosen on measurement

These features are ratios with long tails. Measured on the training CSV with the v6-core
set and an identical CV protocol (**CV AUC / Set B AUC**):

| scaler | linear | rbf | |
|---|---|---|---|
| `standard` | 0.705 / 0.910 | 0.867 / 0.874 | v5 behaviour |
| `robust` | 0.547 / 0.850 | 0.805 / 0.879 | **worst** |
| `log10` | 0.892 / 0.955 | 0.919 / 0.960 | |
| **`yeo-johnson`** | **0.917 / 0.934** | **0.928 / 0.963** | **default** |
| `quantile` | 0.902 / 0.944 | 0.916 / 0.962 | |

Yeo–Johnson wins on both kernels, needs no column subsetting because it handles zero and
negative values, and is a plain scikit-learn built-in so it pickles with no import
coupling — unlike a `log10` transform, which serialises a *reference* to the function and
therefore requires the exact same importable symbol at load time.

> **`RobustScaler` is the trap, and it is worth naming.** The instinct is to reach for "the
> outlier-robust one". It divides by the interquartile range, and this data packs its bulk
> into a tiny IQR with a long tail above it — so the tail is **amplified, not tamed**:
> `peak_SNR |z|max` goes 17.7 → 198.9, `local_crest` 19.6 → 2914, and linear CV AUC
> collapses to 0.547. Bad scaling also made libsvm crawl: the first pass of this comparison
> took over 10 minutes for one linear grid search, where Yeo–Johnson and log10 finish in
> 0.4 s with zero non-convergent fits.

### 12.6 Why the grid scores on `roc_auc`

The grid selects **hyperparameters**; the decision threshold is tuned separately afterwards
from the out-of-fold ROC curve. AUC measures **ranking quality**, which is exactly what that
tuning consumes. `recall` at 0.50 measures a rule that is then discarded — and recall alone
cannot distinguish a good model from one that predicts everything click (recall 1.000,
precision 0.175, AUC 0.500).

This matters most under `class_weight='balanced'`, which pushes toward exactly that
degenerate corner. Measured, rbf + v6-core: **recall scoring reports a CV score of 0.894
while selecting a model whose true out-of-fold AUC is 0.830; `roc_auc` reports 0.911 and
selects one at 0.885.** The better-looking number belongs to the worse model.

`class_weight` was set to `auto`, which puts `[None, 'balanced']` in the grid and lets
cross-validation choose. It chose **`balanced`** — at 947 noise : 189 clicks that is
`w_click = 3.005`, `w_noise = 0.600`, an effective 5.01× reweighting.

### 12.7 NaN policy, and why both ends must agree

The exporter writes `tau_ms = −1.0` and `R2 = 0.0` when the decay fit fails, on **90.2 % of
candidates**. §10.2 explains why those are not measurements.

The trainer converts them to NaN **keyed on `fit_valid`, never on the value** — because 202
rows have `fit_valid == 1` **and** `R2 == 0.0` and are real (a perfectly flat log-envelope
on the normal path). The Pipeline's `SimpleImputer(median)` then fills them.

The choice is stamped into the model as `nan_policy` and read back at inference by
`_stage3_scores`, which applies the same conversion before scoring. **This has to follow
the model, not the pipeline version.** The shipped v5 model was trained *on* the sentinels,
so converting for it would create the very train/serve skew this exists to prevent, in the
opposite direction. A model saved without the key is v5-era by definition, hence the
`'sentinel'` default. At 90.2 % of candidates, a disagreement here would be near-total.

`fit_coverage` is deliberately **not** converted (§10.2).

**The residual hazard, stated rather than papered over.** With the fit gate gone, rows with
no usable decay reach the classifier, where the imputer fills `tau_ms`/`R2` and the model
returns a confident-looking probability computed partly from imputed values. The mitigation
is that **`fit_valid` is itself a feature**, so the model can condition on "this row's decay
features are imputed" — but that only works if `fit_valid` is in `model['features']`. It is
(§12.9), and that is why it is non-optional rather than a nice-to-have.

### 12.8 Feature-set selection — the four-run comparison

Four models were trained on the same CSV with the same protocol, differing only in feature
set. RBF row shown; the linear results agree in ordering.

| run | features | CV AUC-ROC | Set B AUC | threshold |
|---|---:|---:|---:|---:|
| v5 configuration, re-run on the v6 corpus | 17 | 0.781 | 0.790 | 0.167 |
| `v6-core` | 9 | 0.929 | 0.964 | 0.115 |
| 6 best | 6 | 0.927 | 0.940 | 0.117 |
| **6 best + `R2` — DEPLOYED** | **7** | **0.929** | **0.958** | **0.121** |

**The decision, and its reasoning.** Dropping `tau_ms` and `FPE_hz` from `v6-core` moves CV
AUC **0.929 → 0.929** and Set B **0.964 → 0.958**. At 189 positives that is not a
difference worth two extra dimensions: roughly 9–16 features is what this corpus honestly
supports, and every feature that does not earn its slot is variance. **The smaller model
is the one to take when the larger one is not measurably better.**

Each drop also has independent evidence, which is what makes this parsimony rather than
guesswork:

- **`FPE_hz`** — single-feature AUC **0.595 on Set A, 0.475 on Set B**. It does not merely
  weaken; it crosses chance to the *wrong side* on sessions the model has never seen, and
  it ranks **last of nine** on held-out permutation importance (+0.0048 ± 0.0075). That is
  the signature of a training-session property, and it agrees with an independently
  observed artefact: ~20 % of rows park on the 20 kHz band edge, which looks like a
  microphone-normalisation or band-edge effect rather than physics.
- **`tau_ms`** — defined on only **52 % of rows** (473 of 905), and its AUC drifts +0.172
  between the two sets. What it carries about decay duration, `fall_time_ms` (AUC 0.787,
  100 % coverage) already carries better.

**Why `R2` went back in.** Dropping it as well costs 0.002 CV AUC and **0.018 Set B AUC** —
and `R2` is the one feature in the run whose held-out rank *rises*: Set A permutation rank
**9 of 9**, Set B permutation rank **4 of 9**, the largest positive move in the table. A
feature that the fitted model leans on lightly but that still carries signal where the model
has never looked is exactly the kind to keep. It is also the natural companion to
`fit_valid`: together they say *whether* a decay was fitted and *how well*, which neither
answers alone.

Coverage is worth noting beside it: `R2` is measured on 473 of 905 Set A rows (52 %) and
138 of 231 Set B rows (60 %). The rest are imputed, which is precisely the situation
`fit_valid` exists to flag.

### 12.9 Results — the deployed model

```
kernel      rbf
C           50
gamma       0.01
class_weight balanced
pipeline    SimpleImputer(median) → PowerTransformer(yeo-johnson) → SVC(probability=True)
features    peak_SNR, pre_SNR, post_SNR, fall_time_ms, rise_time_ms, fit_valid, R2
threshold   0.1207
nan_policy  nan
```

| metric | CV @ 0.50 | **CV @ 0.121** | Set B |
|---|---:|---:|---:|
| Recall | 0.601 | **0.905** | 0.968 |
| Precision | 0.766 | 0.505 | 0.333 |
| Specificity | 0.961 | 0.813 | 0.700 |
| F1 | 0.674 | 0.649 | 0.496 |
| **AUC-ROC** | 0.929 | **0.929** | **0.958** |
| Accuracy | 0.898 | 0.829 | 0.736 |
| Confusion | TP 95 · FP 29 · FN 63 · TN 718 | TP 143 · FP 140 · FN 15 · TN 607 | TP 30 · FP 60 · FN 1 · TN 140 |

The precision at the operating threshold (0.505) is the deliberate cost of the recall
target, not a defect. The full ladder, from the same out-of-fold curve, shows the trade
explicitly:

| threshold | recall | precision | specificity | F1 | note |
|---:|---:|---:|---:|---:|---|
| 0.5000 | 0.601 | 0.766 | 0.961 | 0.674 | sklearn default |
| 0.3743 | 0.709 | 0.663 | 0.924 | 0.685 | recall ≥ 0.70; **max F1** |
| 0.2408 | 0.816 | 0.561 | 0.865 | 0.665 | recall ≥ 0.80 |
| **0.1207** | **0.905** | **0.505** | **0.813** | **0.649** | **recall ≥ 0.90 — DEPLOYED** |
| 0.0936 | 0.949 | 0.489 | 0.790 | 0.645 | max Youden J |
| 0.0636 | 0.949 | 0.424 | 0.727 | 0.586 | recall ≥ 0.95 |

**Held-out, per session:**

| session | clicks | detected | false positives | recall |
|---|---:|---:|---:|---:|
| `stimolomeccanico_aloe_misurazione1` | 11 | 11 | 15 | **1.00** |
| `stimolomeccanico_cactus_misurazione1` | 20 | 19 | 45 | **0.95** |

**Kernel comparison** (both trained identically, rbf selected):

| kernel | CV AUC-ROC | threshold | Set B AUC |
|---|---:|---:|---:|
| linear | 0.920 | 0.142 | 0.940 |
| **rbf ←** | **0.929** | **0.121** | **0.958** |

The margin over linear is small — 0.009 CV AUC — which is worth stating plainly: the
click/noise boundary in this 7-dimensional space is only mildly non-linear, and the linear
model with `C = 0.1` is a defensible alternative that would be more interpretable. RBF was
selected on the measurement.

### 12.10 Feature importance, and why two measures are reported

**Set A importance says what the fitted model leans on. Set B permutation importance says
what still carries signal where the model has never looked.** A feature ranking high in the
first and near zero in the second is a property of the training sessions, not of clicks.

| # | feature | Set A Δroc_auc | Set B Δroc_auc |
|---:|---|---:|---:|
| 1 | `peak_SNR` | +0.1657 ± 0.0093 | +0.2472 ± 0.0290 |
| 2 | `fall_time_ms` | +0.0612 ± 0.0085 | +0.0855 ± 0.0205 |
| 3 | `pre_SNR` | +0.0609 ± 0.0053 | +0.0264 ± 0.0154 |
| 4 | `post_SNR` | +0.0568 ± 0.0088 | +0.0271 ± 0.0168 |
| 5 | `fit_valid` | +0.0219 ± 0.0030 | +0.0421 ± 0.0133 |
| 6 | `rise_time_ms` | +0.0130 ± 0.0039 | +0.0030 ± 0.0063 |
| 7 | `R2` | +0.0076 ± 0.0023 | +0.0236 ± 0.0069 |

The two orderings agree closely, which is the result one wants: the top three by
generalisation (`peak_SNR`, `fall_time_ms`, `fit_valid`) are all in the top five by fit.
Near-zero can mean unimportant **or** that a correlated feature carries the same
information; and at 31 held-out clicks the ordering is readable but the magnitudes are not.

The corresponding linear-kernel weights give the sign of each effect, which permutation
importance cannot: `peak_SNR` +0.943, `pre_SNR` −0.680, `fall_time_ms` +0.628,
`post_SNR` −0.559, `fit_valid` +0.477, `rise_time_ms` −0.313, `R2` +0.193 (positive pushes
toward *click*). Louder, longer-decaying, better-fitting events preceded and followed by
quiet are clicks — which is the physical picture, recovered from the data rather than
assumed.

> **The object lesson, from the v5 control run.** The same two measures on the v5
> configuration disagree violently:
>
> | feature | Set A rank | Set B rank | move |
> |---|---:|---:|---:|
> | `asymmetry_integral` | 1 | 16 | **−15** |
> | `rise_time_ms` | 12 | 4 | +8 |
> | `ZCR_post` | 17 | 9 | +8 |
> | `fall_time_ms` | 7 | 1 | +6 |
>
> v5's single most important feature was 16th of 17 on held-out sessions, and `peak_SNR` —
> the most robust feature in the entire v6 model — sat at Set A rank 15 with a *negative*
> held-out contribution. **Reporting only training-set importance would have hidden this
> completely.** It is the reason both measures are now printed for every run.

### 12.11 Deployment

The model ships as a joblib archive containing the fitted Pipeline, the operating
threshold, the ordered feature list, and full provenance — training rows and sessions,
scaler, imputer, class weighting, NaN policy, seed, CV split seed, library versions and
the git SHA of the code that produced it.

```python
import joblib, numpy as np

model = joblib.load('src/ml/v6/plantleaf_svm_v6_DEPLOYED.pkl')
pipe  = model['pipeline']
thr   = model['threshold']          # 0.1207

# X: (n, 7), columns in model['features'] order.
# Sentinels must already be converted per model['nan_policy'];
# the Pipeline imputes and scales internally.
proba = pipe.predict_proba(X)[:, 1]
pred  = (proba >= thr).astype(int)  # 1 = click, 0 = noise
```

Three contract points:

- **`model['features']` is authoritative.** Schema membership is not model membership.
  Nothing about Stage 3 is hardcoded in the pipeline module: the feature list, the kernel
  and the threshold all come from the file, which is why deploying v6 required no Stage 3
  code change at all.
- **`model['nan_policy']` must be honoured**, or 90 % of candidates are scored under an
  encoding the model was never fitted on (§12.7).
- **Both model versions ship** with the application. v6 is the default; v5 remains
  selectable from the data-collection dialog for comparison runs, and is correctly treated
  as `nan_policy = 'sentinel'` because it predates the key.

---

## 13. Known Limitations and Open Questions

This section is not a disclaimer. It is what makes the rest of the document usable: a
number quoted without its domain of validity is not a measurement.

### 13.1 The corpus is indoor-only

Every session in the training and held-out sets is an indoor recording — laboratory rooms,
enclosure (`teca`) sessions, empty-room baselines, and potted plants under mechanical or
water stress. The outdoor recordings that motivated v5 were exported but carry no labelled
rows, so they contributed nothing to any threshold or to the model.

**Every performance figure in this document is an indoor figure.** The v6 Stage 2 gates
were derived from indoor distributions; `harmonic_confinement`'s corpus-level 16.9 % noise
cut is dominated by indoor `teca` sessions. Outdoor behaviour is unmeasured, not
guaranteed.

Outdoor analysis and evaluation in currently in progress.

### 13.2 The outdoor noise floor is an open question, not a settled one

Outdoor recordings emit up to **344 773 candidates/hour** — 24 % of all frames, half the
structural cap at `R = 1`. Across 56 recordings, the floor's dynamic range over an entire
recording is p10 1.10×, median 1.30×, max 2.04×, which is a suspiciously narrow range for
an estimator meant to track a changing environment.

The working hypothesis was that the burst gate (`α = 4.0`, whose comment still says *verify
experimentally on outdoor recordings*) freezes the estimator — the mechanism §4.10
describes.

> ⚠️ **The first measurements point AWAY from that hypothesis.** On five indoor recordings
> the gate fires on **0.0–0.01 %** of frames, longest consecutive run 1–20 frames — yet the
> floor range is still 1.11–1.33×. A gate that never fires is not what is holding the floor
> still.
>
> **The noisy outdoor recordings remain untested** — the diagnostic run was interrupted when
> the external drive disconnected, and the script is resumable. This is the first thing to
> finish.

A plausible alternative worth testing: the outdoor noise is **impulsive, not elevated**. The
floor would then be tracking the quiet baseline correctly, the transients genuinely do
exceed `k·Ê_floor`, there is no floor bug at all, the candidate rate is the detector working
as designed in a hostile environment — and Stage 2 is the right fix, which is where this
work already landed.

This matters because `peak_SNR`, `pre_SNR` and `post_SNR` all divide by `noise_floor`, and
`LEVEL` and `decay_end` are `noise_floor + k·std_noise`, so the floor **defines the decay
window** and therefore `n_seg`, `tau_ms`, `R2` and every region-based feature. Of the
features used as Stage 2 gates, only `harmonic_confinement` is floor-independent.

### 13.3 `spectral_entropy` partly encodes region duration

The specification's own mandatory-validation failure condition is met.
`corr(spectral_entropy, n_seg) = −0.488` on a 339-candidate recording: median entropy 0.802
for `n_seg < 45` (n = 163) against 0.625 for `n_seg ≥ 45` (n = 81). Short regions read as
higher entropy — the predicted direction, since a spectrum smeared across correlated bands
looks flatter, and flat is maximum entropy.

**76 % of candidates sit below `V6_MIN_NSEG = 45`**, where the 12-band grid is not
resolvable, and this is not an artefact of failed fits (75.6 % short among `fit_valid == 1`,
76.2 % among `fit_valid == 0`). The real median region length across all candidates is **11
samples (55 µs)**, against the 30–80 the specification assumed.

`fall_time_ms` and `decay_len` already carry duration, so entropy is both confounded with
and partly redundant on them. The feature is exported and **not deployed**; the choice
between emitting NaN below the threshold, residualising on `n_seg`, or revisiting whether a
12-band grid over `[onset, decay_end]` is the right region at all is unresolved. It must not
be silently special-cased away.

### 13.4 Two Stage 1 constants are bounded below, not above

`R = 1` is bounded below by the AC-1 measurement (§5.4) — `R = 2` costs confirmed clicks —
but **nothing bounds it above**; no measurement shows it is optimal. `C = 10` for
`local_crest` is argued from timescales and has never been validated against labels.
`local_crest` is used as a Stage 2 gate on a measured zero-click-cost threshold, which is
safe, but whether it would earn a place in the *classifier's* feature set is unmeasured.

### 13.5 Set B is 31 clicks

Held-out recall of 0.968 means 30 of 31. One click either way moves it by 3 points. The
cross-validated figures rest on 158 clicks and are the ones to quote; Set B corroborates
rather than confirms. The same caveat applies with more force to the per-session table,
where one session contributes 11 positives.

### 13.6 `harmonic_confinement` is validated on four recordings

Its null behaviour is measured (§8.11) and its threshold costs zero clicks, but the
discrimination claim — that harmonic-pair interference separates from *generic* negatives,
not merely from clicks — still needs the tonal-interferer sessions labelled. If it does not
separate from generic negatives, the min-ratio construction is adding nothing over
`spectral_entropy` alone. The correlation check that exists (−0.14) was run on a tone-free
recording and therefore measures only that the two are not redundant in the null case.

### 13.7 Documentation errata found while writing this

Recorded rather than silently corrected, because both live in code that ships:

- **The schema column count is stale in two places.** `SCHEMA_VERSION`'s comment in
  `data_collection_dialog_v5.py` says 51 columns and the Stage 1 implementation record says
  52; `CSV_COLUMNS` actually holds **57**. The schema *contents* are correct in all three;
  only the counts drifted as columns were added.
- **The `v6-final` preset does not reproduce the deployed column order.** `train_svm.py`'s
  `v6-final` preset lists the same seven features as the deployed model but with
  `rise_time_ms`/`fall_time_ms` and `R2`/`fit_valid` swapped. This is **harmless at
  inference** — `model['features']` is authoritative and the deployed model was trained from
  an explicit `--features` list, not from the preset — but `--feature-set v6-final` will not
  reproduce the deployed model's column order.

### 13.8 What a future retrain should decide first

1. Finish the outdoor floor diagnostic (§13.2). `peak_SNR` and `n_seg` are among the
   strongest available features and both depend on the floor; retraining before that
   question closes bakes any error into the model, and the Stage 2 thresholds would need
   re-deriving too.
2. Label outdoor recordings, so that §13.1 stops being true.
3. Run the ambiguous-class sensitivity check (`--ambiguous exclude|click|noise`). The
   distributions place ambiguous rows *between* click and noise on `peak_SNR`, which is
   evidence the label is capturing something real; if recall moves materially between the
   three settings, the set deserves a weighting scheme rather than a hard assignment.
4. Ablate `local_crest`, `shape_novelty` and `FPE_hz_region` against the deployed seven,
   now that the corpus supports it.
5. Consolidate the feature-name list, which is currently restated in seven places.

---

## 14. Algorithm Version History

| Version | Date | Key changes |
|---|---|---|
| v1.0 | August 2025 | Basic energy threshold |
| v2.0 | November 2025 | Added R_spectral, iFFT reconstruction, Hilbert envelope |
| v2.1 | December 2025 | R²_log ≥ 0.60–0.80 as primary gate; 22 % FNR discovered |
| v3.0 | February 2026 | R² removed as gate; 3 criteria (SNR, pre_snr, E_W1 > E_W4) |
| v3.1 | March 2026 | Added asymmetry (C4), τ range (C5); window extended to 300 samples |
| v4.0 | March 2026 | Absolute peak_iFFT (C1 = 130 µV); τ and R² criteria; asymmetry reformulated; Gibbs suppressor v3; Stage 1 k = 5 + MAX_RUN = 4; Stage 2 normalised peak filter |
| v5.0 | May–June 2026 | Adaptive noise estimator; Stage 1 adaptive threshold + MAX_RUN = 3; Stage 2 hard gates (R² < 0.10, SPR ≥ 100); all other thresholds replaced with SVM features; improved fit pipeline; new features: post_SNR, ZCR×3, kurtosis, centroid_shift, rise/fall time, asymmetry_integral; RBF-SVM on 285 labelled events from 38 sessions (16 features, threshold 0.220, CV recall 0.907, Set B AUC 0.925) |
| v5.1 | July 2026 | Frame-grid-independent features: every time-domain feature resolved and measured on a stitched prev\|curr\|next context (§7.1). Stage 4 deduplicates by absolute peak sample (`peak_abs`) instead of frame-index gap (§9). Spectral features still frame-based. **Changes feature values → SVM retrain required.** |
| **v6.0** | **August–September 2026** | **Stage 1 rebuilt: run-length rejection replaced by local peak picking (`R = 1`), measured to recover 45–53 % of above-threshold frames that `MAX_RUN` was deleting in click-bearing recordings; candidate volume +32 %. Stage 2 rebuilt on measurable features (`peak_SNR`, `n_seg`, `local_crest`, `harmonic_confinement`) plus two OOD bounds — 83.6 % of noise for a measured 0.0 % of clicks, replacing a fit gate that cost 12.2 %. The decay fit demoted from gate to feature (`fit_valid`, `R2`, `tau_ms`). Buffer 3 added: per-bin noise PSD by rolling mean, after minimum statistics were measured 82× low per bin. New feature families: v6 excess spectrum (entropy, novelty, tilt, temporal concentration, region FPE/SPR, quantiles) and `harmonic_confinement`. 57-column schema, `peak_abs` label migration key, ambiguous label 2. Sentinels converted to NaN on both training and inference, keyed on `fit_valid`. RBF-SVM on 7 features over 1 136 Stage 2 survivors from 30 sessions (189 confirmed clicks): threshold 0.121, CV AUC-ROC 0.929, CV recall 0.905, Set B AUC 0.958.** |

---

## 15. References

1. **Khait I., et al.** "Sounds emitted by plants under stress are airborne and informative." *Cell*, 186(7):1328–1336, 2023.
2. **Martin R.** "Noise power spectral density estimation based on optimal smoothing and minimum statistics." *IEEE Trans. Speech Audio Process.*, 9(5):504–512, 2001. *(Minimum-statistics noise floor estimator; §4.5, §4.9)*
3. **Stahl V., Fischer A., Bippus R.** "Quantile based noise estimation for spectral subtraction and Wiener filtering." *Proc. IEEE ICASSP*, 1875–1878, 2000. *(Quantile alternative to the minimum; §4.5)*
4. **Rangachari S., Loizou P.C.** "A noise-estimation algorithm for highly non-stationary environments." *Speech Communication*, 48(2):220–231, 2006. *(Why a fixed global percentile is fragile; §4.5)*
5. **Yeo I.-K., Johnson R.A.** "A new family of power transformations to improve normality or symmetry." *Biometrika*, 87(4):954–959, 2000. *(The v6 scaler; §12.5)*
6. **Platt J.** "Probabilistic outputs for support vector machines and comparisons to regularized likelihood methods." In *Advances in Large Margin Classifiers*, MIT Press, 61–74, 1999. *(Calibrated probabilities behind `SVC(probability=True)`; §12.9)*
7. **Tyree M.T., Sperry J.S.** "Do woody plants operate near the point of catastrophic xylem dysfunction?" *Plant Physiology*, 88(3):574–580, 1988.
8. **Knowles Electronics.** *SPU0410LR5H-QB Datasheet*, Rev. H, 2020.
9. **STMicroelectronics.** *STM32F411CEU6 Datasheet*, 2023.

---

*Document maintained by the PlantLeaf project contributors.*
*Last updated: September 2026 — v6.0*
