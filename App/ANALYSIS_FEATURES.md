# Analysis Features Guide

## Overview

PlantLeaf provides two offline analysis modes for reviewing saved recordings:

- **Audio Replay** (`ReplayWindowAudio`): frame-by-frame FFT inspection, adaptive energy filtering, inverse FFT reconstruction, v5 feature computation, and data collection export for the click detection pipeline.
- **Voltage Replay** (`ReplayVoltageWindow`): time-domain voltage signal review, region selection, and mathematical curve fitting for plant electrical events.

---

## Audio Replay Mode

### Opening a Recording

**Launch methods**:
- Home window → "Open Audio File" → select a `.paudio` file
- File menu → Open → choose a `.paudio` file
- Drag and drop a `.paudio` file onto the application window

The window opens maximized (`ReplayWindowAudio`).

**File validation on load**:
- Magic bytes must be `PAUDIO`
- Version must be 3.0 or higher for phase data (iFFT support)
- A progress dialog shows byte count and estimated remaining time for large files

**After loading**, the application pre-computes normalized FFT energy and adaptive noise floor estimates for every frame using `AdaptiveNoiseEstimatorV5`. This one-time pass enables instant threshold filtering and accurate feature computation throughout the session.

---

### FFT Spectrum View

**Tab**: "FFT Spectrum"

**Display**:
- X-axis: Frequency (analysis band, approximately 20–80 kHz)
- Y-axis: Amplitude (V, linear scale)
- Normalized mode (default): curve shown in a slightly darker accent color
- Raw mode: curve shown in the theme's standard accent color

**Frame navigation** (toolbar):
- Frame slider: drag or click-and-drag to scrub through the recording
- Arrow buttons: step one FFT frame forward or backward (~2.56 ms per step, ~390 FPS resolution)
- The current frame position is indicated by a vertical dashed line on the time-domain plot

**Normalization toggle**: Analysis menu → "Toggle Normalized/Raw" applies or removes the 50% SPU0410LR5H-QB microphone correction. The FFT and time-domain plots switch simultaneously; the iFFT window has its own independent toggle.

---

### Time-Domain Energy View

**Tab**: "Time Domain"

This plot shows the per-frame FFT energy [V²] over time, not a reconstructed waveform. Three layers are drawn simultaneously:

| Layer | Color | Meaning |
|-------|-------|---------|
| Main curve | Theme accent / blue | FFT energy per frame (normalized or raw, follows the normalization toggle) |
| Threshold curve (dashed red) | Red dashed | k × Ê_floor(i) — Stage 1 pass criterion at each frame |
| Noise floor curve | Cyan | Ê_floor(i) — raw adaptive noise estimate without multiplier |

The adaptive threshold and noise floor are pre-computed at load time and rendered without further cost.

**Playback**: speed range 0.1× to 1.0× of real recording speed. During playback, the view scrolls a 20-second window around the current position.

---

### Stage 1 Threshold Filter

The right panel contains two tabs:

**"Recorded Events"** — click events embedded in the `.paudio` file at acquisition time (stored by the firmware during live recording). Columns: Timestamp, Frequency, Amplitude, Duration (FFT frames), Notes. Double-click a row to jump to that timestamp.

**"Above Threshold"** — live re-filtering of the loaded recording against the current k value. Consecutive frames whose normalized FFT energy exceeds k × Ê_floor are grouped (maximum 5-frame gap) and listed. Columns: Timestamp, Frequency, Amplitude (mV), Duration (FFT frames), Notes (SNR ratio at peak frame).

**k spinbox** ("Stage 1 threshold multiplier k"): range 0.5–20.0, step 0.5, default 1.5. Lower k casts a wider net; higher k reduces false positives. Press "Apply" or change the spinbox value to rebuild the table. The threshold curve on the time-domain plot updates automatically when k changes.

This filter exposes Stage 1 of the v5 click detection pipeline interactively. See [CLICK_DETECTION_ALGORITHM_v5.md](Automatic_click_detection_algorithm/CLICK_DETECTION_ALGORITHM_v5.md) for the full pipeline (Stages 1–4) and the offline SVM classifier.

---

### Microphone Normalization

**Correction**: 50% conservative correction based on the SPU0410LR5H-QB frequency response datasheet.

**Estimated error**: ±2.9 dB (95% confidence). See [MICROPHONE_NORMALIZATION_TECHNICAL_REPORT.md](MICROPHONE_NORMALIZATION_TECHNICAL_REPORT.md) for the interpolation method and full error analysis.

