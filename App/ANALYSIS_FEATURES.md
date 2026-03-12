# 🔬 Analysis Features Guide

## Overview

PlantLeaf provides advanced post-processing tools for analyzing saved recordings. Analysis features include FFT spectrum visualization, inverse FFT reconstruction, microphone normalization, click detection algorithms, and data export utilities.

---

## Audio Analysis Mode

### 🎯 Purpose

Analyze saved `.paudio` recordings with frame-by-frame precision, reconstruct time-domain signals from phase data, and characterize ultrasonic click events.

---

## Opening Recordings

### 1. Launch Audio Replay Mode

**Methods**:
- **From Home**: Click "Open Audio File" → Select `.paudio` file
- **File Menu**: File → Open → Choose `.paudio`
- **Drag & Drop**: Drag `.paudio` file onto application window (macOS/Windows)
- **macOS Finder**: Double-click `.paudio` file (if file association configured)

**Window Opens**: `ReplayWindowAudio`

### 2. Loading Progress

**Large Files** (>100 MB):
- Progress dialog displays
- Shows: Bytes read / Total size
- Estimated time remaining
- Cancel option (aborts load)

**File Validation**:
- ✅ Check magic bytes: "PAUDIO"
- ✅ Verify version: 3.0 or compatible
- ✅ Validate frame count: Positive integer
- ❌ Error if corrupted: Display diagnostic message

---

## FFT Spectrum Analysis

### 1. Main Spectrum View

**Default Display**:
- **X-axis**: Frequency (20-80 kHz)
- **Y-axis**: Magnitude (Volts, linear scale)
- **Curve Color**: changes based on the theme selected
- **Current Frame**: Indicated by slider position

**Controls**:
- **Frame Slider** (bottom): Navigate through recording
  - Click-and-drag for scrubbing
  - Arrow keys for single-frame steps
  - Home/End keys for first/last frame
- **Play Button**: Auto-advance through frames (5 FPS default)
- **Frame Number Box**: Type frame number for direct jump

**Info Panel** (right side):
- **Frame**: Current frame index (0-based)
- **Timestamp**: Absolute time (MM:SS.mmm)
- **Peak Frequency**: Bin with maximum amplitude
- **Peak Magnitude**: Maximum value (V)
- **Mean Magnitude**: Average across 20-80 kHz
- **Total Energy**: Sum of squared magnitudes (V²)

### 2. Microphone Normalization

**Purpose**: Correct for non-flat frequency response of SPU0410LR5H-QB microphone on both FFT and iFFT window

**Activation**:
- **Menu**: Analysis → Normalize FFT Window

**Effect**:
- **Red Curve**: Normalized spectrum overlays raw curve
- **Legend**: Shows both curves with labels
  - "Raw"
  - "Normalized 50%" (red)

**Technical Details**:
See [MICROPHONE_NORMALIZATION_TECHNICAL_REPORT.md](normalization_feature/MICROPHONE_NORMALIZATION_TECHNICAL_REPORT.md) for:
- Datasheet interpolation method
- Error analysis (±2.9 dB)
- Scientific validation

**Toggle Off**:
- Menu: Analysis → Show Raw FFT Only

### 3. Plot Interaction

**PyQtGraph native controls**:
- **Zoom**: Mouse wheel or right-click drag
- **Pan**: Left-click drag
- **Auto-range**: Press 'A'

---

## Inverse FFT (iFFT) Reconstruction

### 🎯 Purpose

Reconstruct time-domain signal from FFT magnitude + phase data to achieve **sub-frame temporal resolution** (5 μs vs. 2.56 ms frame duration).

### 1. Open iFFT Window

**Action**: Analysis → iFFT Graph (or press `Cmd+I`)

**Requirements**:
- ✅ File version ≥ 3.0 (phase data included)
- ❌ Version < 3.0: Error dialog "No phase data available"

**Window Layout**:
- **Top Panel**: FFT spectrum (current frame)
- **Bottom Panel**: iFFT time-domain waveform (512 samples, 2.56 ms)

### 4. Windowing Options

**Problem**: Abrupt spectral edges (bins 51, 204) cause Gibbs artifacts (ringing)

