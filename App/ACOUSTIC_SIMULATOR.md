 # PlantLeaf Acoustic Simulator

 ## Physical Modeling of Xylem Embolism and Comparison with Real Ultrasonic Clicks

 ## Table of Contents

 1. What It Is and What It Is Used For
2. The Physical Phenomenon: Cavitation and Xylem Embolism
3. Module Architecture
4. The Two Physical Click Models
   - 4.1 Model A — Free Bubble (Minnaert \+ Thermal Damping)
   - 4.2 Model B — Xylem Vessel as a Resonator (Dutta et al. 2022)
   - 4.3 Why Two Models Instead of One
5. The Acoustic Propagation Pipeline
6. The Comparison Engine: `click_model_comparison.py`
7. The End-to-End Simulation Pipeline
8. Systematic Water-Stress Analysis
9. Scientific Report and Data Export
10. User Interface
11. Declared Simplifications and Model Limitations
12. File Structure
13. Bibliographic References

---

 # 1\. What It Is and What It Is Used For

 The `acoustic_simulators` module is the physical modeling engine of PlantLeaf, the platform used to monitor plant bioelectrical activity and ultrasonic emissions.

 Its purpose is not to acquire data — that is handled by the rest of the application through the MEMS microphone and firmware pipeline — but to explain the ultrasonic clicks recorded by the system: where the sound physically originates, what generates it inside the plant, and what can be inferred about the water status of the xylem vessel that produced it.

 In practical terms, the simulator answers a very specific question:

 > **"Given a real click measured by PlantLeaf, what physical phenomenon — and with which parameters — could reproduce it?"**

 To answer this question, the application constructs two alternative physical models of the click, sends both through the exact same measurement chain used for real data (firmware, reconstruction, and v6 estimator), and compares the resulting signal with the observed click.

 This makes it possible not only to validate the physical models, but also to estimate biological quantities that would otherwise be inaccessible — such as the cavitation-bubble radius, the radius and length of the xylem vessel involved, and the plant's xylem water tension — from a single non-invasive acoustic recording.

---

 # 2\. The Physical Phenomenon: Cavitation and Xylem Embolism

 Plants transport water from the roots to the leaves through the xylem, a network of conduits in which water is under negative pressure — literally being "pulled" upward by leaf transpiration.

 When water stress becomes sufficiently severe, this metastable water column can break: a vapor/air bubble forms (cavitation), the water separates inside the vessel, and the conduit stops conducting water (embolism).

 This event is not silent. The rupture of the water column and the subsequent mechanical relaxation of the vessel release elastic energy in the form of a very-high-frequency acoustic impulse, typically in the **20–80 kHz** range, outside the human audible range.

 This is the click recorded by the PlantLeaf system using an ultrasonic MEMS microphone.

 The frequency of the signal and, especially, its damping duration (`τ`) contain information about what generated it. A bubble freely oscillating in water has a different acoustic signature from a vessel resonating as an elastic tube.

 It is precisely this acoustic signature — frequency and decay time — that the two simulator models attempt to reproduce and interpret.

---

 # 3\. Module Architecture

```
acoustic_simulators/
├── acoustic_parameters.py      # physical constants and configuration parameters
├── acoustic_propagation.py     # bubble → tissue → geometry → microphone
├── rayleigh_plesset.py         # Model A: free bubble (Minnaert + Prosperetti)
├── vessel_resonance.py         # Model B: vessel resonator (Dutta 2022)
├── click_model_comparison.py   # comparison of the two models with the real click
├── run_acoustic_simulation.py  # end-to-end simulation pipeline + calibration
├── stress_analysis.py          # systematic P∞ × R0 grid analysis
├── report_acoustic.py          # scientific PDF report generation
└── vessel_resonance.py         # xylem vessel geometry (organ-pipe model)
```

 The module is designed to be completely independent of Qt: none of the calculation files imports PySide6.

 It can therefore be used both by the interactive UI (`MainWindowAcousticSimulator`) and by batch-analysis scripts or notebooks without loading the application's entire graphical stack.

 The only point of contact with the rest of the application is the loading, through `hybrid.pipeline_loader`, of the firmware pipeline constants — sampling frequency, FFT size, and microphone response.

 This ensures that the simulated physics uses exactly the same values as the real acquisition chain rather than duplicated values that could become inconsistent.