**When enabled (default)**:
- The FFT curve switches to the darker accent color
- The time-domain energy plot and threshold curve both use normalized energy values
- The iFFT window applies normalization when opened (can be toggled independently in that window)

**Note**: SPR, R_spectral, and FPE are always computed on normalized data regardless of the display toggle — this matches the SVM training inputs exactly. Analysis menu → "FFT Parameters" shows these three values for the current frame in a dialog.

---

## Inverse FFT (iFFT) Window

### Opening the Window

**Action**: toolbar button or Analysis menu → "iFFT Graph" (`Ctrl+I`)

**Requirements**: file version ≥ 3.0 with real phase data. If phase data is absent, the button is disabled. The window title shows `[Real Phases]` or `[Zero Phases]` accordingly.

The window opens following the main window normalization state: if normalized mode is active, the normalized signal is displayed by default.

The iFFT reconstruction uses `reconstruct_frame_v5`, which applies a Tukey taper internally to suppress Gibbs artifacts at the spectral band edges. This is always active — there is no user-facing toggle for it.

---

### Normalization Toggle

**Button**: "Apply 50% Normalization" / "Show Raw iFFT"

Reconstructs the iFFT with or without the 50% microphone correction, using the same `reconstruct_frame_v5` pipeline as the click detector. The normalized signal is shown in a darker accent color (default); the raw signal uses the standard accent color.

---

### Hilbert Envelope

**Button**: "Show Hilbert Envelope" / "Hide Hilbert Envelope"

Computes the instantaneous amplitude envelope from `scipy.signal.hilbert` on the current signal (raw or normalized) and overlays:
- Hilbert envelope: red thick line
- Peak marker: yellow dashed vertical line at the sample of maximum envelope amplitude

The peak is located on the envelope rather than on the oscillating signal, which avoids misidentifying a zero-crossing as the peak.

---

### Decay Analysis (v5 Features)

**Button**: "Analyze Decay"

Runs `compute_features_v5()` for this frame, fetching per-frame noise estimates from `AudioDataManager` and the adjacent frame envelopes for `pre_SNR` and `post_SNR`. A scrollable dialog shows all 17 v5 features alongside physically motivated expected ranges:

| # | Feature | Expected range (genuine click) |
|---|---------|-------------------------------|
| 1 | peak_SNR | >> 1 (typically 10–1000+) |
| 2 | pre_SNR | ≈ 1.0 (silence before click) |
| 3 | post_SNR | ≈ 1.0 (return to silence) |
| 4 | rise_time_ms | 0.025–0.3 ms |
| 5 | fall_time_ms | > rise_time |
| 6 | asymmetry_integral | positive (decay > rise) |
| 7 | ZCR_pre | low (silence before) |
| 8 | ZCR_click | oscillation rate during click |
| 9 | ZCR_post | decreasing during decay |
| 10 | kurtosis | > 0, typically 0.4–3 (noise ≈ −0.6) |
| 11 | centroid_shift_hz | > 2–5 kHz (high frequencies decay first) |
| 12 | tau_ms | 0.05–1.3 ms (cavitation) |
| 13 | R² | ≥ 0.45 (exponential fit quality) |
| 14 | fit_coverage | 0.7–1.0 (fraction of decay window used) |
| 15 | SPR | ≤ 20 (broadband, not tonal) |
| 16 | R_spectral | descriptive — E[20–40 kHz] / E[40–80 kHz] |
| 17 | FPE_hz | dominant frequency in the analysis band |

These ranges are guidance only. No hard thresholds are applied in the iFFT window — all 17 features are passed to the SVM for the final decision. `fit_coverage` is excluded from the SVM input and shown for diagnostic purposes only.

---

### Fit Curve Overlay

**Button**: "Show Fit Curve" (enabled after "Analyze Decay" has been run)

Overlays on the iFFT plot:

- Exponential fit A₀ · exp(−t/τ) over the identified decay window, color-coded by R²:
  - R² ≥ 0.70: green
  - R² ≥ 0.45: orange
  - R² < 0.45: red
- Yellow dotted vertical line at the peak sample
- Cyan dashed horizontal line at `noise_floor`
- Purple dotted horizontal line at `noise_floor + std_noise`

---

### Show Only Envelope

**Button**: "Show Only Envelope" / "Show iFFT Signal"

Hides or restores the raw iFFT trace to let the Hilbert envelope be inspected without the carrier oscillation.

---

