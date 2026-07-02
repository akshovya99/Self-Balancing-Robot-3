# Self Balancing Robot

Arduino firmware for a two-wheeled self-balancing robot, using an MPU6050 IMU, a Kalman filter for angle estimation, and cascaded PID/PD control loops for balance, speed, and steering.

## Hardware

- Arduino (UNO/Nano or compatible, REV4-style pin mapping)
- MPU6050 accelerometer/gyroscope (I2C)
- TB6612FNG dual motor driver
- Two DC gear motors with Hall-effect speed encoders
- Buzzer and pushbutton for startup calibration
- Optional Bluetooth module for serial remote control

## Pin Mapping

| Function        | Pin |
|-----------------|-----|
| Right motor IN1 | 8   |
| Right motor IN2 | 12  |
| Right motor PWM | 10  |
| Left motor IN1  | 7   |
| Left motor IN2  | 6   |
| Left motor PWM  | 9   |
| Buzzer          | 11  |
| Start button    | 13  |
| Left encoder    | 5   |
| Right encoder   | 4   |

## How It Works

1. **Angle estimation** — Accelerometer and gyroscope readings are fused with a Kalman filter (`Kalman_Filter`) to produce a stable tilt angle estimate, updated every 5 ms via a `MsTimer2` interrupt.
2. **Balance control** — A PD loop (`PD()`) drives motor PWM to keep the robot upright around the calibrated balance angle.
3. **Speed control** — A PI loop (`speedpiout()`) uses wheel encoder pulse counts to regulate forward/backward speed and integrate position.
4. **Steering control** — A PD loop (`turnspin()`) handles left/right turning using gyro yaw rate.
5. **Bluetooth control** — Serial commands (`F`, `B`, `L`, `R`, `S`) drive forward, backward, left, right, and stop.

## Setup

1. Install the required Arduino libraries: `MsTimer2`, `PinChangeInterrupt`, `MPU6050` (i2cdevlib), `Wire`.
2. Wire the hardware according to the pin mapping above.
3. Flash `self_balancing_code.ino` to the board.
4. Power on and press the start button while the robot is upright to calibrate the balance angle.

## Tuning

PID/PD gains are defined near the top of the sketch:

- `kp`, `ki`, `kd` — angle (balance) loop
- `kp_speed`, `ki_speed`, `kd_speed` — speed loop
- `kp_turn`, `ki_turn`, `kd_turn` — steering loop

Adjust incrementally and test on a safe, open surface.

## License

MIT — see [LICENSE](LICENSE).