---

 # 4\. The Two Physical Click Models

 The scientific core of the simulator is the comparison between two alternative physical hypotheses concerning what generates the click.

 These are not two variants of the same model. They represent two conceptually different physical mechanisms, with different free parameters and different predictions.

 ## 4.1 Model A — Free Bubble (Minnaert + Thermal Damping)

 **File:** `rayleigh_plesset.py`

 The hypothesis is that the click is generated by a cavitation bubble freely oscillating in the water contained inside the xylem vessel, around an equilibrium radius `R₀`, with the natural resonance frequency of a gas bubble in a liquid according to Minnaert's theory (1933).

 A historically important implementation detail is documented directly in the code.

 An initial approach attempted to integrate the classical Rayleigh–Plesset equation using the far-field liquid pressure `P∞` as a negative pressure in order to simulate xylem water tension.

 This approach proved physically unstable.

 With negative `P∞` and gas pressure decreasing as the bubble grows, there is no stable equilibrium point. In the simulations, the bubble therefore grew indefinitely instead of oscillating.

 The current model bypasses this problem by directly modeling a damped oscillation around a physically defined equilibrium radius `R₀`.

 This is consistent with the experimental observation that the click is a finite-amplitude oscillatory event rather than an unlimited violent collapse.

 ### Resonance Frequency

 The standard textbook Minnaert equation with a fixed adiabatic exponent `γ` is not used directly, because a bubble with a radius of several tens of micrometers oscillating at tens of kilohertz does not behave as a purely adiabatic system.

 Instead, an effective polytropic exponent `κ` is used, following the linear theory of Prosperetti (1977).

 `κ` lies between the isothermal value `1` and the adiabatic value `γ` and is calculated through fixed-point iteration:

```
ω₀² = [3κ · p_g0 − 2σ/R₀] / (ρ · R₀²)

p_g0 = p₀ + 2σ/R₀
```

 ### Damping

 The bubble oscillation is damped by three independent physical mechanisms, each described by a closed-form expression:

 | Mechanism | Symbol | Formula | Physical origin |
| --- | --- | --- | --- |
| Acoustic radiation | `b_rad` | `ω₀² · R₀ / (2c)` | The oscillating bubble radiates acoustic energy into the surrounding water |
| Viscosity | `b_vis` | `2µ / (ρ · R₀²)` | Viscous friction of the water at the bubble surface |
| Thermal conduction | `b_th` | `p_g0 · Im(Φ) / (2ρ · ω₀ · R₀²)` | Heat exchange between the internal gas and the surrounding liquid |

The decay time is:

```
τ = 1 / (b_rad + b_vis + b_th)
```

 and the quality factor is:

```
Q = ω₀ / (2b) = π · f₀ · τ
```

 An important point, explicitly documented in the code as a correction made during development ("Step B"), is that thermal damping was absent from an earlier version of the model.

 For the relevant frequency range of **20–80 kHz**, this term is actually dominant.

 Including thermal damping reduces the theoretical quality factor `Q` from approximately **55** to approximately **9–14**.

 Without thermal damping, the `τ` predicted by the model was approximately four times larger than the experimentally observed value.

 This was a major error and was corrected by adding the complex Prosperetti function:

```
Φ(R₀, ω)
```

 whose imaginary component quantitatively describes thermal dissipation.

 The bubble radius therefore oscillates according to:

```
R(t) = R₀ + ΔR · rise(t) · exp(−t/τ) · sin(ω₀ · t)
```

 where `rise(t)` is a gradual activation function:

```
rise(t) = (1 − exp(−t/tr))²
```

 with `tr` equal to one quarter of the oscillation period.

 This guarantees zero initial velocity and acceleration and prevents artificial numerical transients at the beginning of the signal.

 ### Free Parameter

 The model has only one free parameter:

```
R₀
```

 Frequency and `τ` are predictions, not calibration parameters.

 Given `R₀`, the physics uniquely determines both the resonance frequency and the damping time.

 This makes the model falsifiable: if the observed `τ` does not correspond to the `τ` predicted by the physical model for the observed frequency, then the free-bubble model does not explain that click.

 ### Radiated Pressure

 The oscillating bubble radiates acoustic pressure according to the monopole-source model:

```
p(r,t) = ρ/r · R · (R·R'' + 2·R'²)
```

 evaluated at:

```
r = R₀
```

 The derivatives are calculated using central numerical differentiation, which provides higher accuracy at the boundaries and is particularly important because the signal begins from a perfectly smooth initial condition.

---

 ## 4.2 Model B — Xylem Vessel as a Resonator (Dutta et al. 2022)

 **File:** `vessel_resonance.py`

 This is the more recent model, added to provide a second physically independent comparison with the free-bubble model.

 The hypothesis is completely different:

 > The click is not primarily the signature of a freely oscillating bubble, but of the xylem vessel itself resonating as a tube after the rupture event.

 The reference is the paper by Dutta et al. (2022):

 > _"Ultrasound Pulse Emission Spectroscopy Method to Characterize Xylem Conduits in Plant Stems"_\
>  Research 2022:9790438.

 The physical interpretation is that bubble formation releases the elastic energy accumulated in the water column under tension.

 This impulse excites a longitudinal standing wave in the vessel element, modeled as a cylindrical cavity with length `L` and radius `R`, which subsequently decays due to the viscosity of the sap.

 ### Resonance Frequency

 For the fundamental mode (`m = 1`):

```
f = m · v_eff / (2L)
```

 where `v_eff` is the effective speed of sound in the elastic-walled tube.

 It accounts for the compliance of the vessel wall:

```
1/v_eff² = 1/v_l² + ρ_l · β

β = 2R / (h · E)
```

 where:

 - `v_l` = speed of sound in water/sap
- `ρ_l` = sap density
- `h` = vessel wall thickness
- `E` = Young's modulus of the vessel wall

 ### Settling Time

 The viscous decay time is:

```
τ_s = ρ_l · R² / (4 · η_l)
```

 A relevant point, explicitly documented as a caution in the code, is that the formula printed in the Dutta et al. paper for this relationship does not algebraically agree with the intermediate equations presented in the same paper.

 There is a factor-of-two discrepancy.

 The simulator deliberately uses the **printed form**, with the factor `4` rather than `2`, because this is the form with which the authors obtained and validated the acoustic radii reported in the paper.

 This choice is explicitly documented together with the warning that, if the alternative formulation were correct, the inferred radii would need to be divided by `√2`.

 ### Inverse Use — From Measurement to Vessel Geometry

 Unlike the bubble model, the vessel model operates in the inverse direction.

 It starts from the measured pair:

```
(f, τ)
```

 and derives the vessel geometry:

```
R = √(4 · η_l · τ_s / ρ_l)
```

 from the decay time, and then:

```
L = m · v_eff(R) / (2f)
```

 from the frequency once `R` is known.

 These two quantities — `R` and `L` — are the scientifically interesting outputs of the vessel model.

 They can be directly compared with the actual plant anatomy measured under a microscope, in the same way as the validation performed in the original paper.

 For _Hydrangea quercifolia_, the reference values included in the code are:

```
Acoustic radius: 11.2 ± 0.5 µm
Length:          0.99 ± 0.08 mm
```

 ### Free Parameters

 The parameters `f` and `τ` are calibrated rather than predicted.

 They are calibrated against the real click, and the calibration is then used to derive:

 - `R`
