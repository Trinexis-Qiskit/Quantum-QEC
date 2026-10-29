# Q-DecodeQEC: Real Quantum Error Correction

**Participant:** Kaustubh Ladda  
**GitHub Username:** kaustubhkladda  
**GitHub Profile:** [https://github.com/kaustubhkladda](https://github.com/kaustubhkladda)  
**Selected Domain:** Quantum Error Correction and Fault Tolerance

Q-DecodeQEC studies syndrome decoding for a distance-3, three-round bit-flip repetition code. It compares three classical decoders using real IBM Quantum measurement records and separately evaluates passive and active memory-correction circuits on real quantum hardware.

## Problem Statement

Predict whether the logical observable has flipped from eight binary detector bits produced by quantum error-correction syndrome measurements. Compare decoding methods using the logical error rate (LER), while also considering uncertainty, latency, probability calibration, reproducibility, and real-hardware behaviour.

The scope is a small bit-flip repetition code. The project does not claim scalable fault tolerance or quantum advantage.

## Proposed Approach

1. **M0: Minimum-Weight Perfect Matching (MWPM)**  
   A PyMatching decoder built from a detector graph, with the measurement-to-data edge-weight ratio tuned using training data.

2. **M1: Multilayer Perceptron (MLP)**  
   A classical neural-network decoder trained on the eight detector bits, using training-only model selection and a validation-tuned decision threshold.

3. **M2: MOR adaptive decoder router**  
   A difficulty-aware router that chooses an identity, matching, or neural path per shot. Neural probabilities are calibrated using Platt scaling.

Two controls are used for comparison: a no-flip baseline and a detector-pattern lookup table. Methods are compared on the same internal-test shots using bootstrap intervals and paired comparisons.

A separate hardware experiment compares passive memory circuits with active circuits that use a conditional X correction based on the most recent syndrome register. The passive shots are also decoded offline with the frozen classical decoders.

## Technologies

- Python 3.12.10
- Qiskit 2.5.0
- IBM Quantum Runtime 0.48.0
- PyMatching 2.4.0
- NumPy 2.1.0 and pandas 2.3.2
- scikit-learn 1.7.1 and SciPy 1.18.0
- Matplotlib 3.10.6 and joblib 1.5.3
- Jupyter Notebook / JupyterLab
- IBM Quantum hardware and dynamic circuits

## Repository Structure

```text
Quantum-QEC/
└── submissions/
    └── kaustubhkladda_Kaustubh_Ladda_FALL_FEST_2026/
        ├── README.md
        ├── src/
        │   └── main.ipynb
        ├── report.pdf
        ├── presentation.pptx
        └── video-link.md
```

## Setup and Execution

### 1. Requirements

Use Python 3.12.10, the version recorded for the experiment. A working internet connection is needed to install packages. Access to an IBM Quantum account is needed only to rerun the hardware-execution portions.

### 2. Create and activate a virtual environment

**Windows (PowerShell):**

```powershell
python -m venv .venv
.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
```

**macOS / Linux:**

```bash
python3.12 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
```

### 3. Install the tested package versions

Run the following command in the activated environment:

```bash
python -m pip install "qiskit==2.5.0" "qiskit-ibm-runtime==0.48.0" "pymatching==2.4.0" "pandas==2.3.2" "numpy==2.1.0" "scipy==1.18.0" "scikit-learn==1.7.1" "matplotlib==3.10.6" "joblib==1.5.3" jupyter
```

These versions match the environment recorded in the technical report. If package installation fails on your platform, check Python and package compatibility before changing versions, because changing dependencies can affect reproducibility.

### 4. Open the notebook

From the submission directory, run:

```bash
jupyter lab
```

Open `src/main.ipynb` in JupyterLab. Follow the notebook sections in order. The notebook documents the data checks, detector construction, decoder training and evaluation, model freezing, hardware circuit execution, and result verification.

### 5. IBM Quantum hardware execution

The documented hardware experiment used `ibm_fez`, including a calibration job and a Part A job. To repeat this portion, configure IBM Quantum Runtime credentials locally using the supported Qiskit IBM Runtime account workflow. Do not commit an API token, private credentials, or a `.env` file to GitHub.

**Hardware cost warning:** the experiment configuration recorded `submit_to_qpu: true`. Review the notebook configuration and hardware-submission cells before executing them. Running those cells can submit jobs to real quantum hardware and consume account resources. Do not run all notebook cells blindly if you only want to inspect the analysis or reproduce classical results. The report recorded 5.0 QPU seconds for the two completed hardware jobs; this is the reported usage of the original run, not a guarantee for a rerun.

## Results and Findings

### Data

- **Labelled source data:** 5,000 shots from `ibm_marrakesh`, split by circuit into 4,000 training shots and 1,000 internal-test shots.
- **Final holdout:** 2,000 unlabelled shots reserved for predictions. Because the target labels are not available, no final-test accuracy or LER is reported.
- **Features:** eight binary detector bits per shot.

### Internal-test benchmark

Logical error rate (LER) is the fraction of shots where the predicted logical-flip label differs from the observed label. Lower is better.

| Method | Internal-test LER | Internal-test accuracy |
|---|---:|---:|
| No-flip baseline | 0.245 (24.5%) | 75.5% |
| Detector-pattern lookup | 0.020 (2.0%) | 98.0% |
| M0: MWPM | 0.028 (2.8%) | 97.2% |
| M1: MLP | 0.018 (1.8%) | 98.2% |
| M2: MOR router | 0.020 (2.0%) | 98.0% |

M1 had the lowest point-estimate LER. However, its improvement over the lookup control was small and **not statistically resolved** by the paired bootstrap interval. The results support decoding over the no-flip baseline, but do not establish a statistically reliable winner among the lookup control, M0, M1, and M2. Under the project’s predeclared selection rule, **M0 MWPM is the exploratory recommendation** because of its low batch-one median latency (30.3 microseconds per shot) among the candidates considered tied by the paired-interval rule.

### Real-hardware memory experiment

The separate experiment ran 16 circuits on `ibm_fez`: eight fault configurations in passive and active versions, with 250 shots per circuit.

| Condition | Pooled logical error rate |
|---|---:|
| Passive circuit, before offline decoding | 25.35% |
| Passive shots decoded offline with M0 | 1.80% |
| Passive shots decoded offline with M1 | 2.15% |
| Passive shots decoded offline with M2 | 2.60% |
| Active circuit, before feedback correction | 25.30% |
| Active circuit, after on-device correction | 4.80% |

Active correction substantially improved the measured result compared with the pre-feedback view, but it did not outperform offline M0 decoding in this experiment. The active rule uses only the latest syndrome register, so it cannot reliably handle every later fault or readout error.

### Reproducibility and integrity

The documented workflow uses a fixed seed, train-only tuning, leakage checks, frozen model artefacts with SHA-256 hashes, validated prediction files, a QPU ledger, and an evidence index. The report records 22 data files checked and 34 indexed evidence files. These checks support traceability of the reported run.

## Limitations

- The code is a distance-3 bit-flip repetition code, not a surface code or a general-purpose quantum error-correction system.
- The labelled data come from one `ibm_marrakesh` job and include deliberately injected X-fault configurations. They do not represent every form of natural hardware noise.
- Only 1,000 labelled internal-test shots are available. Small differences between decoders are not statistically resolved.
- The input space is small, and the pattern-lookup control achieves a 2.0% internal-test LER, making small machine-learning gains difficult to establish.
- The 2,000 final-test shots are unlabelled. Their predictions were generated after the models were frozen, but their accuracy and LER cannot be calculated from the available data.
- The hardware demonstration used a different backend (`ibm_fez`) and a later run than the labelled source data. Differences can reflect backend, calibration, circuit, and time effects.
- The active circuit uses a simple correction rule based on the last syndrome register and can introduce errors in configurations where the fault does not require the commanded correction.
- Decoder latency measurements are classical Python timings on a laptop. They are not measurements of a complete real-time quantum feedback loop.
- Optional stretch methods S1 and S2 were marked `NOT_ATTEMPTED` in the report.
- The project does not demonstrate quantum advantage or scalable fault tolerance.

## Presentation and Demo

- **Presentation:** [presentation.pptx](presentation.pptx)
- **Technical report:** [report.pdf](report.pdf)
- **Notebook:** [src/main.ipynb](src/main.ipynb)
- **Video/demo link:** [`video_link.md`](video_link.md) 

## References and Additional Documentation

The detailed methodology, circuit diagrams, per-configuration results, uncertainty estimates, configuration records, job verification, and evidence inventory are documented in `report.pdf`. The notebook `src/main.ipynb` contains the project workflow and experiment code.

---

**Project status:** Classical decoder benchmark and the documented real-hardware Part A experiment completed. Optional stretch methods not attempted. Results are specific to the experiment and should not be interpreted as a claim of quantum advantage or general fault tolerance.
