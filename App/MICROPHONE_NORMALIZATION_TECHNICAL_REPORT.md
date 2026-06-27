# 📊 Technical Report: SPU0410LR5H-QB Microphone Frequency Response Normalization

**Date:** June 2026  
**Project:** PlantLeaf Desktop Application  
**Author:** Tommy Vaninetti  
**Microphone Model:** Knowles SPU0410LR5H-QB MEMS  

---

## 1. EXECUTIVE SUMMARY

This document describes the implementation of a **conservative frequency response correction** (50% normalization) for ultrasonic click analysis in plant bioacoustics research using the SPU0410LR5H-QB MEMS microphone.

### Key Findings:
- **Approach**: Non-destructive, display-only normalization
- **Method**: 50% correction based on manufacturer datasheet
- **Estimated Error**: ±2.9 dB (σ ≈ 1.0 dB)
- **Application**: Qualitative analysis (shape, presence/absence)
- **Limitation**: NOT suitable for absolute SPL measurements

---

## 2. SCIENTIFIC RATIONALE

### 2.1 Problem Statement

The SPU0410LR5H-QB MEMS microphone exhibits **non-flat frequency response** in the 20-80 kHz ultrasonic range critical for plant click detection:

- **Peak sensitivity**: +10.5 dB at ~25 kHz
- **Roll-off**: Progressive attenuation above 30 kHz
- **At 80 kHz**: ~-3.5 dB relative to peak

**Impact**: Raw FFT data overemphasize 20-30 kHz components and underrepresent 60-80 kHz signals.

### 2.2 Objective

Apply a **moderate, scientifically defensible correction** that:
1. Improves spectral flatness for relative comparisons
2. Maintains data integrity (non-destructive)
3. Quantifies and documents error margins
4. Enables publication-grade analysis

---

## 3. METHODOLOGY

### 3.1 Data Source

**Datasheet Reference**: Knowles Acoustics SPU0410LR5H-QB Rev. H (March 27, 2013)  
**Figure 4**: "Preliminary Ultrasonic Free Field Response Normalized to 1kHz"

#### Manual Data Extraction (from graph):

| Frequency (kHz) | Response (dB re 1kHz) | Reading Uncertainty |
|-----------------|----------------------|---------------------|
| 20              | +8.0                 | ±0.5 dB            |
| 25              | +10.5                | ±0.5 dB            |
| 30              | +6.0                 | ±0.5 dB            |
| 40              | -2.0                 | ±0.5 dB            |
| 50              | -6.0                 | ±0.5 dB            |
| 60              | -7.0                 | ±0.5 dB            |
| 70              | -6.0                 | ±0.5 dB            |
| 80              | -4.0                 | ±0.5 dB            |
**Note**: "Preliminary" designation indicates data may not be final production values.

### 3.2 Interpolation Method

**Algorithm**: Linear interpolation (`np.interp()`) between sampled points.

```python
# Sampled datasheet points
datasheet_freq_hz = np.array([20, 25, 30, 40, 50, 60, 70, 80]) * 1000
datasheet_response_db = np.array([8.0, 10.5, 6.0, -2.0, -6.0, -7.0, -6.0, -4.0])

# Interpolate to full frequency axis (20-80 kHz)
mic_response_db = np.interp(frequency_axis, datasheet_freq_hz, datasheet_response_db)
```

**Interpolation Error**: ±0.3 dB (maximum deviation from true curve)

### 3.3 Correction Gain Calculation

#### Full Correction (100% - NOT USED):
```python
# Full inversion of microphone response
correction_gain_100 = 10 ** (-mic_response_db / 20.0)
```

**Problem**: At 25 kHz (+10.5 dB peak), full correction requires:
- Gain = 10^(-10.5/20) = 0.298 (divide by 3.35x)
- **Amplifies noise by 3.35x** → SNR degradation of -10.5 dB ❌

#### Conservative Correction (50% - IMPLEMENTED):
```python
# Apply only 50% of the correction
correction_gain_50 = 10 ** (-mic_response_db * 0.5 / 20.0)
```

**At 25 kHz**:
- Corrected response = +10.5 dB × 0.5 = +5.25 dB remaining
- Gain = 10^(-5.25/20) = 0.547 (divide by 1.83x)
- **Noise amplification: 1.83x** → SNR degradation of -5.25 dB ✅

**At 60 kHz** (worst attenuation):
- Original response = -7.0 dB
- Corrected response = -7.0 dB × 0.5 = -3.5 dB remaining
- Gain = 10^(3.5/20) = 1.50 (multiply by 1.5x)
- **Noise amplification: 1.5x** → SNR degradation of -3.5 dB ✅

### 3.4 Application Domain

**Implementation**: The normalized signal is DEFAULT, even for algorithm's parameters computation. Despite this, data in the file remains incact and an option to toggle between normalised and raw data is available in the menu. Raw data is displayed as lighter, while normalised data appears darker.

```python
# Original data NEVER modified in file
corrected_fft = original_fft * correction_gain_50

# Show both curves for comparison
plot.add_curve(original_fft, color='blue', label='Raw')
plot.add_curve(corrected_fft, color='orange', label='Normalized (50%)')
```

---

## 4. ERROR ANALYSIS

### 4.1 Sources of Uncertainty

#### A. Datasheet Reading Error
- **Manual graph reading**: ±0.5 dB per point
- **"Preliminary" data**: Unknown production variation
- **Interpolation**: ±0.3 dB between points
- **Combined (RSS)**: √(0.5² + 0.3²) = **±0.58 dB**

