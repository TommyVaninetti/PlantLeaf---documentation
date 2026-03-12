# 📡 Acquisition Features Guide

## Overview

PlantLeaf supports two distinct acquisition modes for plant bioacoustics and electrophysiology research:

1. **Audio Mode**: Ultrasonic click detection (20-80 kHz)
2. **Voltage Mode**: Action potential monitoring (50Hz - 1kHz)

Both modes provide real-time visualization, automated detection, and long-duration recording capabilities.

---

## Audio Mode - Ultrasonic Click Acquisition

### 🎯 Purpose

Capture and analyze ultrasonic sounds emitted by plants under stress conditions (e.g., drought, mechanical damage). These clicks typically occur in the 20-80 kHz range and last 0.1-5 milliseconds.

### 🔧 Hardware Requirements

- **Microphone**: Knowles SPU0410LR5H-QB MEMS ultrasonic microphone
- **ADC**: 12-bit, 200 kHz sampling rate (STM32F411CEU6 microcontroller)
- **FFT Engine**: CMSIS-DSP library (ARM Cortex-M4 hardware acceleration)
- **Connection**: USB CDC (virtual COM port)

### 📊 Technical Specifications

| Parameter | Value | Notes |
|-----------|-------|-------|
| Sampling Rate | 200 kHz | Nyquist frequency: 100 kHz |
| FFT Size | 512 samples | 2.56 ms time window |
| Frame Rate | 390.625 FPS | Real-time spectrum updates |
| Frequency Range | 20-80 kHz | Bandpass filtered (bins 51-204) |
| Frequency Resolution | 390.625 Hz/bin | fs / FFT_size |
| Phase Quantization | 8-bit signed | ±0.71° error |
| Data Rate | 2.4 Mbps | 770 bytes/frame × 390 FPS |

---

## Audio Acquisition Workflow

### 1. Launch Audio Mode

**From Home Screen**:
- Click **"Audio Acquisition"** button
- Alternatively: File → New → Audio Acquisition

**Window Opens**: `MainWindowAudio`

### 2. Configure Serial Port

**Action**: Click **Serial Port** icon (toolbar)

**Dialog Options**:
- **Port List**: Automatically detects available USB CDC devices
- **Baud Rate**: 115200 (nominal, actual USB CDC speed is 12 Mbps)
- **Timeout**: 1 second

**Typical Port Names (without driver installed)**:
- macOS: `/dev/tty.usbmodem123456`
- Windows: `COM3`, `COM4`, etc.
- Linux: `/dev/ttyACM0`

**Click "Connect"**:
- ✅ Success: Status bar shows "Connected to [port]"
- ❌ Failure: Error dialog with troubleshooting tips

### 4. Start Acquisition

**Action**: Click **Start** button (toolbar)

**Real-Time Display**:

#### FFT Spectrum Plot (Main View)
- **X-axis**: Frequency (20-80 kHz)
- **Y-axis**: Magnitude (Volts)
- **Update Rate**: 60 FPS (downsampled from 390 FPS for UI smoothness)
- **Grid**: Major ticks every 10 kHz, minor ticks every 2 kHz

#### Status Bar (Bottom)
- **Frame Counter**: Current frame number (resets at buffer wrap)
- **Timestamp**: Elapsed time since start (MM:SS.mmm)
- **Peak Frequency**: Frequency bin with maximum amplitude
- **Peak Magnitude**: Maximum value in current frame

### 5. Monitor Click Events

**Automatic Detection**:
- Every frame checked against threshold
- If `max(FFT_magnitude) > threshold`: Click detected
- Click event stored with:
  - Frame number
  - Timestamp
  - Peak frequency
  - Peak magnitude
  - Duration (estimated from spectral width)

**Visual Indicators**:
- **Vertical line** on time-domain plot at detected click time
- **Table entry**: Click details in the click events table (side panel)

### 6. Save Recording

**Action**: Click **Save** icon or File → Save

**File Format**: `.paudio` (custom binary format)

**Save Dialog**:
- Choose file name and directory
- File is saved with a 128-byte header (magic number, version, frame count, sampling rate, phase flag)

