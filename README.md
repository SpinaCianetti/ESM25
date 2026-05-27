# ESM25 — Processing Pipeline Notebooks

[![Code License: GPL-3.0](https://img.shields.io/badge/Code-GPL--3.0-blue.svg)](https://www.gnu.org/licenses/gpl-3.0.html)
[![Data License: CC BY-SA 4.0](https://img.shields.io/badge/Data-CC%20BY--SA%204.0-lightgrey.svg)](https://creativecommons.org/licenses/by-sa/4.0/)
[![Python 3.9+](https://img.shields.io/badge/python-3.9+-blue.svg)](https://www.python.org/)
[![SeisBench](https://img.shields.io/badge/platform-SeisBench-orange.svg)](https://github.com/seisbench/seisbench)

This repository contains the Jupyter notebooks used to build **ESM25**, a machine-learning-ready seismic dataset derived from the [Engineering Strong Motion database (ESM-DB)](https://esm-db.eu/). The dataset covers the Euro-Mediterranean and Middle East region and is aligned with the [SeisBench](https://github.com/seisbench/seisbench) platform.

> **Dataset licence:** [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/)  
> **Reference database:** ESM-DB (Luzi et al., 2016; Lanzano et al., 2019)  
> **Modelled after:** INSTANCE dataset (Michelini et al., 2021)

---

## Repository structure

```
.
├── 01-workflow_download_esm.ipynb
├── 02-combine_events_csv.ipynb
├── 03-create_metadata_rename.ipynb
├── 04-extract_waveform_parameters.ipynb
├── 05-check_download_analysis.ipynb
├── 06-create_esm25_padding.ipynb
├── 07-write_h5_seisbench.ipynb
├── 08-ESM25_magnitude_analysis.ipynb
├── 09-dataset_overview_figures.ipynb
└── README.md
```

---

## Workflow overview

```
01-Download
    │
    ├── {event_id}.h5  (ASDF waveforms)
    ├── {event_id}_SA.csv
    └── {event_id}_SD.csv
          │
          ├─── 02-Combine CSVs ────────────────────► events_dataframe.csv
          │
          ├─── 03-Metadata & rename ──────────────► metadata_good.csv
          │                                          metadata_bad.csv
          │
          ├─── 04-Extract waveform parameters ────► waveform_summary.csv
          │
          ├─── 05-Quality checks & magnitude ─────► diagnostic CSVs & figures
          │         (reads from 02 + 04)
          │
          └─── 06-Build HDF5 datasets ─────────────► waveforms_good_cv.h5
                    │                                 waveforms_good_mp.h5
                    │                                 waveforms_bad_cv.h5
                    │                                 spectra_all.h5
                    │                                 metadata_final.csv
                    │                                 metadata_bad_clean.csv
                    │
                    └─── 07-Write SeisBench format ► dataset/good_cv/
                    │                                 dataset/good_mp/
                    │                                 dataset/bad/
                    │                                 dataset/spectra/
                    │
                    └─── 09-Dataset overview ────────► figures/
```

> **Notebook 08** is a standalone analysis tool and can be run independently after notebook 02.

---

## Notebook descriptions

### `01` — ESM-DB Download Workflow

Creates a list of all the events in the ESM-DB, downloads all public available events in ASDF format together with SA and SD metadata flatfiles. Implements an adaptive strategy per event: a single-call download is attempted first; if the server returns HTTP 413, the download falls back to per-letter-initial batching followed by ASDF merging. Supports automatic resume and logs persistent failures to `error_log.csv`.

**Outputs:** `{event_id}.h5`, `{event_id}_SA.csv`, `{event_id}_SD.csv`, `event_list.txt`, `error_log.csv`

---

### `02` — Flatfile Consolidation

Consolidates all per-event `*_SA.csv` flatfiles into a single unified DataFrame sorted by event origin time.

**Inputs:** `{event_id}_SA.csv` (all events)  
**Outputs:** `events_dataframe.csv`

---

### `03` — Metadata Standardisation

Reads the per-event SA and SD flatfiles, merges the spectral metadata, applies the standardised ESM25 column naming convention (`source_`, `station_`, `path_`, `trace_` prefixes), and splits records by quality flag.

**Inputs:** `{event_id}_SA.csv`, `{event_id}_SD.csv`  
**Outputs:** `metadata_all.csv`, `metadata_good.csv`, `metadata_bad.csv`

---

### `04` — Waveform Parameter Extraction

Iterates over all ASDF/HDF5 files and extracts trace-level metadata (station coordinates, event parameters, processing type, duration, sampling rate). Produces a CV vs MP duration comparison table and a list of events with missing depth.

**Inputs:** `{event_id}.h5` (all events)  
**Outputs:** `waveform_summary.csv`, `cv_mp_duration_comparison.csv`, `files_without_mp.txt`, `filenames_missing_depth.txt`

---

### `05` — Quality Checks, Waveform Analysis and Magnitude Analysis

Performs a comprehensive quality control on the downloaded dataset and exploratory analysis of the event catalogue. Triggers automatic re-download of missing or corrupt files.

**Quality control sections:**
- Completeness check and re-download of missing files
- Empty file detection
- Event coverage: waveform summary vs flatfile
- Station elevation cross-check (ASDF vs flatfile, threshold 1 m)
- Geographic filter (Euro-Mediterranean and Middle East)
- Epicentre and station maps
- Annual event count
- Channel code distribution
- Waveform duration statistics, histograms and violin plot
- Sampling rate distribution
- Epoch start-time detection (1970-01-01) and re-download list
- Long-MP waveform impact on event-station coverage
- Three-component consistency check (CV and MP)
- StationXML presence check

**Magnitude analysis sections:**
- Preferred magnitude derivation and type distribution (pie charts)
- Magnitude vs epicentral distance scatter plot

**Inputs:** `waveform_summary.csv` (from `04`), `events_dataframe.csv` (from `02`), raw `.h5` files  
**Outputs:** `empty_files.txt`, `elevation_discrepancies.csv`, `waveform_epoch_start.csv`, `filenames_epoch_start.txt`, `waveform_mp_long.csv`, `events_affected_long_mp.txt`, `cv_inconsistencies.csv`, `mp_inconsistencies.csv`, `missing_stationxml_report.csv`, `figures/magnitude_pie_charts.png`, `figures/magnitude_vs_distance.png`, additional figures in `figures/`

---

### `06` — HDF5 Dataset Construction

Core dataset builder. Creates consolidated HDF5 files from the per-event ASDF files for three record categories (`good_cv`, `good_mp`, `bad`) and extracts response spectra from `AuxiliaryData/Spectra`. Applies resampling to 200 Hz, zero-padding to fixed lengths, and temporal metadata enrichment via join with `waveform_summary.csv`. A front-padding alignment step (section 4.3) synchronises the three CV components (u, v, w) and the corresponding MP waveform to a unified start time by prepending zeros, preserving original per-component timestamps under `_orig_` column names. A final alignment step computes the intersection cv ∩ mp ∩ spectra and writes the definitive `metadata_final.csv`.

| Dataset | CSV file | HDF5 file | Shape | Duration |
|---|---|---|---|---|
| `bad` | `metadata_bad_clean.csv` | `waveforms_bad_cv.h5` | (3, 84000) | 420 s |
| `good_cv` | `metadata_final.csv` | `waveforms_good_cv.h5` | (3, 84000) | 420 s |
| `good_mp` | `metadata_final.csv` | `waveforms_good_mp.h5` | (9, 84000) | 420 s |
| `spectra` | `metadata_final.csv` | `spectra_all.h5` | (6, 105) | — |

**Inputs:** `metadata_good.csv`, `metadata_bad.csv`, `waveform_summary.csv`, raw `.h5` files  
**Outputs:** `waveforms_good_cv.h5`, `waveforms_good_mp.h5`, `waveforms_bad_cv.h5`, `spectra_all.h5`, `metadata_final.csv`, `metadata_bad_clean.csv`

---

### `07` — SeisBench Format Export

**Authors:** Spina Cianetti and Dario Jozinović  Converts the consolidated HDF5 files into the SeisBench bucket format. All four datasets (`bad`, `good_cv`, `good_mp`, `spectra`) are processed automatically in sequence — no manual variable change is required between runs. Each dataset is written to a dedicated subdirectory containing `waveforms.hdf5` and `metadata.csv` (with `trace_name` column for direct SeisBench lookup). Waveforms are written in configurable chunk sizes; failed traces are replaced with NaN placeholders and logged to `errors.csv`. A summary table is printed at the end.

**Inputs:** `waveforms_*.h5`, `spectra_all.h5`, `metadata_final.csv`, `metadata_bad_clean.csv` (all from `SOURCE_DIR`, same as notebook 06's `OUTPUT_DIR`)  
**Outputs:** `dataset/good_cv/`, `dataset/good_mp/`, `dataset/bad/`, `dataset/spectra/`

---

### `08` — Magnitude Distribution Analysis *(standalone)*

Analyses the magnitude distribution of the ESM25 earthquake catalogue: magnitude type coverage, Gutenberg-Richter relation, depth distribution, and magnitude-vs-time trends. Events lacking any valid magnitude are exported to CSV. Can be run independently after notebook `02`.

**Inputs:** `events_dataframe.csv`  
**Outputs:** `nan_magnitude_events.csv`, `magnitude_histogram.png`, `magnitude_histogram_log.png`, `gutenberg_richter.png`, `magnitude_histogram_stacked.png`, `magnitude_analysis_subplots.png`, `magnitude_type.png`, `depth_histogram.png`, `magnitude_vs_depth.png`, `magnitude_vs_depth_subplots.png`, `magnitude_vs_depth_subplots_combined_colored.png`, `magnitude_vs_time_subplots.png`, `magnitude_vs_time_subplots_combined_colored.png`, `magnitude_violinplot.png`

---

### `09` — Dataset Overview Figures

Reads the final metadata CSV files and consolidated HDF5 files to generate publication-ready figures illustrating the ESM25 dataset composition.

**Sections:**
- Dataset statistics (event counts, good vs bad record breakdown)
- Epicentre map (shaded-relief topographic map; circle size ∝ magnitude, colour ∝ focal depth)
- Sample bad waveform (CV acceleration for a bad-quality record)
- Sample good waveform (CV acceleration, MP acc/vel/dis, and response spectra)

**Inputs:** `metadata_final.csv`, `metadata_bad_clean.csv`, `waveforms_good_cv.h5`, `waveforms_good_mp.h5`, `waveforms_bad_cv.h5`, `spectra_all.h5`  
**Outputs:** figures in `FIGURE/`

---

## Requirements

```
obspy
pyasdf
h5py
scipy
numpy
pandas
matplotlib
tqdm
seisbench
seaborn
cartopy
```

Install with:

```bash
pip install obspy pyasdf h5py scipy numpy pandas matplotlib tqdm seisbench seaborn cartopy
```

---

## Configuration

Each notebook contains a **configuration cell** near the top where input/output paths are defined. Before running, update the following variables to match your environment:

| Variable | Notebooks | Description |
|---|---|---|
| `INPUT_DIR` / `H5_DIR` | 01, 02, 03, 04, 05 | Directory containing downloaded ASDF and CSV files |
| `OUTPUT_DIR` | 06 | Directory for consolidated HDF5 files **and** metadata CSVs (`metadata_final.csv`, `metadata_bad_clean.csv`, etc.) |
| `SOURCE_DIR` | 07 | Same directory as `OUTPUT_DIR` in notebook 06 (input for SeisBench conversion) |
| `DATA_DIR` | 09 | Same directory as `OUTPUT_DIR` in notebook 06 (input for overview figures) |
| `WAVE_SUMMARY_CSV` | 06 | Absolute path to `waveform_summary.csv` |

---

## Output file naming conventions

- **ASDF source files:** `{event_id}.h5`, `{event_id}_SA.csv`, `{event_id}_SD.csv`
- **Metadata columns** follow a domain-prefix convention: `source_*`, `station_*`, `path_*`, `trace_*`
- **ESM25 trace names distrubuted through ESM website:** `{source_id}.{network}.{station}.{location}.{channel}`
- **SeisBench trace names:** `bucket{N}${i}`, where `N` is the 1-based bucket number and `i` is the 0-based index of the trace within that bucket

---

## Citation

If you use ESM25 in your research, please cite the ESM-DB source database:

> >Cianetti S., Mascandola C., Faenza L., Felicetta C., Russo E., Jozinović D., Münchmeyer J., Luzi L., Michelini A. (2026).  ESM25: A Machine-Learning-Ready Snapshot of the European Engineering Strong-Motion Database. Istituto Nazionale di Geofisica e Vulcanologia (INGV). https://doi.org/10.xxxxx/esm25

---

## Licence

This repository uses a dual-licence model:

- **Code** (Jupyter notebooks, scripts): [GPL-3.0](https://www.gnu.org/licenses/gpl-3.0.html)
- **Dataset** (ESM25 waveforms, metadata, spectra): [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/)

© Spina Cianetti and co-authors.
