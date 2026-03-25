# CLAUDE.md — AI Assistant Guide for PROSAIL

## Project Overview

PROSAIL is a Python package providing implementations of the PROSPECT and SAIL radiative transfer models for remote sensing and vegetation analysis. It models:

- **PROSPECT**: Leaf optical properties (reflectance/transmittance spectra) from biochemical parameters
- **SAIL (FourSAIL)**: Canopy-level reflectance from leaf optical properties and canopy structure
- **PROSAIL**: Combined leaf+canopy model (the primary user-facing workflow)

**Version:** 2.0.5 | **License:** GPLv3 | **Python:** 3.10, 3.11, 3.12

---

## Repository Structure

```
prosail-1/
├── prosail/                    # Main package
│   ├── __init__.py             # Public API exports
│   ├── prospect_d.py           # PROSPECT model (versions 5, D, PRO)
│   ├── sail_model.py           # run_sail(), run_prosail(), run_thermal_sail()
│   ├── FourSAIL.py             # Core FourSAIL radiative transfer engine
│   ├── spectral_library.py     # Spectral data loader
│   ├── prospect5_spectra.txt   # PROSPECT-5 spectral coefficients
│   ├── prospect_d_spectra.txt  # PROSPECT-D spectral coefficients
│   ├── prospect_pro_spectra.txt# PROSPECT-PRO spectral coefficients
│   ├── soil_reflectance.txt    # Two soil spectra (dry/wet)
│   └── light_spectra.txt       # Illumination spectra
├── tests/
│   ├── test_prospect.py        # PROSPECT unit tests (7 tests)
│   ├── test_prosail.py         # PROSAIL integration tests (4 tests)
│   ├── environment.yml         # Conda test environment spec
│   └── data/                   # Reference data for test validation
│       ├── REFL_CAN.txt
│       ├── prospect5_spectrum.txt
│       ├── prospect_d_test.mat
│       ├── prospect_pro_test.txt
│       ├── tav_alpha40.mat
│       └── verhoef_et_al2007_fig6a.csv
├── recipe/                     # Conda package recipe
├── .github/workflows/main.yml  # GitHub Actions CI
├── requirements.txt            # Runtime dependencies
├── setup.py                    # Package configuration
└── README.md                   # User-facing documentation
```

---

## Development Workflows

### Installation

```bash
# Editable install for development
pip install -e .

# Install dependencies
pip install -r requirements.txt

# Also install test dependencies
pip install pytest pytest-cov coverage flake8
```

### Running Tests

```bash
# Run all tests
pytest

# Run with coverage
pytest --cov

# Run specific test files
pytest tests/test_prospect.py
pytest tests/test_prosail.py
```

Tests validate against MATLAB reference outputs and published spectra. All tests should pass before committing.

### Linting

```bash
# Hard errors (CI will fail on these)
flake8 --select=E9,F63,F7,F82 prosail/

# Full style check (warnings only)
flake8 --max-line-length=127 --max-complexity=10 prosail/
```

CI uses the same flake8 configuration. Fix `E9`, `F63`, `F7`, `F82` errors; treat other style issues as warnings.

### CI/CD

GitHub Actions (`.github/workflows/main.yml`) runs on every push/PR:
1. Install from `requirements.txt`
2. Editable install: `pip install -e .`
3. Flake8 lint check
4. `pytest`

Matrix: Python 3.10, 3.11, 3.12 on Ubuntu latest.

---

## Public API

All public functions are exported from `prosail/__init__.py`:

### `prosail.run_prospect()`

Runs the PROSPECT leaf optical model. Returns wavelengths, reflectance, transmittance (each 2101-element arrays at 1nm resolution, 400–2500nm).

```python
wv, refl, trans = prosail.run_prospect(
    n,          # Leaf layers [-]
    cab,        # Chlorophyll a+b [µg cm⁻²]
    car,        # Carotenoids [µg cm⁻²]
    cbrown,     # Brown pigments [0-1]
    cw,         # Leaf water [cm]
    cm,         # Dry matter [g cm⁻²]
    ant=0.0,    # Anthocyanins [µg cm⁻²] (PROSPECT-D/PRO only)
    prot=0.0,   # Proteins [g cm⁻²] (PROSPECT-PRO only)
    cbc=0.0,    # Carbon-based constituents [g cm⁻²] (PROSPECT-PRO only)
    prospect_version="D",  # "5", "D", or "PRO"
)
```

### `prosail.run_prosail()`

Combined PROSPECT + SAIL model — the primary entry point for most use cases.

```python
refl = prosail.run_prosail(
    n, cab, car, cbrown, cw, cm,  # PROSPECT leaf params
    lai,        # Leaf Area Index [m² m⁻²]
    lidfa,      # Leaf angle distribution mean [°]
    hspot,      # Hotspot parameter [-]
    tts,        # Solar zenith angle [°]
    tto,        # Observer zenith angle [°]
    psi,        # Relative azimuth angle [°]
    ant=0.0, prot=0.0, cbc=0.0,
    prospect_version="D",
    lidf_type="verhoef",  # or "campbell"
    factor="SDR",         # "SDR", "BHR", "DHR", "HDR", "ALL", "ALLALL"
    rsoil0=None,          # Custom soil spectrum (2101 array)
    rsoil=1.0, psoil=1.0, # Soil brightness/moisture scalars
    soil_spectrum1=None, soil_spectrum2=None,
)
```

