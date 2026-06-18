# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

3dentification classifies 3D-printing plastics (PLA, ABS, PETG, etc.) by combining Near-Infrared (NIR) spectroscopy with computer vision for colour. A custom sensor board (based on the open-source [Plastic Scanner](https://github.com/Plastic-Scanner)) sweeps 8 NIR LED wavelengths, a photodiode reads reflected intensity through an ADS1256 ADC, and a scikit-learn model trained on collected scans predicts the material. The repo holds three independent codebases: the device **firmware**, the Python **scripts** (data collection, training, GUI), and **rnd** (experimental prototypes — not production).

## Commands

### Firmware (`firmware/`, PlatformIO + Arduino)
```
pio run                      # build
pio run --target upload      # build & flash
pio device monitor           # serial monitor (115200 baud)
```
Target board is `uno` (see `platformio.ini`). Mode is selected with `#define CLI_MODE` at the top of `app/main.cpp`: `false` = SERIAL_MODE (emits JSON scan data, used by the Python scripts), `true` = interactive CLI.

### Python scripts (`scripts/`)
Always run from inside the `scripts/` directory — modules import via paths relative to it (`from lib.postprocess ...`, `from utils.read_serial ...`).
```
python3 -m venv venv && source venv/bin/activate
pip install -r requirements.txt   # root and scripts/ requirements.txt are identical

python app.py                     # GUI scanner (customtkinter); needs serial board + camera
python -u generation.py           # generate synthetic data from collected ground truth
python -u train.py -e -m -t       # train on mock/synthetic data (smoke test — runs with no hardware)
python data_collection.py         # collect scans over USB serial from the board
python analysis.py                # regenerate poster/report figures
```

`train.py` only acts when `-e`/`--execute` is passed; other flags are no-ops without it. Key flags: `-t` train, `-o` optimize (halving grid search), `-s` save model, `-m` use synthetic data, `-v` verbose (confusion matrices), `-c` check overfitting, `-r` random shuffle.

There is no automated test suite for the Python code; `train.py -e -m -t` against mock data is the closest thing to a runnable check.

## Architecture

### The scan data contract
A single scan is a small CSV/DataFrame with 8 LED columns (`led0`..`led7`) and rows `intensity`, `variance`, `ambient` (plus a `units` column in some formats). This same shape flows from firmware → serial → `data_collection.get_scan()` → `SpectraGen` → model. LED wavelengths in nm are fixed in order as `LEDS = [850, 940, 1050, 890, 1300, 880, 1550, 1650]` (defined in `app.py`); index position, not wavelength, is what ties columns together across the pipeline.

### `lib/postprocess.py` — `SpectraGen` (the core signal engine)
Shared by `app.py` and `train.py`. Turns a raw scan into a feature vector:
- **noise subtraction**: `intensity − ambient`, negatives clamped to 0
- **calibration subtraction**: subtract `cal_values` (an "empty"/no-sample reference scan that captures LED scatter), negatives clamped to 0
- optional **ratios vector** (`create_ratios_vector`): all pairwise LED ratios flattened — an alternate feature representation
- optional **min/max normalization** against reference values

### Calibration (essential — easy to misunderstand)
Calibration does **not** use a known/reference plastic. It is a **full scan taken with an empty chamber (no sample present)**. There are two distinct noise corrections, and they are different things:

1. **Ambient** — captured *inside every scan automatically*. The firmware reads each LED twice (on, then off); the "off" reading is ambient room/sunlight. `subtract_noise()` removes it per-scan: `intensity − ambient`.
2. **Calibration (`cal_values`)** — a *separate, manual* empty-chamber scan. Clicking **Calibrate** in the GUI runs `cal = Spectra.subtract_noise(empty_chamber_scan)` (i.e. `intensity − ambient` for the empty chamber) and stores it. This captures **LED scatter** — light bouncing off the housing/optics back to the photodiode without hitting a sample.

A real sample is then `filtered_spectra() = (intensity − ambient) − cal_values`. So calibration is conceptually a "dark + scatter frame," not a material reference. This is why every dataset ships calibration CSVs and why scanning is gated behind calibration in the GUI — the model was trained on scatter-subtracted features and expects the same. In training, `init_spectra_cal_ref` wires the matching calibration file into a `SpectraGen` before feature extraction (a scan's `id<n>` in its filename selects which calibration CSV to use).

Separately, `SpectraGen` has a `ref_values` concept used only by the optional `normalize()` min/max scaling — *that* is where a known reference would go, but it is mostly unused (e.g. `dataset1/info.txt`: "No reference is created so that data is not normalized"). Don't confuse `ref_values` (normalization, optional) with `cal_values` (scatter subtraction, always used).

### Training pipeline (`train.py`)
`gen_datasets` walks `data/<dataset>/{train,val}/<label>/*.csv`, runs each scan through `SpectraGen`, and builds X/y arrays. Labels are integer-coded (see the `labels` dict in `app.py`: 0=abs, 1=pla, 2=empty, 3=non_plastic, 4=petg, 5=plastic). It trains a panel of scikit-learn classifiers (GradientBoosting, HistGradientBoosting, MLP, SVC/RBF, KNN, VotingClassifier, etc.). Optionally folds in CV colour features (`add_colour_data`).

### Data layout (`scripts/data/`)
`dataset1`..`dataset4` plus `synthetic`. **`dataset4` is the most complete dataset.** Each contains per-session calibration CSVs at its root, a `colours.json`, and `train/`+`val/` split into per-material subfolders. Filenames encode metadata: `bv1_id<calId>_<material>_<colour>_<date>_<unixtime>.csv` — the `id<n>` ties a scan to the calibration scan with the matching id.

### Trained models (`scripts/models/`, `scripts/model_info/`)
Pickled scikit-learn models. The filename is the metadata: `model<n>_<Classifier>_<accuracy>_<labels...>_<date>.pickle`. `app.py` loads a pickle and calls `predict` on the `SpectraGen` feature vector.

### GUI (`scripts/app.py`)
`customtkinter` app: pulls a live scan via `data_collection.get_scan()`, processes it with `SpectraGen`, runs the loaded model, and simultaneously samples average RGB from a `cv2.VideoCapture` feed (`lib/colour.py`: `classify_color` / `get_colour_name`). **The camera device is hardcoded** — `cv2.VideoCapture("/dev/video0")` in `app.py` must be edited to match the local machine.

### Firmware internals (`firmware/`)
`app/main.cpp` is the entry point; `drivers/ads1256` (24-bit ADC, SPI) and `drivers/tlc59208` (LED driver, I²C) are the hardware drivers; `drivers/json` is bundled ArduinoJson for serializing scan output. `app/cli.*` is the optional interactive command shell. `dev/` and `test/` hold alternate `main.cpp` entry points (not built by the default `app` src_dir).

## Reference & experimental code (do not modify as production)
- `ref/plastic-scanner/` — vendored upstream Plastic Scanner firmware + KiCad hardware, kept for reference.
- `rnd/` — experimental prototypes: a Dockerized Flask app, `plastic-cv` (OpenCV object detection), and a Rust serial-reader (`rnd/rust`). Separate from the main pipeline.
- `arduino/test_usb_serial/` — minimal sketch for verifying USB serial connectivity.

## Conventions
- Git commits must never mention AI or Claude.
