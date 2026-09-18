# Codebase Explanation: sfr-fsar-rnt (SKB Repository Safety Assessment)

## Executive Summary
- **What it is:** A radiation transport modeling framework for the Swedish Nuclear Fuel and Waste Management Company (SKB) to assess long-term safety of deep geological repositories for nuclear waste (SFR facility at Forsmark).
- **Main architecture:** Three-stage deterministic/probabilistic simulation pipeline (nearfield → farfield → biosphere) driven by TOML configuration, implemented in Python with NumPy/SciPy for sparse ODE solving.
- **Key technology:** Custom sparse Jacobian ODE solver with Monte Carlo uncertainty quantification, HDF5-based input/output for parametric studies.
- **Getting oriented:** Start with `CLAUDE.html` and `adding_new_CC.html` for overview and calculation-case workflow; read `adding_new_model.html` for adding new vault designs.

## Project Overview
- **What it does:** This codebase implements radionuclide transport calculations for SKB's safety assessments of the SFR repository. It models how radioactive materials might migrate from waste packages through engineered barriers (concrete, bentonite) and host rock to the biosphere over timescales of up to one million years.
- **Who it is for:** SKB safety analysts, nuclear waste disposal researchers, and model developers working on PSAR (Periodic Safety Review) and SAR (Safety Assessment Report) documents.
- **Current state:** Production-grade research code for regulatory submissions. The repository `pgastrand.github.io` hosts generated HTML documentation for the larger `sfr-fsar-rnt` project. The actual code is in three Git submodules (`skbrnt`, `sfr-fsar-code`, `sfr-fsar-data`) managed at `~/git/skb_rnt/` in the developer's workspace.

## Tech Stack
- **Language:** Python 3.11+ (uses `tomllib` from stdlib)
- **Core libraries:** NumPy (array operations), SciPy (sparse linear algebra), H5Py (HDF5 I/O), Joblib (parallelization)
- **Packaging:** No package management - workspace runs from source with `PYTHONPATH=$PWD`
- **Configuration:** TOML files for calculation cases (CC), HDF5 for input parameters and output results
- **Testing:** No automated test suite - validation via deterministic "smoke tests" followed by probabilistic runs
- **Build tools:** Custom scripts (`generate_input_data.py` regenerates HDF5 from Excel sources)

## Project Structure
The `pgastrand.github.io` repository is a documentation site generated from an Obsidian vault. The actual codebase is split across three submodules:

```text
project/
├── index.html              # Homepage (generated from Obsidian)
├── CLAUDE.html            # AI assistant orientation guide
├── README.html            # High-level architecture overview
├── adding_new_CC.html     # Calculation case workflow guide
├── adding_new_model.html  # Vault model addition guide
├── CODE_SMELLS.html       # Structural code audit report
└── [other markdown pages] # Supporting documentation
```

The actual code structure (in submodules under `~/git/skb_rnt/`) is:

```text
skbrnt/                         # Reusable framework library
├── core/
│   └── rnt_model.py           # Sparse ODE solver core
├── nearfield/                 # Nearfield (vault) domain
│   └── sfr/                   # SFR-specific models
│       ├── bma1.py, bma2.py   # BMA vault variants
│       └── nearfield_model.py # Base classes
├── farfield/                  # Farfield (rock) domain
├── biosphere/                 # Biosphere (surface) domain
└── util/                      # Utilities (lookup tables, charting)

sfr-fsar-code/                 # Calculation runner & utilities
├── run_cc.py                  # Main CLI entry point
├── sum_repo_cc.py, aggregate.py # Post-processing scripts
├── cc/                        # CC variant update modules
├── util/
│   ├── cc_config.py           # Configuration management
│   ├── simulation_util_sfr.py # Dispatch table
│   ├── parameter_util_*.py    # Parameter loaders per vault
│   └── input_data_util_sfr.py # Input data utilities

sfr-fsar-data/                 # Input data & configurations
├── config/                    # TOML calculation cases (CC0–CC35, CC516)
├── excel/                     # Source Excel workbooks
├── BaseCase.h5, CC*.h5        # HDF5 parameter files
└── results/                   # Output (generated, not committed)
```

## Architecture and Design
- **Architecture style:** Pipeline architecture with layered domains (nearfield → farfield → biosphere), each domain producing source terms for the next via HDF5 files.
- **Core algorithm:** Sparse ODE system with time-dependent coefficients. Jacobian is pre-assembled in COO format, converted to CSR/CSC for solving. Uses event-driven restarts for multi-phase scenarios (e.g., climate transitions).
- **Data flow:**
  1. TOML config defines calculation case parameters (TOML → CCConfig resolution)
  2. Parameter utility (`parameter_util_2bma.py`) loads parameters from HDF5
  3. `RNTModel` constructs sparse ODE system with time-dependent `de`, `kd`, etc.
  4. Jacobian assembly (once, per iteration) + solve loop
  5. Output to HDF5 with groups `{nearfield,farfield,biosphere}/{repository}/{endpoint}`

- **State management:** State vector (`y`) contains concentrations per compartment. State evolves via ODE `dy/dt = f(t, y)` with time-dependent coefficients from material degradation.
- **Boundaries:** 
  - Domain separation: each domain (`nearfield`, `farfield`, `biosphere`) is a distinct module
  - Configuration separation: `CCConfig` (TOML) vs `Config` (base defaults)
  - I/O separation: HDF5 input/output, Excel sources for regeneration

- **Key patterns:**
  - **Strategy via import:** `cc/<NAME>.py` modules injected via `update = 'GLACIATION'` in TOML
  - **Context manager for simulation:** `SimulationUtilSFR` manages setup/teardown with explicit entry point
  - **Sparse ODE system:** COO → CSR/CSC for efficiency in large systems (100+ compartments)

