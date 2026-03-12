# 🌿 PlantLeaf Desktop Application

[![Version](https://img.shields.io/badge/version-1.0.0--beta-blue.svg)](https://github.com/TommyVaninetti/PlantLeaf-Desktop-App)
[![Python](https://img.shields.io/badge/python-3.8+-blue.svg)](https://www.python.org/)
[![License](https://img.shields.io/badge/license-Custom-green.svg)](LICENSE.rtf)
[![PySide6](https://img.shields.io/badge/PySide6-6.9.0-green.svg)](https://doc.qt.io/qtforpython/)

**PlantLeaf** is a powerful desktop application specifically designed for **plant bioacoustics and bioelectric research**, enabling real-time acquisition and analysis of ultrasonic click events and voltage signals (action potentials) in plants. Developed for the **FAST i Giovani e le Scienze 2026** competition.

---

## 🎯 Project Overview

PlantLeaf bridges the gap between rigorous scientific analysis and creative exploration, providing:

- **Ultrasonic Click Detection**: Capture and analyze plant-emitted ultrasonic clicks (20-80 kHz) with sub-millisecond temporal resolution
- **Action Potential Monitoring**: High-precision voltage acquisition for detecting plant electrical signals
- **Real-Time Acquisition**: Live monitoring with FFT spectrum visualization (390.625 FPS) for audio and up to 1k sampling rate for bioelectric signals
- **Advanced Analysis Tools**: Phase-preserving FFT, inverse FFT reconstruction, click detection algorithms, microphone normalization, automatic fitting for action potential
- **Cheap Instrumentation**: Custom hardware designed to be both cheap and scientifically rigorous 

### 🔬 Scientific Capabilities

- **Frequency Range**: 20-80 kHz (ultrasonic band optimized for SPU0410LR5H-QB microphone)
- **Sampling Rate**: 200 kHz (Audio) / 1-10 kHz (Voltage)
- **FFT Resolution**: 512 samples, 390.625 Hz/bin
- **Phase Preservation**: 8-bit quantized phase data for iFFT reconstruction
- **Temporal Resolution**: 2.56 ms frame duration, 5 μs sub-frame localization
- **Dynamic Range**: 12-bit ADC (74 dB SNR)

---

## 📂 Repository Structure

This repository focuses on:
- **software and firmware** developed by **Tommaso Vaninetti**
- **hardware** developed bu **Abdoellah El Makkaoui**
and wants to provide a deep and open understanding of the process

## 🚀 Key Features

### 📡 Real-Time Acquisition

#### Audio Mode (Ultrasonic Clicks)
- **512-sample FFT** at 390 FFTs per second for real-time spectral analysis
- **Live spectrum visualization** (20-80 kHz bandpass)
- **Phase data preservation** for inverse FFT reconstruction
- **Automatic click detection** with adjustable thresholds
- **Long-duration recording** (hours) with minimal memory footprint

#### Voltage Mode (Action Potentials)
- **High-precision ADC** (12-bit, 0-3.3V range)
- **Variable sampling rates** (up to 1 kHz)
- **Low-pass filtering and notch**
- **Event annotation** with timestamps
- **CSV export** for external analysis

### 🔬 Advanced Analysis Tools

**ULTRASONIC CLICKS**

#### FFT Spectrum Analysis
- **Magnitude + Phase** complex spectrum display
- **50% Conservative Normalization** for SPU0410LR5H-QB microphone response correction
- **Frequency-domain filtering** (Tukey window, 10% taper)
- **Spectral energy plots** with logarithmic scale

#### Inverse FFT (iFFT) Reconstruction
- **Time-domain reconstruction** from complex spectrum (magnitude + phase)
- **Sub-frame temporal localization** (5 μs resolution)
- **Gibbs artifact mitigation** using Tukey windowing on complex spectrum
- **Dual-view comparison**: Raw vs. Normalized waveforms

#### Click Detection Algorithm (still in active development)
- **Multi-parameter detection**: Amplitude, duration, frequency content
- **Adaptive thresholding** based on background noise estimation
- **False-positive suppression** using spectral coherence analysis
- **Batch processing** for entire recordings

**BIOELECTRIC REACTION**

### Mathematical Analysis & Automatic Fitting for Voltage Signals

A dedicated analysis module allows quantitative characterization of plant electrical signals:

#### Automatic Signal Classification
- **Auto-detection** of signal type from shape: Exponential Return vs. Action Potential
- Based on peak structure, rebound ratio, and temporal ordering of events
- 3σ baseline noise threshold for robust detection

#### Exponential Return Model (Variation Potential)
Fitted to the decay phase from the peak:
```
V(t) = A · exp(-(t - t₀) / τ) + V_baseline
```
- Extracts: amplitude `A`, time constant `τ`, peak time `t₀`, baseline `V_baseline`
- Physical meaning: membrane relaxation after mechanical/hydraulic stimulus

#### Action Potential Model (Composite Piecewise)
A physiologically motivated two-phase model:
```
V(t) = A_sin · sin(2πf(t-t₀) + φ)        for t < t_peak   [depolarization]
V(t) = A_exp · exp(-(t-t_peak)/τ) + Vb    for t ≥ t_peak   [repolarization]
```
- Extracts: 7 parameters (amplitude, frequency, phase, time constant, baseline)
- Physical meaning: fast oscillatory depolarization + exponential repolarization

#### Fitting Engine
- **Library**: `scipy.optimize.curve_fit` (Levenberg-Marquardt / Trust Region Reflective)
- **Bounds**: Dynamically derived from signal amplitude range
- **Initial guess**: Auto-estimated from signal shape (63.2% criterion for τ)
- **R² coefficient**: Goodness-of-fit metric, displayed in real-time
- **Manual tuning**: All parameters adjustable via spinboxes with live curve update

#### Signal Energy
- Numerical integration of `(V - V_baseline)²` over event duration (trapezoidal rule)
- Reported per phase (rising, decay) and total

#### Save & Export
- Named analyses saved within the `.pvoltage` file
- Export to CSV (time, measured voltage, fitted curve, residuals)
- Full parameter report for publication

---

## 🛠️ Technology Stack

### Core Technologies
- **Python 3.8+**: Application logic
- **PySide6 (Qt 6.9)**: Cross-platform GUI framework
- **PyQtGraph**: High-performance real-time plotting
- **NumPy + SciPy**: Scientific computing and signal processing

### Hardware Interface
- **PySerial**: USB CDC communication with STM32 microcontroller
- **Custom binary protocol**: 770 bytes/frame (154 bins × 5 bytes)

### Firmware
- **STM32 HAL**: ARM Cortex-M4 microcontroller
- **CMSIS-DSP**: Hardware-accelerated FFT (`arm_rfft_fast_f32`)
- **USB CDC**: Virtual COM port for data streaming

📖 **Detailed library rationale**: See [LIBRARIES.md](app/LIBRARIES.md)

---

## 📚 Documentation

**📖 [Complete Documentation Index](app/DOCUMENTATION_INDEX.md)** - Start here for organized access to all docs

### User Guides
- **[NORMALIZATION_USER_GUIDE.md](app/NORMALIZATION_USER_GUIDE.md)**: How to use the microphone normalization feature
- **[ACQUISITION_FEATURES.md](app/ACQUISITION_FEATURES.md)**: Complete guide to real-time acquisition modes
- **[ANALYSIS_FEATURES.md](app/ANALYSIS_FEATURES.md)**: Advanced analysis tools and workflows

### Technical Specifications
- **[FFT_PHASE_TECHNICAL_SPECIFICATION.md](app/FFT_PHASE_TECHNICAL_SPECIFICATION.md)**: Mathematical foundation of FFT/iFFT processing
- **[MICROPHONE_NORMALIZATION_TECHNICAL_REPORT.md](app/MICROPHONE_NORMALIZATION_TECHNICAL_REPORT.md)**: Error analysis and validation (±2.9 dB accuracy)
- **[CLICK_DETECTION_ALGORITHM_MATHEMATICAL_FRAMEWORK.md](app/CLICK_DETECTION_ALGORITHM_MATHEMATICAL_FRAMEWORK.md)**: Detection algorithm theory

### Architecture
- **[LIBRARIES.md](app/LIBRARIES.md)**: Justification for technology choices

---

## 🔬 Scientific Validation

### Published Research Context
This tool was developed following methodologies from:
- **Khait et al. (2023)**: *Sounds emitted by plants under stress are airborne and informative*. Cell, 186(7), 1328-1336.
- **Baudin et al. (2024)**: *Plant bioacoustics: methods and applications*. Journal of Plant Physiology.

### Error Budget
- **Amplitude accuracy**: ±2.9 dB (95% confidence) after normalization
- **Phase accuracy**: 0.41° RMS (8-bit quantization)
- **Temporal resolution**: 5 μs (sample-level) via iFFT peak detection
- **Frequency resolution**: 390.625 Hz/bin

### Suitable For
- ✅ Qualitative spectral analysis
- ✅ Click presence/absence detection
- ✅ Temporal pattern analysis
- ✅ Before/after stimulus comparisons

### NOT Suitable For
- ❌ Absolute SPL measurements (dB SPL)
- ❌ Quantitative energy budgets
- ❌ Cross-microphone comparisons without calibration

---

## 📄 License & Compliance

### Application License
NOT IMPLEMENTED YET

### Open-Source Components
This application uses the following open-source libraries:
- **Python**: PSF License
- **PySide6**: LGPL v3 (see [LGPL Compliance Notice](README.txt))
- **PyQtGraph**: MIT License
- **PySerial**: BSD License
- **NumPy/SciPy**: BSD License
- **PyInstaller**: GPL (distribution exceptions apply)

Complete license texts: [licenses.txt](licenses.txt)

### Icons
**Uicons** by [Flaticon](https://www.flaticon.com/uicons) - Open-source license

---

## 👥 Team

**Software & Firmware**: Tommaso Vaninetti  
**Hardware Design**: Abdoellah El Makkoui
**Web/Database**: Frida Tirari

**Contact**: tommasovaninetti8@gmail.com

---

## 🏆 Competitions & Recognition

- **FAST i Giovani e le Scienze 2026** - Italian Finals Qualifier
- Target: European Union Contest for Young Scientists (EUCYS)

---

## 🤝 Contributing

This is an academic research project developed for scientific competitions.

For questions or collaboration inquiries, please open an issue or contact the maintainer.

---

## 🔗 Related Resources

- **Official Website**: www.plantleaf.it
- **Research Paper**: -

---

## 📊 Technical Highlights

| Feature | Specification |
|---------|--------------|
| Sampling Rate | 200 kHz (Audio) / 1-10 kHz (Voltage) |
| FFT Size | 512 samples (radix-2 Cooley-Tukey) |
| Frequency Range | 20-80 kHz (ultrasonic) |
| Phase Quantization | 8-bit signed (-127 to +127) |
| Data Throughput | 2.4 Mbps (USB CDC) |
| File Format | Custom binary (`.paudio` / `.pvoltage`) |
| Platform Support | Windows, macOS, Linux (Python-based) |

---

**Last Updated**: March 11, 2026  
**Version**: 1.0.0-beta  
**Project Status**: 🟢 Active Development
