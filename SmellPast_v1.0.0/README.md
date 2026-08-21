# SmellPast

SmellPast is a machine-olfaction dataset for studying **residual odor**: identifying what was recently present in an environment from the chemical trace that remains after it has been removed.

The dataset contains gas-sensor measurements of four spices, **black pepper, nutmeg, paprika, and rosemary**, together with an **ambient empty-enclosure reference**. Substance runs cover the period before the sample is introduced, while it is present, and after it is removed.

Data were collected on 34 days between June 24 and August 5, 2026 by the Multisensory Intelligence Group at MIT.

## Overview

| | |
|---|---|
| Files | 195 CSV files |
| Total duration | approximately 58 hours |
| Classes | `black_pepper`, `nutmeg`, `paprika`, `rosemary`, `ambient` |
| Short runs | 185 files, 37 five-way sets, approximately 12 minutes each |
| Long runs | 10 files, 2 five-way sets, 120–148 minutes each |
| Recorded channels | 27 |
| Derived columns | 3 |
| Acquisition target | 4 Hz |

Each set contains one run from each of the five classes. Set numbers are an organizational pairing: the five runs in a set are matched chronologically within their respective classes and may have been collected on different dates.

## Collection protocol

Each substance run has three phases:

1. **Ambient:** the enclosure is empty for approximately 2 minutes.
2. **Exposure:** the substance is placed inside for approximately 5 minutes.
3. **Decay:** the substance is removed and the enclosure is closed again while the residual odor remains.

Short runs continue for approximately 5 minutes after removal. Long runs continue through an extended decay period. The enclosure is aired out between runs.

Insertion and removal are performed manually and logged at the time of handling, so the exact transition points vary from run to run.

Ambient-class runs contain an empty enclosure throughout.

### Phase labels in the CSV files

Every row has a `derived_phase` value:

- `ambient`: before insertion
- `exposure`: from the logged insertion row up to the row before removal
- `decay`: from the logged removal row to the end of the run

Ambient-class files contain `ambient` on every row.

The word **ambient** therefore appears in two related contexts in the dataset: `ambient` is one of the five class labels, and `ambient` is also the pre-exposure phase of every substance run.

Use `derived_phase` to determine the phase of a row rather than assuming fixed 2/5/5-minute boundaries.

## Files and naming

```text
SmellPast_v1.0.0/
├── README.md
└── data/
    ├── short_runs/
    │   ├── set01/
    │   ├── ...
    │   └── set37/
    └── long_runs/
        ├── set01/
        └── set02/
```

Each set directory contains one CSV file for each class.

File names encode the class, collection date, and start time:

```text
black_pepper_20260716_1833.csv
```

Long-run filenames also include the observed run duration, rounded to minutes:

```text
nutmeg_20260731_1725_148min.csv
```

Times in filenames use the 24-hour clock. File names are unique within the release.

## CSV format

All files use the same 30-column schema.

The first **27 columns** contain the recorded sensor and environmental measurements. These values are preserved as acquired: the release does not resample, interpolate, smooth, replace, or otherwise transform them.

The final **3 columns** were added during dataset preparation:

- **`derived_phase`**: phase of the run at that row (`ambient`, `exposure`, or `decay`).
- **`derived_badchannels`**: semicolon-separated channel-quality flags for that row.
- **`derived_elapsedseconds`**: reconstructed elapsed time in seconds from the beginning of the run.

The `derived_` fields are annotations constructed from the acquisition records. The original 27 recorded columns remain unchanged.

## Time axis and sampling frequency

The acquisition firmware was configured for a 4 Hz sampling frequency. The effective sampling frequency varies across runs because the acquisition loop did not always maintain the target cadence and some runs were recorded at other steady rates.

The release preserves every run at its acquired sampling frequency. **No common-rate resampling has been applied.**

The original sensor stream does not contain a timestamp on every row. `derived_elapsedseconds` supplies a time coordinate for each run using its recorded start and end times together with its number of rows.

For a run containing \(N\) rows over a recorded duration of \(T\) seconds, the mean sampling frequency is

```text
(N - 1) / T
```

and row \(i\) is assigned

```text
i / mean_sampling_frequency
```

seconds from the start of the run.

This construction gives the first row an elapsed time of 0 and the final row the recorded duration of the run. `derived_elapsedseconds` is stored to two decimal places.

For the released short substance runs, the reconstructed time axis was checked against the independently logged insertion and removal timestamps. The largest discrepancy observed at either transition was **2.6 seconds**. Long substance runs were also checked for consistency of the sampling frequency before and after the logged removal event.

For any analysis involving physical time, use `derived_elapsedseconds` rather than converting row number to time under a fixed 4 Hz assumption.

If a model requires every run on the same time grid, resample downstream using `derived_elapsedseconds` as the time coordinate. The appropriate target frequency depends on the analysis, so the release leaves that choice to the user.

## Sensor channels

Every CSV contains **all 27 recorded channels**.

For users who want a common set of raw gas-response inputs across the full dataset, 14 channels provide a conservative starting point:

