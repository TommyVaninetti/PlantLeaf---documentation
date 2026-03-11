# 📖 Documentation Index

## Welcome to PlantLeaf Documentation

This repository contains comprehensive documentation for the **PlantLeaf Desktop Application**, a scientific tool for plant bioacoustics and electrophysiology research developed for the **FAST i Giovani e le Scienze 2026** competition.

---

## 🎯 Quick Start

**New to PlantLeaf?** Start here:
1. **[README.md](../README.md)** - Project overview and installation
2. **[ACQUISITION_FEATURES.md](ACQUISITION_FEATURES.md)** - How to record data
3. **[ANALYSIS_FEATURES.md](ANALYSIS_FEATURES.md)** - How to analyze recordings

**For Developers**:
1. **[LIBRARIES.md](LIBRARIES.md)** - Technology choices

**For Researchers**:
1. **[FFT_PHASE_TECHNICAL_SPECIFICATION.md](FFT_PHASE_TECHNICAL_SPECIFICATION.md)** - Mathematical foundations
2. **[MICROPHONE_NORMALIZATION_TECHNICAL_REPORT.md](MICROPHONE_NORMALIZATION_TECHNICAL_REPORT.md)** - Error analysis

---

## 📚 Documentation Structure

### User Guides

#### 🎬 [ACQUISITION_FEATURES.md](ACQUISITION_FEATURES.md)
**Target Audience**: Experimentalists, lab technicians

**Topics Covered**:
- Real-time audio acquisition (ultrasonic clicks, 20-80 kHz)
- Real-time voltage acquisition (action potentials, 0-500 Hz)
- Serial port configuration
- Sampling settings (frame buffers, detection thresholds)
- Data recording and file formats (`.paudio`, `.pvoltage`)
- Troubleshooting connection issues

**Key Features**:
- ✅ Step-by-step workflows
- ✅ Hardware requirements
- ✅ Best practices for data quality
- ✅ Performance benchmarks

---

#### 🔬 [ANALYSIS_FEATURES.md](ANALYSIS_FEATURES.md)
**Target Audience**: Data analysts, researchers

**Topics Covered**:
- FFT spectrum visualization and navigation
- Inverse FFT (iFFT) reconstruction for sub-frame resolution
- Microphone frequency response normalization (50% conservative)
- Click detection algorithm (multi-parameter)
- Data export (CSV, MATLAB, NumPy)
- Statistical analysis and reporting

**Key Features**:
- ✅ Frame-by-frame analysis tools
- ✅ Time-domain reconstruction (5 μs resolution)
- ✅ Automated click detection
- ✅ Export options for external tools

---

#### 🌿 [NORMALIZATION_USER_GUIDE.md](NORMALIZATION_USER_GUIDE.md)
**Target Audience**: Users applying microphone correction

**Topics Covered**:
- What is microphone normalization?
- When to use it (and when NOT to use it)
- How to apply normalization in FFT and iFFT windows
- Understanding error margins (±2.9 dB)
- Scientific publication guidelines

**Key Features**:
- ✅ Non-technical explanations
- ✅ Visual examples (before/after)
- ✅ FAQ section
- ✅ Peer-review considerations

---

### Technical Specifications

#### ⚙️ [FFT_PHASE_TECHNICAL_SPECIFICATION.md](FFT_PHASE_TECHNICAL_SPECIFICATION.md)
**Target Audience**: Engineers, mathematicians, peer reviewers

**Topics Covered**:
- Hardware acquisition system (STM32, SPU0410LR5H-QB microphone)
- FFT implementation (CMSIS-DSP, 512 samples, 390 FPS)
- Phase quantization (8-bit, ±0.71° error)
- Inverse FFT reconstruction algorithm
- Tukey windowing for Gibbs artifact suppression
- Error analysis and uncertainty budget

**Mathematical Depth**: **High** (equations, proofs, numerical analysis)

**Key Sections**:
- Section 3: FFT parameters and windowing
- Section 4: Phase quantization and transmission
- Section 6: Inverse FFT process
- Section 7: **Critical implementation detail** - window applied to complex spectrum
- Section 8: Total uncertainty budget (±2.9 dB amplitude, ±20 μs temporal)

---

#### 📊 [MICROPHONE_NORMALIZATION_TECHNICAL_REPORT.md](MICROPHONE_NORMALIZATION_TECHNICAL_REPORT.md)
**Target Audience**: Bioacoustics researchers, reviewers

