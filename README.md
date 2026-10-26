# notebooks

Jupyter notebooks for lab measurements and analysis.

| Directory | Contents |
|---|---|
| [longboard_vna/](longboard_vna/) | VNA (S-parameter) characterization of longboard traces: S21 through/crosstalk, S11, and time-domain pulse simulation |
| [foil_irradiation/](foil_irradiation/) | Placeholder; `foil_irradiation.ipynb` is currently empty |

## Setup

A Python 3.14 virtual environment lives in `venv/` (numpy, scipy, pandas, matplotlib, ipykernel, plus uproot/awkward).

```bash
source venv/bin/activate
jupyter lab            # or open the notebooks in VS Code and pick the venv kernel
```

If Jupyter isn't installed in the venv, run `pip install jupyterlab`. Notebooks use relative paths (`data/...`), so run them with their own directory as the working directory.

## longboard_vna

### Data

Exports from a Copper Mountain Technologies PLANAR TR1300/1 VNA, as CSV files in [longboard_vna/data/](longboard_vna/data/). Each file has 3 header lines, then columns:

`freq (Hz), S11 (dB), S11 (deg), S21 (dB), S21 (deg)`

Notebooks load them with `pd.read_csv(..., skiprows=[0,1,2], header=None, names=['freq','s11_m','s11_p','s21_m','s21_p'])`. The sweep starts at 300 kHz.

File naming:

- `L<len>_3.csv`, `vic_L<len>_<n>.csv`: earlier runs for trace lengths L5–L55 (`vic_` = victim/crosstalk-side measurements).
- `high_stat_2/stim_L<a>S<b>_recv_L<c>S<d>.csv`: higher-statistics runs. `L` is the trace length and `S` the strip/channel index; the name gives the stimulated and the received trace. Identical stim/recv gives a through measurement; different ones give crosstalk (e.g. `stim_L05S05_recv_L05S03`).

### Notebooks

| Notebook | Description |
|---|---|
| `longboard_vna.ipynb` | First pass: S21 magnitude/phase vs. frequency for different lengths, relative to the peak with a -3 dB line, plus an FFT demo |
| `longboard_vna2.ipynb` | Copy of the above (same size/contents) |
| `longboard_vna3.ipynb` | Uses the `vic_*` data; builds an exponential-decay test pulse, multiplies its spectrum by the measured S21 and inverse-FFTs to get the filtered time-domain signal |
| `longboard_vna_highstat2.ipynb` | Current/most complete: loads `high_stat_2/` data, separates through and crosstalk keys, fits a single-pole low-pass model (`lowpass_mag`), and plots S21 and S11 filtered pulses. Saves the PDFs below |

### Method (time-domain simulation)

1. Generate a pulse `exp(-t/10 ns)` with `N = 2**16` samples over 1 µs.
2. `rfft` it, interpolate the measured S21 (or S11) magnitude and phase onto the FFT frequency grid.
3. Multiply, `irfft`, and plot against the input.

### Output figures

PDFs written by the notebooks into `longboard_vna/`: `s21_thru_magnitude_*.pdf`, `s21_thru_phase_all.pdf`, `s21_xtalk_magnitude_all.pdf`, `s21_xtalk_phase_all.pdf`, `S21_thru_waveforms.pdf`, `S21_xtalk_waveforms*.pdf`, `S11_tdr_waveforms.pdf`.

### Caveats

- **Transfer-function construction.** `longboard_vna_highstat2.ipynb` builds H(f) with the `transfer_function()` helper: dB to linear with `10**(dB/20)`. The data-loading cell already converts the phase columns to radians and unwraps them in place, so the helper uses them as-is (do not convert again). Below the VNA's 300 kHz start the first point is held; above its 1.3 GHz stop the response is set to zero, so the simulated signal is band-limited to the VNA range. The earlier code used `10**(dB/10)`, which squared the magnitude. The older notebooks (`longboard_vna.ipynb`, `longboard_vna2.ipynb`, `longboard_vna3.ipynb`) still use that code, so don't quote amplitudes from them.
- **Measurement delay is not de-embedded.** The measured S21 includes about 22 ns of group delay for the L05 traces (about 16 ns for L45S05), so the filtered pulses peak at roughly 24 ns (18 ns for L45S05) after the input, not at t = 0. The shorter delay on the longer L45 trace suggests this comes from cables/fixtures rather than the trace itself. Remove or reference it out if you need the trace response alone.
- **Circular convolution.** The filtering is an FFT product over a 1 µs window, so delays approaching 1 µs would wrap around.
- **Running headless.** The notebook can be executed with `nbclient` (installed in the venv) using the `venv` kernel. The S21 plotting cell defines the `xx` frequency grid it needs.