### `prosail.run_sail()`

SAIL-only model accepting pre-computed leaf reflectance/transmittance.

### `prosail.run_thermal_sail()`

Thermal extension of SAIL (experimental).

### `prosail.get_spectra()`

Returns a `Spectra` namedtuple with all spectral coefficient arrays loaded from embedded data files.

---

## Code Conventions

### Naming

- **Functions**: `snake_case` (e.g., `run_prospect`, `volscatt`, `calctav`)
- **Parameters**: Single/double-letter abbreviations matching scientific notation:
  - `n` = leaf layers, `cab` = chlorophyll, `cw` = water, `cm` = dry matter
  - `lai` = Leaf Area Index, `tts` = solar zenith, `tto` = observer zenith
  - `psi` = relative azimuth, `hspot` = hotspot parameter
- **Data structures**: `namedtuple` for spectral groupings (e.g., `Spectra`, `Prospect5Spectra`)

### Spectral Arrays

All spectral data arrays are exactly **2101 elements**, representing 400–2500nm at 1nm resolution. This is a strict invariant — never change array sizes. Functions assert or depend on this shape.

### Performance Patterns

- **Numba JIT**: Performance-critical functions in `FourSAIL.py` use `@numba.jit()` with explicit type signatures in `nopython=True, cache=True` mode:
  ```python
  @numba.jit("Tuple((f8, f8, f8, f8))(f8,f8,f8,f8)", nopython=True, cache=True)
  def volscatt(tts, tto, psi, ttl):
      ...
  ```
- **LRU Cache**: Expensive initialization functions use `@lru_cache(maxsize=16)`
- Do not use Python-level objects, dicts, or control flow inside Numba-jitted functions — only NumPy operations

### Docstrings

Use comprehensive NumPy-style docstrings with parameter units, ranges, and literature references:

```python
def run_prospect(n, cab, ...):
    """The PROSPECT model, versions 5, D and PRO.

    Parameters
    -----------
    n: float
        The number of leaf layers. Unitless [-].
    cab: float
        The chlorophyll a+b concentration. [ug cm^{-2}].
    ...
    Returns
    -------
    3 arrays of the size 2101: wavelengths [nm], reflectance, transmittance.
    """
```

### Error Handling

- Validate `factor` parameter against `["SDR", "BHR", "DHR", "HDR", "ALL", "ALLALL"]`
- Raise `ValueError` for invalid inputs
- Assert array dimensions where shape invariants matter

---

## Key Implementation Details

### Module Dependency Chain

```
spectral_library.py  →  loads embedded .txt data files
prospect_d.py        →  uses spectral_library, implements PROSPECT
FourSAIL.py          →  core RT engine (Numba-optimized)
sail_model.py        →  orchestrates prospect_d + FourSAIL; public API
__init__.py          →  re-exports public API
```

### PROSPECT Versions

| Version | Extra params | File |
|---------|-------------|------|
| "5"     | none        | `prospect5_spectra.txt` |
| "D"     | `ant`       | `prospect_d_spectra.txt` |
| "PRO"   | `ant`, `prot`, `cbc` | `prospect_pro_spectra.txt` |

### Reflectance Factors (`factor` parameter)

| Value    | Description |
|----------|-------------|
| `"SDR"`  | Bi-directional reflectance (default) |
| `"BHR"`  | Bi-hemispherical reflectance |
| `"DHR"`  | Directional-hemispherical reflectance |
| `"HDR"`  | Hemispherical-directional reflectance |
| `"ALL"`  | Returns SDR, BHR, DHR, HDR as tuple |
| `"ALLALL"` | All factors including upward components |

### Leaf Angle Distribution Types (`lidf_type`)

- `"verhoef"`: Two-parameter Verhoef distribution (`lidfa`, `lidfb`)
- `"campbell"`: Ellipsoidal distribution (single `lidfa` parameter)

### Soil Model

Soil reflectance is computed as: `rsoil = psoil * rsoil1 + (1 - psoil) * rsoil2`
where `rsoil1` and `rsoil2` are dry/wet soil spectra from `soil_reflectance.txt`.
A brightness scalar `rsoil` can be applied on top. A fully custom spectrum can be passed via `rsoil0`.

---

## Testing Conventions

- Tests use `pytest` fixtures with `tmpdir` for data file isolation
- Numerical comparisons use `np.testing.assert_array_almost_equal` with appropriate decimal places
- Reference data in `tests/data/` includes MATLAB `.mat` files (loaded with `scipy.io.loadmat`) and text files
- Tests validate against: Jussieu PROSAIL reference implementation, published MATLAB outputs, and Verhoef et al. (2007) Fig. 6a

---

## What NOT to Do

- Do not change spectral array sizes from 2101 elements
- Do not use Python objects or non-NumPy operations inside `@numba.jit` functions
- Do not add Python 2 compatibility (project is Python 3 only despite some legacy comments in `setup.py`)
- Do not modify `.txt` spectral data files — these are validated against external references
- Do not add new public API functions without updating `prosail/__init__.py` exports
