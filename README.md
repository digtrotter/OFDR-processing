# OFDR Acquisition & Processing

A Python application for automated data acquisition, digital signal processing (DSP), and real-time visualization of Optical Frequency Domain Reflectometry (OFDR) measurements in optical fibers.

This tool communicates over Ethernet (SCPI over TCP/IP sockets) with a **Santec TSL-570 Tunable Laser** and a **Tektronix MSO 2 Series (MSO24) Oscilloscope**. It corrects for laser sweep non-linearities using an auxiliary interferometer (K-clock) and maps optical reflections along the fiber after data processing.

---

## Table of Contents
1. [Software Setup](#software-setup)
2. [Physical Setup](#physical-setup)
3. [Theory Overview](#theory-overview)
4. [Signal Processing Pipeline](#signal-processing-pipeline)
5. [Instruments](#instruments)
6. [Data Storage](#data-storage)

---

## Software Setup

### Project Structure

```
OFDR-processing/
|-- setup.py           # Class definitions, equipment configuration and HDF5 saving/loading
|-- processing.py      # DSP routines (scaling, peak detection, resampling, FFT...)
|-- gui.py             # CustomTkinter interface and interactive Matplotlib plots
|-- requirements.txt   # Python package dependencies
`-- README.md          # Project documentation
```

### Installation
```bash
# Clone the repository
git clone https://github.com/digtrotter/OFDR-processing.git
cd OFDR-processing

# Create and activate a virtual environment
python3 -m venv venv
source venv/bin/activate

# Install dependencies
pip install -r requirements.txt
```

---

## Physical Setup

```
                       +-------------------------+
                       |  Santec TSL-570 Laser   |
                       +------------+------------+
                                    |
                                    v
                            +---------------+
                            |  75/25 Split  |
                            +----+------+---+
            75% Measurement Arm  |      |  75% Reference Arm
                                 v      v  
           +-----------------------+  +---------------------+
           |  Main Interferometer  |  |  Auxiliary K-Clock  |
           |  (Fiber Under Test)   |  |   Interferometer    |
           +----------+------------+  +----------+----------+
                      |                         |
                      v                         v
             [Balanced Photodetector]    [Photodetector 2]
                      |                         |
                      v                         v
           +------------------------------------------------+
           |          Tektronix MSO24 Oscilloscope          |
           |   - CH1: Main measurement signal               |
           |   - CH3: Auxiliary K-clock reference signal    |
           +------------------------------------------------+
```

---

## Theory Overview

### What is OFDR?
Optical Frequency Domain Reflectometry (OFDR) is a high-resolution distributed sensing technique usually used to measure temperature and strain across the whole length of an optical fiber cable. 

1. A tunable laser sweeps across a wavelength range.
2. The light is split into a measurement arm (fiber under test) and a reference arm.
3. Reflections along the fiber interfere with the reference beam, producing beat frequencies proportional to the distance of these points.
4. Performing a Fast Fourier Transform (FFT) allows us to calculate the reflection profile along the length of the fiber with millimeter-scale resolution.

### Why use a K-Clock?
In practice, tunable lasers do not sweep at a perfectly constant speed over time. If data is sampled at fixed time intervals, these tuning variations blur and widen the reflection peaks, reducing measurement accuracy.

To fix this, part of the laser output is routed through a separate reference interferometer with a fixed path delay (the **K-clock**):
- The K-clock produces interference fringes with peaks at equal optical frequency steps.
- By finding these peaks and resampling the measurement data at those exact moments, the signal is realigned to be linear in optical frequency.
- The resulting FFT produces sharp, well-defined reflection peaks.

---

## Signal Processing Pipeline

The signal processing workflow is implemented in `processing.py`:

1. **Waveform Restoration**:
   - Converts raw 16-bit integers from the oscilloscope into voltage values using each channel's metadata.
   - Builds the time axis using the oscilloscope time increment.

2. **K-Clock Peak Extraction (`interpolPeaks`)**:
   - Upsamples the K-clock signal by 10x using cubic splines to prepare for peak extraction
   - Detects peaks using `scipy.signal.find_peaks`. These peak positions define points of equal optical frequency intervals.

3. **Optical Frequency Resampling (`interpolData`)**:
   - Interpolates the main measurement signal (CH1) using a cubic spline.
   - Resamples it at the timestamps of the detected K-clock peaks, bringing data to K-domain.
   - This step corrects for non-linear sweep speeds and brings data to K-domain.

4. **Fourier Transform (`process_fft`)**:
   - Removes mean offset and detrends the resampled signal.
   - Computes a real-valued Fast Fourier Transform (`rfft`) to obtain the beat frequency spectrum.
   - This step brings data to the frequency domain

5. **Distance & Reflectivity Mapping (`process_space`)**:
   - Converts beat frequencies to physical distance along the fiber in meters:
     - Distance = (Speed of Light * Beat Frequency) / (2 * Group Index * Sweep Rate)
     - Group refractive index is set to 1.468 (typical for standard single-mode silica fiber like SMF-28).
   - Normalizes magnitudes and computes reflectivity in decibels (dB).

---

## Instruments

### Connections

- Both instruments communicate via TCP/IP using SCPI commands, preferably over ethernet for lower latency
- Instrument communication uses the pure-Python PyVISA backend (`@py`), so no proprietary drivers, such as NI-VISA, are required.

### Automation

- **`Dados` class**:
  - Container for channel data (raw values, oscilloscope scaling parameters, laser sweep settings, and processed axes).
  - Includes `saveFile()` and `loadFile()` methods for saving and loading files.

- **`MSO` class**:
  - Connects to the Tektronix MSO24 oscilloscope over a socket.
  - Configures 16-bit binary waveform transfers (`setupWFMO`) and queries waveforms' curve data.

- **`TSL` class**:
  - Connects to the Santec TSL-570 laser over a socket.
  - Sets wavelength limits and sweep speed, starts the sweep, and monitors completion.

- **Automated Parameter Setup**:
  - Calculates the expected sweep duration based on the chosen wavelength span and speed.
  - Automatically calculates the required oscilloscope sample rate for the selected record length.
  - Adjusts for oscilloscope sample rate quantization: if the oscilloscope rounds up the sample rate, the routine scales the rate down by 0.49x, forcing a round down, so the acquisition window covers the entire sweep duration

---

## Data Storage

Acquired data can be saved in the **HDF5** (`.h5`) format:
- `aq-<name>.h5`: Measurement channel data.
- `clk-<name>.h5`: K-clock channel data.

Each file contains:
- **`valores`**: 1D dataset of raw 16-bit integer samples from the oscilloscope.
- **Attributes**:
  - `numPts`: Number of points in the acquisition.
  - `zero`: Oscilloscope vertical zero offset.
  - `ymult`: Oscilloscope vertical multiplier.
  - `xincr`: Time increment per sample (seconds).
  - `speed`: Laser sweep speed (nm/s).
  - `initial_wavelen`: Start wavelength (nm).
  - `final_wavelen`: Stop wavelength (nm).