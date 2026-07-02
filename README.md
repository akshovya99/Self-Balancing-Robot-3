# ⚖️ Autonomous Self-Balancing Robot

> A two-wheeled inverted-pendulum robot that evolved across three hardware generations — from a jumper-wired proof-of-concept to a precision, encoder-driven production build running a deterministic 5 ms triple-loop PID controller with Kalman-filtered sensor fusion.

The robot continuously estimates its tilt angle from a 6-axis IMU, fuses it with gyroscope rate data via a Kalman filter, and drives two DC gear motors through a cascaded control system (balance → speed → steering) to remain upright, hold position, and respond to Bluetooth-driven movement commands — all while self-correcting in real time.

---

## 🧠 System Architecture Summary

| Layer | Role |
|---|---|
| **Sensing** | MPU6050 6-axis IMU (accelerometer + gyroscope) over I2C |
| **State Estimation** | Kalman filter (angle loop) + first-order complementary filter (secondary axis) |
| **Control** | Cascaded PID/PD: Balance (angle) → Speed (position/velocity) → Steering (yaw) |
| **Actuation** | TB6612-class dual motor driver → DC gear motors |
| **Feedback** | Hall-effect quadrature encoders for closed-loop speed/position |
| **Timing** | Hardware `MsTimer2` interrupt, fixed 5 ms deterministic control tick |
| **I/O** | Onboard push-button calibration, buzzer feedback, Bluetooth serial remote control |

---

## 🏗️ Hardware Evolution: Three Generations

The project was iterated through three distinct hardware revisions, each targeting a specific engineering milestone — from raw signal validation to a rigid, encoder-precise final build.

| | **Generation 1**<br>Proof of Concept | **Generation 2**<br>Structural Refinement | **Generation 3**<br>Final Production Build |
|---|---|---|---|
| **Image** | ![Generation 1](./images/gen1.jpg) | ![Generation 2](./images/gen2.jpg) | ![Generation 3](./images/gen3.jpg) |
| **Chassis** | Circular, multi-tiered acrylic platform | Elongated rectangular custom chassis plate | Clear acrylic sandwich-plate chassis with rigid brass standoffs |
| **Motors** | Standard yellow plastic hobby DC gear motors | Standard DC gear motors, decoupled from control board | Heavy-duty industrial metal gear motors with integrated high-resolution Hall-effect encoders |
| **Electronics** | Jumper-wired MPU6050, breadboard-style wiring | Standalone Arduino Uno + independent motor driver board | Highly integrated dual-layer PCB stack (custom shield) |
| **Mass Distribution** | External power bank used as an improvised counterweight | Top-heavy counterweight layout to shift moment of inertia | Balanced, low-profile mass distribution engineered around encoder feedback |
| **Sensor Fusion** | None — raw accelerometer/gyro readings | Complementary filter | Full Kalman filter (angle) + complementary filter (secondary axis) |
| **Control** | Open-loop motor tests only | Early single-loop PID (angle only) | Deterministic 5 ms triple-loop PID (balance + speed + steering) |
| **Primary Goal** | Validate I2C communication and raw sensor acquisition | Explore chassis geometry, moment of inertia, and stabilize balance PID | Achieve production-grade precision, repeatability, and closed-loop odometry |

### Generation-by-Generation Notes

**🟢 Generation 1 — Proof-of-Concept Breadboard Prototype**
**Hardware Setup:** Circular multi-tiered acrylic chassis, standard yellow plastic hobby DC gear motors, jumper-wired MPU6050, and an external power bank acting as a counterweight.

**Engineering Focus:** Initial validation of the I2C communication protocol with the IMU and basic open-loop motor response. This build was primarily used to establish fundamental raw sensor readings and test the physical limits of standard plastic gear motors under rapid directional switching.

**🟡 Generation 2 — Structural & Component Refinement**
**Hardware Setup:** Elongated rectangular wooden/acrylic custom chassis plate, standalone Arduino Uno, an independent motor driver board, and a top-heavy layout to adjust the center of mass.

**Engineering Focus:** Implementation of the First-Order Complementary Filter (`Yiorderfilter`) and early tuning of the Balance PD loop. The design transitioned to a longer wheelbase to lower the angular acceleration, providing a wider physical window for the control loops to correct tilting errors.

**🔵 Generation 3 — Final Production & High-Precision Build**
**Hardware Setup:** Custom clear acrylic sandwich plate chassis, heavy-duty industrial metal gear motors with integrated high-resolution magnetic Hall-effect encoders, rigid brass standoffs, and a highly integrated dual-layer PCB stack (Mainboard + Shield).