**Topics Covered**:
- Scientific rationale for normalization
- SPU0410LR5H-QB frequency response characterization
- 50% conservative correction method (vs. 100% full correction)
- Error analysis (datasheet reading, unit variability, SNR degradation)
- Validation strategy (known source, frequency sweep, noise floor tests)
- Publication guidelines (methods section template, limitations)

**Error Budget**: ±2.9 dB (95% confidence)

**Key Findings**:
- ✅ Suitable for qualitative analysis (spectral shape, presence/absence)
- ❌ NOT suitable for absolute SPL measurements
- ✅ Peer-review ready with proper disclosure

**Appendices**:
- Datasheet values (Table 1, 8 frequency points)
- Interpolation algorithm (Python pseudocode)
- Future improvements (individual calibration, adaptive normalization)

---

#### 🧮 [CLICK_DETECTION_ALGORITHM_MATHEMATICAL_FRAMEWORK.md](CLICK_DETECTION_ALGORITHM_MATHEMATICAL_FRAMEWORK.md)
**Target Audience**: Algorithm developers, computer scientists

**Topics Covered**:
- Multi-parameter detection criteria (amplitude, frequency, coherence)
- Spectral concentration metric (signal-to-background ratio)
- Temporal persistence filter (consecutive frames)
- False positive suppression strategies
- Performance metrics (sensitivity, specificity, F1-score)

**Mathematical Depth**: **Moderate** (statistical methods, signal processing)

**Validation**:
- Ground truth: Manual annotation by expert
- Cross-validation: 80/20 train/test split
- Confusion matrix analysis

---

### Software Documentation

#### 📚 [LIBRARIES.md](LIBRARIES.md)
**Target Audience**: Developers, technology decision-makers

**Topics Covered**:
- Core technology stack justification
- **PySide6 (Qt)**: GUI framework choice (vs. PyQt, Tkinter, Kivy)
- **PyQtGraph**: Real-time plotting (vs. Matplotlib - 10× faster)
- **NumPy/SciPy**: Scientific computing (no viable alternative)
- **PySerial**: USB CDC communication (vs. PyUSB)
- **PyInstaller**: Deployment (vs. cx_Freeze, Nuitka)
- Performance benchmarks (CPU usage, latency, memory)
- License compliance (LGPL, MIT, BSD)

**Comparison Tables**: Library alternatives with pros/cons

**Benchmarks**:
- PyQtGraph: 3-8 ms plot update (vs. Matplotlib: 50-100 ms)
- NumPy iFFT: 0.2 ms (vs. pure Python: 50 ms)
- Real-time CPU usage: 12-18% (sustainable for hours)

---

## 🎓 For FAST Competition Judges

### Recommended Reading Order

1. **[README.md](../README.md)** - 5 minutes
   - Understand project scope and capabilities

2. **[MICROPHONE_NORMALIZATION_TECHNICAL_REPORT.md](MICROPHONE_NORMALIZATION_TECHNICAL_REPORT.md)** - 15 minutes
   - Scientific rigor and error analysis

3. **[FFT_PHASE_TECHNICAL_SPECIFICATION.md](FFT_PHASE_TECHNICAL_SPECIFICATION.md)** - 20 minutes
   - Mathematical foundations and innovation

4. **[ACQUISITION_FEATURES.md](ACQUISITION_FEATURES.md)** - 10 minutes
   - Practical experimental capabilities

**Total Reading Time**: ~50 minutes for comprehensive understanding

---

## 📊 Scientific Validation Summary

### Key Claims & Evidence

| Claim | Evidence | Documentation |
|-------|----------|---------------|
| ±2.9 dB normalization accuracy | Error budget (RSS calculation) | [MICROPHONE_NORMALIZATION_TECHNICAL_REPORT.md](MICROPHONE_NORMALIZATION_TECHNICAL_REPORT.md), Section 4.2 |
| 5 μs temporal resolution | Phase quantization + iFFT | [FFT_PHASE_TECHNICAL_SPECIFICATION.md](FFT_PHASE_TECHNICAL_SPECIFICATION.md), Section 8.3 |
| 390 FPS real-time acquisition | Frame rate = fs / FFT_size | [FFT_PHASE_TECHNICAL_SPECIFICATION.md](FFT_PHASE_TECHNICAL_SPECIFICATION.md), Section 3.1 |
| <20% CPU usage | Performance benchmarks | [LIBRARIES.md](LIBRARIES.md), Section "Performance Benchmarks" |
| Energy conservation (Parseval) | iFFT validation tests | [FFT_PHASE_TECHNICAL_SPECIFICATION.md](FFT_PHASE_TECHNICAL_SPECIFICATION.md), Section 6.3 |

