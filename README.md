# MS-TCMFNet

Source code and experimental data for MS-TCMFNet-based methane concentration retrieval using WMS-2f signals.

This public package supports **frozen-model measurement replay**: a single-channel WMS-2f waveform is used to retrieve the current methane concentration in ppm. It includes the network implementation, measured waveform CSV files, and five trained checkpoints.

## Download and package layout

Download [MS-TCMFNet_public.zip](./MS-TCMFNet_public.zip) and extract it into an empty working directory. The archive contains:

```text
Code/
  inference.py                  # preprocessing, LS fit, checkpoint loading, replay and metrics
  mstcmfnet.py                  # MSTCMFNet neural module and checkpoint adapter
Data/
  development/*.csv             # measured development waveforms and 50 ppm references
  validation/*.csv              # measured validation waveforms and 50 ppm references
Models/
  H0_seed20261018.pt
  H0_seed20261019.pt
  H0_seed20261020.pt
  H0_seed20261021.pt
  H0_seed20261022.pt
```

The original archive is retained unchanged. Source files are currently distributed inside the ZIP, so download and extraction are required to inspect them.

## Environment and quick start

Dependencies are Python, NumPy and PyTorch; the replay uses CPU and does not require CUDA. A successful independent replay used Linux, Python 3.12.14, NumPy 2.3.5 and PyTorch 2.14.1+cpu. These are the replay-check environment versions, **not a recovered original training environment**. Other versions have not been validated here.

Create an isolated environment and install the two external Python dependencies:

```bash
python -m venv .venv
# Linux/macOS:
source .venv/bin/activate
# Windows PowerShell alternative:
# .venv\Scripts\Activate.ps1
python -m pip install numpy==2.3.5
python -m pip install torch==2.14.1 --index-url https://download.pytorch.org/whl/cpu
```

**Required layout step:** the packaged script defines its data/model root as its own directory, but the archive places the scripts in `Code/` and the data/models beside `Code/`. From the extracted top-level directory, copy both scripts to that top level before running:

```bash
python -c "from pathlib import Path; import shutil; [shutil.copy2(p, Path(p.name)) for p in Path('Code').glob('*.py')]"
python inference.py development
python inference.py validation
```

The resulting working directory must contain `inference.py`, `mstcmfnet.py`, `Data/` and `Models/` side by side. Running `python Code/inference.py` directly against the original extracted layout fails to locate the checkpoints. The copy step changes only the working directory, not the distributed archive.

Each command prints a metric dictionary and writes `predictions_development.csv` or `predictions_validation.csv` beside the copied script. Existing prediction files with those names are overwritten. Omitting the split argument defaults to validation.

## Source, data and model correspondence

| Component | Role and authoritative mapping |
| --- | --- |
| `Code/mstcmfnet.py` | Neural correction model: input shape (batch, 1, 200), output scan-level concentration correction in ppm. |
| `Code/inference.py: RECORDS` | Explicit measurement-to-CSV mapping, reference CSV, scan start and target concentration. Use these targets rather than inferring exact validation concentrations from rounded filenames. |
| `Data/development/` | Development waveform files; replay uses the selected measurements listed in `RECORDS`, with the matching 50 ppm reference. Not every distributed CSV is a scored target. |
| `Data/validation/` | Validation waveform files and reference measurements; `RECORDS` assigns the reference to each target. Reference CSVs are not additional scored targets. |
| `Models/*.pt` | Five frozen seed checkpoints. `MODELS` contains their filenames and SHA-256 hashes; replay verifies each before loading. |

CSV files have two header rows. The reader uses the second column (CH1 voltage), groups consecutive values into 500-point scans, and ignores other columns. Each concentration retrieval uses **24 measured waveforms**, each transformed to 200 points. The replay uses 960 target waveforms in the development set and 504 target waveforms in the validation set; reference waveforms are used separately.

The development replay selects scans 1–24; validation selects scans 97–120 (one-based). The exact file and scan selections are encoded in `RECORDS` and `_read_scans`.

## What the replay computes

1. Locate the spike anchor, cyclically shift, crop, remove an endpoint-fitted linear baseline and resample to 200 points.
2. Form a reference template from the median of the 24 reference waveforms. Fit `x = a*r50 + b0 + b1*t` and compute `C_LS = 50*a`.
3. Divide the fitting residual by the frozen scale `0.0224078329711116`; each checkpoint predicts a correction that is added to `C_LS`.
4. For each seed, aggregate 24 corrected scan predictions using the **lower central value** (index 11 after sorting), matching the packaged implementation rather than the average of the two central values.
5. Average the five seed-level concentration predictions and calculate measurement-level errors.

The output CSV includes target concentration, ensemble prediction, signed error (prediction minus target), and each seed's prediction. Printed metrics include MAE, RMSE, Bias, R², MAPE, median absolute error, P90 absolute error and maximum absolute error. Concentration errors are in ppm; MAPE is percent.

## Verified replay scope and numerical checks

With the layout step above, both replay commands completed and all five checkpoint hashes matched. The following values were recomputed from the released archive on 2026-10-23:

| Replay split | Pooled MAE (ppm) | Pooled RMSE (ppm) | Bias (ppm) | R² |
| --- | ---: | ---: | ---: | ---: |
| Development | 1.473283 | 2.445108 | -0.526509 | 0.992731 |
| Validation | 2.516535 | 3.182711 | +0.749921 | 0.985412 |

These are the pooled metrics printed by this public inference script. **Development replay of these already-trained weights is not a reconstruction of cross-validation, out-of-fold predictions or model selection.** Do not substitute these development replay values for a paper's development cross-validation results. Small numerical differences may occur with other numerical-library versions.

This package enables inspection and replay of the released model on the explicitly mapped measured data. It does **not** contain training scripts, synthetic-data generation, optimizer/scheduler settings, complete training/selection logs, formal competing-baseline implementations, ablation/hyperparameter experiment runners, or figure-generation scripts. The embedded LS fitting is part of the inference pipeline; the current script does not export a separate LS-baseline evaluation. Full retraining and reproduction of every manuscript table/figure are therefore outside the verified public-package scope.

The target values and reference pairings can be inspected in `RECORDS`; their upstream acquisition/calibration provenance is not independently established by this package alone. No explicit license file is included in the current repository/archive.

## Version and reviewer verification path

- Original public archive commit: [6b2534a1f553cb4d85cbc19fea5beed0b743ec71](https://github.com/Lintherain/MS-TCMFNet/commit/6b2534a1f553cb4d85cbc19fea5beed0b743ec71), committed 2026-10-22.
- Public-entry documentation revision: 2026-10-23.
- Archive size: **7,531,537 bytes**.
- Archive SHA-256: `0125c1b2bbe0d59e3689af6d96636ddbd7aa6a61b9b4ad45c8472b3feab7e9d9`.

Reviewer path: download the ZIP → verify its SHA-256 → extract → inspect the two source files and `RECORDS`/`MODELS` → apply the layout step → run both replay commands → compare the generated validation metrics with the numerical checks above.

Cross-platform checksum command, run beside the downloaded archive:

```bash
python -c "import hashlib; from pathlib import Path; print(hashlib.sha256(Path('MS-TCMFNet_public.zip').read_bytes()).hexdigest())"
```

The initial archive and checkpoint bytes are unchanged by this documentation update. The archive checksum identifies the artifact even when the default branch receives later documentation changes.
