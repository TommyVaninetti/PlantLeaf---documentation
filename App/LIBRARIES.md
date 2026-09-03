# Libraries and Technologies

## Overview

This document provides detailed justification for the technology choices in **PlantLeaf Desktop Application**, explaining why each library was selected and how it contributes to the application's scientific and performance goals.

---

## Core Technology Stack

### 1. Python 3.8+

**Why Python?**
- **Rapid Prototyping**: Ideal for research software with evolving requirements
- **Scientific Ecosystem**: Extensive libraries for signal processing (NumPy, SciPy)
- **Cross-Platform**: Single codebase runs on Windows, macOS, Linux
- **Readable Code**: Maintainable for academic collaboration and peer review
- **Performance Trade-off**: Lower speed than C/C++, mitigated by NumPy/CMSIS-DSP offloading

**Alternatives Considered**:
- **C++/Qt**: Higher performance but steeper learning curve, slower iteration
- **MATLAB**: Excellent for prototyping but expensive, not deployable as standalone app

**Decision**: Python's balance of development speed and scientific library support outweighed performance concerns, especially since FFT computation is offloaded to firmware (STM32F411CEU6 CMSIS-DSP).

---

## GUI Framework

### 2. PySide6 (Qt for Python) 6.9.0

**Official Documentation**: [https://doc.qt.io/qtforpython/](https://doc.qt.io/qtforpython/)

**Why PySide6?**
- **Professional UI**: Native look-and-feel on all platforms (Windows, macOS, Linux)
- **Rich Widget Library**: Complex layouts, dialogs, menus, toolbars
- **Signal/Slot Architecture**: Clean event-driven programming model
- **Theme Support**: Custom CSS-like styling (8 themes: dark/light × 4 color schemes)
- **LGPL License**: More permissive than PyQt (GPL), allows commercial distribution
- **Active Maintenance**: Official Qt Company support, frequent updates

**Installation**:
```bash
pip install PySide6==6.9.0
```
Please note that version 6.9.0 was adopted because new versions were causing problems with PyQtGraph.

**Key Modules Used**:
- `QtWidgets`: Core UI elements (QMainWindow, QDialog, QPushButton, etc.)
- `QtCore`: Event loop, threading (QThread), timers (QTimer)
- `QtGui`: Icons, fonts, keyboard/mouse events

**Alternatives Considered**:
- **PyQt6**: Similar API but GPL license (restrictive for distribution)

---

## Real-Time Plotting

### 3. PyQtGraph

**GitHub**: [https://github.com/pyqtgraph/pyqtgraph](https://github.com/pyqtgraph/pyqtgraph)

**Why PyQtGraph?**
- **High Performance**: Hardware-accelerated OpenGL rendering for >1000 FPS
- **Real-Time Capable**: Updates at 60 FPS (though the FFT frame rate is 390, it'd be useless and impossible to plot all data on a normal screen) without lag
- **Qt Integration**: Native PySide6/PyQt6 compatibility
- **Scientific Features**: Logarithmic axes, crosshairs, region selection
- **Zero Dependencies**: Pure Python + NumPy (no external C libraries)
- **MIT License**: Permissive, no restrictions

**Installation**:
```bash
pip install pyqtgraph
```

**Performance Comparison** (1000 data points, 60 FPS):

| Library | CPU Usage | Latency | Memory |
|---------|-----------|---------|--------|
| **PyQtGraph** | 5-10% | <5 ms | ~50 MB |
| Matplotlib | 40-60% | 50-100 ms | ~200 MB |

**Key Features Used**:
- `PlotWidget`: Main plotting canvas for time-series and FFT spectra
- `LinearRegionItem`: Interactive time window selection for iFFT
- `InfiniteLine`: Crosshairs for precise data reading
- `ViewBox`: Custom zoom/pan behavior
- `GraphicsLayoutWidget`: Multi-plot layouts (e.g., FFT + time-domain)

**Alternatives Considered**:
- **Matplotlib**: Standard library for scientific plotting, but **too slow** for real-time

**Decision**: PyQtGraph's combination of **real-time performance** (critical real time display) and **Qt-native integration** made it indispensable. No other library could maintain smooth 60 FPS UI updates while processing 200 kHz audio streams.

---

## Scientific Computing

### 4. NumPy

**Official Site**: [https://numpy.org/](https://numpy.org/)

**Why NumPy?**
- **Foundation for Scientific Python**: De facto standard for numerical arrays
- **Vectorized Operations**: 10-100× faster than pure Python loops
- **Memory Efficiency**: Contiguous C arrays, minimal overhead
- **FFT/IFFT**: `numpy.fft` module (FFTPACK backend)
- **Broadcasting**: Elegant syntax for element-wise operations
- **BSD License**: Fully permissive

**Installation**:
```bash
pip install numpy
```

**Critical Use Cases**:
1. **FFT/iFFT Operations**:
   ```python
   # Inverse FFT from complex spectrum
   time_signal = np.fft.irfft(complex_spectrum, n=512)
   ```

2. **Microphone Normalization**:
   ```python
   # Interpolate datasheet response curve
   mic_response_db = np.interp(frequency_axis, datasheet_freq, datasheet_db)
   correction_gain = 10 ** (-mic_response_db * 0.5 / 20.0)
   ```

3. **Data Buffers**:
   ```python
   # Circular buffer for real-time streaming
   audio_buffer = np.zeros((10000, 154), dtype=np.float32)
   ```

**Performance Metrics**:
- **512-point iFFT**: ~0.2 ms (pure Python: ~50 ms) → **250× speedup**
- **Tukey window generation**: ~0.01 ms
- **Complex spectrum construction**: ~0.05 ms

---

### 5. SciPy

**Official Site**: [https://scipy.org/](https://scipy.org/)

**Why SciPy?**
- **Advanced Signal Processing**: `scipy.signal` for filtering, windowing, spectral analysis
- **Optimization**: `scipy.optimize` for curve fitting (future: peak fitting)
- **Sparse Arrays**: Efficient storage for large FFT spectra
- **BSD License**: Permissive

**Installation**:
```bash
pip install scipy
```

**Key Modules Used**:

1. **Signal Processing (`scipy.signal`)**:
   ```python
   # Butterworth low-pass filter for voltage data
   from scipy.signal import butter, filtfilt
   b, a = butter(4, 500, fs=10000, btype='low')
   filtered_voltage = filtfilt(b, a, raw_voltage)
   ```

**Alternatives Considered**:
- **NumPy-only**: Limited signal processing functions

---

## Machine Learning

### 6. scikit-learn

**Official Site**: [https://scikit-learn.org/](https://scikit-learn.org/)

**Why scikit-learn?**
- **SVM with probability output**: `SVC(probability=True)` enables Platt scaling — calibrated probabilities that can be thresholded independently of the training decision boundary, which is essential for the recall-optimised threshold used in v6 (0.121, tuned from the out-of-fold ROC curve)
- **Pipeline API**: chains `SimpleImputer → scaler → SVC` into a single serialisable object; the entire preprocessing + model is loaded and applied in one call at inference time with no risk of mismatched transforms
- **`PowerTransformer` (Yeo–Johnson)**: the v6 scaler. These features are ratios with long tails, and Yeo–Johnson was selected on measurement — it beat `StandardScaler`, `RobustScaler`, `QuantileTransformer` and a log10 transform on cross-validated AUC, handles zero and negative values without column subsetting, and pickles with no import coupling
- **Session-level cross-validation**: `StratifiedGroupKFold` prevents recording-level data leakage by keeping all candidates from the same `.paudio` file in the same fold
- **Hyperparameter search on ranking quality**: `GridSearchCV` scores on `roc_auc`, because the grid selects hyperparameters while the decision threshold is tuned separately afterwards from the out-of-fold ROC curve — AUC measures ranking quality, which is exactly what that tuning consumes
- **Model-agnostic feature importance**: `permutation_importance` is run on both the training set and the held-out sessions. The first says what the fitted model leans on, the second what still carries signal where it has never looked — a feature ranking high in one and near zero in the other is a property of the training sessions, not of clicks
- **BSD License**: Permissive

**Installation**:
```bash
pip install scikit-learn
```

**Key components used**:
- `SVC(probability=True, kernel='rbf', C=50, gamma=0.01, class_weight='balanced')` — RBF-kernel classifier with Platt-calibrated probabilities
- `Pipeline([('imputer', SimpleImputer(strategy='median')), ('scaler', PowerTransformer()), ('svm', SVC(...))])` — end-to-end preprocessing + inference object
- `StratifiedGroupKFold(n_splits=5)` — session-aware cross-validation
- `GridSearchCV(scoring='roc_auc')` — hyperparameter search over C, gamma and class_weight
- `permutation_importance` — feature ranking by AUC drop, on both Set A and the held-out Set B

**Use in PlantLeaf**:
- **Training** (`src/ml/train_svm.py`): full cross-validated training pipeline; model saved as `.pkl` via joblib
- **Inference** (`src/core/click_pipeline_v5.py`): the loaded `Pipeline` object runs Stage 3 of the real-time detection loop; joblib is imported lazily so the module remains usable without scikit-learn if Stage 3 is disabled

**Alternatives considered**:
- **TensorFlow / PyTorch**: neural networks require far more labeled data than currently available (91 confirmed clicks); SVMs generalise well in the hundreds-of-samples regime
- **Custom libsvm wrapper**: lower-level, no Pipeline API; scikit-learn's wrapper provides everything needed with no extra integration work

---

### 7. joblib

**Official Site**: [https://joblib.readthedocs.io/](https://joblib.readthedocs.io/)

**Why joblib?**
- **Efficient model serialisation**: saves and loads NumPy arrays inside scikit-learn Pipelines more efficiently than standard `pickle` (memory-mapped arrays, no redundant copies)
- **Automatic parallelism**: used internally by scikit-learn for `GridSearchCV` and `permutation_importance` with no extra configuration
- **BSD License**: Permissive

**Installation**:
```bash
pip install joblib
```

joblib is a hard dependency of scikit-learn and is installed automatically with it.

**Use in PlantLeaf**:
- **Training** (`src/ml/train_svm.py`): `joblib.dump(model_dict, 'model.pkl')` saves the fitted pipeline, threshold, kernel, feature list, and CV log
- **Inference** (`src/core/click_pipeline_v5.py`): imported lazily at the first call to `load_svm_model()`; `joblib.load('model.pkl')` restores the full pipeline in one call
- **Evaluation scripts** (`src/ml/evaluate_candidates.py`, `src/ml/analyze_dataset.py`): load the same `.pkl` to run batch inference on candidate CSV files

---

## Offline Analysis

### 8. pandas

**Official Site**: [https://pandas.pydata.org/](https://pandas.pydata.org/)

**Why pandas?**
- **CSV I/O with heterogeneous types**: reads `*_candidates.csv` files (mix of float features, integer indices, string file names, and optional integer labels) in one call, preserving column types
- **Italian locale robustness**: the `str.replace(',', '.')` + `pd.to_numeric(..., errors='coerce')` pattern handles decimal-comma CSVs exported from Excel without requiring a separate CSV dialect or re-export
- **GroupBy**: per-file confusion matrix and click-rate statistics are computed with `df.groupby('file')`, avoiding explicit loops over file names
- **BSD License**: Permissive

**Installation**:
```bash
pip install pandas
```

**Key use cases**:
- `evaluate_candidates.py`: loads multi-file candidate CSVs, appends `svm_probability`, `svm_prediction`, `stage_blocked` columns, writes annotated output CSV
- `analyze_dataset.py`: computes per-file TP/FP/FN/TN, duration-weighted click rates, and passes the annotated DataFrame to the plot generator
- `train_svm.py`: loads the labeled training CSV, splits into Set A / Set B by session

**Note**: pandas is used only in the offline ML/analysis scripts (`src/ml/`), not in the real-time application loop.

**Alternatives considered**:
- **Pure NumPy / csv module**: no native support for named columns with mixed types; manual parsing of Italian-locale decimals and optional label columns would require significant boilerplate

---

### 9. matplotlib

**Official Site**: [https://matplotlib.org/](https://matplotlib.org/)

**Why matplotlib (for offline analysis)?**
- **Headless rendering**: `matplotlib.use('Agg')` generates PNG files without opening any window or requiring a display — no Qt application instance needed for batch processing
- **`constrained_layout`**: automatically prevents title/tick-label overlap across multi-panel figures without manual spacing tuning; this was the primary reason for switching from PyQtGraph for this use case (PyQtGraph's layout engine required significant effort to avoid clipping and overlap in the multi-row overview + zoom-panel layout)
- **`vlines` API**: clean per-spike rendering for the click-distribution raster plot — no NaN-separator trick needed unlike PyQtGraph line plots
- **Subplot stacking**: `plt.subplots(n_rows, 1)` with a single `figsize` argument creates the overview + per-cluster zoom panels in one call
- **BSD License**: Permissive

**Installation**:
```bash
pip install matplotlib
```

**Key features used** (`src/ml/analyze_dataset.py`):
- `plt.subplots(n_rows, 1, constrained_layout=True)` — multi-row figure (overview + per-cluster zoom)
- `ax.vlines(click_times, 0, 1)` — spike raster for click-distribution timeline
- `ax.axvspan(x_lo, x_hi, alpha=0.15)` — semi-transparent cluster-window markers on the overview panel
- `mticker.MultipleLocator` + `FuncFormatter` — clock-aligned tick intervals (10 s, 1 min, 5 min, 1 h, …) labelled as MM:SS / H:MM:SS
- `fig.savefig(..., facecolor=BG)` — consistent dark background in saved PNGs

**Performance note**: matplotlib is not used for real-time in-app plotting. As shown in §3, it is 40–60× slower than PyQtGraph for live updates, which makes it unsuitable for the 390 FPS FFT stream. Its use is intentionally restricted to the offline analysis pipeline where rendering latency is irrelevant.

---

## Hardware Communication

### 10. PySerial

**Official Site**: [https://pythonhosted.org/pyserial/](https://pythonhosted.org/pyserial/)

**Why PySerial?**
- **Cross-Platform Serial I/O**: Works on Windows, macOS, Linux without modification
- **USB CDC Support**: Virtual COM port communication with STM32F411CEU6
- **Reliable**: Mature library (18+ years), widely used in industry
- **Simple API**: Minimal learning curve
- **BSD License**: Permissive

**Installation**:
```bash
pip install pyserial
```

**Usage Example**:
```python
import serial

# Open USB CDC port (STM32F411CEU6 microcontroller)
ser = serial.Serial('/dev/tty.usbmodem1234', baudrate=115200, timeout=1)

# Read binary frame (770 bytes: 154 bins × 5 bytes)
frame_data = ser.read(770)

# Parse magnitude (float32) and phase (int8)
magnitude = np.frombuffer(frame_data[:616], dtype=np.float32)
phase = np.frombuffer(frame_data[616:], dtype=np.int8)
```

**Data Protocol**:
- **Baud Rate**: 115200 (nominal, USB CDC ignores this)
- **Throughput**: 2.4 Mbps (770 bytes/frame × 390 FPS)
- **Latency**: <10 ms (USB Full Speed: 12 Mbps available)

**Alternatives Considered**:
- **PyUSB**: Lower-level, requires libusb, more complex setup
- **Platform-Specific APIs**: Not cross-platform (e.g., Windows CreateFile, macOS IOKit)

**Decision**: PySerial abstracts platform differences and provides a simple, reliable interface for USB CDC communication. Perfect for scientific instruments.

---

## Deployment

### 11. PyInstaller

**Official Site**: [https://www.pyinstaller.org/](https://www.pyinstaller.org/)

**Why PyInstaller?**
- **Single-File Executables**: Bundles Python + dependencies into `.exe` (Windows) or `.app` (macOS)
- **Cross-Platform**: Same spec file works on all platforms
- **PySide6 Support**: Automatic detection of Qt plugins
- **Code Signing Compatible**: Supports macOS notarization, Windows Authenticode
- **GPL License** (with distribution exceptions): Allows proprietary apps

**Installation**:
```bash
pip install pyinstaller
```

**Build Configuration** (`PlantLeaf.spec`):
```python
# -*- mode: python ; coding: utf-8 -*-

a = Analysis(
    ['src/main.py'],
    pathex=[],
    binaries=[],
    datas=[
        ('assets', 'assets'),
        ('themes', 'themes'),
        ('docs', 'docs')
    ],
    hiddenimports=['PySide6', 'pyqtgraph', 'scipy'],
    hookspath=[],
    runtime_hooks=[],
    excludes=[],
    win_no_prefer_redirects=False,
    win_private_assemblies=False,
    cipher=None,
    noarchive=False
)

pyz = PYZ(a.pure, a.zipped_data, cipher=None)

exe = EXE(
    pyz,
    a.scripts,
    [],
    exclude_binaries=True,
    name='PlantLeaf',
    debug=False,
    bootloader_ignore_signals=False,
    strip=False,
    upx=True,
    console=False,
    icon='assets/logo.ico'  # Windows
)

coll = COLLECT(
    exe,
    a.binaries,
    a.zipfiles,
    a.datas,
    strip=False,
    upx=True,
    upx_exclude=[],
    name='PlantLeaf'
)

app = BUNDLE(
    coll,
    name='PlantLeaf.app',
    icon='assets/logo_for_app.icns',  # macOS
    bundle_identifier='com.plantleaf.app',
    info_plist={
        'NSHighResolutionCapable': 'True',
        'CFBundleShortVersionString': '1.0.0',
        'CFBundleDocumentTypes': [
            {
                'CFBundleTypeName': 'PlantLeaf Audio File',
                'CFBundleTypeRole': 'Viewer',
                'LSItemContentTypes': ['com.plantleaf.paudio'],
                'LSHandlerRank': 'Owner'
            }
        ]
    }
)
```

**Build Command**:
```bash
pyinstaller PlantLeaf.spec
```

**Output**:
- **Windows**: `dist/PlantLeaf.exe` (~120 MB)
- **macOS**: `dist/PlantLeaf.app` (~140 MB)

---

## UI Enhancements

### 12. Custom Theming System

**Implementation**: CSS-like stylesheets for Qt

**Why Custom Themes?**
- **User Preference**: Dark mode for low-light labs, light mode for presentations
- **Color Schemes**: Amber (warm), Blue (cool), Green (plant-themed)
- **Accessibility**: High contrast options

**Theme Files** (`themes/`):
```
dark.css         # Base dark theme
dark_amber.css   # Dark + amber accents
dark_blue.css    # Dark + blue accents
dark_green.css   # Dark + green accents
light.css        # Base light theme
light_amber.css
light_blue.css
light_green.css
```

**Example** (`dark_green.css`):
```css
QMainWindow {
    background-color: #1e1e1e;
    color: #d4d4d4;
}

QPushButton {
    background-color: #2d5016;  /* Green accent */
    border: 1px solid #4a7c25;
    border-radius: 4px;
    padding: 6px 12px;
    color: white;
}

QPushButton:hover {
    background-color: #3a6b1e;
}
```

**Themes are managed by theme_manager.py**:

---

## Summary Table

| Library | Version | License | Purpose | Used in |
|---------|---------|---------|---------|---------|
| **Python** | 3.8+ | PSF | Core language | App + scripts |
| **PySide6** | 6.9.0 | LGPL v3 | GUI framework | App |
| **PyQtGraph** | Latest | MIT | Real-time plotting | App |
| **NumPy** | Latest | BSD | Numerical arrays | App + scripts |
| **SciPy** | Latest | BSD | Signal processing | App + scripts |
| **scikit-learn** | 1.6.1 | BSD | SVM training & inference | App (Stage 3) + scripts |
| **joblib** | 1.5.3 | BSD | Model serialisation | App (Stage 3) + scripts |
| **pandas** | 2.3.3 | BSD | CSV I/O & data manipulation | ML scripts only |
| **matplotlib** | 3.9.4 | BSD | Offline click-distribution plots | ML scripts only |
| **PySerial** | Latest | BSD | USB communication | App |
| **PyInstaller** | Latest | GPL | Deployment packaging | Build only |

---

## License Compliance

All libraries used in PlantLeaf comply with open-source licenses compatible with academic and commercial distribution:

- **LGPL (PySide6)**: Dynamic linking allowed, no source code disclosure required
- **MIT/BSD (PyQtGraph, NumPy, SciPy, PySerial)**: Fully permissive, minimal restrictions
- **PSF (Python)**: Permissive, GPL-compatible

**LGPL Compliance Note** (PySide6):
- PySide6 libraries are **dynamically linked** (not compiled into executable)
- Users can replace PySide6 libraries with compatible versions
- Original PySide6 source code available at: [https://code.qt.io/cgit/pyside/pyside-setup.git/](https://code.qt.io/cgit/pyside/pyside-setup.git/)

---

## References

1. **PySide6 Documentation**: [https://doc.qt.io/qtforpython/](https://doc.qt.io/qtforpython/)
2. **PyQtGraph Documentation**: [https://pyqtgraph.readthedocs.io/](https://pyqtgraph.readthedocs.io/)
3. **NumPy User Guide**: [https://numpy.org/doc/stable/user/](https://numpy.org/doc/stable/user/)
4. **SciPy Reference**: [https://docs.scipy.org/doc/scipy/reference/](https://docs.scipy.org/doc/scipy/reference/)
5. **PySerial Documentation**: [https://pythonhosted.org/pyserial/](https://pythonhosted.org/pyserial/)
6. **PyInstaller Manual**: [https://pyinstaller.readthedocs.io/](https://pyinstaller.readthedocs.io/)

---

**Last Updated**: June 2026  
**Author**: Tommaso Vaninetti  
**Version**: 1.1