---

## 🔗 External Resources

### Microphone Datasheet
- **SPU0410LR5H-QB Rev. H** (Knowles Acoustics)
- [Download PDF](https://media.digikey.com/pdf/Data%20Sheets/Knowles%20Acoustics%20PDFs/SPU0410LR5H-QB_RevH_3-27-13.pdf)
- Referenced in: Normalization report, FFT spec

### Research Papers
1. **Khait et al. (2023)**: Plant stress sounds, *Cell*
   - Methodology inspiration for click detection
2. **Baudin et al. (2024)**: Plant bioacoustics methods, *Journal of Plant Physiology*
   - Validation of frequency range (20-80 kHz)

### Software Libraries
- **CMSIS-DSP**: [ARM Documentation](https://arm-software.github.io/CMSIS_5/DSP/html/index.html)
- **PySide6**: [Qt for Python](https://doc.qt.io/qtforpython/)
- **PyQtGraph**: [Official Site](https://www.pyqtgraph.org/)

---

## 🛠️ Contributing to Documentation

### Style Guidelines

**Markdown Formatting**:
- Headings: `#` for main, `##` for sections, `###` for subsections
- Code blocks: Use triple backticks with language identifier
- Tables: Align columns with pipes `|`
- Math: Use LaTeX notation in code blocks (e.g., `σ = √(Σx²/N)`)

**Technical Writing**:
- **Be precise**: Specify units, ranges, tolerances
- **Be reproducible**: Provide enough detail to replicate
- **Be honest**: Document limitations, not just successes
- **Be cited**: Reference sources (datasheets, papers, standards)

**File Naming**:
- `UPPERCASE_WITH_UNDERSCORES.md` for major docs
- `lowercase_with_underscores.md` for minor docs
- Use descriptive names (e.g., `FFT_PHASE_TECHNICAL_SPECIFICATION.md`, not `tech_spec.md`)

### Adding New Documentation

1. **Create File**: Use template structure (Overview, Sections, References)
2. **Update Index**: Add entry to this file (`DOCUMENTATION_INDEX.md`)
3. **Cross-Link**: Reference from related docs (e.g., link to user guide from technical spec)
4. **Review**: Check for broken links, typos, clarity

---

## 📅 Documentation Roadmap

### Planned Additions

- [ ] **Video Tutorials**: Screen recordings for acquisition and analysis workflows
- [ ] **API Reference**: Auto-generated from docstrings (Sphinx)
- [ ] **Troubleshooting Database**: Common issues with step-by-step solutions
- [ ] **Hardware Build Guide**: STM32 firmware flashing, circuit diagrams
- [ ] **Multi-Language Support**: Italian translation for local outreach

---

## 📞 Contact & Support

**Author**: Tommaso Vaninetti  
**Email**: tommy.vaninetti@[your-domain]  
**GitHub Issues**: [PlantLeaf-Desktop-App/issues](https://github.com/TommyVaninetti/PlantLeaf-Desktop-App/issues)

**For Questions About**:
- **Software**: Open GitHub issue with `[SOFTWARE]` tag
- **Scientific Methods**: Email with `[SCIENCE]` subject line
- **Hardware**: Contact hardware team members (see README)

---

## 📜 License

All documentation is released under the same license as the PlantLeaf software. See [LICENSE.rtf](../LICENSE.rtf) for details.

**Citations**: If you use PlantLeaf in published research, please cite:
```
Vaninetti, T. (2026). PlantLeaf Desktop Application: A Tool for Plant Bioacoustics 
and Electrophysiology. FAST i Giovani e le Scienze 2026. 
GitHub: https://github.com/TommyVaninetti/PlantLeaf-Desktop-App
```

---

**Last Updated**: March 11, 2026  
**Documentation Version**: 1.0  
**Total Pages**: ~150 pages equivalent (estimated)
