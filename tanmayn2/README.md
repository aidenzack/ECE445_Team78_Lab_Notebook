## **Tanmay's Worklog**

### Week of 2026-02-18 - System Architecture Layout
Early planning of our system established design constraints we wanted to stick to, such as 
having 2 seperate USB-C connections:
- MCU (Arduino Nano ESP32) for 5V power and UART connection
- Battery Charging as part of our custom power management system to produce 3.3V power for peripherals

Which IMU we went with was based on high precision, tolerance for high-speed (up to 2000 degrees/sec motions) and force,
and SPI communication. This led us to choosing the:

**Adafruit TDK InvenSense ICM-20948 IMU**

<img width="800" height="590" alt="image" src="https://github.com/user-attachments/assets/a01be986-59fb-4cf9-85b7-db0722510b76" />

Synchronization across all 4 sensors (Placed at the wrist, elbow, hip, and knee to capture the full kinematic chain) is key so we intend
to set up an FSYNC, with our fall back being that we can increase our sampling rate.

**Timing Analysis:**

- With a sampling rate of 200 Hz, we have a period _T_ = 5 ms.
- Our IMUs have an approximate read time of ~0.1ms
- Even during the fastest movements such as a wrist flick (~30 ms), we can capture 5-6 samples and grab precise data.

### Weeks of 2026-02-25 & 2026-03-02 - Full PCB Schematic Finalized + 1st Round PCB Ordered

I completed the first revision of our PCB Schematic and design, involving the following components:

- USB-C Receptacle (Power Input)
- Charger + Power Path IC (BQ24074RGT)
- 3.3V, 500mA LDO (TLV75533PDBVR)
- Arduino Nano ESP32
- Headers for Li-Po battery, IMUs
- LEDs for Debugging
- Designed 8-pin JST-style connectors for each IMU
- Implemented:
  - **Series resistors (22–47 Ω)** on SPI lines
  - **ESD protection** at connectors
  - **Local decoupling** at each connector

***PCB Schematic (Revision A)***
<img width="2048" height="1410" alt="image" src="https://github.com/user-attachments/assets/a9238047-a672-4ee6-ab0d-fb7ea4a45b40" />


After the PCB Design Review, we talked with our TA about whether or not we are allowed to implement the full Arduino Nano ESP32 devboard or only allowed to use the chip.

Result - Allowed to use full Arduino Nano ESP32 Dev Board (w/ USB-C port, RESET button). Our PCB will be designed as expansion board for the MCU

---

### Week of 2026-03-08 - Tolerance Analysis of SPI Bus and Cable Delay

### SPI Throughput & Utilization Analysis

_Assumptions_

- 4 IMUs (wrist, elbow, hip, knee)
- Sampling rate: **220 Hz**
- Data per sample:
  - Accelerometer: 6 bytes
  - Gyroscope: 6 bytes
  - Register overhead: 1 byte

**Total per sample:**

```
13 bytes / sample / IMU
```

### Total Data Rate

```
13 bytes × 4 IMUs × 220 Hz = 11,400 bytes/s
11,440 × 8                  = 91,520 bits/s
```

### SPI Bus Capacity (2 MHz)

```
2,000,000 bits/s ÷ 8 = 250,000 bytes/s
```

### Bus Utilization

```
91,520 / 2,000,000 = 4.576%
```

### Interpretation

- Only **~4.576%** SPI bus utilization
- System is far from bandwidth-limited
- Allows operation at a **reduced SPI clock for robustness**

### SPI Timing & Cable Delay Analysis

### SPI Clock

Selected SPI clock: **2 MHz**

```
T            = 1 / 2 MHz = 500 ns
Half-cycle   = 250 ns
```

### Cable Length Estimates

| Sensor | Length (m) |
| :----- | :--------- |
| Wrist  | 1.00       |
| Elbow  | 0.45       |
| Hip    | 0.60       |
| Knee   | 0.90       |

### Propagation Delay (≈ 5 ns/m)

**One-way delay:**

```
t_pd = 5 ns/m × L
```

| Sensor | Delay (ns) |
| :----- | :--------- |
| Wrist  | 5.00       |
| Elbow  | 2.25       |
| Hip    | 3.00       |
| Knee   | 4.50       |

**Round-trip delay:**

```
t_rt ≈ 2 × t_pd
```

| Sensor | Delay (ns) |
| :----- | :--------- |
| Wrist  | 10.0       |
| Elbow  | 4.5        |
| Hip    | 6.0        |
| Knee   | 9.0        |

### Timing Comparison (Worst Case — Wrist)

```
One-way    : 5 ns  / 250 ns = 2.0%
Round-trip : 10 ns / 250 ns = 4.0%
```

### Interpretation

- Cable delay is **small relative to the SPI timing budget**
- 2 MHz provides **comfortable timing margin**
- System remains robust despite off-board wiring

---
## Week of 2026-03-08 - Testing and Validation
 
> **Date:** 2026-03-08
 
While awaiting delivery of the Rev B PCB, the entire signal chain was validated on a breadboard. This allowed us to verify SPI bus integrity and IMU synchronization in parallel with PCB fabrication.
 
### Breadboard Setup
 
