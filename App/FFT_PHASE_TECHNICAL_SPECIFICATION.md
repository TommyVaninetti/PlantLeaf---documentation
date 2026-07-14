# FFT and Phase Data Technical Specification

**Last Updated:** July 2026  
**Author:** Tommaso Vaninetti
**Target Audience:** Advanced users, researchers, algorithm developers

---

## Table of Contents

1. [Overview](#overview)
2. [Hardware Acquisition System](#hardware-acquisition-system)
3. [FFT Implementation Details](#fft-implementation-details)
4. [Phase Quantization and Transmission](#phase-quantization-and-transmission)
5. [Spectral Reconstruction](#spectral-reconstruction)
6. [Inverse FFT (iFFT) Process](#inverse-fft-ifft-process)

---

## 1. Overview

PlantLeaf audio acquisition system captures ultrasonic signals (20-80 kHz) using a specialized pipeline that performs **real-time FFT analysis** on embedded hardware (STM32F411CEU6) and transmits **both magnitude and phase information** to the host computer.

### Key Design Decisions

**Why FFT on Hardware?**
- **More efficient than time signal** sending 200k floats/s is impossible with USB CDC
- **Real-time processing**: 200 kHz sampling requires ~390 FFT/s (2.56 ms frames)
- **Bandwidth optimization**: Transmitting 154 bins (20-80 kHz) requires 785 bytes/frame vs 200k samples/s (800 KB/s as float32) for the raw time signal
- **Computational efficiency**: Hardware FFT accelerators (CMSIS-DSP) enable low-latency processing
- **Filtering**: sending only 20-80kHz band already removes low frequencies noise

**Why Preserve Phase Information?**
- **Time-domain reconstruction**: Enable inverse FFT for temporal analysis (click detection, decay analysis)
- **Signal fidelity**: Magnitude-only FFT loses temporal structure critical for click characterization
- **Click localization**: Sub-frame temporal resolution via iFFT peak detection

---

## 2. Hardware Acquisition System

### 2.1 Microphone Specifications

**Model:** Knowles SPU0410LR5H-QB  
**Type:** MEMS ultrasonic microphone  
**Frequency Response:** 20 Hz - 80 kHz (±3 dB)  
**Sensitivity:** -38 dBV @ 1 kHz (12.6 mV/Pa)  
**Dynamic Range:** 60 dB  
**SNR:** 64 dB(A)  

**Ultrasonic Response Characteristics** (from datasheet):

| Frequency (kHz) | Response (dB re: 1 kHz) | Notes |
|-----------------|-------------------------|-------|
| 20 | +8.0 | Resonance peak region |
| 25 | +10.5 | Maximum gain |
| 30 | +6.0 | Transition zone |
| 40 | -2.0 | Flat region start |
| 50 | -6.0 | Roll-off begins |
| 60 | -7.0 | Maximum attenuation |
| 70 | -6.0 | Partial recovery |
| 80 | -4.0 | Upper limit |

**Key Observations:**
- **Resonance peak** at 25 kHz (+10.5 dB): Natural amplification of low-ultrasonic signals
- **Attenuation valley** at 60 kHz (-7 dB): Mechanical response limitation
- **Non-flat response**: ±9 dB variation across 20-80 kHz band
- **Measurement uncertainty**: ±0.5 dB (manual datasheet reading)

**The problem was addressed with a conservative normalization. All the information about it are available here: [MICROPHONE_NORMALIZATION_TECHNICAL_REPORT](MICROPHONE_NORMALIZATION_TECHNICAL_REPORT.md) **

### 2.2 ADC Configuration

**Resolution:** 12-bit (4096 levels)  
**Sampling Rate (fs):** 200,000 Hz (200 ksps)  
**Voltage Range:** 0 - 3.3 V  (offset at 1.65V)
**Quantization Step:** 3.3V / 4096 = 0.8057 mV  

## 3. FFT Implementation Details

### 3.1 FFT Parameters

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| FFT Size (N) | 512 samples | Power-of-2 for radix-2 algorithm |
| Window Duration | 2.56 ms | N/fs = 512/200000 |
| Frame Rate | 390.625 FPS | fs/N = 200000/512 |
| Frequency Resolution | 390.625 Hz/bin | fs/N |
| Total Bins | 256 | N/2 (real FFT) |
| Transmitted Bins | 154 | Bins 51-204 (20-80 kHz) |

### 3.2 Windowing Function

**Type:** ⚠️ **None (Rectangular Window)**

**Current Implementation:**
The firmware does **NOT** apply any windowing function before FFT. Samples are converted directly from ADC values to voltage and fed to the FFT algorithm:

```c
// Firmware code (main_with_phase.c)
for (int i = 0; i < FFT_BUFFER_SIZE; i++) {
    fftBufIn[i] = (float)(adc_buffer[i]) * ADC_TO_VOLTAGE_FACTOR - ADC_OFFSET;
}
arm_rfft_fast_f32(&fftHandler, fftBufIn, fftBufOut, 0);  // No windowing
```

**Rectangular Window Properties:**
- **Mainlobe Width:** 4π/N (1 bin) - narrowest possible
- **First Sidelobe:** -13 dB (poor)
- **Sidelobe Roll-off:** -6 dB/octave (slow)
- **Scalloping Loss:** 3.92 dB (worst case between bins)

**Consequences:**
- **Best frequency resolution**: 390.625 Hz/bin (no mainlobe widening)
- **Best temporal localization**: 2.56 ms (no envelope spreading)
- **High spectral leakage**: -13 dB sidelobes cause strong inter-bin interference
- **High scalloping loss**: Up to 3.92 dB amplitude error between bins

**Why No Window Currently?**
- **Click detection**: Click events are typically broadband (low spectral resolution needed)
- **Temporal precision**: Rectangular window preserves sharp temporal boundaries

**Time-Frequency Trade-off:**
```
Δt · Δf ≥ 1/(4π)    (Uncertainty Principle)

With rectangular window:
Δt ≈ 2.56 ms (optimal - no spreading)
Δf ≈ 390.625 Hz (optimal - single bin)
Δt · Δf ≈ 1.0 > 1/(4π) ≈ 0.08  ✓
```

### 3.3 FFT Algorithm

**Implementation:** CMSIS-DSP Library `arm_rfft_fast_f32()`

**Algorithm:** Radix-2 Cooley-Tukey Decimation-in-Time (DIT)

**Computational Complexity:**
```
Multiplications: (N/2) · log₂(N) = 256 · 9 = 2304 ops
Additions: N · log₂(N) = 512 · 9 = 4608 ops
Total: ~7000 FLOPs per FFT
```

### 3.4 FFT Output Format

**Complex Spectrum:**
```
X[k] = Re[k] + j·Im[k]    for k ∈ [0, N/2]

Magnitude: A[k] = √(Re[k]² + Im[k]²)
Phase:     φ[k] = atan2(Im[k], Re[k]) made with a look-up table (see 4.2)
```

**Frequency Mapping:**
```
f[k] = k · (fs / N) = k · 390.625 Hz

Bin 0:   0 Hz (DC)
Bin 51:  19,921.875 Hz ≈ 20 kHz
Bin 204: 79,687.5 Hz ≈ 80 kHz
Bin 256: 100 kHz (Nyquist)
```

---

## 4. Phase Quantization and Transmission

### 4.1 Phase Representation Challenge

**Problem:** Phase values φ[k] ∈ [-π, π] are continuous (floating-point)

**Solution:** Quantize to 8-bit signed integer for efficient transmission

### 4.2 Quantization Formula

The phase scale is **128 counts per π radians**, end to end: firmware encoder,
wire format and host decoder all use it.

**Encoding (firmware, `fast_atan2.c`):**

The firmware never calls a floating-point `atan2`. It computes the phase
directly on the quantized scale with an octant-folded lookup table:

1. `(imag, real)` is folded into the first octant via absolute values and a
   `min/max` ratio ∈ [0, 1], quantized (round-to-nearest) to an index i ∈ [0, 255].
2. The LUT returns the first-octant angle in counts:
   ```
   lut[i] = round( atan(i / 255) · 128 / π )        # 0..32 counts = 0..45°
   ```
3. Octant and quadrant reconstruction on the same scale: 90° = 64 counts,
   quadrant II = `128 − α`, quadrant III = `−128 + α`, quadrant IV = `−α`.
   The value +128 does not exist in int8; it is exactly +180°, which this
   scale represents as −128 (the two describe the same angle).

The closed-form equivalent of the encoder is:
```c
int8_t phase_quantized = (int8_t)round((phase_rad / π) * 128.0);   // +π wraps to -128

Range: [-128, +127] maps to [-π, +π)
```

**Decoding (Host, `click_pipeline_v5.py`):**
```python
phase_rad = (phase_int8 / 128.0) * π
```

**Quantization Step:**
```
Δφ = 2π / 256 ≈ 0.0245 rad ≈ 1.41°
```

### 4.3 Phase Quantization Error

**Error budget:**
```
Output quantization:  ±Δφ / 2 = π / 256 ≈ 0.70°
LUT index rounding:   ≤ ~0.11°
```

**Measured** (dense sweep of the encoder+decoder pair against float64 `atan2`):
**0.81° worst case, 0.35° mean**, zero systematic bias in any quadrant. On a
synthetic ultrasonic click (50 kHz carrier, τ = 0.3 ms exponential decay), the
phase encoding contributes ~0.5% RMS-normalized waveform error and −0.2%
envelope-peak error after reconstruction; the envelope peak position is
unaffected. The error is at the floor set by 8-bit phase encoding — reducing
it further would require widening the phase field in the wire format.

### 4.4 Data Transmission Format

**Wire format — one frame per FFT (little-endian):**

| Field | Type | Bytes | Content |
|-------|------|-------|---------|
| Sync | uint8 × 2 | 2 | `0xAA 0x55` |
| Payload length | uint16 | 2 | Always **781** |
| Max amplitude | float32 | 4 | Peak magnitude in the band [V] |
| Peak bin | uint16 | 2 | Index of the peak, **relative to bin 51** (0–153) |
| Above threshold | uint8 | 1 | 1 if max amplitude > threshold |
| Threshold | float32 | 4 | Active detection threshold [V] |
| Magnitudes | float32 × 154 | 616 | Bins 51–204, amplitude-normalized [V] |
| Phases | int8 × 154 | 154 | Bins 51–204, 128 counts per π, −128 to +127 |
| **Frame total** | | **785** | header 4 + payload 781 |

**Framing invariant:** the payload length field is always 781 for this
protocol version. The host reader resynchronizes on the byte stream by
scanning for `0xAA 0x55` and **accepts a candidate header only if its length
field equals 781** — the sync pair can legitimately occur inside float32
payload data, so the length check is what makes recovery from a truncated or
corrupted frame deterministic (within ~2 frames). Frames whose decoded
content is invalid (non-finite floats, peak bin ≥ 154) are dropped.

**Bandwidth Requirement:**
```
Data rate = 785 bytes/frame × 390.625 FPS = 306.6 KB/s
Bit rate ≈ 2.45 Mbps

USB 2.0 Full Speed: 12 Mbps available
Utilization: 2.45 / 12 ≈ 20% ✓
```

---

## 5. Spectral Reconstruction

### 5.1 Received Spectrum (20-80 kHz)

**Transmitted Data:**
- **Bins:** 51 to 204 (154 bins)
- **Frequency Range:** 19.92 kHz to 79.69 kHz
- **Magnitude:** A[51:204] (154 float32 values)
- **Phase:** φ[51:204] (154 int8 values)

### 5.2 Full Spectrum Reconstruction (0-100 kHz)

**Objective:** Reconstruct 256-bin complex spectrum for inverse FFT

**Process:**

1. **Initialize Full Spectrum (256 bins):**
   ```python
   full_magnitude = np.zeros(256, dtype=np.float32)
   full_phase = np.zeros(256, dtype=np.int8)
   ```

2. **Insert Received Data:**
   ```python
   full_magnitude[51:205] = received_magnitude[:]  # 154 bins
   full_phase[51:205] = received_phase[:]
   ```

3. **Zero-Padding Regions:**
   - **DC to 20 kHz:** Bins 0-50 = 0 (environmental noise filtered)
   - **80 kHz to Nyquist:** Bins 205-256 = 0 (above microphone response)

4. **Complex Spectrum Construction:**
   ```python
   phase_rad = (full_phase / 128.0) * π      # 128 counts per π — see §4.2
   complex_spectrum = full_magnitude * exp(j * phase_rad)
   complex_spectrum *= (fft_size / 2)        # restore raw-FFT scale — see §6.1
   ```

### 5.3 Spectral Discontinuities (Gibbs Phenomenon)

**Problem:** Abrupt transitions at bin 51 and bin 204 cause temporal artifacts

**Mathematical Basis:**

The Fourier transform of a rectangular window is:
```
W(f) = sin(πfT) / (πf)    (sinc function)
```

Spectral discontinuities introduce **Gibbs phenomenon** in time domain:
- **Overshoot:** ~9% of step height
- **Ringing:** Decays as 1/t
- **Duration:** ~2-3 oscillations per transition
Gibbs suppression is handled by [CLICK_DETECTION_ALGORITHM_v5](Automatic_click_detection_algorithm/CLICK_DETECTION_ALGORITHM_v5.md) and descibed in section 7 below

**Example:**

If spectral edge has magnitude jump from 0 to A:
```
Overshoot amplitude ≈ 0.09 × A
First zero-crossing ≈ 1/(2Δf) ≈ 1.3 ms from edge
```

---

## 6. Inverse FFT (iFFT) Process

### 6.1 IFFT Mathematical Definition

**Discrete Inverse Fourier Transform:**
```
x[n] = (1/N) · Σ(k=0 to N-1) X[k] · exp(j·2π·k·n/N)

For real signals (using rfft/irfft), where X[k] are RAW (unnormalized) coefficients:
x[n] = (2/N) · Σ(k=0 to N/2) |X[k]| · cos(2π·k·n/N + φ[k])
```

**Implementation:**
```python
complex_spectrum *= (fft_size / 2)                          # restore raw-FFT scale
time_domain_signal = np.fft.irfft(complex_spectrum, n=512)
```

**Output:**
- **Samples:** 512 points
- **Duration:** 2.56 ms
- **Sampling Rate:** 200 kHz
- **Amplitude:** Volts (ADC scale)

### 6.2 Temporal Resolution

**Frame Boundaries:**
```
Frame i start time: t_i = i × (N/fs) = i × 2.56 ms
Frame i end time:   t_i + 2.56 ms
```

**Sub-Frame Localization:**

Using Hilbert envelope peak detection:
```
Peak position: t_peak = t_i + (n_peak / fs)
Resolution: Δt = 1/fs = 5 μs (single sample)
```

**Localization Accuracy:**
- **Frame-level:** 2.56 ms (FFT duration)
- **Sub-frame (iFFT):** 5 μs (sample-level)
- **Improvement:** 512× better temporal resolution

## 7. Windowing and Artifact Mitigation

### 7.1 Tukey Window (Cosine Taper)

**Purpose:** Eliminate Gibbs phenomenon at spectral edges (bins 51 and 204)

**Mathematical Definition:**

For a spectrum of length L with taper length α·L:

```
        ⎧ 0.5 · (1 - cos(π·n/α·L))           for n < α·L
w[n] = ⎨ 1.0                                 for α·L ≤ n < L-α·L
        ⎩ 0.5 · (1 - cos(π·(L-n-1)/α·L))     for n ≥ L-α·L
```

**PlantLeaf Implementation (α = 10%):**

For 154 bins (20-80 kHz):
```python
taper_bins = max(5, 154 // 10) = 15 bins

# Left taper (bins 51-65, 20-25.5 kHz)
for i in range(15):
    alpha = i / 15
    window[51 + i] = 0.5 * (1 - cos(π * alpha))

# Center plateau (bins 66-189, 25.5-73.8 kHz)
window[66:190] = 1.0

# Right taper (bins 190-204, 73.8-80 kHz)
for i in range(15):
    alpha = i / 15
    window[204 - i] = 0.5 * (1 - cos(π * alpha))
```

### 7.2 Window Properties

**Minimum Gain:** 0.5 (-6 dB) at edges (bins 51, 204)

**Transition Width:** 15 bins ≈ 5.86 kHz

**Energy Loss:**
```
Theoretical: 10% of edge bins
Practical: (2 × 15 bins) / 154 bins × 50% ≈ 9.7% energy reduction
```

**Artifact Reduction:**

| Metric | Without Window | With Tukey (10%) | Improvement |
|--------|----------------|------------------|-------------|
| Edge Overshoot | 9% of step | <1% of step | ~9× reduction |
| Ringing Amplitude | -20 dB | -40 dB | +20 dB SNR |
| Temporal Artifacts | Visible | Negligible | 10× cleaner |

---

## 8. References

1. **CMSIS-DSP Library Documentation**, ARM Ltd., v1.10.0
2. **Knowles SPU0410LR5H-QB Datasheet**, Rev. H, 2023
3. **Oppenheim, A. V., & Schafer, R. W.** (2010). *Discrete-Time Signal Processing*, 3rd ed., Prentice Hall
4. **Harris, F. J.** (1978). "On the Use of Windows for Harmonic Analysis with the Discrete Fourier Transform", *Proceedings of the IEEE*, 66(1), 51-83
5. **Nuttall, A. H.** (1981). "Some Windows with Very Good Sidelobe Behavior", *IEEE Transactions on Acoustics, Speech, and Signal Processing*, 29(1), 84-91
6. **Smith, J. O.** (2011). *Spectral Audio Signal Processing*, W3K Publishing, https://ccrma.stanford.edu/~jos/sasp/

---

**Document Revision History:**

**For questions or corrections, please contact Tommaso Vaninetti**
