# Guide: Adding New MC Samples to cmsdb

A step-by-step reference for adding new Monte Carlo datasets and process definitions to the [uhh-cms/cmsdb](https://github.com/uhh-cms/cmsdb) repository.

---

## 1. Repository Structure

```
cmsdb/
├── cmsdb/
│   ├── __init__.py
│   ├── constants/              # Physics constants (BRs, masses)
│   │   └── __init__.py         # br_h, br_hh, br_w, br_z, etc.
│   ├── processes/              # Physics process definitions
│   │   ├── __init__.py         # Imports all process modules
│   │   ├── hh.py               # Parent HH processes (ggF/VBF couplings)
│   │   ├── hh2ml.py            # HH → multilepton decay processes
│   │   ├── hh2bbtautau.py      # HH → bbττ decay processes
│   │   ├── hh2bbvv.py          # HH → bbVV decay processes
│   │   └── ...
│   ├── campaigns/              # Campaign-specific dataset definitions
│   │   ├── run3_2024_nano_v15/
│   │   │   ├── __init__.py     # Campaign object + trailing imports
│   │   │   ├── hh2ml.py        # HH→ML datasets for this campaign
│   │   │   ├── hh2bbtautau.py  # HH→bbττ datasets
│   │   │   ├── top.py, ewk.py, qcd.py, ...
│   │   └── ...
│   └── util.py                 # Helper functions (multiply_xsecs, etc.)
├── scripts/
│   └── get_das_info.py         # DAS query helper script
├── tests/
│   ├── test_campaigns.py       # Tests: unique names/IDs, required attrs
│   └── test_processes.py       # Tests: process definitions
└── requirements.txt
```

### Key Concepts

| Concept | Description |
|---------|-------------|
| **Process** | A physics process (e.g., `hh_ggf_htt_htt_kl1_kt1`). Defined in `cmsdb/processes/`. Has a name, ID, parent process, cross section. |
| **Campaign** | A data-taking era + processing version (e.g., `run3_2024_nano_v15`). Defined in `cmsdb/campaigns/`. |
| **Dataset** | A specific MC sample in a campaign. Links a process to its DAS key, file count, and event count. |

---

## 2. Process Definitions (`cmsdb/processes/`)

Processes form a hierarchy using the [order](https://github.com/riga/order) library. Example from `hh2ml.py`:

```python
import cmsdb.constants as const
from cmsdb.processes.hh import (
    hh_ggf, hh_ggf_kl1_kt1, hh_ggf_kl0_kt1, hh_ggf_kl2p45_kt1, hh_ggf_kl5_kt1,
    hh_vbf, hh_vbf_kv1_k2v1_kl1, ...
)
from cmsdb.util import multiply_xsecs

# Parent process (no cross section, but has a label)
hh_ggf_htt_htt = hh_ggf.add_process(
    name="hh_ggf_htt_htt",
    id=91000,
    label=r"$HH_{ggf} \rightarrow 4\tau$",
)

# Specific coupling point (with cross section AND label including coupling info)
hh_ggf_htt_htt_kl1_kt1 = hh_ggf_kl1_kt1.add_process(
    name="hh_ggf_htt_htt_kl1_kt1",
    id=91001,
    label=r"$HH_{ggf} \rightarrow 4\tau$ ($\kappa_{\lambda}=1$)",
    xsecs=multiply_xsecs(hh_ggf_kl1_kt1, const.br_hh.tttt),
)
```

### Naming Convention for Processes

| Production | Pattern | Example |
|-----------|---------|---------|
| **ggF** | `hh_ggf_{channel}_{coupling}` | `hh_ggf_htt_htt_kl1_kt1` |
| **VBF** | `hh_vbf_{channel}_{coupling}` | `hh_vbf_htt_htt_kv1_k2v1_kl1` |

**Channel names:**
- `htt_htt` = HH → 4τ
- `hvv_hvv` = HH → 4V (inclusive)
- `htt_hvv` = HH → 2τ2V
- `hww_hzz_3l` = HH → WWZZ (3-lepton filter)
- `hvv_hvv_2lplus` = HH → 4V (≥2 lepton filter)

**Coupling suffixes:**
- ggF: `_kl0_kt1`, `_kl1_kt1`, `_kl2p45_kt1`, `_kl5_kt1`
- VBF: `_kv1_k2v1_kl1`, `_kvm0p758_k2v1p44_klm19p3`, etc.

> ⚠️ **VBF naming gotchas:** Trailing zeros are dropped (`0p030` → `0p03`, `1p60` → `1p6`). The benchmark `CV=-2.12` is stored as `kv2p12` (positive) in the existing definitions.

### Branching Ratios (from `cmsdb/constants/__init__.py`)

**Inclusive HH BRs:**
`tttt`, `vvvv`, `ttvv`, `wwzz`, `wwww`, `zzzz`, `bbtt`, `bbww`, `bbzz`, `bbvv`, `bbbb`, `bbgg`, `wwgg`

**Filtered 4V BRs** (added for multilepton samples):
| Key | Description | Fraction of 4V |
|-----|-------------|----------------|
| `vvvv_2lplus` | 4V → ≥2 charged leptons | 37.1% |
| `vvvv_1l` | 4V → exactly 1 charged lepton | 38.6% |
| `vvvv_0l` | 4V → 0 charged leptons + ≥6 jets | 23.8% |
| `wwzz_veto_nunu_3l` | WWZZ (veto Z→νν) → 3 leptons | 6.2% of WWZZ |
| `wwzz_veto_nunu_4lplus` | WWZZ (veto Z→νν) → ≥4 leptons | 2.5% of WWZZ |

These filtered BRs are computed by enumerating individual decay modes (4W, 2W2Z, 4Z), following the official methodology from [genproductions PR #3537](https://github.com/cms-sw/genproductions/pull/3537). The intermediate variables use underscore prefix (`_br_4w_2l2nu4q`, etc.) to mark them as private.

**Usage:** `xsecs=multiply_xsecs(parent_coupling, const.br_hh.vvvv_2lplus)`

### ID Allocation

> **There are two different types of IDs in cmsdb — don't confuse them!**

#### Process IDs (in `cmsdb/processes/`)

These are **manually assigned** integers that uniquely identify each physics process across the entire repository. They are internal to cmsdb and have no connection to CMS/DAS.

**How it works:** Each process file "owns" a range of IDs. Within that range, the parent (inclusive) process gets the base ID (e.g., `91000`), and each coupling variant gets the next sequential integer (`91001`, `91002`, ...).

| ID range | File | Channel |
|----------|------|--------|
| `20000`–`20xxx` | `hh.py` | Parent HH process |
| `21000`–`21xxx` | `hh.py` | ggF parent couplings |
| `21100`–`21xxx` | `hh2bbtautau.py` | ggF HH → bbττ |
| `22000`–`22xxx` | `hh.py` | VBF parent couplings |
| `23100`–`23xxx` | `hh2bbtautau.py` | VBF HH → bbττ |
| `91000`–`91xxx` | `hh2ml.py` | ggF HH → 4τ |
| `92000`–`92xxx` | `hh2ml.py` | ggF HH → 4V (inclusive) |
| `92100`–`92xxx` | `hh2ml.py` | ggF HH → 4V (2L+) |
| `92200`–`92xxx` | `hh2ml.py` | ggF HH → 4V (1L) |
| `92300`–`92xxx` | `hh2ml.py` | ggF HH → 4V (0L) |
| `93000`–`93xxx` | `hh2ml.py` | ggF HH → 2τ2V |
| `94000`–`94xxx` | `hh2ml.py` | ggF HH → 2W2Z (3L) |
| `94100`–`94xxx` | `hh2ml.py` | ggF HH → 2W2Z (4L+) |
| `95000`–`95xxx` | `hh2ml.py` | VBF HH → 4τ |
| `96000`–`96xxx` | `hh2ml.py` | VBF HH → 4V (inclusive) |
| `96100`–`96xxx` | `hh2ml.py` | VBF HH → 4V (2L+) |
| ... | ... | ... |

**How to pick a new process ID:**
1. Check existing IDs in the same file: `grep 'id=' cmsdb/processes/hh2ml.py`
2. Find the last used ID in your range and increment by 1
3. Run `python3 -m pytest tests/test_processes.py -v` to verify no duplicates

#### Dataset IDs (in `cmsdb/campaigns/`)

These are the **DAS dataset_id** obtained from `dasgoclient`. They are NOT manually assigned — you query them from DAS and copy them directly. See Step 3 below.

---

## 3. Campaign Datasets (`cmsdb/campaigns/{campaign}/`)

Each campaign directory has an `__init__.py` that defines the campaign and imports dataset modules:

```python
# __init__.py
from order import Campaign
campaign_run3_2024_nano_v15 = Campaign(
    name="run3_2024_nano_v15", id=32024115, ecm=13.6, bx=25, ...
)
import cmsdb.campaigns.run3_2024_nano_v15.hh2ml  # noqa
```

Dataset entries in `hh2ml.py`:

```python
import cmsdb.processes as procs
from cmsdb.campaigns.run3_2024_nano_v15 import campaign_run3_2024_nano_v15 as cpn

cpn.add_dataset(
    name="hh_ggf_htt_htt_kl1_kt1_powheg",      # process_name + _generator
    id=15543589,                                  # DAS dataset_id
    processes=[procs.hh_ggf_htt_htt_kl1_kt1],    # linked process
    keys=[
        "/GluGlutoHHto4Tau_.../NANOAODSIM",       # DAS key (NanoAOD path)
    ],
    n_files=24,                                    # number of good files in DAS
    n_events=1000000,                              # total events in good files
)
```

### Dataset Naming Convention

```
{process_name}_{generator}
```
- Generator: `powheg` (ggF) or `madgraph` (VBF)
- Example: `hh_ggf_htt_htt_kl1_kt1_powheg`, `hh_vbf_htt_htt_kv1_k2v1_kl1_madgraph`

---

## 4. Step-by-Step: Adding New Samples

### Step 1: Fork & Clone

```bash
# Fork uhh-cms/cmsdb on GitHub, then:
git clone https://github.com/YOUR_USERNAME/cmsdb.git
cd cmsdb
git checkout -b add_my_new_samples
```

### Step 2: Get McM Request Info

Samples are typically communicated as McM request ranges (e.g., `HIG-RunIII2024Summer24wmLHEGS-01535` to `01548`).

**McM web UI:** Browse requests at:
```
https://cms-pdmv-prod.web.cern.ch/mcm/requests?range=HIG-RunIII2024Summer24wmLHEGS-01535,HIG-RunIII2024Summer24wmLHEGS-01548
```

**McM REST API** (to get dataset name and NanoAOD path programmatically):
```python
import urllib.request, json

prepid = "HIG-RunIII2024Summer24wmLHEGS-01535"
url = f"https://cms-pdmv-prod.web.cern.ch/mcm/public/restapi/requests/get/{prepid}"
data = json.loads(urllib.request.urlopen(url).read())["results"]

dataset_name = data["dataset_name"]   # e.g. "GluGlutoHHto4Tau_Par-c2-0p00-kl-2p45-kt-1p00_..."
status = data["status"]               # should be "done"

# Extract NanoAOD path from the chain output:
for rmn in data.get("reqmgr_name", []):
    for ds_path in rmn.get("content", {}).get("pdmv_dataset_statuses", {}):
        if "NanoAOD" in ds_path:
            print(f"NanoAOD: {ds_path}")
```

### Step 3: Get DAS Info (n_files, dataset_id, n_events)

Requires a valid grid proxy:
```bash
voms-proxy-init -voms cms -rfc -valid 196:00
```

**Using `dasgoclient`:**
```bash
# Get file count, event count, etc. as JSON
dasgoclient --query="summary dataset=/GluGlutoHHto4Vto2Lplus_Par-c2-0p00-kl-2p45-kt-1p00_TuneCP5_13p6TeV_powheg-pythia8/RunIII2024Summer24NanoAODv15-150X_mcRun3_2024_realistic_v2-v2/NANOAODSIM" --json
```

```bash
# Get dataset details including ID
dasgoclient --query="dataset dataset=/GluGlutoHHto4Vto2Lplus_Par-c2-0p00-kl-2p45-kt-1p00_TuneCP5_13p6TeV_powheg-pythia8/RunIII2024Summer24NanoAODv15-150X_mcRun3_2024_realistic_v2-v2/NANOAODSIM" --json
```

**Using the repo's helper script:**
```bash
cd cmsdb
python3 scripts/get_das_info.py -d "/GluGlutoHHto4Tau_.../NANOAODSIM"
```

This prints a ready-to-paste `cpn.add_dataset(...)` block with the correct `id`, `n_files`, `n_events`, and `keys`.

**Programmatic DAS query (Python):**
```python
import subprocess, json

nano_path = "/GluGlutoHHto4Tau_Par-c2-0p00-kl-1p00-kt-1p00_TuneCP5_13p6TeV_powheg-pythia8/RunIII2024Summer24NanoAODv15-150X_mcRun3_2024_realistic_v2-v2/NANOAODSIM"

cmd = f"dasgoclient -query='file dataset={nano_path}' -json"
result = subprocess.run(cmd, shell=True, capture_output=True, text=True)
file_infos = json.loads(result.stdout)

seen = set()
n_files, n_events, dataset_id = 0, 0, None

for fi in file_infos:
    fdata = fi["file"][0]
    if fdata["name"] in seen:
        continue
    seen.add(fdata["name"])
    if fdata.get("is_file_valid", 1) and fdata.get("nevents", 0) > 0:
        n_files += 1
        n_events += fdata["nevents"]
        if dataset_id is None:
            dataset_id = fdata["dataset_id"]

print(f"Dataset ID:   {dataset_id}")
print(f"Valid files:  {n_files}")
print(f"Total events: {n_events:,}")
```

### Step 4: Add Process Definitions (if needed)

Check if the process already exists in `cmsdb/processes/`:
```bash
grep -r "hh_ggf_htt_htt_kl2p45" cmsdb/processes/
```

If not, add it to the appropriate file (e.g., `cmsdb/processes/hh2ml.py`):

```python
hh_ggf_htt_htt_kl2p45_kt1 = hh_ggf_kl2p45_kt1.add_process(
    name="hh_ggf_htt_htt_kl2p45_kt1",
    id=91004,                                         # unique ID
    xsecs=multiply_xsecs(hh_ggf_kl2p45_kt1, const.br_hh.tttt),
)
```

**Don't forget** to add the new name to `__all__` at the top of the file.

### Step 5: Add Dataset Entries

Add `cpn.add_dataset()` calls to the campaign file (e.g., `cmsdb/campaigns/run3_2024_nano_v15/hh2ml.py`):

```python
cpn.add_dataset(
    name="hh_ggf_htt_htt_kl2p45_kt1_powheg",
    id=15544701,
    processes=[procs.hh_ggf_htt_htt_kl2p45_kt1],
    keys=[
        "/GluGlutoHHto4Tau_Par-c2-0p00-kl-2p45-kt-1p00_.../NANOAODSIM",  # noqa
    ],
    n_files=139,
    n_events=989671,
)
```

### Step 6: Validate

The repository has existing test scripts in `tests/` — use them instead of writing custom checks.

```bash
cd cmsdb
```

**Quick import test** (catches syntax errors, missing processes, duplicate names immediately):
```bash
python3 -c "import cmsdb.campaigns.run3_2024_nano_v15; print('OK')"
```

**Run process tests** — checks for unique process names/IDs and valid properties:
```bash
python3 -m pytest tests/test_processes.py -v
```

**Run campaign tests** — checks all datasets have required attributes (`name`, `id`, `keys`, `n_files`, `n_events`), unique names/IDs, lowercase names, and correct generator suffix:
```bash
python3 -m pytest tests/test_campaigns.py -v
```

> **Note:** pytest's `-k` flag filters by test **function name**, not campaign name, so `-k "run3_2024_nano_v15"` won't work here. The full suite tests all campaigns at once but only takes ~5 seconds.

**Optional: DAS consistency check** — verifies that `n_files`, `n_events` and `id` in your code match what DAS actually reports. Requires grid proxy and takes a long time:
```bash
# Edit tests/test_campaigns.py and set check_das_info = True, then run:
python3 -m pytest tests/test_campaigns.py -v
```

#### What the tests check

| Test file | What it verifies |
|-----------|------------------|
| `test_processes.py` | Unique process names & IDs, lowercase names, name matches variable |
| `test_campaigns.py` | Campaign has `name`, `id`, `ecm`, `bx`; datasets have all required attrs; dataset name ends with generator name (`powheg`/`madgraph`/`amcatnlo`/`pythia`); no duplicate dataset names or IDs; optionally checks DAS consistency |

### Step 7: Commit & PR

```bash
git add cmsdb/processes/hh2ml.py cmsdb/campaigns/run3_2024_nano_v15/hh2ml.py
git commit -m "Add HH multilepton 2024 samples (RunIII2024Summer24NanoAODv15)"
git push origin add_my_new_samples
```

**Squashing commits for a clean PR** (if you made many incremental commits during development):
```bash
# Squash all branch commits into one
git reset --soft master
git commit -m "Add HH multilepton processes and datasets for Run3 2024 campaign"
git push --force origin add_my_new_samples
```

Then open a PR at: `https://github.com/YOUR_USERNAME/cmsdb/pull/new/add_my_new_samples`

Target: `uhh-cms/cmsdb` main branch.

---

## 5. Quick Reference

### McM API URL Pattern
```
https://cms-pdmv-prod.web.cern.ch/mcm/public/restapi/requests/get/{PREPID}
```

### DAS Query Patterns
```bash
# File info (for n_files, n_events, dataset_id)
dasgoclient --query='dataset dataset={DAS_KEY}' --json

# Dataset existence check
dasgoclient --query='dataset={DAS_KEY}'

# Using wildcards
dasgoclient --query='dataset=/GluGlutoHHto4Tau*/RunIII2024*/NANOAODSIM'
```

### VBF Coupling Name Map (McM → cmsdb)

| McM CV | McM C2V | McM C3 | cmsdb suffix |
|--------|---------|--------|--------------|
| 1 | 0 | 1 | `_kv1_k2v0_kl1` |
| 1 | 1 | 1 | `_kv1_k2v1_kl1` |
| 1p74 | 1p37 | 14p4 | `_kv1p74_k2v1p37_kl14p4` |
| m0p012 | 0p030 | 10p2 | `_kvm0p012_k2v0p03_kl10p2` |
| m0p758 | 1p44 | m19p3 | `_kvm0p758_k2v1p44_klm19p3` |
| m0p962 | 0p959 | m1p43 | `_kvm0p962_k2v0p959_klm1p43` |
| m1p21 | 1p94 | m0p94 | `_kvm1p21_k2v1p94_klm0p94` |
| m1p60 | 2p72 | m1p36 | `_kvm1p6_k2v2p72_klm1p36` |
| m1p83 | 3p57 | m3p39 | `_kvm1p83_k2v3p57_klm3p39` |
| m2p12 / 2p12 | 3p87 | m5p96 | `_kv2p12_k2v3p87_klm5p96` |

> **Key rule:** Drop trailing zeros (`0p030`→`0p03`, `1p60`→`1p6`). The `CV=m2p12` benchmark maps to `kv2p12` (no minus sign).

### Process Labels

Every process should have a LaTeX `label` for use in plots:

| Process type | Label pattern | Example |
|-------------|---------------|--------|
| Placeholder (no coupling) | `$HH_{mode} \rightarrow {decay}$` | `$HH_{ggf} \rightarrow 4\tau$` |
| ggF + coupling | `$HH_{ggf} \rightarrow {decay}$ ($\kappa_{\lambda}=X$)` | `$HH_{ggf} \rightarrow 4V$ ($\kappa_{\lambda}=1$)` |
| VBF + coupling | `$HH_{vbf} \rightarrow {decay}$ ($\kappa_{V}=X$, $\kappa_{2V}=Y$, $\kappa_{\lambda}=Z$)` | see file |
| Filtered | Append filter info after decay | `$HH_{ggf} \rightarrow 4V$ (2L+) ($\kappa_{\lambda}=1$)` |

### Filtered BR Methodology

For generator-level filtered samples (e.g., `HHto4Vto2Lplus`), the effective cross section is:
```
σ_filtered = σ_parent × BR(HH→4V→filtered)
```

The filtered BRs are computed by **enumerating individual decay modes**, not by applying a simple filter efficiency. This follows the official methodology from [genproductions PR #3537](https://github.com/cms-sw/genproductions/pull/3537).

**Key references:**
- [Notebook by ktht](https://cernbox.cern.ch/s/1NxyzV8kAYewuSv) — 4V filter BRs
- [AN-2024/245 Table 20](https://cds.cern.ch/record/2905000) — 2W2Z decay mode BRs
- Pythia fragments: [genproductions HH directory](https://github.com/cms-sw/genproductions/tree/master/genfragments/ThirteenPointSixTeV/Higgs/HH)

**Conventions:**
- Filtered BRs go in `cmsdb/constants/__init__.py` (not in process files)
- Intermediate variables use underscore prefix (`_br_4w_2l2nu4q`)
- Public BRs are added to `br_hh` DotDict (`br_hh.vvvv_2lplus`)
- Processes reference them with `multiply_xsecs(parent, const.br_hh.vvvv_2lplus)`