**Engineering Focus:** The culmination of the project. Features high-torque actuators to eliminate mechanical backlash, clean wire routing to reduce signal noise, and a deterministic 5 ms hardware timer interrupt system. This build fully integrates the Kalman Filter with the complete nested triple-loop PID control architecture (Balance, Speed, and Steering) to achieve pristine, rock-solid dynamic stability.

---

## 🎥 Video Demos

| Generation 1 | Generation 2 | Generation 3 |
|---|---|---|
| [▶ Watch](./videos/version_1_video.mp4) | [▶ Watch](./videos/version_2_video.mp4) | [▶ Watch](./videos/version_3_video.mp4) |

> GitHub renders these as downloadable/playable links when viewing the README on github.com. Click through to view each generation in action.

---

## 🔁 Software & Control Loop Topology

All control logic runs inside a single hardware-timed interrupt (`MsTimer2`, 5 ms period), guaranteeing deterministic loop timing independent of `loop()` jitter.

```
                         ┌─────────────────────────────┐
                         │   MsTimer2 ISR — every 5ms   │
                         └───────────────┬─────────────┘
                                          │
                    ┌─────────────────────┼─────────────────────┐
                    ▼                     ▼                     ▼
           ┌─────────────────┐  ┌──────────────────┐  ┌──────────────────┐
           │  Read MPU6050    │  │  Count encoder    │  │   (every tick)   │
           │  ax, ay, az,     │  │  pulses (Hall     │  │                  │
           │  gx, gy, gz      │  │  effect ISR)      │  │                  │
           └────────┬─────────┘  └─────────┬─────────┘  └──────────────────┘
                    ▼                      ▼
           ┌─────────────────┐   ┌──────────────────┐
           │  Kalman Filter   │   │  countpluse()     │
           │  angle_calculate │   │  direction-signed  │
           │  → tilt angle    │   │  pulse accumulate  │
           └────────┬─────────┘   └─────────┬─────────┘
                    ▼                      │
        ┌─────────────────────┐            │
        │  BALANCE LOOP (PD)   │            │
        │  every 5ms            │           │
        │  PD_pwm = kp·angle    │           │
        │        + kd·angle_dot │           │
        └──────────┬───────────┘            │
                    │        ┌───────────────┘
                    │        ▼
                    │  ┌─────────────────────┐
                    │  │  SPEED LOOP (PI)      │
                    │  │  every 40ms (8 ticks) │
                    │  │  PI_pwm = ki·(pos_err)│
                    │  │       + kp·(vel_err)  │
                    │  └──────────┬───────────┘
                    │             │
                    │  ┌─────────────────────┐
                    │  │  STEERING LOOP (PD)   │
                    │  │  every 20ms (4 ticks) │
                    │  │  Turn_pwm = -turnout  │
                    │  │      ·kp_turn         │
                    │  │      - Gyro_z·kd_turn │
                    │  └──────────┬───────────┘
                    ▼             ▼
           ┌─────────────────────────────────┐
           │   anglePWM() — motor mixing      │
           │   pwm1 = -PD -PI -Turn (right)   │
           │   pwm2 = -PD -PI +Turn (left)    │
           │   ± clamp to [-255, 255]         │
           │   safety cutoff if |angle| > 45° │
           └─────────────────────────────────┘
```

### 1. Kalman Filter — Angle Estimation
Fuses accelerometer-derived tilt (`-atan2(ay, az)`) with gyroscope angular rate to produce a drift-free, low-noise angle estimate. Tunable via:

| Parameter | Value | Effect |
|---|---|---|
| `Q_angle` | `0.0003` | Process noise — lower = smoother angle |
| `Q_gyro` | `0.002` | Gyro drift noise — lower = trust gyro bias more |
| `R_angle` | `3.0` | Measurement noise — lower = trust accelerometer more |

A first-order complementary filter (`Yiorderfilter`) is run in parallel on the secondary axis as a lightweight cross-check / redundancy path.

### 2. Balance Loop (Angle PD) — 5 ms
The innermost, highest-priority loop. Keeps the robot upright around the calibrated `angle0` set-point using proportional-derivative control on tilt angle and filtered angular velocity:

```
PD_pwm = kp * (angle + angle0) + kd * angle_speed
```

### 3. Speed Loop (Position/Velocity PI) — 40 ms
Uses signed, direction-corrected Hall-effect encoder pulses to regulate forward/backward velocity and accumulated position, with anti-windup clamping (`±3550`):