- 3 MQ channels
- 9 MEMS channels
- 2 raw SGP30 channels

The other 13 recorded channels remain in every file. They include processed gas indices, environmental measurements, and channels that show documented fault-state or constant-value periods in part of the collection.

### Raw gas-response channels

| Channels | Sensor | Measurement |
|---|---|---|
| `mqAlc`, `mqCOComb`, `mqAirQ` | MQ-3, MQ-9, MQ-135 | raw 12-bit ADC counts |
| `memsVOC`, `memsH2S`, `memsNH3`, `memsEtOH`, `memsH2`, `memsHCHO`, `memsSmoke`, `memsCH4`, `memsOdor` | DFRobot 9-channel MEMS array | raw 12-bit ADC counts |
| `sgpH2`, `sgpEth` | Sensirion SGP30 | raw H₂ and ethanol signal ticks |

These 14 channels are the most straightforward choice when a model requires the same raw gas-response feature set across the collection.

### Additional recorded channels

| Channels | Sensor | Measurement | Data characteristics |
|---|---|---|---|
| `sgpTVOC`, `sgpeCO2` | Sensirion SGP30 | TVOC and CO₂-equivalent indices | processed outputs; extended plateaus occur in some runs |
| `bmeTemp`, `bmePres`, `bmeHum` | Bosch BME680 | temperature (°C), pressure (hPa), relative humidity (%) | environmental measurements; some humidity and pressure values are quality-flagged |
| `gravO2`, `gravTemp` | DFRobot Gravity O₂ | oxygen concentration (% vol) and sensor temperature | some runs contain constant-value periods |
| `groveNO2`, `groveC2H5OH`, `groveVOC`, `groveCO` | Seeed Grove Multichannel Gas Sensor v2 | metal-oxide gas responses | later runs contain extended zero-valued fault states |
| `mhCO2`, `mhTemp` | Winsen MH-Z19C | CO₂ (ppm) and sensor temperature (°C) | later runs commonly contain the fault pattern `mhCO2 = 0` and `mhTemp = -273.15` |

`sgpeCO2` is the SGP30's computed CO₂-equivalent output. Direct CO₂ measurements, where available, are recorded in `mhCO2`.

The BME680 contributes temperature, pressure, and humidity to this dataset. Its gas-resistance output was not recorded.

## Channel-quality annotations

`derived_badchannels` records channels whose values meet predefined quality-control criteria. The recorded value is always retained in the CSV; the annotation simply identifies it for downstream handling.

A channel is flagged on every row of a run when its value is constant throughout that run, except that an ambient run with `sgpeCO2 = 400` throughout is left unflagged.

The following row-level conditions are also flagged:

| Channel(s) | Condition |
|---|---|
| `groveNO2`, `groveC2H5OH`, `groveVOC`, `groveCO` | value = 0 |
| `mhCO2` | value = 0 |
| `mhTemp` | value = -273.15 |
| MQ and MEMS channels | value ≥ 4095 |
| `sgpTVOC` | value ≥ 60000 |
| `sgpeCO2` | value ≥ 57330 |
| `bmeHum` | value ≥ 100 |
| `bmePres` | value outside 900–1100 hPa |

A flag means that the reading met one of the screening conditions above. Depending on the channel, this can correspond to a constant signal, a known fault-state value, a plateau, saturation, or an out-of-range measurement.

Multiple flagged channels are separated by semicolons. An empty field means that no channel was flagged on that row. CSV readers that treat empty strings as missing values may display an empty `derived_badchannels` field as `NaN`.

The field is intended to support filtering, masking, and sensitivity checks while leaving the underlying measurements available.

## Practical notes

### Raw and processed SGP30 outputs

`sgpH2` and `sgpEth` are raw SGP30 signals.

`sgpTVOC` and `sgpeCO2` are processed indices generated by the sensor. Some runs contain long plateaus in these processed outputs, so users should inspect them separately from the raw SGP30 channels.

### Environmental measurements

Temperature, pressure, and humidity are recorded alongside the gas-response channels. They describe conditions during acquisition and may also capture environmental variation across collection days.

Some BME680 humidity and pressure values meet the quality-control criteria described above.

### Ambient reference

The `ambient` class represents the empty enclosure. No substance is inserted or removed during an ambient run.

The ambient class therefore provides an empty-enclosure reference. It does not reproduce the physical insertion and removal steps used during substance runs.

## Dataset preparation

The release adds structure and timing annotations while preserving the recorded measurements.

During preparation:

- the 27 recorded columns were retained in their original order and with their original values;
- insertion and removal logs were converted into the row-level `derived_phase` annotation;
- run-level timing information and row counts were used to construct `derived_elapsedseconds`;
- deterministic channel-quality rules were applied to produce `derived_badchannels`;
- filenames were standardized to encode class, date, and start time;
- runs were organized into five-way short and long sets.

No sensor values were imputed or corrected during this process.

## Citation

The SmellPast paper is currently in preparation.

If you use this dataset in academic work, please cite the paper once a public version is available. The full citation will be added here when the preprint is released.

## Contact

josephfn@mit.edu