The breadboard mirrors the central PCB topology:
 
- 1 × Arduino Nano ESP32 (MCU)
- 4 × ICM-20948 IMU breakout boards (wrist, elbow, hip, knee positions)
- Shared SPI bus: `SCLK`, `MOSI`, `MISO`
- Independent `CS_<IMU>` lines per IMU
- Shared `FSYNC` line (driven directly at 3.3 V on breadboard for testing)
> Built in collaboration with my teammate's Arduino firmware to read all 4 IMUs across the shared SPI bus.

**_Breadboard Setup_**
<img width="1875" height="2500" alt="image" src="https://github.com/user-attachments/assets/58448d1e-8b77-41fa-97fd-1c3e9194fb9e" />
 
### Test 1 — SPI Bus Synchronization
 
**Goal:** Verify that the MCU correctly reads all 4 IMUs across the shared SPI bus within a single sampling frame, and that each IMU is uniquely identifiable via its `CS` line.
 
**Method:** Since all 4 IMUs are physically located on the same plane on the breadboard, their accelerometer and gyroscope readings should be **closely aligned** at any given instant — both at rest and under shared motion.

**_Breadboard Setup at Rest_**
<img width="1502" height="782" alt="image" src="https://github.com/user-attachments/assets/40426e03-201f-4555-a15f-1358a6073fd0" />

**_Breadboard Setup in Motion_**
<img width="1450" height="747" alt="image" src="https://github.com/user-attachments/assets/e68ef88f-f3fb-4132-bb7c-0cd6750ed259" />

- The `Δ` values represent **sequential read delay** on the SPI bus, **not** sampling-time difference between IMUs.

### Test 2 — FSYNC Clock Alignment
 
**Goal:** Verify that the shared `FSYNC` line successfully aligns the internal sampling clocks of all 4 IMUs to a common reference.
 
**Method:** The firmware waits for the data-ready condition on all 4 IMUs, then assigns a shared frame number. By tracking the **frame index of peak motion** for each IMU, we can detect whether the IMUs' internal clocks are drifting independently or marching in lockstep.
 
**Before FSYNC Trigger (at rest, clocks free-running):**
<img width="1623" height="402" alt="image" src="https://github.com/user-attachments/assets/7d29fa7a-0bc1-4836-a87e-9e4867fefbf6" />

> Peak-motion frames are **scattered** across IMU1=1, IMU2=32, IMU3=72, IMU4=34 — the internal sampling clocks are running independently.
 
**After FSYNC Trigger (sharp flick of breadboard provides impulse):**
<img width="1644" height="328" alt="image" src="https://github.com/user-attachments/assets/ccd23ce9-4640-4224-b750-6f1963f5230a" />

> All 4 IMUs report peak motion at the **same frame index (2266)** — the FSYNC impulse successfully aligned their internal clocks.
---
## Week of 2026-03-23 - PCB Revisions

At the onset of delivery of our 1st Round PCB order, several issues were identified that required revisions before fabrication of Rev B.
 
### Battery Connector Footprint Mismatch
 
- **Issue:** The Adafruit 3.7 V 1200 mAh Li-Po battery ships with **JST-PH 2.00 mm** connectors.
- **Original footprint:** Molex KK-254 **2.54 mm** male headers.
- **Resolution:** Updated the Li-Po battery connector footprint on the PCB to match JST-PH 2.00 mm.
### Trace Width Revisions for Power Signals
 
Several power-carrying traces needed to be widened to safely handle higher current loads:
 
| Signal      | Path                              | Old Width | New Width |
| :---------- | :-------------------------------- | :-------- | :-------- |
| `VDD5V`     | USB-C input → Charger IC          | 10 mils   | 20 mils   |
| `VBAT`      | Charger IC → Li-Po battery        | 10 mils   | 20 mils   |
| `+3V3_MAIN` | LDO → 4 IMUs + debugging LED      | 10 mils   | 20 mils   |

### FSYNC Logic-Level Mismatch
 
**Issue:** Per the Adafruit TDK InvenSense **ICM-20948** datasheet, the `FSYNC` pin requires an approximate **1.8 V** logic input. However, the Arduino Nano ESP32's `FSYNC` GPIO drives at **3.3 V**, ruling out a direct connection.
 
**Resolution:** Designed a **resistor voltage divider** to step down the 3.3 V signal to ~1.8 V before feeding it to the IMUs.

Used Resistors 2.2kΩ and 1.91kΩ as per ECE Shop availability

**Calculation:**
```
FSYNC_B = 3.3 V
 
FSYNC = FSYNC_B × ( 2.2 kΩ / (1.91 kΩ + 2.2 kΩ) )
      = 3.3 V × (2.2 / 1.91 + 2.2)
      = 1.767 V ~ 1.8V
```

**_Resistor Network for Stepdown to FSYNC_**
<img width="111" height="216" alt="image" src="https://github.com/user-attachments/assets/5aec7859-4038-47c4-be71-de0257a29411" />

**_FINALIZED PCB DESIGN_**
<img width="1614" height="1453" alt="image" src="https://github.com/user-attachments/assets/155a42be-902a-4f00-96fb-fdb90156b940" />