```
PI_pwm = ki_speed * (setp0 - positions) + kp_speed * (setp0 - speeds_filter)
```

### 4. Steering Loop (Yaw PD) — 20 ms
Drives differential wheel speed for turning using gyro Z-axis rate as damping, with a ramped turn-rate output for smooth rotation:

```
Turn_pwm = -turnout * kp_turn - Gyro_z * kd_turn
```

### 5. Motor Mixing & Safety
The three loop outputs are summed and split across left/right motors, clamped to the PWM range, and force-zeroed if the chassis tips beyond ±45° (fall detection cutoff).

### Default Gains (Generation 3 tune)

| Loop | kp | ki | kd |
|---|---|---|---|
| Balance (angle) | 15 | 4 | 0.2 |
| Speed | 2.0 | 0.03 | 0 |
| Steering (turn) | 24 | 0 | 0.08 |

---

## 🔌 Hardware & Pin Mapping (Generation 3)

| Function | Pin |
|---|---|
| Right motor IN1 | 8 |
| Right motor IN2 | 12 |
| Right motor PWM | 10 |
| Left motor IN1 | 7 |
| Left motor IN2 | 6 |
| Left motor PWM | 9 |
| Buzzer | 11 |
| Calibration button | 13 |
| Left encoder | 5 |
| Right encoder | 4 |
| MPU6050 | I2C (SDA/SCL) |

**Bill of Materials (Gen 3):**
- Arduino Uno (or REV4-pin-compatible board)
- MPU6050 6-axis IMU
- TB6612-class dual H-bridge motor driver
- 2× industrial metal gear motors with integrated Hall-effect quadrature encoders
- Custom acrylic sandwich-plate chassis + brass standoffs
- Custom dual-layer PCB shield
- Piezo buzzer, momentary push-button

---

## 📦 Software Dependencies

Install via the Arduino Library Manager (or place in `~/Documents/Arduino/libraries`):

- [`MsTimer2`](https://github.com/PaulStoffregen/MsTimer2) — hardware timer interrupt for the deterministic 5 ms control tick
- [`PinChangeInterrupt`](https://github.com/NicoHood/PinChangeInterrupt) — enables interrupt-driven encoder counting on arbitrary digital pins
- [`MPU6050`](https://github.com/jrowberg/i2cdevlib) (i2cdevlib) — IMU driver
- `Wire` — I2C bus communication (bundled with Arduino IDE)

---

## ⚙️ Setup & Calibration

1. **Wire the hardware** according to the [pin mapping](#-hardware--pin-mapping-generation-3) above.
2. **Install dependencies** listed above via Library Manager.
3. **Flash the firmware**: open `self_balancing_code.ino` in the Arduino IDE and upload.
4. **Calibrate the balance point:**
   - Power on the robot and hold it upright at its true mechanical balance angle.
   - Press and hold the onboard **button** — the robot captures the current tilt angle as `angle0` (its zero reference).
   - Two short **buzzer beeps** confirm calibration is complete.
   - ⚠️ Calibration only runs once per power cycle — recalibrate after any chassis or weight distribution change.
5. **Set it down and let go.** The balance loop engages immediately after calibration.

### Bluetooth Remote Control

Connect a serial Bluetooth module (e.g. HC-05/HC-06) to the Arduino's UART and send single-character commands at 9600 baud:

| Command | Action |
|---|---|
| `F` | Move forward |
| `B` | Move backward |
| `L` | Turn left |
| `R` | Turn right |
| `S` | Stop / hold position |
| `D` | Print current tilt angle to serial |

---

## 🛠️ Tuning Guide

If the robot oscillates, drifts, or falls:

1. **Falls immediately / can't find balance** → re-run button calibration; verify `angle0` sign matches your mounting orientation.
2. **Fast, jittery oscillation** → reduce `kp` (angle loop) or increase `kd`.
3. **Slow front-back drifting/rocking** → increase `kp_speed` slightly, or reduce `ki_speed` to prevent integral windup.
4. **Sluggish or delayed response** → increase `kp`, ensure the 5 ms interrupt isn't being starved by long blocking code elsewhere.
5. **Drifts to one side over time** → check `Q_angle`/`Q_gyro`/`R_angle` Kalman tuning, and confirm encoder wiring direction on both wheels.

Always tune on a soft, open surface, and be ready to catch the robot.

---

## 📄 License

MIT — see [LICENSE](LICENSE).
