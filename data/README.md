# Data card: real IBM Quantum repetition-code syndromes

This folder holds every file you may train on. Nothing here is simulated. Read Section 3 of the main `README.md` for the rules of use.

## 1. Provenance
| Item | Value |
|---|---|
| Experiment | Distance-3 bit-flip repetition-code memory, 3 syndrome rounds, passive circuits (no on-device correction) |
| Backend | `ibm_marrakesh` (physical IBM Quantum processor) |
| Labelled job | `dbbq3agc77vs7384te2g`, 8 circuits x 625 shots = 5,000 shots (details in `dataset_card.json` and `raw/.../manifest.json`) |
| Final-test job | A later, separate job on the same backend (passive circuits only, 8 circuits x 250 shots = 2,000 shots); circuit identity and labels are withheld |
| Verification | `raw/syndromes_d3_r3_v1/ibm_result_verification.json` records that the saved per-shot registers were re-checked against the IBM result of the job |

Deliberate faults are real `X` gates on a physical data qubit after a chosen syndrome round, so the fault location is known exactly while the syndrome is a genuine hardware measurement. Hardware noise is present in every circuit.

## 2. Files
| File | Rows | Columns |
|---|---|---|
| `train_4000.parquet` | 4,000 | full labelled schema (Section 3) |
| `internal_test_1000.parquet` | 1,000 | full labelled schema |
| `final_test_inputs.parquet` | 2,000 | `record_id`, `syndrome`, `detectors` |
| `raw/syndromes_d3_r3_v1/*.npz` | 8 files | Per-shot raw registers `syn0`, `syn1`, `syn2`, `final` as bitstrings |
| `raw/syndromes_d3_r3_v1/*.qpy` | 8 files | The transpiled circuits that ran (`qiskit.qpy.load`) |
| `raw/syndromes_d3_r3_v1/manifest.json` | 1 | Layout, calibration snapshot at submission, per-circuit gate counts and digests |
| `dataset_card.json` | 1 | Summary of the labelled job |
| `SHA256SUMS.txt` | 1 | Checksums (verify before use) |

The labelled shots were split **stratified by circuit**: each of the 8 circuits contributes 500 shots to train and 125 to the internal test. Train and internal test therefore come from the same job and the same hour; the final test comes from a later job.

## 3. Schema (train and internal test)
| Column | Type | Meaning | Allowed as model input? |
|---|---|---|---|
| `record_id` | int | Row identifier | no |
| `shot_id` | int | Shot index within its circuit | no |
| `circuit_name` | str | Circuit identifier, encodes the injected fault | no (label-related) |
| `job_id`, `backend` | str | Provenance | no |
| `distance`, `rounds` | int | Code distance (3) and syndrome rounds (3) | no (constants) |
| `initial_bit` | int | Prepared logical bit | **no** (determines the label) |
| `fault_type` | str | `natural` or `injected_x` | no (label-related) |
| `injected_x_qubit` | int | Data qubit of the injected `X`, or -1 | no (label-related) |
| `injected_after_round` | int | Round after which it was injected, or -1 | no (label-related) |
| `fault_label` | int | 0 for none, otherwise `1 + 3*round + qubit` | no (label-related) |
| `syndrome` | 6 x uint8 | Raw parity-check outcomes, index `2*round + check` | **yes** |
| `detectors` | 8 x uint8 | Detector events, index `2*t + check` (Section 4) | **yes** |
| `final_data` | 3 x uint8 | Final readout of the three data qubits | **no** (determines the label) |
| `observable_error` | uint8 | **Target.** 1 if data qubit 0 was read differently from the prepared bit | target only |
| `detector_event_count` | int | Sum of `detectors` | no (derived from input, kept for EDA) |
| `has_detection_event` | uint8 | 1 if any detector fired | no (derived, for EDA) |

`final_test_inputs.parquet` has only `record_id` (fresh identifiers 0 to 1999, unrelated to the labelled files), `syndrome` and `detectors`.

## 4. Detector convention
- Check `i` is the parity of data qubits `i` and `i + 1` (check 0: data0 and data1; check 1: data1 and data2), measured by ancilla `i`.
- `detectors[2*t + i]`: for `t = 0, 1, 2` the change of check `i` between syndrome round `t` and round `t - 1` (round `-1` is all zeros); for `t = 3` the parity of the final data readout for check `i` compared with the last syndrome round.
- Raw hardware registers are little-endian: character 0 of a bitstring is the **last** bit.

## 5. Statistics you can rely on
- Labelled shots: 5,000 (train 4,000, test 1,000); final-test shots: 2,000.
- Share of shots with `observable_error = 1`: about 25 % in train and about 24.5 % in the internal test.
- About 130 distinct detector patterns occur in the 5,000 labelled shots.
- Compute every other statistic yourself in your EDA; do not take numbers from this card.

## 6. Caveats
- Labels come from real readouts. Some logical flips are **not visible in the detectors** (for example relaxation of a prepared state or a readout error), so no decoder can reach zero error.
- Most shots in the labelled files carry a deliberately injected `X` fault, so the fault mix is **not** the natural noise mix. Do not present results as natural-noise performance.
- Train and internal test are drawn from one job; the final test is a different job. Report any gap between them as a drift observation, not as a method property.
- The injected-fault columns describe the cause of a flip. They are for EDA, stratification and reporting only.

## 7. Loading and verifying
1. **Verify integrity.** For each line of `SHA256SUMS.txt` (hash, then relative path), compute the SHA-256 of that file and confirm it matches before you use any data.
2. **Load the tables** with `pandas.read_parquet`. The `detectors` and `syndrome` columns are fixed-length lists of 0/1 values; stack them into a 2-D array (shots by 8 and shots by 6) for modelling. The target is `observable_error`.
3. **Inspect the circuits** that produced the labelled shots by loading any `.qpy` file in `raw/syndromes_d3_r3_v1/` with `qiskit.qpy.load` and printing it. They are the transpiled passive circuits with the physical layout of the job.