**File Size Calculation**:
```
Size (bytes) = Header (128) + Frames × 770
Example (1 minute):
  Frames = 60 sec / 0.00256 sec = 23,437 frames
  Size = 128 + 23,437 × 770 = 18,046,618 bytes ≈ 17.2 MB
```

### 7. Stop Acquisition

**Action**: Click **Stop** button

**Behavior**:
- Acquisition stops immediately
- Current buffer contents preserved
- Prompt to save if unsaved changes exist

---

## Voltage Mode - Action Potential Acquisition

### 🎯 Purpose

Monitor slow electrical signals in plants (action potentials, variation potentials). These signals indicate plant responses to stimuli (light, touch, wounding).

### 🔧 Hardware Requirements

- **Electrodes**: Ag/AgCl surface electrodes or microelectrodes
- **Amplifier**: Differential amplifier (optional, depends on signal strength)
- **ADC**: 12-bit, up to 1 kHz sampling rate (STM32F411CEU6 microcontroller)
- **Connection**: USB CDC (virtual COM port)

### 📊 Technical Specifications

| Parameter | Value | Notes |
|-----------|-------|-------|
| Sampling Rate | 50 Hz – 1 kHz | User-configurable in Sampling Settings |
| Microcontroller | STM32F411CEU6 | USB CDC, same as audio mode |
| ADC Resolution | 12-bit | 0.8 mV quantization step |
| Voltage Range | 0 – 3.3 V | ±1.65 V with DC offset removal |
| Data Rate | up to 4 kB/s | 4 bytes/sample × sampling rate |

---

## Voltage Acquisition Workflow

### 1. Launch Voltage Mode

**From Home Screen**:
- Click **"Voltage Acquisition"** button
- Alternatively: File → New → Voltage Acquisition

**Window Opens**: `MainWindowVoltage`

### 2. Configure Serial Port

**Same as Audio Mode** (see section 2 above)

### 3. Adjust Sampling Settings

**Action**: Click **Sampling Settings** icon (toolbar)

**Dialog Parameters**:

#### Sampling Rate
- **Options**: 50Hz-1kHz

### 4. Start Acquisition

**Action**: Click **Start** button

**Real-Time Display**:

#### Time-Domain Plot (Main View)
- **X-axis**: Time (seconds)
- **Y-axis**: Voltage (V or mV, depending on amplifier setting)
- **Update rate**: ~20 FPS (50 ms timer interval)
- **Buffer**: Last 60,000 samples kept in memory for the rolling plot view

#### Rolling Window
- **Auto-Scroll**: New data appears on right, old data scrolls left

### 5. Save Data

**Saving is automatic during acquisition**: Data is flushed to a temporary `.pvoltage` file every 1,000 samples.

**Permanent save**: File → Save — prompts for file name and location; the temporary file is finalized and renamed.

### 6. Stop Acquisition

**Same as Audio Mode** (see section 7 above)

---

## Troubleshooting

### Issue: No Serial Ports Detected

**Causes**:
1. USB cable not connected
2. Device not powered on
3. Driver not installed (Windows only)

