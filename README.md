# eDisGo-lobaflex – Load Balancing Flexibilities

![load_balance_flat](https://user-images.githubusercontent.com/51374526/215474357-33537736-33fa-4777-b681-f1d957aac03e.png)

**lobaflex** is the code base of the Master's thesis

> **Evaluating load-balancing potential for the energy system provided through distribution grid flexibilities**
> Julian Endres – Technische Universität Berlin, Department of Digital Transformation in Energy Systems, in cooperation with the [Reiner Lemoine Institut (RLI)](https://reiner-lemoine-institut.de/). Submitted April 10, 2023.
>
> Primary reviewer: Prof. Dr. Tom Brown (TU Berlin) · Supervisors: Dr.-Ing. Fabian Neumann (TU Berlin), M.Sc. Birgit Schachler & M.Sc. Anya Heider (RLI)

📄 [Thesis (PDF)](2023-04-10_Thesis.pdf) · 📊 [Defense presentation (PDF)](2023-04-26_MA_presentation.pdf)

---

## Contents

- [Motivation](#motivation)
- [Approach](#approach)
- [Key findings](#key-findings)
- [Repository structure](#repository-structure)
- [Installation](#installation)
- [Configuration](#configuration)
- [Running the workflow](#running-the-workflow)
- [Results and analysis](#results-and-analysis)
- [Limitations](#limitations)
- [Related tools](#related-tools)
- [License](#license)

## Motivation

The energy transition brings large numbers of *new consumers* such as heat pumps (HP)
and battery electric vehicles (BEV) into the distribution grids. Operated naively, they
drive expensive grid reinforcement. Operated flexibly, they can avoid part of that
reinforcement **and** offer flexibility to the overlying transmission system.

The thesis asks two questions:

1. **How much grid reinforcement can be avoided** if flexible HPs (with thermal storage)
   and BEVs are dispatched to minimise line loading?
2. **How much load-balancing potential (LBP)** can a distribution grid offer to the
   overlying grid at the HV/MV substation, and how does it grow with additional grid
   reinforcement?

## Approach

The analysis was carried out for six synthetic medium-voltage grid districts (MVGDs)
generated with [ding0](https://github.com/openego/ding0). Load and generation follow
the *eGo100* scenario of the open_eGo project, and the penetration of flexible units
follows NEP C 2035. All time series are hourly for the 2035 horizon.

| MVGD | 1690 | 1811 | 1056 | 176 | 177 | 2534 |
|---|---|---|---|---|---|---|
| Characteristic | Wind | Wind | PV | PV / Load | Load | Load |

The workflow (thesis Fig. 4.1) consists of these steps:

```text
 Grid topologies (ding0 + eGon-data: RES, conventional load, BEV, HP/TES)
                                │
                     Observation periods (residual load)
          ├─ reinforcement week: max / min mean residual load
          └─ potential week:     most balanced residual load
                                │
        ┌───────────────────────┼─────────────────────────────┐
        ▼                       ▼                             ▼
 Reference                Minimal line loading           (based on minimal)
 - immediate BEV charging - feeder separation            Partial-load scenarios
 - HP without TES         - LOPF: min line loading       - 20/40/60/80/100 % of
 - reinforcement          - concat & integrate dispatch    installed flexible load
                          - reinforcement                - reinforcement
        │                       │                             │
        └───────────────────────┴──────────────┬──────────────┘
                                               ▼
                          Determine potential (per feeder, parallel)
                          - max / min grid power
                          - max / min energy level
                                               ▼
                         Flex-bands & load balancing potential
```

**Methods:**

- **Spatio-temporal decomposition:** Each grid is split into its MV feeders, which are
  optimised in parallel. Optimisation runs over selected observation weeks instead of
  the whole year, optionally with a rolling horizon.
- **Dispatch optimisation:** A linearised optimal power flow (LOPF, branch flow model
  formulated in Pyomo on top of `edisgo.opf.lopf`) with models for BEV charging
  (flex-bands for the *home* and *work* charging sectors) and HPs with thermal energy
  storage. Objectives: `minimize_loading`, `maximize/minimize_grid_power`,
  `maximize/minimize_energy_level`.
- **Enhanced reinforcement:** eDisGo's reinforcement heuristic is wrapped in an
  iterative fallback. It first tries combined MV-LV reinforcement. If the power flow
  does not converge, it reinforces MV only, then MV plus MV/LV transformers, then each
  LV grid separately, and finally combined again. Time steps that converge are always
  handled first.
- **Flex-bands & LBP:** The results of the min/max optimisations are aggregated at the
  HV/MV station into upper and lower bands for power and energy. From these bands the
  load balancing potential is derived.

## Key findings

- An optimised dispatch of HPs and BEVs reduced reinforcement costs by **up to one third**
  compared to the reference (uncontrolled) case.
- When the grid is reinforced only to the minimum the optimised dispatch requires,
  **hardly any flexibility** is left to offer to the overlying grid.
- Additional reinforcement unlocks significant flexibility, but the relation is mostly
  **non-linear**. All grids reached their maximum potential at or below the 60 %
  flexible-load scenario, with reinforcement
  costs of about **150–250 k€ per installed flexible peak load**. One grid behaved linearly and cost about twice as much.
- Compared to the minimal reinforcement, the extra cost was **2–3×** for half of the
  grids and **≥10×** for the rest.
- A reinforcement level designed for **40–60 % of the installed flexible load** appears
  to be the most effective.

Full discussion: thesis chapters 5 and 6.

## Repository structure

```text
eDisGo-lobaflex/
├── config/
│   ├── model_config.yaml            # main configuration (split into .<section>.yaml)
│   └── egon-data.configuration.yaml # DB / SSH-tunnel settings for eGon-data
├── data/                            # input grids (eDisGo exports), not versioned
├── results/                         # pipeline output per run_id / mvgd
├── logs/
├── src/
│   ├── workflow.py                  # early linear prototype of the workflow
│   └── lobaflex/
│       ├── grids/                   # grid & data preparation (pydoit: dodo_grids.py)
│       │   ├── egon_db.py, db_data.py   # access to the eGon-data database
│       │   ├── load_integration.py      # conventional load & generation
│       │   ├── emob_integration.py      # BEV charging points & flex-bands
│       │   └── hp_integration.py        # heat pumps & thermal storage
│       ├── opt/                     # optimisation pipeline (pydoit: pipeline.py)
│       │   ├── pipeline.py              # task graph: init, ref/min/scn × exp/pot
│       │   ├── tasks.py                 # pydoit task definitions
│       │   ├── timeframe_selection.py   # observation periods
│       │   ├── feeder_extraction.py     # split MVGD into feeders
│       │   ├── dnm_generation.py        # downstream-nodes matrix
│       │   ├── dispatch_optimization.py # LOPF (long-term / rolling horizon)
│       │   ├── result_concatination.py  # merge feeder results
│       │   ├── dispatch_integration.py  # write dispatch back into eDisGo
│       │   ├── grid_reinforcement.py    # iterative reinforcement
│       │   └── expansion_scenario.py    # partial-load scenarios
│       ├── analysis/                # papermill notebook templates
│       └── tools/                   # config, logging, plots, pydoit helpers
├── 2023-04-10_Thesis.pdf
└── 2023-04-26_MA_presentation.pdf
```

## Installation

The code was developed with Python 3.8 and conda.

```bash
git clone https://github.com/iuljan/eDisGo-lobaflex.git
```

```bash
cd eDisGo-lobaflex
```

```bash
conda env create -f environment.yml
```

```bash
conda activate myvenv
```

```bash
pip install -e .
```

Requirements:

- **eDisGo** with the `edisgo.opf.lopf` module. In 2023 this module was only on the
  development branch of [openego/eDisGo](https://github.com/openego/eDisGo), so you
  may need to install a matching version from source.
- **Gurobi** with a valid licence. This is the default solver; see `solver` in the
  config.
- Grid preparation (`lobaflex/grids`) also needs access to an
  [eGon-data](https://github.com/openego/eGon-data) database, optionally through an SSH
  tunnel (`config/egon-data.configuration.yaml`).
- `pydoit`, `papermill`, `kaleido` and `saio` are listed in `requirements.txt`.

## Configuration

All settings live in `config/model_config.yaml`. At runtime each top-level section is
written to a hidden sub-config, e.g. `opt:` → `config/.opt.yaml`. Important keys in the
`opt` section:

| Key | Meaning |
|---|---|
| `run_id` | Name of the run; results go to `results/<run_id>/` |
| `import_dir` | Folder in `data/` with prepared grids (`data/<import_dir>/<mvgd>/`) |
| `mvgds` | List of grid district IDs to process |
| `solver`, `options` | Solver and solver options (e.g. Gurobi threads) |
| `flexible_loads` | Which units are flexible (`hp`, `bev`, `bess`, BEV sectors) |
| `start_datetime`, `total_timesteps` | Optimisation horizon |
| `timesteps_per_iteration`, `overlap_iterations`, `iterations_per_era` | Rolling-horizon settings |
| `rolling_horizon.pot / .load` | Use a rolling horizon for potential / loading runs |
| `min_potential`, `scn_potential`, `ref_potential` | Objectives used to compute flex-bands per scenario |
| `fix_preparation` | Skip re-running preparation tasks |

Optional: put a `config/telegram.env` file with `TOKEN` and `CHAT_ID` to receive pipeline
status messages through the `TelegramReporter`. Never commit this file.

## Running the workflow

The workflow is orchestrated with [pydoit](https://pydoit.org/). Each stage has a
`doit.cfg` that points to its task file.

**1. Prepare grids** (load, BEV and HP integration from eGon-data):

```bash
cd src/lobaflex/grids
```

```bash
doit
```

**2. Optimisation & reinforcement pipeline:**

```bash
cd src/lobaflex/opt
```

```bash
doit list
```

```bash
doit
```

The default task groups of `pipeline.py` are:

| Task group | Content |
|---|---|
| `init` | Observation periods & feeder separation |
| `ref_exp` / `ref_pot` | Reference reinforcement / its flex potential |
| `min_exp` / `min_pot` | Minimal-loading optimisation + reinforcement / its flex potential |
| `scn_exp` / `scn_pot` | Partial-load scenarios (20–100 %) / their flex potential |
| `*_pot_2` | Same potential runs for the second observation period |
| `analysis` | Evaluation notebooks via papermill |

You can run a single group or subtask, e.g. `doit min_exp` or
`doit min_exp:minimize_loading_1056/03`. pydoit stores task state in
`src/lobaflex/pydoit/`, so finished tasks are skipped on the next run.

## Results and analysis

Results are stored per run and grid, for example:

```text
results/<run_id>/<mvgd>/
├── initial/            # observation period grid + feeders
├── reference/          # reference reinforcement
├── minimize_loading/   # optimised dispatch, reinforced grid, feeders
├── scenarios/<xx>_pct_reinforced/
└── potential/          # flex-band optimisation results
```

`lobaflex/analysis/templates/` contains Jupyter templates (`analyse_potential.ipynb`,
`grid_analysis_minimize_loading_template.ipynb`). The pipeline runs them per grid with
papermill. Plot functions are in `lobaflex/tools/plots.py`.

## Limitations

- The study covers only six heterogeneous grids, so no general patterns can be derived.
- The grid is the only restricting component. Market and transmission-level effects
  were not modelled.
- There is no sensitivity analysis on installed capacities or the penetration of
  flexible units.
- The code is research code from a Master's thesis. It depends on specific eDisGo and
  eGon-data versions and on Gurobi, and is not maintained as a general-purpose package.

See thesis section 5.4 for details.

## Related tools

[eDisGo](https://github.com/openego/eDisGo) ·
[ding0](https://github.com/openego/ding0) ·
[eGon-data](https://github.com/openego/eGon-data) ·
[SimBEV](https://github.com/rl-institut/SimBEV) ·
[TracBEV](https://github.com/rl-institut/tracbev) ·
[Pyomo](http://www.pyomo.org/) ·
[pydoit](https://pydoit.org/)

## License

[GNU Affero General Public License v3.0](LICENSE)

## Citation

```bibtex
@mastersthesis{endres2023lobaflex,
  author  = {Endres, Julian},
  title   = {Evaluating load-balancing potential for the energy system provided
             through distribution grid flexibilities},
  school  = {Technische Universit{\"a}t Berlin},
  year    = {2023},
  month   = {4},
  note    = {In cooperation with the Reiner Lemoine Institut}
}
```
