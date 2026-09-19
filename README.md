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
| OpenFAST + TurbSim binaries | `openfast_repo` (default `OpenFAST/openfast`) at `openfast_version` (default `v4.2.0`) | Public or private — see [OpenFAST binary source](#openfast-binary-source) |
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
| `openfast_repo` | no | `OpenFAST/openfast` | Repo (`owner/repo`) whose release hosts `openfast_x64.exe` and `TurbSim_x64.exe`; may be private |
| `openfast_version` | no | `v4.2.0` | Release tag in `openfast_repo` to download from |
| `github_token` | no | `''` | Token used by `gh` to download the assets; required for a private `openfast_repo`, falls back to `github.token` |
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
| `dlc_file` | `dlc_11_details.txt` | DLC config file (repo-root-relative). Ignored when `dlc_file_b64` is set — used only as the display filename in that case. |
| `dlc_file_b64` | `''` | Base64-encoded contents of an uploaded DLC file. When set, takes precedence over `dlc_file`; see [Uploading a DLC file from the dashboard](#uploading-a-dlc-file-from-the-dashboard). |
| `model_repo` | `NatLabRockies/ROSCO` | Public repo hosting the NREL-5MW turbine model and TurbSim template |
| `model_tag` | `v2.10.5` | Release tag for the turbine model and TurbSim template |
| `controller_repo` | `NatLabRockies/ROSCO` | Public repo hosting the controller release assets |
| `controller_tag` | `v2.9.0` | Release tag to download the controller from (must include a 64-bit binary) |
| `fastprep_version` | `latest` | fastprep release tag |
| `openfast_repo` | `OpenFAST/openfast` | Repo hosting the OpenFAST + TurbSim release assets (public or private) |
| `openfast_version` | `v4.2.0` | OpenFAST release tag in `openfast_repo` |
| `postfast_version` | `latest` | postfast release tag |

### `dlc11-smoke.yml` — smoke test

Triggers: push to `main`, pull request targeting `main`, `workflow_dispatch`.

Calls `dlc11-sweep.yml` with `dlc_file: dlc_smoke.txt`.

Manual (`workflow_dispatch`) runs accept `wind_speed`, `controller_repo`, `controller_tag`,
`openfast_repo` and `openfast_version`; push / pull-request runs use the sweep defaults
(`OpenFAST/openfast` @ `v4.2.0`).

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

## OpenFAST binary source

`openfast_x64.exe` and `TurbSim_x64.exe` are **not built** by this repo; they are
downloaded as release assets. By default they come from the official
[`OpenFAST/openfast`](https://github.com/OpenFAST/openfast) repository at tag `v4.2.0`,
but the source repository is configurable in exactly the same way as the controller:
`openfast_repo` (`owner/repo`) selects the repo and `openfast_version` selects a release
tag in it. The repo can be a **private** repo, e.g. an internal build or fork.

### What happens on each simulation job (`action.yml`)

1. **Validate.** `openfast_repo` must match `owner/repo` (letters, digits, `_`, `.`, `-`)
   and `openfast_version` must be non-empty. Both values are passed to the script through
   environment variables, never pasted into the script text.
2. **Cache lookup.** The key is

   ```
   openfast-turbsim-<owner>__<repo>-<openfast_version>-win64
   ```

   The repo is part of the key because a tag name only identifies a release *within one
   repo*: the official `v4.2.0` and a private build that is also tagged `v4.2.0` are
   different binaries and must not share a cache entry. (`/` is replaced by `__` so the
   repo can sit inside a key.) On a hit the download is skipped.
3. **Download (cache miss).** `gh release download <tag> --repo <owner/repo>` with
   `--pattern openfast_x64.exe --pattern TurbSim_x64.exe`, authenticated by
   `GH_TOKEN = github_token` (the sweep workflow passes `secrets.ORG_TOKEN`), falling
   back to the workflow's own `github.token`, which is enough for public repos only.
4. **Verify.** Both files must exist after the download, otherwise the job fails with a
   message naming the repo, tag and missing asset.
5. **Run.** TurbSim (skipped for steady `STD` cases) and OpenFAST are executed from the
   cached `.bin` folder exactly as before.

### Requirements for a custom `openfast_repo`

- A GitHub **release** for the chosen tag with assets named exactly `openfast_x64.exe`
  **and** `TurbSim_x64.exe` (TurbSim is taken from the same repo as OpenFAST).
- For a private repo, the token behind `secrets.ORG_TOKEN` needs read access to that
  repo (fine-grained token: *Contents: read* on it; classic token: `repo` scope).
- Private binaries end up in this repo's Actions cache, which any workflow in this repo
  can read. Runs triggered from forked pull requests have no access to secrets, so they
  fall back to `github.token` and can only use public sources.

### Dashboard (`docs/index.html`)

*Tool versions* has an **OpenFAST repo** field (default `OpenFAST/openfast`) and an
**OpenFAST version** dropdown (default `v4.2.0`) that behave like the Controller pair:
changing the repo or pressing ↻ reloads the tags using the token you connected.

- The dropdown lists only releases that contain **both** `openfast_x64.exe` and
  `TurbSim_x64.exe` (draft releases are ignored). If none qualifies it shows all releases
  with a warning.
- Official repo: `v4.2.0` is pre-selected (and always offered, even if it is beyond the
  first 30 releases). Custom repo: the newest qualifying release is pre-selected.
- If the release list cannot be read for a custom repo (typically a token without access
  to a private repo) the dropdown shows *tags unavailable* and **Launch is blocked**
  rather than silently sending the default tag, which would not exist in that repo.
- The smoke test button sends no inputs, so it always uses the defaults.

## Uploading a DLC file from the dashboard

The **Browse** button next to the DLC file field lets you launch a sweep
against a file from your own machine, instead of one already committed to
the repo. The dashboard reads the file, base64-encodes it, and sends it as
a new `dlc_file_b64` input; the `prepare` job decodes it to a temp file and
uses that everywhere `dlc_file` used to be read. Max upload size is ~44 KB
(GitHub's `workflow_dispatch` payload limit) — plenty for these DLC files,
but larger files still need to be committed to the repo and referenced by
path as before.

## ServoDyn controller path patching

The ROSCO `NRELOffshrBsline5MW_Onshore_ServoDyn.dat` references ROSCO's default
controller. The `prepare` job replaces three parameters with the correct values
for the fetched controller before fastprep runs:

| Parameter | Value |
|---|---|
| `DLL_FileName` | Absolute path to the `.dll` from the controller release |
| `DLL_InFile` | Absolute path to the `.IN` file (controller release, falling back to model repo) |
| `DLL_ProcName` | `DISCON` if the DLL name contains `discon` or `rosco` (case-insensitive); otherwise the DLL filename stem |


## Calling this from another repo

Run a full DLC sweep from another workflow:

```yaml
jobs:
  sweep:
    uses: openfast-tools/fastforge/.github/workflows/dlc11-sweep.yml@main
    secrets: inherit
    with:
      dlc_file: 'my_campaign.txt'
      model_tag: 'v2.10.5'
      openfast_repo: 'OpenFAST/openfast'   # or a private owner/repo
      openfast_version: 'v4.2.0'
```

Run a single simulation case directly:

```yaml
- uses: actions/checkout@v4
  with:
    repository: openfast-tools/fastforge
    path: openfast-action

- uses: ./openfast-action
  with:
    fst_rel_path: 'FST/DLC1.1/DLC1.1_NTM_U12_S01_Yaw000_IA00.fst'
    campaign_dir: '${{ github.workspace }}/work/my_project'
```