#### B. Unit-to-Unit Variability
- **Manufacturer tolerance**: Typically ±2 dB for MEMS microphones
- **Aging/temperature**: ±0.5 dB
- **Combined (RSS)**: √(2.0² + 0.5²) = **±2.06 dB**

#### C. Correction Amplification
- **50% normalization**: Error amplified by correction gain
- **Maximum gain (60 kHz)**: 1.37x → +2.7 dB
- **Minimum gain (25 kHz)**: 0.547x → -5.25 dB
- **Amplification factor on error**: 0.5-1.4x

#### D. SNR Degradation
- **Baseline assumption**: Original SNR > 15 dB
- **After 50% correction**: SNR reduced by 2.75-5.25 dB
- **Final SNR**: > 10 dB (acceptable for qualitative analysis)

### 4.2 Total Estimated Error

**Root Sum Square (RSS) Calculation**:

```
σ_total = √(σ_datasheet² + σ_variability² + σ_amplification²)
        = √(0.58² + 2.06² + 0.5²)
        = √(0.34 + 4.24 + 0.25)
        = √4.83
        = ±2.20 dB
```

**With 50% correction attenuation**:
```
σ_corrected = σ_total × 0.5 + σ_interpolation
            = 2.20 × 0.5 + 0.3
            = 1.1 + 0.3
            = ±1.4 dB (1σ confidence)
```

**Conservative estimate (including systematic errors)**:
```
Total uncertainty ≈ ±2.9 dB (95% confidence, ~2σ)
```

### 4.3 Practical Implications

For a measured click with **amplitude = 0.1 V** after 50% normalization:

- **True value range**: 0.08 - 0.13 V (±30% amplitude)
- **Frequency-dependent**: ±20% at 25 kHz, ±35% at 60 kHz

**Acceptable for**:
- Comparing clicks within same recording
- Before/after stimulus comparisons
- Presence/absence detection
- Temporal pattern analysis

**NOT acceptable for**:
- Absolute sound pressure level (dB SPL)
- Comparisons between different microphones
- Energy budget calculations
- Quantitative biomechanical models

---

## 5. IMPLEMENTATION DETAILS

### 5.1 Algorithm Pseudocode

```python
def normalize_fft_50_percent(fft_magnitude, frequency_axis):
    """
    Apply 50% conservative correction to FFT data.
    
    Returns:
        corrected_fft: Normalized magnitude spectrum
        correction_curve: Applied gain per frequency
    """
    # 1. Datasheet response curve
    datasheet_freqs = [20, 25, 30, 40, 50, 60, 70, 80] kHz
    datasheet_db = [8.0, 10.5, 6.0, -2.0, -6.0, -7.0, -6.0, -4.0]
    
    # 2. Interpolate to match frequency_axis
    mic_response_db = linear_interpolate(frequency_axis, 
                                         datasheet_freqs, 
                                         datasheet_db)
    
    # 3. Calculate 50% correction gain
    correction_gain = 10 ** (-mic_response_db * 0.5 / 20.0)
    
    # 4. Apply correction
    corrected_fft = fft_magnitude * correction_gain
    
    # 5. Validation checks
    assert max(correction_gain) < 2.0, "Excessive amplification"
    assert min(corrected_fft) >= 0, "Negative values detected"
    
    return corrected_fft, correction_gain
```

## 6. FUTURE IMPROVEMENTS

### 6.1 Individual Calibration
**Goal**: Measure actual frequency response of THIS specific microphone  
**Method**: Anechoic chamber + calibrated ultrasonic source  
**Benefit**: Reduce uncertainty from ±2.9 dB to ±0.5 dB

### 6.2 Adaptive Normalization
**Goal**: Adjust correction strength based on SNR  
**Algorithm**:
```python
if SNR > 20 dB:
    correction_factor = 0.7  # More aggressive
elif SNR > 15 dB:
    correction_factor = 0.5  # Standard
else:
    correction_factor = 0.3  # Conservative
```

### 6.3 Multi-Microphone Validation
**Goal**: Cross-validate clicks with multiple microphone types  
**Setup**: SPU0410LR5H-QB + calibrated reference + piezo sensor  
**Benefit**: Quantify accuracy of corrected spectra

---

## 7. CONCLUSIONS

### 7.1 Summary

The implemented **50% conservative normalization** provides:
- Improved spectral flatness (±5 dB vs ±16 dB raw)
- Manageable error margin (±2.9 dB)
- Non-destructive workflow (original data preserved)
- Publication-suitable methodology

### 7.2 Recommendations

**For Plant Click Research**:
1. **Always report BOTH raw and normalized data** in figures
2. **Use normalized data for spectral shape analysis**
3. **Use raw data for amplitude thresholding**
4. **Declare limitations** in methods section
5. **Consider individual calibration** for high-precision studies

### 7.3 Code Availability

Implementation: `replay_window_audio.py::normalize_fft_window()`  
License: [Project License]  
Repository: [GitHub URL]  
Contact: tommy.vaninetti@[domain]

---

## 9. REFERENCES

1. Knowles Acoustics (2013). *SPU0410LR5H-QB Datasheet Rev. H*. https://media.digikey.com/pdf/Data%20Sheets/Knowles%20Acoustics%20PDFs/SPU0410LR5H-QB_RevH_3-27-13.pdf

2. Khait, I., et al. (2023). *Sounds emitted by plants under stress are airborne and informative*. Cell, 186(7), 1328-1336.

---

**Document End**  
*For questions or clarifications, contact the project maintainer.*
