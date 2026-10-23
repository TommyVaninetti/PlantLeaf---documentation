# 🔉 PlantLeaf ASEB — Audio Signal Elaboration Board
![Version](https://img.shields.io/badge/version-1.0-blue.svg)


The **Audio Signal Elaboration Board (ASEB)** is a custom analog front-end designed to **detect, filter, amplify, and condition ultrasonic acoustic emissions produced by plants** before digital acquisition.

Recent research suggests that plants under stress may emit **ultrasonic acoustic signals**, typically in the **20–100 kHz frequency range**. These emissions are extremely weak and require **low-noise analog filtering and amplification** before they can be analyzed digitally.

The current ASEB revision is optimized around a **20–80 kHz useful acquisition band** and operates from a **3.3 V single supply** with a **1.65 V analog reference (VREF)**.

The ASEB converts the signal captured by an **ultrasonic MEMS microphone** into a conditioned analog voltage compatible with **0–3.3 V microcontroller ADC inputs**.

It is part of the **PlantLeaf experimental platform**, enabling the study and monitoring of **plant ultrasonic activity**.

---

# 📡 Ultrasonic Emissions in Plants

Recent studies suggest that plants under stress conditions, such as drought or mechanical damage, may emit airborne ultrasonic acoustic signals.

These emissions are typically referred to as **ultrasonic clicks** and consist of very short acoustic pulses in the ultrasonic frequency range.

These signals are believed to originate from **cavitation events in the xylem**, where small air bubbles form and collapse during water transport inside the plant vascular system.

---

# 🔬 Acoustic Characteristics of Plant Emissions

Experimental measurements from recent studies have identified several key properties of plant ultrasonic emissions.

Typical signal characteristics include:

## Frequency Range

- Typically between **20 kHz and 100 kHz**
- Most observed peaks occur between **40 kHz and 80 kHz**
- Some species show dominant spectral peaks around **50–60 kHz**
- The current ASEB analog chain is optimized for a **20–80 kHz useful acquisition band**

## Click Duration

- Extremely short impulsive events
- Typical duration between **0.1 ms and 0.5 ms**

## Sound Pressure Level

- Approximately **60–70 dB SPL at 10 cm**
- Can still be detected at distances of **3–5 meters** under controlled conditions

## Event Rate

- Healthy plants produce **very few clicks**
- Plants under stress can produce **tens of acoustic events per hour**
- These emissions appear as **isolated ultrasonic impulses** rather than continuous signals

---

# 🌿 Biological Origin of the Signals

The currently accepted explanation links these emissions to **xylem cavitation**.

During water transport:

- Plants pull water upward through the **xylem vessels**
- Under stress conditions, especially drought, the tension inside the xylem increases
- This tension can cause **air bubbles to form and collapse**
- These microscopic events generate **mechanical vibrations**
- The vibrations propagate through the plant tissue and into the air as **ultrasonic acoustic clicks**

While this mechanism is widely supported, the exact relationship between cavitation and airborne ultrasonic emissions is still under investigation.

---

# 📊 Scientific Background

One of the most cited recent studies was conducted by researchers at **Tel Aviv University**, who recorded ultrasonic emissions from plants such as tomato and tobacco.

Key findings from this research include:

- Stressed plants emitted significantly more ultrasonic clicks than unstressed plants
- The signals could be detected several meters away

However, the study also highlights several limitations:

- Only a **limited number of plant species** were tested
- Experiments were conducted under **controlled laboratory conditions**

As a result, plant bioacoustics remains a **relatively unexplored research field**, also due to the elevated cost of the instrumentation.

---

# 🎯 Relevance for the PlantLeaf Project

The existence of ultrasonic emissions from plants opens the possibility of **non-invasive plant monitoring systems**.

Potential applications include:

- Early detection of **plant water stress**
- Monitoring **plant health conditions**
- Developing **precision agriculture sensors**

However, detecting these signals is technically challenging because:

- The signals are **very weak**
- They occur in the **ultrasonic frequency range**
- They are often masked by **environmental and electronic noise**

For this reason, the PlantLeaf platform focuses on developing a **low-noise ultrasonic acquisition system** capable of capturing, conditioning, digitizing, and analyzing these acoustic events.

---

# 🎯 System Overview

The ASEB board is the **analog interface between the ultrasonic MEMS sensor and the digital acquisition system**, conditioning weak ultrasonic signals before ADC conversion.

## Main Objectives

- **Low-noise conditioning and amplification of ultrasonic signals**
- **Band limitation around the 20–80 kHz region of interest**
- **Fixed and repeatable analog gain**
- **Stable 1.65 V-centered output for 3.3 V ADC acquisition**
- **Compact, low-power, and fully SMD implementation**

The analog chain consists of a **dual OPA2365 active filtering stage** followed by a **fixed-gain OPA365 amplifier (~151×)**.

The conditioned signal is then digitized by an **STM32F411 microcontroller** for further processing and analysis.

The signal path is:

```text
MEMS Microphone
      │
      ▼
Active Low-Pass Filter
OPA2365
      │
      ▼
Active High-Pass Filter
OPA2365
      │
      ▼
Fixed-Gain Amplifier
OPA365
~151×
      │
      ▼
STM32F411 ADC
```

The highest amplification is intentionally placed **after the filtering stages**, so that strong out-of-band disturbances are not amplified by the full system gain.

---

# 🔬 Hardware Architecture

## 🎤 Ultrasonic Microphone

The ASEB uses the **SPU0410LR5H-QB MEMS microphone manufactured by Knowles** as the primary ultrasonic sensing element.

This sensor converts **acoustic pressure variations into an analog electrical signal**, allowing the detection of ultrasonic emissions potentially produced by plants.

Typical characteristics:

| Parameter | Typical Value |
|---|---|
| Sensor type | Analog MEMS microphone |
| Frequency response | Up to ~80 kHz |
| Output | Analog voltage |
| Package | Surface-mount (SMD) |

The microphone captures pressure fluctuations in the air and outputs a **very small analog voltage signal**, typically in the microvolt-to-millivolt range, which is then conditioned by the ASEB analog front-end.

### What is a MEMS Microphone?

A **MEMS (Micro-Electro-Mechanical System) microphone** is a sensor that combines microscopic mechanical structures and electronic circuits on a silicon chip.

Inside the device, a **tiny movable diaphragm** reacts to sound pressure.

The diaphragm movement changes the **capacitance between microscopic electrodes**, which is then converted into an electrical signal by integrated circuitry.

MEMS microphones offer several advantages for scientific instrumentation:

- **Very small size**
- **High manufacturing consistency**
- **Affordable cost**
- **Good sensitivity and stability**

These characteristics make them well suited for **compact acoustic sensing systems**, including ultrasonic detection applications.

---

# 🎛️ Signal Processing Components

The current ASEB analog front-end is built around the following main components:

- **OPA2365 dual operational amplifier**
- **OPA365 single operational amplifier**
- **Sallen-Key RC filtering networks**
- **1.65 V analog reference (VREF)**
- **Precision SMD resistors**
- **Stable ceramic capacitors**

The previous **variable-gain architecture and analog switching network have been removed**.

The current ASEB uses a **fixed final gain**, simplifying the signal path and making the complete analog transfer function easier to characterize and reproduce.

---

# 1️⃣ Active Filtering — OPA2365

The **OPA2365 dual operational amplifier** implements the two active filtering stages.

The device operates from:

- **VCC = 3.3 V**
- **GND = 0 V**
- **VREF = 1.65 V**

The AC signal is processed around the **1.65 V analog midpoint**, allowing positive and negative signal excursions to remain inside the 0–3.3 V supply range.

The OPA2365 was selected because it provides:

- **Rail-to-rail input and output capability**
- **3.3 V operation**
- **High bandwidth**
- **Low noise**
- **Low distortion**
- **Fast transient response**
- **Two amplifier channels in one package**

The two OPA2365 channels are used primarily for **frequency-selective conditioning** and operate at approximately **unity gain**.

The main voltage amplification is performed by the OPA365 after the filtering stages.

---

# 2️⃣ Fixed-Gain Amplification — OPA365

The final amplification stage uses an **OPA365 configured as a non-inverting amplifier**.

This stage is placed **after the active filtering chain**, so the largest gain is applied mainly to the conditioned ultrasonic signal.

The principal components are:

| Component | Value | Function |
|---|---:|---|
| R12 | **75 kΩ** | Feedback resistor |
| R11 | **499 Ω** | Gain-setting resistor |
| C14 | **22 µF** | AC coupling in feedback path |
| C11 | **1 µF** | Input AC-coupling capacitor |
| R10 | **10 kΩ** | Input bias resistor to VREF |

At ultrasonic frequencies, the closed-loop gain is approximately:

`Av = 1 + (R12 / R11)`

Using the current resistor values:

`Av = 1 + (75,000 / 499)`

`Av ≈ 151.3`

Therefore, the final amplifier provides approximately:

- **151× voltage gain**
- **43.6 dB voltage gain**

The gain is **fixed** and is no longer dynamically selectable.

---

## Input Coupling Network

The OPA365 input is AC coupled through:

- **C11 = 1 µF**
- **R10 = 10 kΩ to VREF**

The approximate high-pass corner is calculated using:

`fc = 1 / (2πRC)`

Using:

`R = 10 kΩ`

`C = 1 µF`

gives:

`fc ≈ 15.9 Hz`

This is far below the **20–80 kHz acquisition band** and therefore has negligible influence on the ultrasonic response.

---

## Feedback Coupling Network

The lower feedback path contains:

- **R11 = 499 Ω**
- **C14 = 22 µF**

Its approximate transition frequency is:

`fc = 1 / (2πRC)`

Using:

`R = 499 Ω`

`C = 22 µF`

gives:

`fc ≈ 14.5 Hz`

This is also negligible relative to the ultrasonic acquisition band.

The feedback arrangement maintains the required **1.65 V DC operating point** while providing approximately **151× AC gain**.

Because the two preceding OPA2365 filter stages operate at approximately unity gain, the nominal maximum analog-chain gain is approximately:

`Atotal ≈ 151×`

The effective gain inside the useful band is slightly lower because of the frequency-dependent attenuation introduced by the filter stages.

---

# 3️⃣ Analog Filtering

The ASEB uses **two second-order active Sallen-Key filter sections**:

- **Low-pass section**
- **High-pass section**

Together they form a band-limiting analog network optimized around the **20–80 kHz useful measurement region**.

The filter characteristic frequencies are intentionally positioned **outside the useful band**:

- the high-pass transition is below 20 kHz
- the low-pass transition is above 80 kHz

This reduces amplitude attenuation and phase rotation inside the region of interest.

---

## Low-Pass Stage

The low-pass section uses a **2nd-order unity-gain Sallen-Key topology**.

Current component values:

| Component | Value |
|---|---:|
| R1 | **5.76 kΩ** |
| R2 | **5.76 kΩ** |
| C1 | **220 pF** |
| C2 | **220 pF** |

For equal resistor and capacitor values, the characteristic frequency is approximately:

`f0 = 1 / (2πRC)`

Using:

`R = 5.76 kΩ`

and:

`C = 220 pF`

gives:

`f0,LP ≈ 125.6 kHz`

The low-pass characteristic frequency is deliberately placed **above 80 kHz**.

This allows signals near the upper edge of the useful band to pass with lower attenuation and reduced phase rotation.

Above the transition region, the second-order response progressively suppresses unwanted high-frequency content.

---

## High-Pass Stage

The high-pass section uses a **2nd-order unity-gain Sallen-Key topology**.

Current component values:

| Component | Value |
|---|---:|
| R3 | **8.2 kΩ** |
| R4 | **8.2 kΩ** |
| C3 | **1.5 nF** |
| C4 | **1.5 nF** |

For equal resistor and capacitor values:

`f0 = 1 / (2πRC)`

Using:

`R = 8.2 kΩ`

and:

`C = 1.5 nF`

gives:

`f0,HP ≈ 12.9 kHz`

The high-pass characteristic frequency is deliberately placed **below 20 kHz**.

This reduces attenuation and phase shift at the lower edge of the useful acquisition band while progressively rejecting unwanted lower-frequency signals.

---

## Combined Filter Behaviour

Together, the two active filter stages create a response optimized around the **20–80 kHz region of interest**.

The analog chain is:

```text
Microphone
    │
    ▼
Low-Pass Filter
f0 ≈ 125.6 kHz
    │
    ▼
High-Pass Filter
f0 ≈ 12.9 kHz
    │
    ▼
OPA365 Fixed-Gain Amplifier
Gain ≈ 151×
```

The filter suppresses unwanted signals outside the useful measurement band.

### Below the Useful Band

The high-pass stage helps reduce:

- audible-frequency signals
- mechanical vibration
- low-frequency environmental noise
- baseline drift
- DC and slow disturbances

### Above the Useful Band

The low-pass stage helps reduce:

- high-frequency electronic noise
- unwanted RF-related interference
- out-of-band sensor noise
- unnecessary frequency content before ADC conversion

The design therefore prioritizes:

- **Amplitude preservation in the useful band**
- **Reduced phase distortion**
- **Controlled group delay**
- **Good transient response**
- **Progressive rejection of out-of-band signals**

These characteristics are especially important because plant ultrasonic emissions are expected to behave as **short broadband impulses rather than continuous single-frequency tones**.

---

# 📉 Signal Conditioning

Ultrasonic plant emissions are **very low-amplitude acoustic events**, requiring careful analog signal conditioning.

Typical signal characteristics and current ASEB design targets are:

| Parameter | Typical / Design Value |
|---|---|
| Microphone signal amplitude | **µV – mV** |
| Useful acquisition band | **20–80 kHz** |
| Active filter gain | **~1× per stage** |
| Final amplifier gain | **~151×** |
| Final amplifier gain | **~43.6 dB** |
| Analog supply | **3.3 V** |
| Analog reference | **1.65 V** |
| ADC-compatible output | **0–3.3 V** |
| Minimum acquisition rate | **≥200 kS/s** |

The **1.65 V mid-supply reference** allows the ultrasonic AC waveform to swing above and below the reference while remaining compatible with a unipolar 3.3 V ADC.

The output can therefore be represented as:

`VOUT(t) = 1.65 V + vAC(t)`

where `vAC(t)` is the amplified ultrasonic waveform.

---

## Approximate Signal Gain

Ignoring frequency-dependent filter attenuation:

| Microphone AC Amplitude | Approximate Output at 151× |
|---:|---:|
| 100 µV | 15.1 mV |
| 500 µV | 75.7 mV |
| 1 mV | 151 mV |
| 2 mV | 303 mV |
| 5 mV | 757 mV |
| 10 mV | 1.51 V |

The fixed gain significantly improves the detectability of weak ultrasonic signals, while making output headroom and saturation analysis important.

---

# ⚙️ Digital Acquisition

After analog processing, the signal is sampled and digitized by an **STM32F411 microcontroller**.

The microcontroller provides:

- **12-bit ADC conversion**
- **High-speed sampling**
- **USB communication**
- **Real-time data streaming**

The ADC receives an analog waveform centered around:

`VREF = 1.65 V`

For a maximum useful frequency of approximately **80 kHz**, the Nyquist criterion requires:

`fs > 2 × 80 kHz`

therefore:

`fs > 160 kS/s`

A practical minimum design target is:

`fs ≥ 200 kS/s`

Higher sampling rates improve:

- waveform reconstruction
- FFT analysis
- transient characterization
- event timing
- digital filtering
- spectral estimation

The digitized data is transmitted to the **PlantLeaf Desktop Application**, where ultrasonic activity can be visualized and analyzed.

---

# 🔬 Measurement Capabilities

The ASEB enables the detection and characterization of ultrasonic acoustic events potentially associated with:

- **Plant stress**
- **Water transport anomalies**
- **Cavitation events in xylem**
- **Environmental responses**

The current hardware revision is optimized for signals whose relevant spectral content lies approximately within:

`20 kHz – 80 kHz`

The PlantLeaf analysis software can then perform:

- Waveform visualization
- Ultrasonic event detection
- FFT and spectral analysis
- Frequency-domain characterization
- Event amplitude measurement
- Event duration analysis
- Comparison of ultrasonic activity over time
- Correlation with plant or environmental conditions

---

# 🧪 Design Philosophy

The ASEB was designed with the following goals:

- **Maximum practical sensitivity for weak ultrasonic signals**
- **Low-noise analog conditioning**
- **Controlled 20–80 kHz useful bandwidth**
- **Fixed and repeatable amplification**
- **Reduced phase and amplitude distortion in the useful band**
- **Predictable analog transfer function**
- **3.3 V compatibility**
- **Compact SMD implementation**
- **Low component count**
- **Accessible and affordable instrumentation**

The previous concept used electronically selectable gain through an analog switching network.

The current revision removes this architecture and instead uses a **fixed ~151× final amplification stage** after the active filtering.

This provides:

- A simpler analog signal path
- Fewer components
- No gain-switching artifacts
- No gain-control firmware
- Easier calibration
- Easier simulation
- More reproducible analog behaviour

The filter characteristic frequencies are deliberately positioned outside the nominal **20–80 kHz measurement region**.

This improves:

- Signal amplitude preservation
- Phase behaviour
- Group-delay behaviour
- Transient fidelity

These characteristics are particularly important when analyzing **short ultrasonic clicks**, because an impulsive signal contains many frequency components simultaneously.

---

# 🔧 Current Analog Signal Chain

```text
Ultrasonic MEMS Microphone
          │
          ▼
2nd-Order Active Low-Pass
OPA2365
R = 5.76 kΩ
C = 220 pF
f0 ≈ 125.6 kHz
Gain ≈ 1×
          │
          ▼
2nd-Order Active High-Pass
OPA2365
R = 8.2 kΩ
C = 1.5 nF
f0 ≈ 12.9 kHz
Gain ≈ 1×
          │
          ▼
AC Coupling
C11 = 1 µF
R10 = 10 kΩ to VREF
          │
          ▼
Fixed-Gain Amplifier
OPA365
Rf = 75 kΩ
Rg = 499 Ω
Gain ≈ 151.3×
Gain ≈ 43.6 dB
          │
          ▼
ADC Input
0–3.3 V
Centered around VREF = 1.65 V
          │
          ▼
STM32F411
Digital Acquisition
```

---

# ⚡ Power and Analog Reference

The analog front-end operates from a **3.3 V single supply**.

The main supply rails are:

```text
VCC  = 3.3 V
GND  = 0 V
VREF = 1.65 V
```

Because the operational amplifiers operate from a single positive supply, VREF acts as the **analog signal midpoint**.

Filter nodes that would conventionally reference ground in a dual-supply circuit are instead referenced to:

`VREF = 1.65 V`

The actual **0 V GND remains the power-supply ground**.

The VREF node should therefore be:

- **Stable**
- **Low noise**
- **Low impedance**
- **Properly decoupled**

Noise on VREF can directly couple into the analog signal path.

---

# 📐 Current Key Component Values

| Function | Component | Value |
|---|---|---:|
| LP filter | R1 | **5.76 kΩ** |
| LP filter | R2 | **5.76 kΩ** |
| LP filter | C1 | **220 pF** |
| LP filter | C2 | **220 pF** |
| HP filter | R3 | **8.2 kΩ** |
| HP filter | R4 | **8.2 kΩ** |
| HP filter | C3 | **1.5 nF** |
| HP filter | C4 | **1.5 nF** |
| Final amplifier input | C11 | **1 µF** |
| Final amplifier bias | R10 | **10 kΩ** |
| Final amplifier feedback | R12 | **75 kΩ** |
| Final amplifier gain resistor | R11 | **499 Ω** |
| Feedback coupling | C14 | **22 µF** |
| Analog supply | VCC | **3.3 V** |
| Analog reference | VREF | **1.65 V** |

Precision resistors and stable capacitor technologies are preferred in the active filter networks because component tolerance directly affects the filter transfer function.

For the small frequency-setting capacitors, **C0G/NP0 ceramic dielectric** is preferred whenever practical because of its:

- Low temperature coefficient
- Low voltage coefficient
- Low dielectric absorption
- High stability
- Low distortion

---

# 👥 Development

**Hardware Design**  
Abdoellah El Makkaoui

**Software Integration**  
Tommaso Vaninetti

Developed as part of the **PlantLeaf research project**.

The design of this board is still **WIP** and may continue to evolve following prototype testing, acoustic characterization, and comparison between simulation and measured hardware performance.

---

**Last Updated:** September 15th, 2026  
**ASEB Version:** 1.0
![Status](https://img.shields.io/badge/status-Active-green.svg)