- `L`
- `v_eff`
- `Q = π · f · τ`

---

 ## 4.3 Why Two Models Instead of One

 The two models are not redundant.

 They represent alternative physical interpretations of the same event and have different epistemic status.

 The **bubble model** is predictive and parsimonious:

 - it has only one free parameter, `R₀`;
- if the real click resembles its prediction, the physics of a free bubble in water may be sufficient to explain the event without explicitly invoking vessel geometry.

 The **vessel model** is descriptive:

 - it assumes that the vessel itself acts as the resonator;
- it uses the measured click to infer vessel properties (`R`, `L`);
- it provides biologically interpretable information even when the free-bubble model does not describe the event well.

 Comparing the two models on the same click using the same measurement procedure also makes it possible to examine the difference between:

 - the "true" `τ`, calibrated by the vessel model without assumptions about free-bubble physics;
- the `τ` predicted by the free-bubble physical model.

 Their ratio:

```
tau_ratio_true_over_pred
```

 is therefore a diagnostic quantity.

 A ratio close to `1` is consistent with the free-bubble model, whereas a substantially larger ratio indicates additional damping that may be associated with interaction with the vessel wall.

---

 # 5\. The Acoustic Propagation Pipeline

 **File:** `acoustic_propagation.py`

 A click generated at the source, inside the xylem vessel, is not identical to what the microphone records.

 Between the source and the sensor, three physical transformations are applied sequentially:

 1. **Plant-tissue attenuation** (`apply_tissue_attenuation`)\
    The leaf/parenchymal tissue attenuates sound in a frequency-dependent manner. The attenuation coefficient `α`, expressed in dB/cm/kHz, is taken from Khait et al. (2023).
2. **Spherical geometric decay** (`apply_geometric_decay`)\
    The amplitude of a spherical wave decreases as `1/r` with increasing distance between the bubble and the microphone.
3. **Microphone frequency response** (`apply_microphone_response`)\
    The frequency response of the Knowles SPU0410H5H MEMS microphone is applied. The microphone has a resonance peak around **25 kHz**, approximately **+10.5 dB**.

 An important implementation choice, explicitly justified in the code, is that both tissue attenuation and microphone gain are calculated only once, at the dominant frequency of the signal, and then applied as a uniform scalar factor rather than bin-by-bin in the frequency domain.

 For a narrow-band signal such as a damped bubble resonance, which is essentially monochromatic, filtering each spectral bin independently would differentially attenuate nearby spectral components around the resonance peak.

 This could artificially distort the time-domain decay shape and increase the measured `τ`.

 Applying a single gain at the dominant frequency is therefore considered physically appropriate for this type of source and avoids this artifact.

---

 # 6\. The Comparison Engine: `click_model_comparison.py`

 This is probably the most conceptually sophisticated component of the simulator.

 It was added more recently together with the vessel model.

 The problem it solves is subtle but fundamental, and it is described directly in the module docstring:

 > The real click is not the emitted pressure: it is what remains after the microphone, firmware FFT, reconstruction, and v6 estimator.

 The real acquisition chain includes:

 - microphone response;
- firmware FFT;
- frame segmentation;
- quantization;
- spectral reconstruction;
- microphone correction;
- tapering;
- Gibbs handling;
- feature extraction;
- decay estimation.

 Therefore, comparing the raw simulated pressure directly with the reconstructed real click would produce quantities that do not have a meaningful physical interpretation.

 The solution is to make the simulated signal pass through exactly the same chain.

