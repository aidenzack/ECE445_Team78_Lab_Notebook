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
