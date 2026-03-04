# 🎯 User Guide: FFT Normalization Feature

## Quick Start

### What is Normalization?

The **50% Conservative Normalization** corrects for the non-flat frequency response of the SPU0410LR5H-QB microphone used in ultrasonic plant click detection (20-80 kHz range).

**Why do you need it?**
- The microphone has peak sensitivity at ~25 kHz (+10 dB)
- It attenuates frequencies above 30 kHz (up to -5.5 dB at 60 kHz)
- This makes raw spectra **biased** toward low frequencies

**What does it fix?**
- ✅ Flatter spectral representation
- ✅ Better comparison between different frequency bands
- ✅ More accurate click shape visualization

---

## How to Use

### 1. FFT Spectrum Normalization

**Location**: Analysis Menu → "Normalize FFT Window" (or keyboard shortcut)

**Steps**:
1. Open a `.paudio` file in Audio Replay mode
2. Navigate to the click/frame of interest
3. Click **Analysis → Normalize FFT Window**
4. Two curves will appear:
   - **Cyan**: Raw data (original)
   - **Orange**: Normalized data (50% corrected)

**Visual Example**:
```
Before:                    After (normalized):
Peak at 25 kHz ↑          More balanced spectrum
Roll-off at 60 kHz ↓      Flatter response
```

### 2. iFFT Signal Normalization

**Location**: iFFT Window → "Apply 50% Normalization" button

**Steps**:
1. Open iFFT window (Analysis → iFFT Graph)
2. Click **"Apply 50% Normalization"** button
3. View overlay:
   - **Orange (solid)**: Normalized iFFT
   - **Cyan (dashed)**: Raw iFFT
4. Toggle back with **"Show Raw iFFT"**

---

## When to Use Normalization

### ✅ RECOMMENDED FOR:
- **Comparing frequency content** between clicks
- **Analyzing spectral shape** (peak frequency, bandwidth)
- **Before/after stimulus comparisons** (same microphone)
- **Presence/absence detection** of ultrasonic components
- **Qualitative analysis** for research papers

### ❌ NOT RECOMMENDED FOR:
- **Absolute sound pressure level** (dB SPL) measurements
- **Energy calculations** requiring quantitative accuracy
- **Comparisons between different microphones**
- **When SNR is poor** (< 10 dB)

---

## Understanding the Correction

### What is "50% Conservative"?

Instead of fully correcting the microphone response (which would amplify noise), we apply **only 50%** of the correction:

| Frequency | Mic Response | Full Correction | 50% Correction |
|-----------|-------------|-----------------|----------------|
| 25 kHz    | +10.5 dB    | -10.5 dB (÷3.35x) | -5.25 dB (÷1.83x) |
| 40 kHz    | 0 dB        | 0 dB            | 0 dB           |
| 60 kHz    | -5.5 dB     | +5.5 dB (×1.78x)  | +2.75 dB (×1.37x) |

**Benefits**:
- ✅ Reduces bias while keeping noise low
- ✅ Scientifically defensible approach
- ✅ Suitable for publication

### Error Margins

**Estimated Total Error**: ±2.9 dB (95% confidence)

**Sources of error**:
1. Datasheet reading: ±0.6 dB
2. Mic unit-to-unit variability: ±2.0 dB
3. Interpolation: ±0.3 dB
4. Temperature/aging: ±0.5 dB

**Practical impact**:
- A click measured at **0.1 V** could be **0.08-0.13 V** (±30%)
- Acceptable for **relative comparisons**
- Not for **absolute measurements**

---

## Data Safety

### ⚠️ IMPORTANT: Original Data Never Modified

- Normalization is **display-only** (overlay graphics)
- The original `.paudio` file remains **unchanged**
- You can always return to raw data
- Both raw and normalized curves shown for transparency

### Reproducibility

All normalization parameters are documented:
- Datasheet values: Fixed (see technical report)
- Correction factor: 50% (hardcoded)
- Algorithm: Linear interpolation + dB-to-gain conversion

**For publications**: Cite the technical report and mention:
> *"FFT spectra were corrected using a 50% conservative normalization based on manufacturer datasheet (SPU0410LR5H-QB Rev. H, 2013). Estimated error: ±2.9 dB."*

---

## Troubleshooting

### "No Phase Data" Error
- **Cause**: File version < 3.0 (no phase information)
- **Solution**: iFFT normalization requires phase data. Use FFT normalization instead.

### Normalized Curve Looks Strange
- **Check**: Is the raw signal saturated? (> 3.0 V)
- **Check**: Is SNR sufficient? (> 10 dB)
- **Solution**: If signal quality is poor, normalization may amplify noise

### Can't See Orange Curve
- **Check**: Zoom out on Y-axis (normalization changes amplitude)
- **Check**: Legend visibility (toggle in plot settings)

---

## Advanced: Understanding the Algorithm

### Step-by-Step Process

1. **Load datasheet curve** (8 frequency points, 20-80 kHz)
2. **Interpolate** to match your frequency axis
3. **Convert dB to linear gain**: `gain = 10^(-dB * 0.5 / 20)`
4. **Apply to FFT**: `normalized_fft = raw_fft * gain`
5. **Display overlay** with both curves

### Code Reference

See: `replay_window_audio.py::normalize_fft_window()`

---

## Validation

### How to Check if Normalization is Working

1. **Known frequency test**: Generate pure tone (e.g., 25 kHz)
   - Raw: Should show +10 dB peak
   - Normalized: Should reduce peak by ~5 dB

2. **Flat noise test**: White noise 20-80 kHz
   - Raw: Peaked at 25 kHz
   - Normalized: Flatter spectrum

3. **Cross-check**: Compare with reference microphone (if available)

---

## FAQ

**Q: Will normalization improve click detection?**  
A: No. Detection uses raw amplitude. Normalization is for **visualization and analysis** only.

**Q: Should I always use normalized data?**  
A: Show **both** in your analysis. Raw data for transparency, normalized for spectral interpretation.

**Q: Can I export normalized data?**  
A: Currently display-only. For export, use screenshot or manual export feature (future update).

**Q: Why 50%? Can I change it?**  
A: 50% balances correction vs. noise amplification. Higher % = more correction but more noise. Not user-configurable to ensure consistency.

**Q: Is this peer-review ready?**  
A: **Yes**, if you:
- Cite the technical report
- Show both raw and normalized data
- Declare limitations (±2.9 dB error)
- Don't claim absolute SPL accuracy

---

## Related Documentation

- 📖 **Technical Report**: `MICROPHONE_NORMALIZATION_TECHNICAL_REPORT.md` (detailed math & error analysis)
- 📊 **Datasheet**: [SPU0410LR5H-QB Rev. H](https://media.digikey.com/pdf/Data%20Sheets/Knowles%20Acoustics%20PDFs/SPU0410LR5H-QB_RevH_3-27-13.pdf)
- 🔬 **Scientific Background**: See references in technical report

---

## Support

For questions or issues:
- **GitHub Issues**: [PlantLeaf-Desktop-App/issues]
- **Email**: tommy.vaninetti@[your-domain]
- **Documentation**: Check technical report for in-depth details

---

**Last Updated**: February 26, 2026  
**Version**: 1.0  
**Feature Status**: ✅ Production Ready
