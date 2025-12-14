# Neurofeedback Experimentation Suite

This project is a modular neurofeedback system designed to provide real-time visual feedback based on a user's frontal theta brainwave activity. The system is capable of connecting to an EEG data stream, processing it in real-time, and displaying feedback. It includes modes for both live neurofeedback and user-specific calibration.

## Core Components

The project is broken down into several key modules:

### `frontalTheta/` - The Main Application
This directory contains the core real-time neurofeedback application.

-   **`nfrun.py`**: The main orchestrator script for running a session. It connects all the other modules and manages the main data processing loop.
-   **`nfprocess.py`**: Contains the core scientific logic. It handles signal filtering, artifact detection, Fast Fourier Transform (FFT), and the adaptive feedback algorithm.
-   **`nfcomm.py`**: Manages all network communication. `lslreader` receives live EEG data from a Lab Streaming Layer (LSL) stream, and `udpfeedback` sends the feedback score to the visualizer via UDP packets.
-   **`nfshowsquare.py`**: A simple, standalone `pygame` application that acts as the visual feedback client. It listens for UDP packets and displays a colored square that changes based on the user's brain activity.

### `Final.py` - The Simulator
This is a self-contained `pygame` script that simulates the entire neurofeedback loop. It's useful for understanding the core logic and for demonstrations without needing a live EEG device.

### `F1/` - Data Acquisition
This directory contains C++ source code (`F1toLSLInterface`) that appears to be an interface for a specific "F1" model EEG device. Its purpose is to read data from the hardware and stream it onto the network via the LSL protocol, which `nfrun.py` can then connect to.

### `Artifact_Detection/`
This module contains scripts for the offline, manual analysis of EEG data to identify and inspect artifacts (like eye blinks or muscle movements) using the MNE library.

## How to Run the System

The system has two main modes: `calibrate` to create a user-specific model, and `nf` to run a live neurofeedback session using that model.

### Step 1: Calibration
First, you must run a calibration session to generate a model of a user's baseline brain activity. This only needs to be done once per user.

Run this command in a terminal, replacing `<your_subject_code>` with a unique identifier for the user:
```bash
python frontalTheta/nfrun.py -m calibrate -s <your_subject_code>
```
This will create a `_model.mat` file (e.g., `test_model.mat`) in the `data/` directory.

### Step 2: Neurofeedback Session
To run a live neurofeedback session, you need to run two scripts in two separate terminals.

**In your first terminal**, start the visual feedback GUI:
```bash
python frontalTheta/nfshowsquare.py
```
A `pygame` window will open.

**In your second terminal**, start the main neurofeedback engine, using the same subject code as before:
```bash
python frontalTheta/nfrun.py -m nf -s <your_subject_code>
```
The `nfrun.py` script will now connect to the EEG stream, process the data, and send feedback to the `nfshowsquare.py` window.

## Dependencies
This project requires the following Python libraries:
- `numpy`
- `scipy`
- `pygame`
- `pylsl`
- `matplotlib`

**Note on `pylsl` versioning:** This codebase was likely written for an older version of `pylsl`. If you encounter an `ImportError` for `resolve_stream`, it has been renamed to `resolve_byprop`. This change has been applied to the code.

## Signal Processing Details (`nfprocess.py`)

- **Data Window:** The system processes data in **1.0-second windows** (250 samples at 250 Hz), with the window advancing every 0.25 seconds (a 75% overlap).
- **Processing Pipeline:**
    1.  **Channel Selection**: The 'Fz' channel (index 1) is selected for processing.
    2.  **Filtering**: A 50 Hz notch filter is applied to remove electrical noise, followed by a 0.5 Hz high-pass filter to remove slow signal drift.
    3.  **Frequency Analysis**: A Fast Fourier Transform (FFT) is performed to get the power spectrum.
    4.  **Theta Power Calculation**: The system calculates the mean of the **log power** at three specific frequency bins: 4 Hz, 5 Hz, and 6 Hz. This serves as an approximation of the user's current theta power.
- **Feedback Algorithm:** The final feedback value (0.0 to 1.0) is calculated using an adaptive thresholding algorithm based on the work of Brandmeyer et al. (2020), which compares the current theta power to a dynamically adjusting window.

## Potential Improvements

- **Theta Band Power:** The current method of averaging the power of three single frequency bins (4, 5, 6 Hz) is brittle. A more robust and scientifically standard method would be to calculate the mean power across a continuous frequency band (e.g., 4 Hz to 7 Hz).
- **Artifact Rejection:** The artifact detection logic is currently disabled by default (`detectartifacts = False`) and the re-referencing code is being bypassed. Enabling and validating this logic would improve data quality.