```
p(t) model
    ↓
fine grid (16× oversampling at 200 kHz)
    ↓
decimation to 200 kHz (band-limited)
    ↓
microphone response
(hybrid.channel_model.colorize)
    ↓
firmware framing
(hybrid.frame_emulator.forward)
    ↓
reconstruct_frame_v5
    ↓
build_click_context
    ↓
resolve_click
    ↓
compute_features_v5
    ↓
τ, R², f
```

 These are extracted using exactly the same procedures used by the PlantLeaf v6 detector.

 Only at this point is the simulated signal considered a "measured" signal.

 The comparison with the real click can then be interpreted physically.

 ## Calibration

 For each model, an iterative fixed-point calibration is performed for up to **10 iterations**.

 ### Vessel Model

 The parameters `f` and `τ` are adjusted until, after passing through the complete acquisition chain, they reproduce the frequency and decay time observed in the real click.

 ### Bubble Model

 Only the frequency is calibrated.

 The frequency determines `R₀` through bisection using:

```
_bubble_R0_for_frequency
```

 The `τ` is **not** a free parameter.

 It is predicted by the bubble physics and is subsequently compared with the measured real value.

 The simulated amplitude is always scaled to the peak amplitude of the real click because detector thresholds depend on the signal-to-noise ratio.

 The simulated onset is also aligned with the real click peak.

 ## Waveform Comparison

 The waveform comparison is performed only inside the window:

```
[onset, decay_end]
```

 that the v6 detector actually considers part of the click.

 No arbitrary additional margins are introduced.

 The amplitude and phase of the simulated signal are allowed to vary freely and are estimated through least-squares fitting against:

 - the simulated signal;
- its 90° phase-shifted quadrature component, obtained using the Hilbert transform.

 This produces an `R²` value describing the waveform similarity.

 The amplitude is intentionally left free.

 Fixing it to the absolute peak penalized the model because real clicks sometimes contain an initial impulse that is not explained by either physical model.

 In some clicks, the first one or two cycles can be **3–5 times larger** than the subsequent ringing.

 A pure damped sinusoid cannot reproduce this behavior.

 Instead of interpreting the mismatch as a poor ringing fit, this initial impulse is measured separately through:

```
impulse_factor
```

 defined as the ratio between the peak real envelope and the peak envelope of the fitted model.

 ## Thread-Safety Implementation Detail

 The code deliberately avoids `np.linalg` and the `@` matrix multiplication operator for the linear algebra used in the fitting procedure.

 As documented in the source code, the underlying BLAS routines can cause segmentation faults when executed inside a `QThread` on macOS.

 The implementation therefore uses elementary sums for this part of the calculation.

 This is a concrete engineering consideration arising from the combination of PySide6, multi-threaded execution, NumPy, and macOS.

 ## Batch Comparison Output

 The `analyse_click` function generates a complete result for every click, including the results of both models.

 When both calibrations succeed, the output also contains the ratio between:

 - the "true" `τ` calibrated by the vessel model;
- the `τ` predicted by the free-bubble physical model.

 This provides a direct diagnostic quantity for evaluating the physical plausibility of the two mechanisms for an individual event.

---

 # 7\. The End-to-End Simulation Pipeline

 **File:** `run_acoustic_simulation.py`

 The repository contains two generations of this module.

 They are worth distinguishing because they document the evolution of the project.

 ## Base Version

 The original implementation takes `R0` and `P_inf` as direct parameters and performs:

```
simulate_bubble_collapse
        ↓
apply_propagation
        ↓
extract_diagnostics
        ↓
resample_to_plantleaf
```

 It returns the simulated click together with diagnostic parameters such as:

 - `τ` measured using the Hilbert envelope;
- `SPR`;
- asymmetry;
- `R_spectral`.

 ## More Recent "Physical" Version

 The newer implementation adds an entire level of automatic calibration.

 ### Frequency Calibration

```
run_simulation(freq_target_hz=...)
```

 calibrates `R₀` so that the simulated signal reproduces the target frequency after passing through the entire pipeline:

```
bubble
→ tissue propagation
→ measurement chain
```

 The calibration uses a robust grid scan rather than relying exclusively on bisection because some regions of the biological parameter range may not produce valid measurements.

 ### Decay-Time Calibration

