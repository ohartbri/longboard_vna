# longboard_vna

VNA (S-parameter) characterization of longboard traces: S21 through and crosstalk, S11, and time-domain pulse simulation using the measured transfer functions. Analysis is in `longboard_vna_highstat2.ipynb`.

## Setup

Required packages (numpy, scipy, pandas, matplotlib, ipykernel) are pinned in [dependencies.txt](dependencies.txt). To recreate the environment:

```bash
python3 -m venv venv
source venv/bin/activate
pip install -r dependencies.txt
pip install jupyterlab   # optional, for the browser UI; VS Code can use the venv kernel directly
```

Notebooks use relative paths (`data/...`), so run them with this directory as the working directory.

## Data

Exports from a Copper Mountain Technologies PLANAR TR1300/1 VNA, as CSV files in [data/](data/). Each file has 3 header lines, then columns:

`freq (Hz), S11 (dB), S11 (deg), S21 (dB), S21 (deg)`

The notebook loads them with `pd.read_csv(..., skiprows=[0,1,2], header=None, names=['freq','s11_m','s11_p','s21_m','s21_p'])`. The sweep starts at 300 kHz.

File naming (the notebook uses only the `high_stat_2/` files):

- `high_stat_2/stim_L<a>S<b>_recv_L<c>S<d>.csv`: higher-statistics runs. `L` is the trace length and `S` the strip/channel index; the name gives the stimulated and the received trace. Identical stim/recv gives a through measurement; different ones give crosstalk (e.g. `stim_L05S05_recv_L05S03`).

## Notebook

`longboard_vna_highstat2.ipynb` loads the `high_stat_2/` data, separates through and crosstalk traces, fits a single-pole low-pass model (`lowpass_mag`), and plots S21 and S11 filtered pulses. It saves the PDFs listed below.

## Method (time-domain simulation)

1. Generate a pulse `exp(-t/10 ns)` with `N = 2**16` samples over 1 µs.
2. `rfft` it, interpolate the measured S21 (or S11) magnitude and phase onto the FFT frequency grid.
3. Multiply, `irfft`, and plot against the input.

## Output figures

PDFs written by the notebook into this directory: `s21_thru_magnitude_*.pdf`, `s21_thru_phase_all.pdf`, `s21_xtalk_magnitude_all.pdf`, `s21_xtalk_phase_all.pdf`, `S21_thru_waveforms.pdf`, `S21_xtalk_waveforms*.pdf`, `S11_tdr_waveforms.pdf`.

## Caveats

- **Transfer-function construction.** The notebook builds H(f) with the `transfer_function()` helper: dB to linear with `10**(dB/20)`. The data-loading cell already converts the phase columns to radians and unwraps them in place, so the helper uses them as-is (do not convert again). Below the VNA's 300 kHz start the first point is held; above its 1.3 GHz stop the response is set to zero, so the simulated signal is band-limited to the VNA range.
- **Measurement delay is not de-embedded.** The measured S21 includes about 22 ns of group delay for the L05 traces (about 16 ns for L45S05), so the filtered pulses peak at roughly 24 ns (18 ns for L45S05) after the input, not at t = 0. The shorter delay on the longer L45 trace suggests this comes from cables/fixtures rather than the trace itself. Remove or reference it out if you need the trace response alone.
- **Circular convolution.** The filtering is an FFT product over a 1 µs window, so delays approaching 1 µs would wrap around.
- **Running headless.** The notebook can be executed with `nbclient` (installed in the venv) using the `venv` kernel. The S21 plotting cell defines the `xx` frequency grid it needs.
