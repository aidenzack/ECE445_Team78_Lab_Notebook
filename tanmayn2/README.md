## **Tanmay's Worklog**

### 2026-02-18 - System Architecture Layout
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

### 2026-02-25 - Full PCB Schematic Finalized

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

***PCB Schematic***
<img width="2048" height="1410" alt="image" src="https://github.com/user-attachments/assets/a9238047-a672-4ee6-ab0d-fb7ea4a45b40" />

---

## 3-2-2026 - SPI Throughput & Utilization Analysis

### Assumptions

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

---

## SPI Timing & Cable Delay Analysis

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