## Click Detection Algorithm

The v5 pipeline (Stages 1–4) runs in real-time during acquisition and can be re-applied offline via the data collection export. The threshold filter in the replay window exposes Stage 1 interactively.

For the complete algorithm specification — including Stage 2 hard gates (R² ≥ 0.10, SPR < 100), Stage 3 SVM classifier, Stage 4 deduplication, all 17 feature definitions, training protocol, and evaluation results — see:

**[CLICK_DETECTION_ALGORITHM_v5.md](Automatic_click_detection_algorithm/CLICK_DETECTION_ALGORITHM_v5.md)**

---

## Data Collection Export

**Action**: File menu → "Export Data Collection" (audio replay only)

**Purpose**: exports Stage 1 survivors from one or more `.paudio` files as a labeled CSV and two-panel screenshots, ready for manual labeling and SVM training.

**Dialog** (`DataCollectionDialogV5`):
- Select `.paudio` files from a folder
- Set the Stage 1 multiplier k (default 1.5 — intentionally wide to avoid missing any click)
- Run export: for each candidate, all 17 v5 features and noise estimates are computed and written

**CSV schema** (one row per Stage 1 candidate):

| Column | Description |
|--------|-------------|
| file | `.paudio` filename |
| frame_idx | Frame index (0-based) |
| timestamp_s | Absolute time (s) |
| noise_floor_uV | Adaptive noise floor at this frame (µV) |
| std_noise_uV | Noise standard deviation (µV) |
| E_hat_floor | Ê_floor energy threshold (V²) |
| peak_SNR … FPE_hz | 17 v5 features |
| label | Fill manually: 1 = click, 0 = noise, empty = unknown |

**Screenshots**: two-panel PNG per candidate (FFT spectrum + iFFT waveform with feature table), rendered with QPainter without opening display windows.

---

## Data Export — Trimmed Region

**Action**: File menu → "Export Trimmed Region..." (`Ctrl+T`)

Available in both audio and voltage replay windows.

**Workflow**:
1. File → Export Trimmed Region...
2. Select start and end time in the dialog
3. Choose output file name

The trimmed file is written with a valid header preserving the sampling rate and all metadata. For voltage files, saved analyses within the selected time range have their timestamps adjusted to the new time origin.

---

## Voltage Replay Mode

### Opening a Recording

Same methods as audio, but selecting a `.pvoltage` file. Opens `ReplayVoltageWindow`.

**File format**: magic bytes `PLANTVOLT`, versions 1.0 and 2.0. The header flag `amplified` determines Y-axis scaling:
- Amplified recording: ±1.7 V
- Non-amplified: ±130 mV

Large files are loaded in 100,000-sample chunks with a progress dialog.

---

### Main Voltage Plot

**Display**: single time-domain trace showing the full recording at rest. On playback, the view scrolls a 15-second window centered on the current position.

**Playback controls** (toolbar):
- Play / Pause: scroll through the recording in real time
- Stop: reset to position 0 and restore the full-recording view
- Speed: playback speed multiplier

**Time slider**: drag to jump to any position.

---

### Mathematical Analysis and Curve Fitting

**Purpose**: automated parameter extraction and model fitting for plant electrical signals.

**Activation**:
1. Analysis menu → "Select Region for Analysis" — a movable 5-second selection region appears centered on the current view. Click again to remove it.
2. Drag the region boundaries to frame the event (maximum 30 seconds)
3. Double-click the region, or Analysis menu → "Analyze Region" — opens the `MathOperations` dialog

During playback, region selection is disabled and the action button is grayed out.

#### Signal Type Auto-Detection

**Exponential Return** (variation potential / slow relaxation): single dominant peak, monotonic return to baseline, no significant opposite rebound. Detection criterion: rebound amplitude < 30% of main peak amplitude.

**Action Potential**: main peak followed by a rebound in the opposite direction (≥ 30% of main peak), rebound occurring after the main peak, time between peaks < 2.0 s, both peaks above 3σ of baseline noise.

#### Mathematical Models

**Exponential Return**:
```
V(t) = A · exp(-(t - t₀) / τ) + V_baseline
```
Fitted over the decay phase (from peak onwards). Parameters: A (amplitude relative to baseline), t₀ (peak time), τ (time constant), V_baseline (resting potential).

