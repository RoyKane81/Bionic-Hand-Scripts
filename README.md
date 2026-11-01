# Bionic Hand Scripts

Arduino and Python code for an EMG-controlled bionic hand. A MyoWare 2.0 muscle
sensor on the forearm is read by an Arduino, which drives a DC motor (through a
TB6612 motor driver) that opens and closes the hand. The Python side is used to
record sensor data, label it, and train open/closed classifiers.

## Hardware

| Part | Connection |
|---|---|
| MyoWare 2.0 **ENV** output | `A0` (`A4` in `with_several_sensors`) |
| TB6612 `AIN1` | pin `13` |
| TB6612 `AIN2` | pin `4` |
| TB6612 `PWMA` | pin `11` |
| TB6612 `STBY` | tie HIGH (no sketch drives it) |
| Push button (button sketches) | pin `8` |

Older sketches in `old_scripts/` use different pins; see each file below.

## Repository layout

```
arduino_scripts/   Sketches that run on the Arduino
python_scripts/    Data recording, labeling, and model training
data/              Recorded EMG sessions and labeling reference images
```

---

## `arduino_scripts/`

Each sketch lives in its own folder so it opens directly in the Arduino IDE.

### `proportional_control/proportional_control.ino` — **current controller**
Proportional grip force control. Instead of an on/off threshold, the EMG
envelope (averaged over the last `AVG_WINDOW` readings) is scaled to a 0–1 activation between your calibrated rest level
and max contraction, and that sets the closing PWM. Once the fingers stall on an
object, motor torque is roughly proportional to PWM, so squeezing harder grips
harder. Relaxing drives the hand open for `OPEN_TIME_MS`, then releases the motor.

- **Calibration:** starts with default values (`DEFAULT_REST_MEAN`,
  `DEFAULT_REST_STD`, `DEFAULT_MVC_LEVEL`). Press the button on pin 8 to
  recalibrate: relax for 3 s, then squeeze as hard as possible for 3 s.
- **Serial:** 115200 baud, prints `envelope,activation%,pwm` for Serial Plotter
  (pwm is negative while opening).
- **Tune:** `MIN_PWM` (lowest PWM that moves the fingers), `MVC_FRACTION`
  (lower = less effort for full force), `AVG_WINDOW`, `REST_STD_MULT`
  (raise if the hand twitches at rest), `OPEN_PWM` / `OPEN_TIME_MS`.
- **Caution:** holding a grip stalls the motor, which draws max current and heats
  the motor and TB6612 (≈1.2 A continuous). Lower `MAX_PWM` if your motor's
  stall current exceeds that.

### `arduino_read_myo/arduino_read_myo.ino`
Runs the trained logistic regression model on the Arduino. Normalizes `A0`
readings with hardcoded `CALLIBRATED_MEAN` / `CALLIBRATED_STD`, collects
10-sample windows, extracts features (mean, endpoints, skew, kurtosis, std of
differences), and prints `Open` or `Close`. Does not drive the motor. Note the
feature list and intercept do not exactly match `make_predictions_from_sensor.py`.

### `process_signal_tools/process_signal_tools.ino`
Threshold-based toggle. Keeps a 20-sample window of first differences; when the
raw reading is > 100 and the window's std dev is > 10, it toggles between
clench (motor forward 1.5 s) and unclench (reverse 0.5 s).

### `button_actuated/button_actuated.ino`
Push button on pin 8 toggles the hand: each press alternates clench (forward
1.5 s) and unclench (reverse 0.5 s). Useful for testing the mechanism without EMG.

### `motor_spin/motor_spin.ino`
Motor runs only while the button on pin 8 is held; direction alternates on each
new press. Useful for manually jogging the fingers.

### `with_several_sensors/with_several_sensors.ino`
Data logger. Prints `reading,micros_since_start` from `A4` every 10 ms at
115200 baud. This is the format of the `data/Spring 2026/` recordings. The
`NUM_SENSORS` / `WINDOW_SIZE` defines are placeholders for future multi-sensor work.

### `old_scripts/`
Earlier versions, kept for reference.

| File | Description |
|---|---|
| `hand_movement_button_4_3_24.ino` | Based on the SparkFun SIK motor example. Button on pin 7 (pull-up) toggles direction. Pins: `AIN1=13`, `AIN2=12`, `PWMA=11`. |
| `motors_button.ino` | Button on pin 13 toggles CW/CCW for 1 s using two direct motor pins (12, 11). |
| `reverse_motor_with_sensor.ino` | First EMG attempt: reverses the motor when `A0` exceeds a threshold. Pins `9/10/11`. `thresh` is never assigned, so this does not work as-is. |

---

## `python_scripts/`

Requires `pyserial`, `numpy`, `pandas`, `matplotlib`, `scikit-learn`, and
`pygame` (calibration GUI only). Serial ports (`com3`, `com5`) and many file
paths are hardcoded to individual team members' machines — edit them before running.

### Scripts

