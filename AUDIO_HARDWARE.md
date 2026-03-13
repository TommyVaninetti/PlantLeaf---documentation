# 🔉 PlantLeaf ASEB — Audio Signal Elaboration Board
![Version](https://img.shields.io/badge/version-WIP-purple.svg)

The **Audio Signal Elaboration Board (ASEB)** is a custom analog front-end designed to **detect, amplify, and condition ultrasonic acoustic emissions produced by plants** before digital acquisition.

Recent research suggests that plants under stress may emit **ultrasonic acoustic signals**, typically in the **20–100 kHz frequency range**. These emissions are extremely weak and require **low-noise analog amplification and filtering** before they can be analyzed digitally.

The ASEB converts the signal captured by an **ultrasonic MEMS microphone** into a **clean analog voltage compatible with microcontroller ADC inputs (0–3.3 V)**.

It is part of the **PlantLeaf experimental platform**, enabling the study and monitoring of **plant ultrasonic activity**.

---

# 📡 Ultrasonic Emissions in Plants
Recent studies suggest that plants under stress conditions (such as drought or mechanical damage) may emit airborne ultrasonic acoustic signals. These emissions are typically referred to as **ultrasonic clicks** and consist of very short acoustic pulses in the ultrasonic frequency range.
These signals are believed to originate from **cavitation events in the xylem,** where small air bubbles form and collapse during water transport inside the plant vascular system.
 
# 🔬 Acoustic Characteristics of Plant Emissions
Experimental measurements from recent studies have identified several key properties of plant ultrasonic emissions.
Typical signal characteristics include:

**Frequency range**

- Typically between **20 kHz and 100 kHz**
- Most observed peaks occur between **40 kHz and 80 kHz**
- Some species show dominant spectral peaks around **50–60 kHz**
  
**Click duration**

- Extremely short impulsive events
- Typical duration between **0.1 ms and 0.5 ms**

**Sound pressure level**

- Approximately 6**0–70 dB SPL at 10 cm**
- Can still be detected at distances of **3–5 meters** under controlled conditions

**Event rate**

- Healthy plants produce **very few clicks**
- Plants under stress can produce **tens of acoustic events per hour**
These emissions appear as **isolated ultrasonic impulses** rather than continuous signals.
 
# 🌿 Biological Origin of the Signals
The currently accepted explanation links these emissions to **xylem cavitation.**
During water transport:
- Plants pull water upward through the **xylem vessels**
- Under stress conditions (especially drought), the tension inside the xylem increases
- This tension can cause **air bubbles to form and collapse**
- These microscopic events generate **mechanical vibrations**
- The vibrations propagate through the plant tissue and into the air as **ultrasonic acoustic clicks**
While this mechanism is widely supported, the exact relationship between cavitation and airborne ultrasonic emissions is still under investigation.
 
# 📊 Scientific Background
One of the most cited recent studies was conducted by researchers at **Tel Aviv University,** who recorded ultrasonic emissions from plants such as tomato and tobacco.
Key findings from this research include:
- Stressed plants emitted significantly more ultrasonic clicks than unstressed plants
- The signals could be detected several meters away
- Machine learning models were able to classify plant conditions from the acoustic data
However, the study also highlights several limitations:
- Only a **limited number of plant species** were tested
- Experiments were conducted under **controlled laboratory conditions**
- The biological origin and ecological role of the signals are **not yet fully understood**
As a result, plant bioacoustics remains a **relatively unexplored research field.**
 
# 🎯 Relevance for the PlantLeaf Project
The existence of ultrasonic emissions from plants opens the possibility of **non-invasive plant monitoring systems.**
Potential applications include:
- Early detection of **plant water stress**
- Monitoring **plant health conditions**
- Developing **precision agriculture sensors**
However, detecting these signals is technically challenging because:
- The signals are **very weak**
- They occur in the **ultrasonic frequency range**
- They are often masked by **environmental noise**
For this reason, the PlantLeaf platform focuses on developing a **low-noise ultrasonic acquisition system** capable of capturing and analyzing these acoustic events.

---

# 🎯 System Overview

The ASEB board acts as the **analog interface between the ultrasonic sensor and the digital acquisition system**.

Its main objectives are:

- **Low-noise amplification of ultrasonic signals**
- **Adjustable gain for different signal amplitudes**
- **Bandwidth limitation to reduce noise**
- **Stable analog output for ADC acquisition**
- **Compact and low-power hardware design**

The architecture uses a **two-stage analog signal processing chain** followed by **digital acquisition with an STM32 microcontroller**.

---

# 🔬 Hardware Architecture

🎤 **Ultrasonic Microphone**

The ASEB uses the **SPU0410LR5H-QB MEMS microphone manufactured by Knowles** as the primary ultrasonic sensing element.
This sensor converts **acoustic pressure variations into an analog electrical signal**, allowing the detection of ultrasonic emissions potentially produced by plants.

Typical characteristics:

| Parameter | Typical Value |
|-----------|---------------|
| Sensor type | Analog MEMS microphone |
| Frequency response | Up to ~100 kHz (ultrasonic capable) |
| Output | Analog voltage |
| Package | Surface-mount (SMD) |

The microphone captures pressure fluctuations in the air and outputs a **very small analog voltage signal**, which is then amplified and conditioned by the ASEB analog front-end.

***What is a MEMS Microphone?***

A **MEMS (Micro-Electro-Mechanical System) microphone** is a sensor that combines **microscopic mechanical structures and electronic circuits on a silicon chip**.

Inside the device, a **tiny movable diaphragm** reacts to sound pressure.  
The diaphragm's movement changes the **capacitance between microscopic electrodes**, which is then converted into an electrical signal by integrated circuitry.

MEMS microphones offer several advantages for scientific instrumentation:

- **Very small size**
- **High manufacturing consistency**
- **very affordable**
- **Good sensitivity and stability**

These characteristics make them well suited for **compact acoustic sensing systems**, including ultrasonic detection applications.

---

🎛️ **Signal Processing Components**

The ASEB analog front-end is built around three main components:

- **OPA1652 dual operational amplifier**
- **ADG704 analog switch**
- **RC filtering networks**

These components provide **low-noise amplification, configurable gain, and signal conditioning** before the signal reaches the digital acquisition stage.

---

## 1️⃣ Low-Noise Amplification — OPA1652

The first stage of the ASEB uses the **OPA1652 low-noise operational amplifier**.

This amplifier was selected because of its characteristics:

- **Very low noise density**
- **High bandwidth**
- **Excellent performance for audio and ultrasonic signals**
- **Low distortion**

The amplifier boosts the weak microphone signal to a level that can be processed by subsequent stages.

Multiple amplification stages may be used to reach the required signal amplitude while maintaining stability and low noise.

---

## 2️⃣ Configurable Gain — ADG704

The ASEB includes an **ADG704 analog switch** to allow **dynamic selection of different gain configurations**.

The ADG704 is a **quad SPST analog switch** that can connect different resistor networks into the amplifier feedback path.

This allows the system to **select between multiple gain values**, adapting the amplification depending on the detected signal strength.

Advantages of this approach:

- adjustable sensitivity  
- improved measurement flexibility  
- optimized signal-to-noise ratio  

The switches are controlled digitally by the **STM32 microcontroller**.

---

## 3️⃣ Analog Filtering

To improve signal quality, the ASEB uses **RC low-pass filters**.

These filters are implemented using **resistor–capacitor networks** that limit the signal bandwidth and suppress unwanted noise.

Filtering helps remove:

- high-frequency electronic noise  
- switching artifacts  
- environmental electromagnetic interference  

This ensures that the signal reaching the ADC is **stable and representative of the ultrasonic emissions**.

---

# 📉 Signal Conditioning

Ultrasonic plant emissions are **very low amplitude acoustic events**, requiring careful signal conditioning.

Typical signal characteristics:

| Parameter | Typical Range |
|-----------|---------------|
| Signal amplitude | µV – mV |
| Frequency range | ~20 – 100 kHz |
| Acquisition rate | ≥200 kHz recommended |

Proper amplification and filtering are essential to ensure reliable detection.

---

# ⚙️ Digital Acquisition

After analog processing, the signal is **sampled and digitized** by an **STM32 microcontroller**.

The microcontroller provides:

- **12-bit ADC conversion**
- **High-speed sampling**
- **USB communication**
- **Real-time data streaming**

The digitized data is then transmitted to the **PlantLeaf Desktop Application**, where ultrasonic activity can be visualized and analyzed.

---

# 🔬 Measurement Capabilities

The ASEB system enables the detection of ultrasonic acoustic events potentially associated with:

- **plant stress**
- **water transport anomalies**
- **cavitation events in xylem**
- **environmental responses**

These signals can then be processed in the **PlantLeaf analysis software**, enabling further research into plant bioacoustics.

---

# 🧪 Design Philosophy

The ASEB was designed with the following goals:

- **Maximum sensitivity for weak ultrasonic signals**
- **Low-noise analog amplification**
- **Flexible gain configuration**
- **Compact and efficient hardware**
- **accesible and affordable instrumentation**
Traditional scientific ultrasonic microphones used in this field can cost **hundreds of dollars**.  
The ASEB approach aims to provide a **low-cost alternative based on MEMS sensors**, enabling wider experimentation and research.

---

# 👥 Development

**Hardware Design**  
Abdoellah El Makkaoui  

**Software & Firmware Integration**  
Tommaso Vaninetti  

Developed as part of the **PlantLeaf research project**; the design of this board is still **WIP**.

---

**Last Updated**: March 13, 2026  
**ASEB Version**: WIP  
![status](https://img.shields.io/badge/status-Active-green.svg)