## Entry Points and Runtime Behavior
- **Main entry point:** `sfr-fsar-code/run_cc.py`
  - CLI flags: `--cc <N>`, `--repository <name>`, `--domain <nearfield|farfield|biosphere>`, `--n-iter <N>`, `--n-cores <N>`
  - `--deterministic-only` (n_iter=1) vs `--probabilistic-only` (Monte Carlo)
- **Calculation cases:** `sfr-fsar-data/config/CC*.toml` files define parameters like repositories, radionuclides, time steps
- **CC variant modules:** Dynamically imported when TOML sets `update = '<NAME>'` (e.g., `cc/GLACIATION.py` for climate transitions)

## Key Abstractions and Domain Concepts
- **`RNTModel` (skbrnt/core/rnt_model.py):** Abstract base class for all vault models. Implements sparse ODE assembly and solve loop. Subclassed by vault-specific implementations (BMA1, BMA2, BRT, etc.)
- **`NearfieldModel` / `FarfieldModel` / `BiosphereModel`:** Domain-specific extensions of `RNTModel`
- **`ParameterUtil*` classes:** Adapters between HDF5 parameter groups and model instances. Each vault type has its own (e.g., `ParameterUtil2BMA`, `ParameterUtilSilo`)
- **`Compartment`:** Physical region with defined geometry, material, and sorption properties. A vault contains multiple compartments (e.g., `WasteWall`, `ConstructionConcrete`, `Gravel`).
- **Material properties:** Six core parameters:
  - `porosity`: Porosity evolution over time (degradation states)
  - `kd`: Distribution coefficient (sorption)
  - `srf`: Sorption ratio (default 1)
  - `de`: Effective diffusivity (time-dependent via degradation)
  - `density`: Bulk density
  - `volume`: Volume (constant)

## Configuration and Environment
- **Environment:** No environment variables required - all configuration via TOML/HDF5
- **Configuration precedence (highest to lowest):**
  1. User overrides at `~/.psu/fsar_sfr_python.toml`
  2. CC config at `sfr-fsar-data/config/CC<N>.toml`
  3. `CCConfig._default` in `sfr-fsar-code/util/cc_config.py`
  4. `Config._default` in `skbrnt/util/config.py`
- **HDF5 structure:** Groups organized as `{global}/{nearfield}/{farfield}/{biosphere}/{repository}/{parameters}`

## Development Workflow
- **Install dependencies:** `pip install numpy scipy h5py tqdm joblib` (no `pyproject.toml`)
- **Run locally:** 
  ```bash
  cd sfr-fsar-code
  export PYTHONPATH="$PWD:$PYTHONPATH"  # Required
  python run_cc.py --cc CC1
  ```
- **Run tests:** No automated tests. Validation workflow:
  1. Deterministic smoke test (fast): `python run_cc.py --cc CC1 --repository 2BMA --domain nearfield --deterministic-only`
  2. Full chain deterministic run
  3. Probabilistic validation with small N if deterministic looks good
- **Lint/typecheck:** None - no linting setup
- **Build:** Regenerate HDF5 from Excel: `python generate_input_data.py`
- **Output:** Results stored in `results/nearfield/hdf/`, `results/farfield/hdf/`, etc.

## Testing Strategy
- **No automated test suite exists.** Validation relies on:
  - Deterministic runs as "smoke tests" before probabilistic runs
  - Cross-validation with Ecolego (SKB's legacy tool)
  - Physical sanity checks on output (mass balance, expected time scales)
- **Gap:** Missing unit/integration test coverage - code is in production but lacks formal verification

## Operational Notes
- **Deployment model:** Local HPC/cluster execution - no containerization or cloud deployment
- **Parallelization:** Joblib with `loky` backend (prevents nested BLAS threading issues on macOS)
- **Output format:** HDF5 files with groups `{domain}/{repository}/{endpoint}` containing results per radionuclide
- **Scaling:** Deterministic single solve ≈ seconds; probabilistic 1000 iterations ≈ hours on multi-core machine
- **Failure modes:** 
  - Sparse matrix Ill-conditioning (handled by ODE solver tolerance)
  - Missing HDF5 groups (validation in `parameter_util_*.py`)
  - Configuration drift (Shotgun Surgery pattern makes adding new repositories error-prone)

## Onboarding Guide
1. **Start with `README.html`** - Provides high-level architecture overview with mermaid diagrams showing the three-domain pipeline
2. **Read `CLAUDE.html`** - Contains common commands, submodule structure, and working conventions
3. **Run a smoke test**:
   ```bash
   cd sfr-fsar-code
   python run_cc.py --cc CC1 --repository 2BMA --domain nearfield --deterministic-only
   ```
4. **Study `adding_new_CC.html`** - Walks through the full calculation case lifecycle: TOML → solution → output
5. **Read `adding_new_model.html`** - Line-by-line checklist for adding new vault models (3BMA example)

## Open Questions or Uncertainties
- **Missing source code:** The actual Python code is in Git submodules not present in this repo (`skbrnt`, `sfr-fsar-code`, `sfr-fsar-data`)
- **Test coverage:** No automated tests found - need to confirm if tests exist elsewhere or if validation is done via manual comparison
- **Submodule pointers:** The `.gitmodules` file and submodule commit references are not visible in this docs-only repo
- **Dependency versions:** `numpy`, `scipy`, `h5py` dependencies are not pinned - need to confirm compatible versions used in production
- **HDF5 format stability:** The HDF5 schema is not documented outside of the code - schema evolution history is unclear

---

**Conversation saved on:** 2026-05-15
**Repository:** `/Users/pg/git/pgastrand.github.io`
