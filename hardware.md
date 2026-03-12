# ⚡ ESEB — Electric Signal Elaboration Board

The **Electric Signal Elaboration Board (ESEB)** is a custom analog front-end designed to **acquire, amplify, and condition extremely small bioelectric signals in plants** before digital acquisition.

Plant electrophysiological signals (such as **action potentials** and **variation potentials**) typically have amplitudes in the **microvolt–millivolt range** and occur at **very low frequencies**. Because of this, direct acquisition with a microcontroller is not possible without dedicated analog signal conditioning.

The ESEB solves this problem by converting weak electrode signals into a **stable analog voltage compatible with microcontroller ADC inputs (0–3.3 V)**.

It is part of the **PlantLeaf experimental platform**, enabling the measurement and analysis of plant electrical activity in real time.

---

# 🎯 System Overview

The ESEB board acts as the **analog interface between plant electrodes and the digital acquisition system**.

Its main objectives are:

- **High input impedance** to avoid loading biological signals
- **Low-noise differential amplification**
- **Strong rejection of environmental interference**
- **Stable low-frequency signal conditioning**
- **Compatibility with low-cost microcontroller ADC systems**

The architecture uses a **two-stage analog signal processing chain** followed by **digital acquisition with an STM32 microcontroller**.

---

# 🔬 Hardware Architecture

🔌**Electrode Connection**

The plant electrodes connect to the ESEB board via a **standard XLR-3 connector**, which provides a robust and noise-resistant connection for bioelectric measurements.

Pin Configuration:
| Pin | Signal |
|-----|--------|
| 1 | − (Negative/Reference) |
| 2 | + (Positive/Signal) |
| 3 | GND (Ground) |

This three-pin configuration enables differential measurement while maintaining a dedicated ground reference, minimizing environmental noise coupling.



🎛️ **Signal Processing Components**

The ESEB analog front-end is built around 3 key components:

- **INA128 instrumentation amplifier**
- **LMC6484 quad operational amplifier**
- **RC networks (Low-Pass Filtering)**: Implementing Resistor–Capacitor(RC) networks limits bandwidth and suppresses high-frequency noise before ADC acquisition.

These components perform **precision differential amplification and signal conditioning** before the signal reaches the digital system.

---

## 1️⃣ Differential Amplification — INA128

The first stage uses the **INA128 precision instrumentation amplifier** to measure the **differential voltage between plant electrodes**.

Instrumentation amplifiers are essential in bioelectric measurements because they provide:

- **Extremely high input impedance**
- **High common-mode rejection (CMRR)**
- **Low offset voltage**
- **Accurate differential amplification**

This stage extracts the **true biological signal** while rejecting common-mode disturbances such as electromagnetic interference or ground noise.

The gain of the INA128 is controlled by an external resistor according to:

G = 1 + (50 kΩ / R_G)

Where:
- G = Gain
- R_G = Gain resistor

This allows amplification of signals ranging from **tens of microvolts to several millivolts**, making them measurable by the following stages.

---

## 2️⃣ Signal Elaboration — LMC6484

After the initial amplification, the signal is processed by the **LMC6484 quad operational amplifier**, which performs the analog **signal elaboration** stage.

The LMC6484 was selected due to its characteristics:

- **Ultra-low input bias current**
- **Rail-to-rail input and output**
- **High stability with high-impedance sources**
- **Low power consumption**

These properties make it particularly suitable for **bioelectric measurements**, where electrode impedance can be very high.

The LMC6484 stage performs:

- signal buffering  
- additional amplification  
- analog filtering  
- output stabilization  

This ensures that the signal delivered to the ADC is **clean and stable**.

---

# 📉 Signal Conditioning

Plant electrical signals are typically **low profile biological events**, often occurring below **tens of Hertz**.

The ESEB therefore includes **analog filtering** to suppress unwanted noise sources such as:

- electromagnetic interference
- environmental electrical noise
- high-frequency artifacts

Typical signal characteristics:

| Parameter | Typical Range |
|-----------|---------------|
Signal amplitude | µV – tens of mV |
Frequency range | ~0.01 – 50 Hz |
Acquisition rate | 50 Hz – 1 kHz |

---

# ⚙️ Digital Acquisition

After analog processing, the signal is **sampled and digitized** by an **STM32 microcontroller**.

The microcontroller provides:

- **12-bit ADC conversion**
- **Sampling rates up to ~1 kHz**
- **USB CDC communication**
- **Real-time data streaming**

The digitized data is then transmitted to the **PlantLeaf Desktop Application**, where the signal can be visualized, recorded, and analyzed.

---

# 🔬 Measurement Capabilities

The ESEB system enables the detection and analysis of several plant electrophysiological phenomena:

- **Action potentials**
- **Variation potentials**
- **Electrical responses to mechanical or environmental stimuli**
- **Slow bioelectric fluctuations**

These signals can then be analyzed in the **PlantLeaf software environment**, where advanced tools allow **event detection, signal modeling, and quantitative analysis**.

---

# 🧪 Design Philosophy

The ESEB was designed with the following goals:

- **Maximum sensitivity for weak biological signals**
- **Minimal signal distortion**
- **High input impedance**
- **Low noise amplification**
- **Affordable and reproducible hardware**

This makes the system suitable for **plant electrophysiology experiments, educational laboratories, and open scientific instrumentation**.

---

# 👥 Development

**Hardware Design**  
Abdoellah El Makkaoui  

**Software & Firmware Integration**  
Tommaso Vaninetti  

Developed as part of the **PlantLeaf research project**.