**Action Potential** (piecewise composite):
```
         ⎧ A_sin · sin(2π · f · (t - t₀) + φ)           for t < t_peak
V(t) =  ⎨
         ⎩ A_exp · exp(-(t - t_peak) / τ) + V_baseline   for t ≥ t_peak
```
Parameters: A_sin, f, φ (depolarization oscillation); A_exp, τ (repolarization exponential); V_baseline.

#### Baseline Estimation

Mean of a pre-event window: last 50 samples of the 1 second preceding the selected region. If less than 1 second is available, the first 50 samples of the selection are used instead.

#### Fitting Engine

`scipy.optimize.curve_fit` (Levenberg–Marquardt / Trust Region Reflective), maximum 10,000 function evaluations. Initial parameters are estimated analytically from signal shape before optimization (τ from the 63.2% criterion, f from zero-crossing timing). Parameter bounds are derived dynamically from the signal amplitude range.

#### Goodness of Fit

```
R² = 1 - SS_res / SS_tot
```
| R² | Quality |
|----|---------|
| 0.99–1.00 | Excellent |
| 0.95–0.99 | Good |
| 0.85–0.95 | Acceptable |
| < 0.85 | Poor — review parameters or region |
| < 0 | Model worse than flat line (warning shown automatically) |

#### Signal Energy

Trapezoidal numerical integration of (V(t) − V_baseline)² over the fitted region. Units: V²·s (proportional to energy dissipated per unit resistance).

#### Manual Parameter Adjustment

All fitted parameters can be adjusted via spinboxes after auto-fitting. The curve, R², and energy update in real time. "Auto-Fit" re-runs `curve_fit` using the current spinbox values as the initial guess.

---

### Saving and Exporting Analyses

**Save**: appends the analysis as a JSON record in the footer of the `.pvoltage` file without overwriting any signal data. Stores model type, all fitted parameters, R², energy, time range, name, and save timestamp. Multiple analyses per file are supported, each identified by a UUID.

**Open Saved Analysis**: Analysis menu → "Open Saved Analysis" lists all saved analyses for the current file. Select one to reopen the `MathOperations` dialog pre-filled with saved parameters. Analyses can be deleted from this dialog (permanent, with confirmation prompt).

**Export to CSV**: exports one row per sample in the fitted region with columns: Time_s, Voltage_V, Fitted_V, Residual_V.

---

## Troubleshooting

**iFFT window shows noise, not a waveform**: the frame has no signal above background, or the file lacks phase data (version < 3.0). Navigate to a frame with a visible FFT peak before opening the iFFT window.

**No candidates in the Above-Threshold table**: lower k toward 0.5. If the table is still empty, the recording may contain no frames above the adaptive noise floor estimate.

**Normalization makes the FFT spectrum look wrong**: the 50% correction is calibrated only for the SPU0410LR5H-QB microphone. Use raw mode if a different transducer was used.

**MathOperations fit does not converge**:
1. Check that the selected region contains a single clean event.
2. Try switching signal type (Exponential Return ↔ Action Potential).
3. Manually adjust initial parameters before clicking "Auto-Fit".
4. Narrow or widen the region to better isolate the event.

---

## Best Practices

### Audio Analysis

1. Load the recording and let the pre-computation finish.
2. Start at k=1.5. Review the "Above Threshold" table to estimate the false positive rate.
3. Increase k to reduce false positives; lower it only if genuine clicks are missing.
4. For any suspicious candidate: step to the frame, open the iFFT window, run "Analyze Decay" to inspect the full feature vector.
5. Use "Export Data Collection" to batch-export Stage 1 survivors across multiple recordings for offline SVM evaluation.

### Voltage Analysis

1. Play the recording at reduced speed to locate events of interest.
2. Pause at the event and use "Select Region for Analysis" to frame it.
3. If R² > 0.95, the fit is reliable. If R² < 0.85, adjust the region boundaries or switch signal type.
4. Save each analysis before moving to the next event.

---

## References

1. **Click Detection Algorithm v5**: [CLICK_DETECTION_ALGORITHM_v5.md](../autoclick/v5/CLICK_DETECTION_ALGORITHM_v5.md)
2. **FFT Phase Technical Specification**: [FFT_PHASE_TECHNICAL_SPECIFICATION.md](FFT_PHASE_TECHNICAL_SPECIFICATION.md)
3. **Microphone Normalization Report**: [MICROPHONE_NORMALIZATION_TECHNICAL_REPORT.md](MICROPHONE_NORMALIZATION_TECHNICAL_REPORT.md)

---

**Last Updated**: June 2026  
**Author**: Tommaso Vaninetti  
**Version**: 2.0