```
run_simulation(tau_target_ms=...)
```

 calibrates `R₀` to reproduce the target `τ`.

 Alternatively, if the frequency is already known, the method estimates an additional vessel damping term:

```
extra_damping_rate
```

 required to reproduce the observed `τ` beyond what is predicted by the pure free-bubble physics in an infinite liquid.

 This additional damping explicitly represents frictional interaction with the xylem vessel wall.

 The code deliberately does not treat this as a universal physical law.

 The observed variability between clicks, in some cases up to **3–4×**, suggests that individual vessels may have physically different damping behavior.

 ### Real-Signal Shape Calibration

 The most sophisticated calibration mode is:

```
run_simulation(real_signal_for_fit=...)
```

 Instead of independently calibrating frequency and `τ` and hoping that the waveform shape follows automatically, this procedure searches directly for the pair:

```
(R₀, extra_damping_rate)
```

 that maximizes the correlation between the real click envelope and the simulated envelope.

 The procedure consists of:

 1. a parameter grid search;
2. local refinement around the best region.

 All diagnostic measurements — `τ`, `SPR`, asymmetry, and `R_spectral` — are calculated using the same procedure as the PlantLeaf v4.0 production algorithm so that simulated and real values remain directly comparable.

---

 # 8\. Systematic Water-Stress Analysis

 **File:** `stress_analysis.py`

 This module automates exploration of the physical parameter space.

 It constructs a two-dimensional grid:

```
P∞ × R₀
```

 with a default size of:

```
20 × 10 = 200 combinations
```

 The simulator is executed for every point in the grid and the resulting `τ` is extracted.

 This produces the full distribution of expected decay times as a function of:

 - xylem water tension;
- bubble radius.

 The support functions make it possible to:

 - statistically compare the distribution of simulated `τ` values with the distribution measured from real clicks;
- estimate `P∞`, the xylem water tension, a biological parameter that cannot be measured directly;
- identify the grid value whose simulated `τ` distribution is closest to the observed distribution.

 The comparison function:

```
compare_with_real_data
```

 uses an overlap index based on the distance between the simulated and observed means normalized by their standard deviation.

 The estimation function:

```
estimate_p_inf
```

 identifies the most compatible `P∞` value and then assigns a stress state using:

```
XylemPressure.get_stress_label
```

 with categories ranging from:

```
"Well hydrated"
```

 to:

```
"Severe water stress"
```

---

 # 9\. Scientific Report and Data Export

 **File:** `report_acoustic.py`

 The module generates a multi-page PDF using:

 - `matplotlib`;
- `reportlab`.

 The report is designed to be scientifically self-contained.

 It includes:

 ### Bubble Dynamics

 - bubble radius `R(t)`;
- emitted pressure `p_source(t)`.

 ### Real vs Simulated Signal

 Comparison in:

 - time domain;
- frequency domain.

 ### Decay-Time Distributions

 When a stress analysis is available:

 - simulated `τ` distribution;
- measured `τ` distribution;
- overlapping histograms.

 ### Fitted Parameters

 A table containing:

 - fitted parameters;
- parameter uncertainties.

 A second table explicitly lists the declared model simplifications and provides a qualitative assessment of their expected impact.

 For example:

```
Spherical bubble geometry
→ Low impact
→ Valid while R ≪ vessel diameter
```

 ## CSV Export

 The function:

```
click_model_comparison.result_to_row
```

 together with:

```
CSV_COLUMNS
```

 flattens the complete multi-model comparison result into a single row per click.

 The resulting CSV is ready for external statistical analysis.

 It contains:

 - real `τ`;
- real frequency;
- calibration outcome;
- model-specific parameters;
- waveform `R²`;
- waveform correlation;
- parameters from both physical models.