**Solutions**:
1. Check physical connections
2. Verify device power LED is on
3. Install STM32 USB CDC driver, but note that it's not required nor with macOS/Linux nor Windows:
   - Windows: [ST VCP Driver](https://www.st.com/en/development-tools/stsw-stm32102.html)
   - macOS/Linux: Built-in kernel driver (no installation needed)

### Issue: Connection Fails

**Error**: "Failed to open /dev/tty.usbmodem123456: Permission denied"

**Solution (Linux/macOS)**:
```bash
# Add user to dialout group
sudo usermod -aG dialout $USER

# Apply changes (logout/login or reboot)
```

**Solution (macOS specific)**:
```bash
# Grant Terminal access to USB devices
# System Preferences → Security & Privacy → Files and Folders
```

### Issue: Dropped Frames

**Symptoms**: Frame counter increments irregularly, plot freezes

**Causes**:
1. USB bandwidth saturation
2. CPU overload (other processes)
3. Faulty USB cable

**Solutions**:
1. Close other USB-intensive applications
2. Use USB 2.0 port (not USB 1.1 hub)
3. Replace USB cable

### Issue: Many Automatic Click Candidates With No Real Events

**Symptoms**: Click table fills up with noise hits

**Causes**:
1. Energy threshold too low for the noise floor of this recording
2. Electrical interference

**Solutions**:
1. Open Click Detector dialog (Analysis → Automatic Click Detector) and raise the threshold spinbox
2. Shield the microphone from electromagnetic sources

### Issue: Low Signal Amplitude

**Voltage Mode**:
- Check electrode contact (impedance test)
- Verify plant is properly grounded
- Increase amplifier gain (if using external amp)

**Audio Mode**:
- Move microphone closer to plant (5-10 cm optimal)
- Check for ultrasonic interference (fans, electronics)
- Verify microphone orientation (diaphragm faces sound source)

---

## Best Practices

### 1. Pre-Recording Checklist

**Audio Mode**:
- [ ] Microphone within 10 cm of plant
- [ ] No ultrasonic noise sources nearby (motors, speakers)
- [ ] Room temperature stable (20-25°C)
- [ ] Test recording: 30 seconds to verify signal quality
- [ ] Check mean energy μ and σ in the Click Detector dialog after loading a test file

**Voltage Mode**:
- [ ] Electrodes properly attached (skin contact gel if needed)
- [ ] Reference electrode on soil or stem base
- [ ] Wires shielded from noise
- [ ] Ground loop avoided (single ground point)
- [ ] Baseline voltage stable for 60 seconds

### 2. During Recording

- **Minimize Movement**: Avoid touching setup to prevent artifacts
- **Monitor Status Bar**: Check frame counter increments smoothly
- **Save Frequently**: Every 15-30 minutes for long recordings
- **Log Events**: Write down manual annotations (e.g., "Watered plant at t=600s")

### 3. Post-Recording

- **Verify File Integrity**: Re-open saved file to confirm
- **Backup Data**: Copy to external drive or cloud storage
- **Document Metadata**: Create text file with:
  - Date, time, duration
  - Plant species, age, condition
  - Experimental conditions (light, temperature, humidity)
  - Electrode/microphone positions
  - Any observed events

---

## Performance Metrics

### Audio Mode Benchmarks

**Test System**: MacBook Pro M1, 16 GB RAM, macOS 13 ; Custom Thinkpad

| Metric | Value |
|--------|-------|
| Frame rate (actual) | 389.8 ± 1.2 FPS |
| Plot update latency | 7.2 ± 2.1 ms |
| CPU usage | 12-18% (single core) |
| Memory usage | 45 MB (10K frame buffer) |
| Dropped frames | 0 in 10 min test |

### Voltage Mode Benchmarks

**Test System**: Same as above

| Metric | Value |
|--------|-------|
| Sampling rate (500 Hz) | 500 ± 2 Hz |
| Plot update rate | ~20 FPS (50 ms timer) |
| CPU usage | 8-12% |
| Memory usage | ~10 MB (60K sample plot buffer) |

---

## Data Integrity

### Timestamp Accuracy

**Audio Mode**:
- **Frame timestamp**: Based on frame number × 2.56 ms
- **Error**: ±0.5 ms (USB latency jitter)
- **Resolution**: 2.56 ms (frame duration)

**Voltage Mode**:
- **Sample timestamp**: Based on sample number / sampling_rate
- **Error**: ±0.1 ms
- **Resolution**: 0.1 ms (at 10 kHz)

### Lossless Recording

Both modes use **lossless binary formats**:
- No compression artifacts
- No quantization beyond ADC (12-bit)
- Exact reconstruction of acquired data

---

## References

1. **Knowles SPU0410LR5H-QB Datasheet**: [Link](https://media.digikey.com/pdf/Data%20Sheets/Knowles%20Acoustics%20PDFs/SPU0410LR5H-QB_RevH_3-27-13.pdf)
2. **CMSIS-DSP Documentation**: [ARM CMSIS-DSP](https://arm-software.github.io/CMSIS_5/DSP/html/index.html)
3. **USB CDC Specification**: USB Implementers Forum
4. **Khait et al. (2023)**: Plant ultrasonic clicks methodology. *Cell*.

---

**Last Updated**: March 11, 2026  
**Author**: Tommaso Vaninetti  
**Version**: 1.0