**Solution**: Apply Tukey window (10% taper) to complex spectrum

**Controls**:
- **Checkbox**: "Apply Tukey Window (10% taper)"
  - ✅ Checked (default): Smooth edges, minimal ringing
  - ❌ Unchecked: Raw spectrum, visible Gibbs artifacts

**Visual Comparison**:
- **With Window**: Clean start/end of waveform
- **Without Window**: ~9% overshoot, decaying oscillations

**Technical Explanation**:
See [FFT_PHASE_TECHNICAL_SPECIFICATION.md](FFT_and_acquisition_specifications/FFT_PHASE_TECHNICAL_SPECIFICATION.md), Section 7

### 5. Normalization in iFFT

**Purpose**: Apply 50% microphone correction to time-domain signal

**Activation**: Click **"Apply 50% Normalization"** button


**Toggle Back**: Click **"Show Raw iFFT"** button

**Important Note**:
- Normalization applied to **complex spectrum** before iFFT
- Not a simple amplitude scaling of time signal
- Preserves phase relationships

### 6. Envelope Analysis

**Hilbert Envelope**:
- **Button**: "Show Hilbert Envelope" in iFFT window
- **Method**: Scipy `hilbert()` transform to compute instantaneous amplitude
- **Display**: Red thick line overlaid on iFFT waveform
- **Toggle**: Click button again to hide envelope

**Decay Analysis**:
- **Button**: "Analyze Decay" in iFFT window
- **Method**: Logarithmic fit of envelope decay over a 0.6 ms post-peak window (120 samples at 200 kHz)
- **Output**: Decay time constant τ (ms) and R² of fit
- **Purpose**: Confirm exponential decay typical of genuine ultrasonic clicks

---

## Click Detection Algorithm

### 🎯 Purpose

Automatically identify frames containing ultrasonic clicks based on a **4-stage pipeline** algorithm.

### 1. Open Click Detector Dialog

**Action**: Analysis → 🔍 Automatic Click Detector... (or `Ctrl+D`)

**Window**: `ClickDetectorDialog`

**File Info panel** (auto-populated from recording):
- Total duration (s)
- Total frames
- Mean energy μ (mV)
- Std deviation σ (mV)

### 2. Detection Pipeline

The algorithm runs 4 sequential stages:

#### Stage 1: Energy Threshold
- **Parameter**: Energy threshold (spinbox, units: mV)
- **Default**: μ + 4σ (auto-calculated from the loaded file)
- **Step size**: 1σ (spinbox step adapts to the file's noise floor)
- **Logic**: Frames with mean FFT amplitude above threshold are **candidates**

#### Stage 2: Spectral Ratio (descriptive)
- **Computes**: R = E_low / E_high (energy ratio between frequency sub-bands)
- **Purpose**: Descriptive feature only — saved per click, **not used as a pass/fail criterion**
- **Optional**: "Apply 50% microphone correction" checkbox normalizes the FFT before computing R

#### Stage 3: Three-Criterion Validation
Candidate frames must pass **all three criteria**:

| Criterion | Parameter | Default | Description |
|-----------|-----------|---------|-------------|
| 1. SNR | Min. SNR spinbox | 5.0 | peak_amplitude / noise_rms > threshold |
| 2. PRE_ratio | Max. PRE_ratio spinbox | 0.15 | E_pre / E_post < threshold (silence before click) |
| 3. Decay | — | fixed | E_W1 > E_W4 (energy decreases across sub-windows: global decay) |

**Noise estimation**: Before stage 1, an offline noise estimation scans the file to compute `noise_rms` from frames below the energy threshold (up to 500 samples).

#### Stage 4: Deduplication
- Merges detections that are too close in time to be separate events

**Check the detailed documentation to have a deeper understanding** [CLICK_DETECTION_ALGORITHM_MATHEMATICAL_FRAMEWORK.md](Automatic_click_detection_algorithm/CLICK_DETECTION_ALGORITHM_MATHEMATICAL_FRAMEWORK.md)

NOTE THAT THE ALGOROTHM IS STILL IN DEVELOPMENT

### 3. Run Detection

**Action**: Click **"▶ Run Detection"** button

**Process**:
1. Offline noise estimation from empty frames
2. Stage 1 candidate selection
3. Stage 2 spectral ratio computation (descriptive)
4. Stage 3 three-criterion validation
5. Stage 4 deduplication
6. Results table populated

### 4. Results Table

**Columns**:
| Column | Description |
|--------|-------------|
| Timestamp | Absolute time (s) |
| Amplitude | Peak FFT amplitude (V) |
| SNR | Signal-to-noise ratio (criterion 1) |
| PRE_ratio | Pre/post energy ratio (criterion 2) |
| E_W1/E_W4 | First vs. last sub-window energy ratio (criterion 3) |
| Energy FFT | Total frame energy |
| Ratio R | E_low / E_high spectral ratio (descriptive) |
| R² (log) | R² of logarithmic decay fit (descriptive) |
| τ (ms) | Decay time constant from envelope fit (descriptive) |
| Classification | Validated click / Rejected |
| Notes | Additional information |

**Navigation**: Double-click a row to jump to that frame in the main window.

### 5. Export Results

**Action**: Click **"Export Results..."** button (enabled after detection)

**Format**: CSV with all table columns

---

## Data Export

### 1. Export Trimmed Region (Audio & Voltage)

**Action**: File → Export Trimmed Region... (or `Ctrl+T`)

**Purpose**: Save a selected time sub-range of the recording as a new `.paudio` or `.pvoltage` file.

**Workflow**:
1. Open a recording
2. File → Export Trimmed Region...
3. Select start/end time in the dialog
4. Choose output file name
5. Trimmed file is written with a proper header (preserving sampling rate, metadata)

---

## Voltage Analysis Mode

### 🎯 Purpose

Analyze saved `.pvoltage` recordings with statistical tools, event detection, and **automated mathematical fitting** for plant electrical signals (action potentials and exponential relaxation events).

### 1. Open Voltage Recording

**Same as Audio** (see section above), but select `.pvoltage` file

**Window Opens**: `ReplayVoltageWindow`

### 2. Main Voltage Plot

**Display**:
- **X-axis**: Time (seconds)
- **Y-axis**: Voltage (mV or V, depending on whether the amplifier was used during acquisition)
- **Curve**: Single trace (full recording visible at rest)

**Playback controls** (toolbar):
- **Play / Pause**: Scroll through the recording in real time
- **Stop**: Returns to full-recording view (position reset to 0)
- **Speed**: Playback speed multiplier

**Time slider**: Drag to jump to any position in the recording.

### 3. Export Voltage Data

**Trimmed Region Export**:
- File → Export Trimmed Region... (`Ctrl+T`)
- Select start/end time → saves a new `.pvoltage` file

---

## 📐 Mathematical Analysis & Automatic Fitting

### 🎯 Purpose

The **Mathematical Analysis module** (`MathOperations` dialog) provides **automated parameter extraction and curve fitting** for plant electrical signals. It automatically classifies the signal type and fits a physiologically motivated mathematical model, extracting quantitative parameters for scientific comparison.

**Activation**:
1. In `ReplayVoltageWindow`, click **Analysis → Select Region for Analysis** — the cursor enters region-selection mode
2. Click and drag on the voltage plot to define the time region of interest
3. Once a region is selected, click **Analysis → Analyze Region** to open the `MathOperations` dialog

---

### Signal Type Auto-Detection

The algorithm automatically classifies the selected signal as one of two types:

#### A. Exponential Return (Variation Potential / Slow Relaxation)
**Biological meaning**: Slow membrane depolarization or repolarization after a stimulus, typical of variation potentials or wound-induced responses.

**Detection criteria**:
- Single dominant peak (either upward or downward)
- No significant counter-oscillation (rebound ratio < 30% of main amplitude)
- Monotonic decay/rise toward baseline after peak

#### B. Action Potential
**Biological meaning**: Fast oscillatory membrane event characteristic of true action potentials in excitable plant cells (e.g., Mimosa, Venus flytrap).

**Detection criteria**:
- Main peak followed by a **significant rebound in opposite direction** (rebound ratio ≥ 30%)
- Peaks in correct temporal order (rebound comes after main peak)
- Time between peaks < 2.0 seconds
- Both peaks exceed 3σ above baseline noise

**Decision logic** (simplified):
```
if (rebound_amplitude > 30% of main_amplitude)
   AND (rebound occurs AFTER main peak)
   AND (time_between_peaks < 2.0 s):
   → Action Potential
else:
   → Exponential Return
```

---

### Baseline Estimation

**Method**: Mean of a pre-event window (~1 second before the selected region)

```
V_baseline = mean(V[pre-event window])
σ_baseline = std(V[pre-event window])
```

**Purpose**:
- Zero-reference for amplitude measurements
- Noise threshold (3σ criterion for detection)
- Fitting parameter `Vb` (asymptote of exponential)

**Pre-event window selection**:
- If ≥ 1s of data precedes the selection: uses last 50 samples of that second
- If < 1s available: uses first 50 samples of the selection
- Edge case (< 50 samples total): uses first 1/3 of available data

---

### Mathematical Models

#### Model 1: Exponential Return

Fitted to the **decay phase** (from peak onwards):

```
V(t) = A · exp(-(t - t₀) / τ) + V_baseline
```

| Parameter | Symbol | Physical Meaning |
|-----------|--------|-----------------|
| Amplitude | A | Peak voltage relative to baseline (V) |
| Time origin | t₀ | Time of peak (s) |
| Time constant | τ | Characteristic decay time (s) |
| Baseline | V_baseline | Resting membrane potential (V) |

**Initial parameter estimation** (before `curve_fit`):
- `A` = `V_peak - V_baseline`
- `τ` = time for signal to reach `V_baseline + 0.632 × |A|` after peak (63.2% criterion, i.e., one time constant)
- `t₀` = time of peak
- `V_baseline` = pre-event mean

**Fitting domain**: Only samples **from t₀ onwards** (peak to end of event), to avoid distortion from the rising phase which follows a different kinetics.

---

#### Model 2: Action Potential (Composite Model)

A **piecewise function** combining a sinusoidal depolarization phase and an exponential repolarization phase:

```
         ⎧ A_sin · sin(2π · f · (t - t₀) + φ)           for t < t_peak
V(t) =  ⎨
         ⎩ A_exp · exp(-(t - t_peak) / τ) + V_baseline   for t ≥ t_peak
```

| Parameter | Symbol | Physical Meaning |
|-----------|--------|-----------------|
| Sine amplitude | A_sin | Half-amplitude of depolarization oscillation (V) |
| Oscillation frequency | f | Characteristic frequency of the depolarization (Hz) |
| Phase | φ | Phase offset of the sine wave (rad), default ≈ 5° |
| Exp. amplitude | A_exp | Amplitude of repolarization exponential (V) |
| Peak time | t_peak | Time of peak / transition between phases (s) |
| Time constant | τ | Repolarization time constant (s) |
| Baseline | V_baseline | Resting membrane potential (V) |

**Physical interpretation**:
- **Sine phase** (t < t_peak): Models the oscillatory depolarization event (fast voltage change)
- **Exponential phase** (t ≥ t_peak): Models the membrane repolarization back to resting potential

**Initial parameter estimation**:
- `A_sin` = `(V_first_peak - V_second_peak) / 2`
- `f` estimated from zero-crossing time after main peak
- `A_exp` = `V_peak - V_baseline`
- `τ` = time to reach `V_baseline - 0.632 × |A_exp|` (63.2% criterion)

---

### Curve Fitting Engine

**Library**: `scipy.optimize.curve_fit` (Levenberg-Marquardt / Trust Region Reflective)

**Strategy**: Two-step approach:
1. **Auto-detect** initial parameters from signal shape (see above)
2. **Bounded optimization** with physically motivated parameter bounds

**Bounds** (exponential return example):
```python
# Bounds derived dynamically from signal range
V_span = V_max - V_min
bounds = (
    [-1.2 * V_span,          # A_min
     t_peak - 0.5,           # t0_min
     0.0001,                  # tau_min (0.1 ms)
     V_min - V_span],        # Vb_min
    [+1.2 * V_span,          # A_max
     t_peak + 0.5,           # t0_max
     signal_duration,         # tau_max
     V_max + V_span]         # Vb_max
)
```

**Convergence**: Max 10,000 function evaluations

---

### Goodness-of-Fit: R² Coefficient

**Formula**:
```
R² = 1 - SS_res / SS_tot

where:
  SS_res = Σ(V_measured - V_fitted)²    (residual sum of squares)
  SS_tot = Σ(V_measured - V_mean)²      (total sum of squares)
```

**Interpretation**:
| R² Value | Quality |
|----------|---------|
| 0.99 - 1.00 | Excellent fit |
| 0.95 - 0.99 | Good fit |
| 0.85 - 0.95 | Acceptable |
| < 0.85 | Poor fit, review parameters |
| < 0 | Model worse than flat line (auto-warning) |

**Automated warning**: If R² < 0, a dialog prompts the user to review signal type selection, time range, or initial parameters.

---

### Signal Energy (Numerical Integration)

The dialog computes the **electrical energy** of the event above baseline using the trapezoidal rule:

```
E = ∫ (V(t) - V_baseline)² dt  ≈  Σ [(V[i] - V_b)² + (V[i+1] - V_b)²] / 2 × Δt
```

**Units**: V²·s (proportional to energy dissipated per unit resistance)

**Reported value**: Total energy over the fitted region.

---

### Manual Parameter Adjustment

After auto-fitting, users can **manually fine-tune** all parameters via spinboxes:

**Real-time update**: Modifying any spinbox instantly recalculates:
1. The fitted curve
2. The R² coefficient
3. The energy integral
4. The displayed formula

**Controls available**:

*Exponential Return*:
- `A` (amplitude): Slider + spinbox, range ±2× signal span
- `τ` (tau): Slider + spinbox, range 0.1 ms to 10 s
- **Direction toggle**: Upward / Downward (inverts amplitude sign)

*Action Potential*:
- `A_sin`, `f`, `φ` (sine component)
- `A_exp`, `τ` (exponential component)
- **Direction toggle**: Upward / Downward
- **Signal type selector**: Switch between Exponential Return and Action Potential

**Auto-Fit button**: Re-runs `curve_fit` with current spinbox values as initial guess.

---

### General Variables Panel

A separate display shows auto-detected event properties regardless of model:

| Variable | Description |
|----------|-------------|
| V_baseline | Mean baseline voltage (V) |
| V_max | Maximum voltage in event (V) |
| V_min | Minimum voltage in event (V) |
| t_start | Event start time (s) |
| t_end | Event end time (s) |
| t_peak | Time of peak voltage (s) |
| Direction | Upward / Downward |

These values are also editable for manual correction, and draggable reference lines appear on the plot for visual confirmation.

---

### Saving & Exporting Analysis Results

**Save Analysis** (button: "Save"):
- Appends the analysis as a JSON record in a footer at the end of the `.pvoltage` file (no data is overwritten)
- Stores: analysis name, model type, all fitted parameters, R², energy, time range, and timestamp
- Multiple analyses can be saved per file (different time regions), each identified by a UUID

**Re-open a saved analysis**:
- Analysis → Open Saved Analysis
- A dialog lists all saved analyses with name and save date
- Select one and click OK to re-open the `MathOperations` dialog pre-filled with saved parameters
- Analyses can also be deleted from this dialog (permanent, with confirmation prompt)

**Export to CSV** (button: "Export CSV"):
```csv
Time_s, Voltage_V, Fitted_V, Residual_V
12.000, 1.6523, 1.6488, 0.0035
12.001, 1.6721, 1.6695, 0.0026
...
```

**Report Summary** (shown in dialog):
```
=== Analysis Report ===
Signal Type: Action Potential
Direction: Upward
V_baseline: 1.648 V
V_peak: 1.892 V
Peak time: 12.234 s
Duration: 1.45 s

Fitted Parameters:
  A_sin = 0.142 V    f = 2.31 Hz    φ = 0.087 rad
  A_exp = 0.183 V    τ = 0.312 s    Vb = 1.648 V

Goodness of Fit:
  R² = 0.9723

Signal Energy (above baseline):
  Total:  8.42 × 10⁻⁴ V²·s
```

**For publications**: Cite fitted parameters (A, τ, R²) as quantitative descriptors of the response, comparable across experiments and species.

---

## Troubleshooting

### Issue: iFFT Window Shows Noise

**Symptoms**: Time-domain signal looks like random noise, no clear waveform

**Causes**:
1. No phase data (file version < 3.0)
2. Phase data corrupted
3. Selected frame has no signal (background noise)

**Solutions**:
1. Check file version displayed in the window title
2. Navigate to a frame with a known click (high FFT amplitude)
3. If "Zero Phases" is shown in the title, the file lacks phase data and iFFT will not be meaningful

### Issue: Click Detection Finds Nothing

**Symptoms**: "0 clicks detected" despite visible events

**Causes**:
1. Threshold too high (μ + Nσ too large for the recording)
2. SNR criterion too strict

**Solutions**:
1. Lower threshold spinbox value by one or more σ steps
2. Reduce Min. SNR below 5.0

### Issue: Normalization Makes Spectrum Look Wrong

**Symptoms**: Normalized curve is strange or noisier

**Causes**:
1. Microphone model mismatch (correction only valid for SPU0410LR5H-QB)
2. Very low SNR (noise gets amplified by the correction)

**Solutions**:
1. Verify microphone model: correction is based on SPU0410LR5H-QB datasheet only
2. If SNR is very low, use raw spectrum instead

### Issue: MathOperations Fit Does Not Converge

**Symptoms**: Fitted curve does not match signal, R² is very low or negative

**Solutions**:
1. Check that the selected region contains a single clean event (no multiple overlapping signals)
2. Try switching signal type (Exponential Return ↔ Action Potential)
3. Manually adjust initial parameters via spinboxes before clicking "Auto-Fit"
4. If R² < 0, a warning dialog is shown automatically

---

## Best Practices

### 1. Analysis Workflow

**Recommended Order**:
1. **Overview**: Scroll through the entire recording to identify regions of interest
2. **Rough Detection**: Run click detector with the default threshold (μ + 4σ), then adjust
3. **Manual Review**: Inspect each detected click (FFT spectrum + iFFT waveform)
4. **Voltage Fitting**: Select regions of interest in voltage recordings, run MathOperations, save analyses
5. **Export**: Export click detector results (CSV) and voltage fitted curves (CSV)

### 2. Metadata Documentation

**Recommended Practice**: Create a `metadata.txt` alongside each recording

**Contents**:
```
Date: 2026-03-11
Time: 14:30:00
Plant species: Solanum lycopersicum
Plant ID: TOM-42
Age: 6 weeks
Condition: Drought stress (3 days no water)
Microphone position: 5 cm above stem, 2 cm lateral
Room temperature: 23°C
Humidity: 45%
Background noise level: estimated from μ ± σ displayed in Click Detector dialog
Experimenter: T. Vaninetti
Notes: Watered at t=600s
```

---

## References

1. **FFT Phase Technical Specification**: [FFT_PHASE_TECHNICAL_SPECIFICATION.md](FFT_and_acquisition_specifications/FFT_PHASE_TECHNICAL_SPECIFICATION.md)
2. **Microphone Normalization Report**: [MICROPHONE_NORMALIZATION_TECHNICAL_REPORT.md](normalization_feature/MICROPHONE_NORMALIZATION_TECHNICAL_REPORT.md)
3. **Click Detection Algorithm**: [CLICK_DETECTION_ALGORITHM_MATHEMATICAL_FRAMEWORK.md](CLICK_DETECTION_ALGORITHM_MATHEMATICAL_FRAMEWORK.md)
4. **User Guide (Normalization)**: [NORMALIZATION_USER_GUIDE.md](normalization_feature/NORMALIZATION_USER_GUIDE.md)

---

**Last Updated**: March 11, 2026  
**Author**: Tommaso Vaninetti  
**Version**: 1.0