---

 # 10\. User Interface

 **Class:** `MainWindowAcousticSimulator`

 The interface integrates the entire workflow described above into an interactive environment.

 ## Recording Loading

 A `.paudio` file can be loaded using the native PlantLeaf format through the shared application loader.

 This ensures that the clicks displayed by the simulator are exactly the same clicks that would be seen in the replay window or exported for data collection.

 ## Re-detection

 The v6 detector is re-run on the entire recording.

 Clicks already embedded in the `.paudio` file are not reused because they may have been generated by an earlier detector version.

 ## Click Selection

 The user can select a click from the table.

 The corresponding real signal is immediately displayed in:

 - time domain;
- frequency domain.

 ## Model Selection

 The `model_combo` selector allows the user to choose which model is displayed in the tables and graphs.

 Importantly, the UI always calculates **both models** for every click.

 Changing the selector therefore does not trigger another simulation.

 It only changes the displayed result.

 ## Simulation Execution

 The simulator supports:

 - single-click analysis;
- `"Analyze All"` batch processing.

 Batch processing runs on a separate thread through:

```
ModelComparisonWorker
```

 so that the GUI remains responsive.

 The UI provides:

 - a progress bar;
- batch cancellation.

 ## Graphical Views

 ### Time Domain

 Displays:

 - real click;
- simulated click.

 The time interval actually used by the v6 detector for waveform `R²` calculation is highlighted.

 ### Frequency Domain

 Displays the spectrum in the:

```
20–80 kHz
```

 analysis range.

 ### Bubble Dynamics

 Displays the evolution of:

```
R(t)
```

 for the bubble model at the calibrated `R₀`.

 ### τ–f Map

 The map displays each analyzed click as a point:

```
(frequency, τ)
```

 with both quantities corrected for the measurement chain.

 The points are plotted together with:

 - the theoretical `τ(f)` curve for a free bubble;
- iso-radius lines for the vessel model:

```
R = 10 µm
R = 20 µm
R = 30 µm
R = 40 µm
R = 50 µm
```

 A point lying close to the theoretical bubble curve is compatible with free-bubble behavior.

 A point substantially above the curve represents ringing that is too long to be explained by the free-bubble model alone.

 A point close to an iso-radius line corresponds to a vessel with that inferred radius under the vessel model.

 ### Diagnostic Tables

 The interface provides:

 - model-specific simulated parameters;
- real-vs-simulated comparison metrics.

 These include:

 - `τ`;
- frequency;
- waveform `R²`;
- waveform correlation;
- impulse factor.

 ## Export

 The UI supports:

 - CSV export of the complete batch analysis;
- scientific PDF report generation.

 In the current implementation, the PDF report always reports the **bubble model**.

 ## Vessel Parameters

 The Young's modulus:

```
E
```

 and vessel-wall thickness:

```
h
```

 are exposed as adjustable controls in the UI.

 As documented in the source code, only the vessel length `L` depends on these parameters.

 The vessel radius `R`, derived from `τ`, is not affected by them.

---

 # 11\. Declared Simplifications and Model Limitations

 The project explicitly documents its modeling assumptions both in the internal documentation and in the generated scientific report.

 The principal simplifications and their expected impact are:

 | Simplification | Declared impact |
| --- | --- |
| Spherical bubble geometry | Low — valid while `R ≪` vessel diameter |
| Adiabatic gas law (`γ = 1.4`) as starting point | Low — subsequently corrected using effective `κ` from Prosperetti |
| Uniform xylem pressure `P∞` | Medium — actual `P∞` varies along the vessel |
| Homogeneous tissue attenuation | Medium — real plant tissue is heterogeneous |
| Microphone response from datasheet | Low — unit-to-unit variation is `< 2 dB` |
| No bubble-bubble interaction | Low — a single embolism event is assumed |
| Dutta et al. (2022) formula for `τ_s` used "as printed" in the paper | To be verified — possible `√2` uncertainty, explicitly discussed in the code |
| Additional vessel damping (`extra_damping_rate`) | Empirical fitting parameter, not a universal physical law — varies by `3–4×` between clicks |
| Below approximately `0.06 ms` `τ`, the firmware + v6 estimator no longer resolves the decay reliably | Results for shorter clicks are flagged (`below_resolution`) rather than discarded |

