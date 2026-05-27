# FAST Forge

A GitHub Actions workflow and composite action for running OpenFAST DLC simulation
campaigns. Turbine models are fetched from a configurable public repository
(default [NatLabRockies/ROSCO](https://github.com/NatLabRockies/ROSCO)); the controller binary and tuning files are
downloaded from a separately configurable public release (default `NatLabRockies/ROSCO`).

---

### External dependencies fetched at runtime

| Dependency | Source | Access |
|---|---|---|
| NREL-5MW turbine model | `model_repo` (default `NatLabRockies/ROSCO`) at `model_tag` | Public |
| TurbSim template (`90m_12mps_twr.inp`) | Same model repo release | Public |
| Controller DLL + IN + Cp/Ct files | `controller_repo` (default `NatLabRockies/ROSCO`) at `controller_tag` | Public |
| `fastprep` binary | Private repo set via `FASTPREP_REPO` variable | Private |
| OpenFAST + TurbSim binaries | OpenFAST public GitHub releases | Public |
| `postfast` binary | Private repo set via `POSTFAST_REPO` variable | Private |

---

## Pipeline: `dlc11-sweep.yml`

Three sequential jobs run on Windows runners.

### Job 1 — `prepare`
### Job 2 — `simulate`
### Job 3 — `post_process` (`if: always()`)
---

## `action.yml` — single-case composite action

Runs one OpenFAST simulation case. Called by `dlc11-sweep.yml` (simulate job) but
can also be called directly from any workflow.

### Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `fst_rel_path` | yes | — | Path to `.fst` relative to campaign dir root |
| `campaign_dir` | yes | — | Absolute path to the downloaded campaign directory |
| `openfast_version` | no | `v4.2.0` | OpenFAST release tag |
| `upload_artifact` | no | `true` | Upload the `.outb` as a workflow artifact |
| `artifact_retention_days` | no | `90` | Artifact retention period [days] |

### Outputs

| Output | Description |
|---|---|
| `case_stem` | `.fst` filename without extension |
| `outb_path` | Absolute path to the produced `.outb` (empty if not produced) |


## Workflows

### `dlc11-sweep.yml` — full DLC sweep

Triggers: `workflow_call` (called by other workflows) and `workflow_dispatch`
(manual trigger from the GitHub Actions UI).

#### Inputs

| Input | Default | Description |
|---|---|---|
| `dlc_file` | `dlc_11_details.txt` | DLC config file (repo-root-relative) |
| `model_repo` | `NatLabRockies/ROSCO` | Public repo hosting the NREL-5MW turbine model and TurbSim template |
| `model_tag` | `v2.10.5` | Release tag for the turbine model and TurbSim template |
| `controller_repo` | `NatLabRockies/ROSCO` | Public repo hosting the controller release assets |
| `controller_tag` | `v2.9.0` | Release tag to download the controller from (must include a 64-bit binary) |
| `fastprep_version` | `latest` | fastprep release tag |
| `openfast_version` | `v4.2.0` | OpenFAST release tag |
| `postfast_version` | `latest` | postfast release tag |

### `dlc11-smoke.yml` — smoke test

Triggers: push to `main`, pull request targeting `main`, `workflow_dispatch`.

Calls `dlc11-sweep.yml` with `dlc_file: dlc_smoke.txt`.

---

### DLC table

One row per condition set, whitespace-delimited:

```
% DLC_ID  TurbModel  WSP_min  WSP_max  WSP_step  N_seeds  Shear  Yaw_deg  IA_deg  Active
DLC1.1    NTM        4        20       2         3        0.14   0.0      0.0     1
```

| Column | Type | Description |
|---|---|---|
| `DLC_ID` | string | e.g. `DLC1.1` |
| `TurbModel` | string | `NTM`, `ETM`, `EWM1`, `EWM50`, or `STD` |
| `WSP_min` | float | lowest hub-height wind speed [m/s] |
| `WSP_max` | float | highest hub-height wind speed [m/s] |
| `WSP_step` | float | wind speed increment [m/s] |
| `N_seeds` | int | number of random turbulence realisations |
| `Shear` | float | power-law wind shear exponent |
| `Yaw_deg` | float | yaw error [deg] |
| `IA_deg` | float | inflow (upflow) angle [deg] |
| `Active` | int | `1` = include, `0` = skip |

## ServoDyn controller path patching

The ROSCO `NRELOffshrBsline5MW_Onshore_ServoDyn.dat` references ROSCO's default
controller. The `prepare` job replaces the `DLL_FileName` and `DLL_InFile` lines
with absolute paths to the fetched controller before fastprep runs.


## Calling this from another repo

Run a full DLC sweep from another workflow:

```yaml
jobs:
  sweep:
    uses: turbinesim/fastforge/.github/workflows/dlc11-sweep.yml@main
    secrets: inherit
    with:
      dlc_file: 'my_campaign.txt'
      model_tag: 'v2.10.5'
      openfast_version: 'v4.2.0'
```

Run a single simulation case directly:

```yaml
- uses: actions/checkout@v4
  with:
    repository: turbinesim/fastforge
    path: openfast-action

- uses: ./openfast-action
  with:
    fst_rel_path: 'FST/DLC1.1/DLC1.1_NTM_U12_S01_Yaw000_IA00.fst'
    campaign_dir: '${{ github.workspace }}/work/my_project'
```