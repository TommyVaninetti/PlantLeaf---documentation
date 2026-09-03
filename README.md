# PlantLeaf Desktop Application

[![Version](https://img.shields.io/badge/version-1.0.0--beta-blue.svg)](https://github.com/TommyVaninetti/PlantLeaf-Desktop-App)
[![Python](https://img.shields.io/badge/python-3.8+-blue.svg)](https://www.python.org/)
[![License](https://img.shields.io/badge/license-AGPLv3-green.svg)](https://www.gnu.org/licenses/agpl-3.0.html)
[![PySide6](https://img.shields.io/badge/PySide6-6.9.0-green.svg)](https://doc.qt.io/qtforpython/)

**PlantLeaf** is a desktop application designed for **plant bioacoustics and bioelectric research**, enabling real-time acquisition and analysis of ultrasonic click events and voltage signals (action potentials) in plants. All source code is available under the AGPLv3 licence at [PlantLeaf-Desktop-App](https://github.com/TommyVaninetti/PlantLeaf-Desktop-App).

---

## Overview

PlantLeaf bridges the gap between rigorous scientific analysis and accessible tooling, providing:

- **Ultrasonic Click Detection**: capture and analyze plant-emitted ultrasonic clicks (20–80 kHz) with sub-millisecond temporal resolution
- **Action Potential Monitoring**: high-precision voltage acquisition for plant electrical signals
- **Real-Time Acquisition**: live FFT spectrum visualization at 390.625 FPS for audio and up to 1 kHz sampling for bioelectric signals
- **Machine Learning Pipeline**: SVM classifier (v6) trained on hand-crafted acoustic features for high-recall click detection, with every gate threshold quoted against its measured cost in confirmed clicks
- **Advanced Analysis Tools**: phase-preserving FFT, inverse FFT reconstruction, microphone normalization, automatic curve fitting for voltage signals

### Scientific Capabilities

- **Frequency Range**: 20–80 kHz (ultrasonic band, SPU0410LR5H-QB microphone)
- **Sampling Rate**: 200 kHz (audio) / 50 Hz–1 kHz (voltage)
- **FFT Resolution**: 512 samples, 390.625 Hz/bin
- **Phase Preservation**: 8-bit quantized phase data for iFFT reconstruction
- **Temporal Resolution**: 2.56 ms frame duration, 5 μs sub-frame localization
- **Dynamic Range**: 12-bit ADC (74 dB SNR)

---

## Repository Structure

This repository focuses on:
- **software and firmware** developed by **Tommaso Vaninetti**
- **hardware** developed by **Abdoellah El Makkaoui**

---

## Key Features

### Real-Time Acquisition

#### Audio Mode (Ultrasonic Clicks)
- **512-sample FFT** at 390 FPS for real-time spectral analysis
- **Live spectrum visualization** (20–80 kHz bandpass)
- **Phase data preservation** for inverse FFT reconstruction
- **Adaptive click detection** with real-time Stage 1 threshold display
- **Long-duration recording** (hours) with multi-level memory architecture

#### Voltage Mode (Action Potentials)
- **High-precision ADC** (12-bit, 0–3.3 V range)
- **Variable sampling rates** (up to 1 kHz)
- **Low-pass filtering and notch filter**
- **Event annotation** with timestamps
- **CSV export** for external analysis

---

### Ultrasonic Click Detection Algorithm — v6

The current version uses a 4-stage pipeline:

| Stage | Description |
|-------|-------------|
| Stage 1 | Adaptive energy threshold (frame energy > k × Ê_floor, k = 1.5), then **local peak picking** over ±1 frame. Nothing is rejected for run length |
| Stage 2 | Gates on measurable features: `peak_SNR ≥ 4.5`, `n_seg ≥ 10`, `local_crest ≥ 1.2`, `harmonic_confinement ≤ 1.6`, plus two out-of-distribution bounds |
| Stage 3 | SVM classifier on 7 acoustic features (RBF kernel, threshold = 0.121 for recall ≥ 0.90) |
| Stage 4 | Deduplication on the absolute peak sample (`peak_abs`) |

The SVM (scikit-learn `Pipeline`: `SimpleImputer(median) → PowerTransformer(yeo-johnson) → SVC`, `class_weight='balanced'`) was trained with session-level cross-validation (`StratifiedGroupKFold`) on 1,136 Stage-2 survivors (189 confirmed clicks, 30 sessions, 4 plant species). Cross-validated AUC-ROC = 0.929 at recall 0.905; held-out AUC-ROC = 0.958 at recall 0.968.

Every Stage 2 threshold is quoted in the specification with the measured percentage of confirmed clicks it costs — the v6 rule removes 83.6 % of noise for a measured 0.0 % of clicks, where the v5 rule it replaced cost 12.2 %.

For the full algorithm specification, feature definitions, training protocol, and evaluation results, see [CLICK_DETECTION_ALGORITHM_v6.md](App/Automatic_click_detection_algorithm/CLICK_DETECTION_ALGORITHM_v6.md).

---

### Audio Replay and Analysis

- **FFT Spectrum View**: frame-by-frame spectrum (20–80 kHz), normalized or raw, with per-frame color coding
- **Time-Domain Energy View**: FFT energy [V²] over time with adaptive threshold curve (k × Ê_floor) and noise floor overlay
- **Stage 1 Filter**: interactive k spinbox; "Above Threshold" table shows real-time candidates grouped by energy bursts
- **iFFT Window**: reconstructed time-domain signal (512 samples, 2.56 ms) with Hilbert envelope, exponential fit overlay, and full 17-feature Analyze Decay dialog
- **Data Collection Export**: batch export of Stage 1 survivors across multiple recordings as CSV (17 features + label column) and two-panel PNG screenshots for manual labeling

---

### FFT Spectrum Analysis

- **Magnitude + Phase** complex spectrum display
- **50% Conservative Normalization** for SPU0410LR5H-QB frequency response correction (±2.9 dB, 95% confidence)
- **Gibbs artifact suppression**: Tukey taper applied internally to the complex spectrum before iFFT
- **FFT Parameters**: Analysis menu shows SPR, R_spectral, FPE for the current frame (always on normalized data, matching SVM inputs)

---

### Mathematical Analysis and Automatic Fitting for Voltage Signals

A dedicated analysis module allows quantitative characterization of plant electrical signals:

#### Automatic Signal Classification
- Auto-detection of signal type: Exponential Return vs. Action Potential
- Detection criteria: peak structure, rebound ratio (≥ 30% triggers Action Potential), peak ordering, 3σ baseline threshold

#### Exponential Return Model (Variation Potential)
Fitted to the decay phase from the peak:
```
V(t) = A · exp(-(t - t₀) / τ) + V_baseline
```
Extracts: A (amplitude), τ (time constant), t₀ (peak time), V_baseline (resting potential).

#### Action Potential Model (Composite Piecewise)
```
V(t) = A_sin · sin(2πf(t-t₀) + φ)        for t < t_peak   [depolarization]
V(t) = A_exp · exp(-(t-t_peak)/τ) + Vb    for t ≥ t_peak   [repolarization]
```
Extracts 7 parameters across the depolarization and repolarization phases.

#### Fitting Engine
- `scipy.optimize.curve_fit` (Levenberg–Marquardt / Trust Region Reflective)
- Bounds derived dynamically from signal amplitude range
- Initial parameters auto-estimated from signal shape (63.2% criterion for τ)
- R² goodness-of-fit updated in real time; warning shown if R² < 0

#### Save and Export
- Named analyses saved as JSON in the footer of the `.pvoltage` file without overwriting signal data
- Multiple analyses per file, each identified by UUID
- Export to CSV: time, measured voltage, fitted curve, residuals

---

## Technology Stack

### Core Technologies
- **Python 3.8+**: application logic
- **PySide6 6.9.0**: cross-platform GUI framework (LGPL v3)
- **PyQtGraph**: high-performance real-time plotting (MIT)
- **NumPy + SciPy**: scientific computing and signal processing (BSD)

### Machine Learning (offline analysis)
- **scikit-learn 1.6.1**: SVM Pipeline training and inference (BSD)
- **joblib 1.5.3**: model serialisation / `.pkl` loading (BSD)
- **pandas 2.3.3**: CSV I/O and feature aggregation in ML scripts (BSD)
- **matplotlib 3.9.4**: offline click-distribution plots (BSD, Agg backend)

### Hardware Interface
- **PySerial**: USB CDC communication with STM32 microcontroller (BSD)
- **Custom binary protocol**: 770 bytes/frame (154 bins × 5 bytes)

### Firmware
- **STM32 HAL**: ARM Cortex-M4 microcontroller
- **CMSIS-DSP**: hardware-accelerated FFT (`arm_rfft_fast`)
- **USB CDC**: virtual COM port for data streaming

**Detailed library rationale**: see [LIBRARIES.md](App/LIBRARIES.md)

---

## Documentation

### User Guides
- **[ACQUISITION_FEATURES.md](App/ACQUISITION_FEATURES.md)**: complete guide to real-time acquisition modes
- **[ANALYSIS_FEATURES.md](App/ANALYSIS_FEATURES.md)**: advanced analysis tools and workflows

### Technical Specifications
- **[FFT_PHASE_TECHNICAL_SPECIFICATION.md](App/FFT_PHASE_TECHNICAL_SPECIFICATION.md)**: mathematical foundation of FFT/iFFT processing
- **[MICROPHONE_NORMALIZATION_TECHNICAL_REPORT.md](App/MICROPHONE_NORMALIZATION_TECHNICAL_REPORT.md)**: error analysis and validation (±2.9 dB)
- **[CLICK_DETECTION_ALGORITHM_v6.md](App/Automatic_click_detection_algorithm/CLICK_DETECTION_ALGORITHM_v6.md)**: click detection algorithm v6 (current)
- **[CLICK_DETECTION_ALGORITHM_v5.md](App/Automatic_click_detection_algorithm/CLICK_DETECTION_ALGORITHM_v5.md)**: click detection algorithm v5 (historical)
- **[CLICK_DETECTION_ALGORITHM_v4.md](App/Automatic_click_detection_algorithm/CLICK_DETECTION_ALGORITHM_v4.md)**: click detection algorithm v4 (historical)
- **[AUDIO_HARDWARE.md](Hardware/AUDIO_HARDWARE.md)**: ASEB board design and specifications
- **[VOLTAGE_HARDWARE.md](Hardware/VOLTAGE_HARDWARE.md)**: ESEB board design and specifications

### Architecture
- **[LIBRARIES.md](App/LIBRARIES.md)**: justification for all technology choices

---

## Technical Highlights

| Feature | Specification |
|---------|--------------|
| Sampling Rate | 200 kHz (audio) / 50 Hz–1 kHz (voltage) |
| FFT Size | 512 samples (radix-2 Cooley-Tukey) |
| Frequency Range | 20–80 kHz (ultrasonic) |
| Phase Quantization | 8-bit signed (−127 to +127) |
| Data Throughput | 2.4 Mbps (USB CDC) |
| File Format | Custom binary (`.paudio` / `.pvoltage`) |
| SVM Features | 7 (~26 computed, 57 exported per candidate) |
| SVM AUC-ROC | 0.929 cross-validated / 0.958 held out |
| Platform Support | Windows, macOS, Linux |

---

## Scientific Validation

### Published Research Context
- **Khait et al. (2023)**: *Sounds emitted by plants under stress are airborne and informative*. Cell, 186(7), 1328–1336.

### Error Budget
- **Amplitude accuracy**: ±2.9 dB (95% confidence) after normalization
- **Phase accuracy**: 0.41° RMS (8-bit quantization)
- **Temporal resolution**: 5 μs via iFFT peak detection
- **Frequency resolution**: 390.625 Hz/bin

### Suitable For
- Qualitative spectral analysis
- Click presence/absence detection
- Temporal pattern analysis (click rate, clustering)
- Before/after stimulus comparisons

### Not Suitable For
- Absolute SPL measurements (dB SPL)
- Quantitative energy budgets
- Cross-microphone comparisons without calibration

---

## License and Compliance

The software is licensed under the **AGPLv3 licence**.

Open-source components:
- **Python**: PSF License
- **PySide6**: LGPL v3
- **PyQtGraph**: MIT License
- **NumPy / SciPy / pandas / matplotlib**: BSD License
- **scikit-learn / joblib**: BSD License
- **PySerial**: BSD License
- **PyInstaller**: GPL (distribution exceptions apply)

Icons: **Uicons** by [Flaticon](https://www.flaticon.com/uicons) — open-source license

---

## Team

**Software & Firmware**: Tommaso Vaninetti  
**Hardware Design**: Abdoellah El Makkaoui  
**Web/Database**: Frida Tirari

**Contact**: tommasovaninetti8@gmail.com, abdoellah.elmakkaoui@gmail.com, fridatirari@gmail.com

---

## Competitions and Recognition

- **FAST i Giovani e le Scienze 2026** — Italian Finals 1st place overall
- **EUCYS — European Union Contest for Young Scientists 2026** — final in September 2026

---

## Contributing

Our mission is to make plant bioacoustics accessible to everyone. We welcome collaboration from anyone interested — open an issue on GitHub or contact us directly.

---

## Related Resources

- **Official Website**: [www.plantleaf.it](https://www.plantleaf.it)
- **Research Paper**: planned for future publication

---

**Last Updated**: September 2026  
**Project Status**: Active Development