This transparency regarding assumptions is a recurring characteristic of the implementation.

 Every non-obvious modeling choice — including:

 - monochromatic attenuation instead of bin-by-bin attenuation;
- free amplitude during waveform fitting;
- the "printed" versus "derived" version of the Dutta equation;

 is accompanied by an explicit physical justification in the source code.

 In several cases, the documentation also records a previous implementation that was tested and found to be unsuitable.

---

 # 12\. File Structure

```
acoustic_parameters.py
    Physical constants:
    water, gas, bubble, vessel parameters,
    PlantLeaf configuration (FS, FFT, frequency range),
    and microphone response.

acoustic_propagation.py
    apply_propagation():
    bubble → tissue → geometry → microphone → spectrum.

rayleigh_plesset.py
    Model A: free-bubble resonance
    (Minnaert + Prosperetti).

    minnaert_frequency
    damping_time_constant
    synthesize_bubble_oscillation
    solve_R0_for_freq/tau
    (inverse solution through bisection).

vessel_resonance.py
    Model B: xylem vessel as a resonator
    (Dutta et al. 2022).

    vessel_from_click
    synthesize_vessel_pressure
    effective_sound_speed
    settling_time

click_model_comparison.py
    Measurement of the real click through the firmware chain
    (measure_frames).

    Calibration of both models against the same click
    (fit_model).

    Waveform comparison
    (_waveform_fit).

    CSV export.

run_acoustic_simulation.py
    End-to-end simulation pipeline and calibration:

    - frequency calibration
    - τ calibration
    - maximum-correlation calibration against a real signal

stress_analysis.py
    Systematic P∞ × R0 grid analysis,
    statistical comparison with real data,
    and estimation of P∞ from the observed τ distribution.

report_acoustic.py
    Four-page scientific PDF report
    generated using matplotlib + reportlab.
```

---

 # 13\. Bibliographic References

 1. **Minnaert, M. (1933).** _On musical air-bubbles and the sounds of running water._ Philosophical Magazine.
2. **Plesset, M. S. & Prosperetti, A. (1977).** _Bubble dynamics and cavitation._ Annual Review of Fluid Mechanics.
3. **Prosperetti, A. (1977).** _Thermal effects and damping mechanisms in the forced radial oscillations of gas bubbles in liquids._ Journal of the Acoustical Society of America, 61:17.
4. **Brennen, C. E. (1995).** _Cavitation and Bubble Dynamics._ Oxford University Press.
5. **Devin, C. (1959).** _Journal of the Acoustical Society of America_, 31:1654.
6. **Ainslie, M. A. & Leighton, T. G. (2011).** _Journal of the Acoustical Society of America_, 130:3184.
7. **Tyree, M. T. & Sperry, J. S. (1988).** _Do woody plants operate near the point of catastrophic xylem dysfunction caused by dynamic water stress?_ — theory of cavitation in xylem.
8. **Khait, I. et al. (2023).** Acoustic modeling and tissue propagation model for ultrasonic emissions from plants.
9. **Dutta, S. et al. (2022).** _Ultrasound Pulse Emission Spectroscopy Method to Characterize Xylem Conduits in Plant Stems._ Research, 2022:9790438.\
    DOI: `10.34133/2022/9790438`\
    — xylem vessel resonator model.
10. **Knowles SPU0410LR5H.** Datasheet for the MEMS microphone used by the PlantLeaf acquisition system.

 ## Development

 **Acoustic Simulator**\
 Frida Tirari

 Developed as part of the PlantLeaf research project.

 **Last Updated:** September, 2026\
 **Version:** 1.0