| File | Description |
|---|---|
| `read_sensor.py` | Records serial data from the Arduino to CSV, prepending a PC timestamp in ms. Edit the port and output filename before each session. Note: it does `from time import time` but calls `time.time()`, which fails — change the import to `import time`. |
| `calibrate_device.py` | Pygame GUI (from an EECE5552 Assistive Robotics project, see header) that cues REST / FLEX cycles and records an 8-electrode serial stream labeled with the stimulus to `calibration_data.csv`. Run with `--port <port> --baud_rate <rate>`. Expects 8 comma-separated values per line, so it does not work with the current single-sensor setup. |
| `make_predictions_from_sensor.py` | Live open/close prediction on the PC. Reads the serial stream, calibrates for 1500 samples (open, then closed), normalizes, and runs the logistic regression weights on 10-sample windows. |
| `moving_normalization.py` | Demonstrates sliding-window (200-sample) z-score normalization on `lk_pressed.csv` and plots the result. |
| `read_data.py` | Quick plot of a CSV grouped by `Time`. Set `csv_file` first. |
| `with_states.csv` | Output of `data_cleaning.ipynb`: normalized signal with open (0) / closed (1) `State` labels. |

### Notebooks

All notebooks follow the same pipeline: load a recording → spread duplicate
timestamps evenly → normalize → label open/closed using stopwatch lap times
(`interval_times`, see `data/images/`) → slide a window over the signal and
extract features (mean, max, min, range, last values, 20th percentile, skew,
kurtosis, std of differences) → train a classifier.

| Notebook | Data | Description |
|---|---|---|
| `data_cleaning.ipynb` | `lk_pressed_stdpos.csv` | First labeling pipeline; writes `with_states.csv`; trains a random forest on mean/max/min/range. |
| `data_cleaning2.ipynb` | `lk_pressed_sidepos.csv` | Labeling for the side-position recording (lap times set 3). Nearly identical to `Side-Training.ipynb` without the training cells. |
| `data_cleaning3.ipynb` | `lk_pressed_sidepos.csv` | Refactor of the pipeline into `read_data()` / `clean_data()` functions with min-max scaling. |
| `make_csv_with_states.ipynb` | `data3.csv` | Labels `data3.csv` with states and plots signal vs. state. |
| `model_training.ipynb` | `LK_pressed_with_states.csv` | Trains the logistic regression model; prints the coefficients and intercept used in `make_predictions_from_sensor.py` and `arduino_read_myo.ino`. |
| `logistic_regression.ipynb` | `lk_pressed_stdpos.csv` | 10-sample windows, full feature set; compares random forest and logistic regression and prints the first tree's decision rules. |
| `High-Training.ipynb` | `lk_pressed_highpos.csv` | Same pipeline on the high-arm-position recording (lap times set 2), 50-sample windows. |
| `Side-Training.ipynb` | `lk_pressed_sidepos.csv` | Same pipeline on the side-arm-position recording (lap times set 3), 50-sample windows. |
| `Spring '26/Untitled.ipynb` | `lk_pressed_sidepos.csv` | Scratch notebook for plotting / normalizing the side-position data. |

---

## `data/`

All files are headerless two-column CSVs unless noted. Values are MyoWare ENV
readings (0–1023 ADC counts; rest is roughly 30–60).

| File | Columns | Description |
|---|---|---|
| `Fall24Data/as.csv`, `ag.csv`, `afg.csv` | unix time (s), signal | Early recordings, Feb 2024. Many samples share each second. |
| `Fall24Data/3_13_v2.csv`, `3_13_v3.csv` | unix time (s), signal | Recordings from Mar 13, 2024. |
| `lk_pressed.csv` | `Time` (unix s), `Signal` (header) | Oct 2024 session, repeated hand presses. |
| `LK_pressed_with_states.csv` | index, `Time` (s from start), `Signal`, `State` (header) | `lk_pressed.csv` with open/closed labels; used by `model_training.ipynb`. |
| `data3.csv` | unix time (ms), signal | Feb 2025 session at ~50 ms per sample; labeled in `make_csv_with_states.ipynb`. |
| `Spring 2026/lk_pressed_stdpos.csv` | signal, µs since start | Arm in standard position, ~10 ms per sample. |
| `Spring 2026/lk_pressed_sidepos.csv` | signal, µs since start | Arm held to the side. |
| `Spring 2026/lk_pressed_highpos.csv` | signal, µs since start | Arm raised. |
| `Spring 2026/zp_pressed_stdpos.csv` | signal, µs since start | Second subject, standard position. Starts with a partial line (`4.0`) and reads much higher (~930) — check sensor placement/gain before using. |

### `data/images/`
Stopwatch lap screenshots used as ground truth for when the hand was open vs.
closed. Each lap is one open or closed interval, alternating; the times are
typed into the notebooks as `interval_times_1/2/3`. `P1`–`P3` are successive
scrolls of the same lap list.

| Files | Lap set |
|---|---|
| `1stP1.PNG`, `1stP2.PNG` | `interval_times_1` |
| `2ndP1.jpg`, `2ndP2.PNG`, `2ndP3.PNG` | `interval_times_2` (high position) |
| `3rdP1.PNG`, `3rdP2.PNG`, `3rdP3.PNG` | `interval_times_3` (side position) |

---

## Typical workflow

1. **Record:** flash `with_several_sensors.ino`, run `read_sensor.py`, and time
   open/close intervals with a stopwatch.
2. **Label & train:** copy the lap times into a training notebook, label the
   data, and train a classifier.
3. **Control:** flash `proportional_control.ino` for variable grip force, or
   `arduino_read_myo.ino` to test the classifier on-device.
