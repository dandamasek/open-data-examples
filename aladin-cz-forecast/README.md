# ALADIN Czech Republic 1 km Weather Forecasts — data documentation

> **DRAFT** — items marked `DOPLNIT` are still open (bucket name, region, maintainer, a few parameter definitions).

Hourly 72-hour weather forecasts for the Czech Republic from the **ALADIN** limited-area numerical weather
prediction model run by the **Czech Hydrometeorological Institute (ČHMÚ / CHMI)** on its 1 km `CZ_1km` domain.
ČHMÚ publishes the raw GRIB files on <https://opendata.chmi.cz/> for only a few days. This dataset keeps them as a
continuous archive converted to cloud-optimized **Zarr v3**.

- **Source:** ČHMÚ open data, `https://opendata.chmi.cz/meteorology/weather/nwp_aladin/CZ_1km/`
- **License:** [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). When you use the data, credit
  *"Czech Hydrometeorological Institute (ČHMÚ), ALADIN CZ_1km, opendata.chmi.cz"*.
- **Coverage:** June 2025 onward. Model runs at 00, 06, 12 and 18 UTC. Forecast hours 0–72 at 1 h steps.
- **Update frequency:** daily. Each day the four runs from the previous day are appended.
- **Tutorial:** [`get-to-know-a-dataset.ipynb`](get-to-know-a-dataset.ipynb)

## Accessing the data (no AWS account needed)

The bucket is public, so requests can be anonymous (unsigned).

```bash
aws s3 ls --no-sign-request s3://DOPLNIT-bucket-name/
aws s3 ls --no-sign-request s3://DOPLNIT-bucket-name/meteo_data/
aws s3 ls --no-sign-request s3://DOPLNIT-bucket-name/meteo_data/202602/
```

```python
import xarray as xr

url = "s3://DOPLNIT-bucket-name/meteo_data/202602/CLSTEMPERATURE.zarr"
ds = xr.open_zarr(url, consolidated=True, storage_options={"anon": True})   # needs s3fs, zarr>=3
t2m = ds.CLSTEMPERATURE.sel(latitude=slice(49, 50), longitude=slice(14, 16)).isel(time=0, step=0)
```

Only the chunks you select are downloaded.

## Bucket layout

```
s3://DOPLNIT-bucket-name/
  README.txt                         short description, license, link to this page
  meteo_data/
    202506/                          one prefix per calendar month (YYYYMM)
      CLPVEIND_MOD_XFU.zarr/         one Zarr store = one parameter x one calendar month
        zarr.json                    consolidated metadata of the whole store
        CLPVEIND_MOD_XFU/            data array (chunks under c/<time>/<step>/<lat>/<lon>)
        time/  step/  valid_time/    coordinate arrays
        latitude/  longitude/
        heightAboveGround/           scalar level coordinate (meanSea/ for MSLPRESSURE)
      CLSHUMI_RELATIVE.zarr/
      CLSTEMPERATURE.zarr/
      ... (24 stores)
    202507/
    ...
```

The path to any store is `meteo_data/<YYYYMM>/<PARAMETER>.zarr`. Month prefixes and store names sort chronologically and
alphabetically.

## Zarr store structure

Each store holds **exactly one data variable**, named after the parameter (for example `CLSTEMPERATURE`).

| Property | Value |
|---|---|
| Format | Zarr v3, consolidated metadata (`zarr.json`) |
| Compression | Zstd (level 0), no checksum |
| Data variable | 1 per store, `float32`, named after the parameter |
| Dimensions | `(time, step, latitude, longitude)` = `(≤124, 72, 290, 501)` |
| Chunks | `time=1, step=12, latitude=97, longitude=126` → 72 chunks per model run (~0.6 MB each, uncompressed) |
| Grid | regular lat/lon (`regular_ll`), latitude 48.5 → 51.1 °N (290 points, ascending), longitude 12.0 → 19.0 °E (501 points, 0.014°) |
| Size | ~5 GB uncompressed / ~1–1.3 GB stored per store-month; ~25–32 GB per month for all 24 parameters |

### Coordinates

| Coordinate | Dims | Meaning |
|---|---|---|
| `time` | `time` | **Model run (analysis) time**, `datetime64`, UTC. Only 00, 06, 12, 18 UTC; sorted, no duplicates. |
| `step` | `step` | **Forecast lead time in hours**, `float64` with `units: hours` (not a timedelta). |
| `valid_time` | `(time, step)` | Time the forecast is valid for = `time + step` hours. |
| `latitude`, `longitude` | 1-D | Grid point centres in degrees. |
| `heightAboveGround` / `meanSea` | scalar | Level metadata inherited from GRIB (`meanSea` for `MSLPRESSURE`). |

```python
import pandas as pd
lead = pd.to_timedelta(ds.step.values, unit="h")   # NOT pd.to_timedelta(ds.step.values): that gives nanoseconds
valid = ds.valid_time.isel(time=0)                  # or simply use valid_time
```

### Two kinds of `step` axis

- **Instantaneous parameters**: `step` = 0, 1, …, 71.
- **Accumulated parameters**: `step` = 1, 2, …, 72. At step 0 the accumulation is zero by definition, so the archive keeps
  the first 72 non-trivial hours.

Because of this, **do not align different parameters by step index**. Align them by `step` value or by `valid_time`.

### Accumulated parameters

Accumulated parameters are summed **from the start of the model run** up to `step`. To get hourly values, difference
along `step`:

```python
acc = ds.SURFPREC_TOTAL                         # mm since run start
hourly = acc.diff("step")                       # mm in hour (step-1, step]
first = acc.isel(step=0)                        # hour (0, 1]
```

GRIB packing adds small noise, so consecutive differences can be slightly negative (for example −0.1 mm or −800 J m⁻²).
Clip at 0 if needed.

## Parameters

There are 24 parameters per month. ⚠️ The GRIB-derived attributes in the stores (`units`, `long_name`, `GRIB_*`) come from
generic tables and are **often wrong or `unknown`**. ČHMÚ uses local GRIB codes. For example, `SURFRAINFALL` is labelled
*"Snow depth [m]"*, and all parameters carry `GRIB_stepType=instant`. **Trust the variable name and the table below.**

| Variable | Description | Units (as stored) | Type | `step` |
|---|---|---|---|---|
| `CLSTEMPERATURE` | Air temperature at 2 m | K | instantaneous | 0–71 |
| `CLSHUMI_RELATIVE` | Relative humidity at 2 m | **fraction 0–1** (attribute says %) | instantaneous | 0–71 |
| `CLSWIND_SPEED` | Wind speed at 10 m | m s⁻¹ | instantaneous | 0–71 |
| `CLSWIND_DIREC` | Wind direction at 10 m (direction the wind blows from) | degrees | instantaneous | 0–71 |
| `CLPVEIND_MOD_XFU` | DOPLNIT – definition to be confirmed with ČHMÚ (values of order 10⁴) | DOPLNIT | instantaneous | 0–71 |
| `MSLPRESSURE` | Mean sea level pressure | Pa | instantaneous | 0–71 |
| `CLS_VISICLD` | Visibility (cloud / fog) | m (DOPLNIT confirm) | instantaneous | 0–71 |
| `CLS_VISIPRE` | Visibility in precipitation | m (DOPLNIT confirm) | instantaneous | 0–71 |
| `SURFCAPE_POS_F00` | Convective available potential energy (CAPE) | J kg⁻¹ | instantaneous | 0–71 |
| `SURFNEBUL_BASSE` | Low cloud cover | fraction 0–1 | instantaneous | 0–71 |
| `SURFNEBUL_MOYENN` | Medium cloud cover | fraction 0–1 | instantaneous | 0–71 |
| `SURFNEBUL_HAUTE` | High cloud cover | fraction 0–1 | instantaneous | 0–71 |
| `SURFNEBUL_TOTALE` | Total cloud cover | fraction 0–1 | instantaneous | 0–71 |
| `SURFRESERV_NEIGE` | Snow on the ground, water equivalent | kg m⁻² | instantaneous | 0–71 |
| `SURFPREC_TOTAL` | Total precipitation | kg m⁻² (= mm) | accumulated from run start | 1–72 |
| `SURFRAINFALL` | Liquid precipitation (rain) | kg m⁻² (= mm) (attribute says "Snow depth, m") | accumulated from run start | 1–72 |
| `SURFSNOWFALL` | Solid precipitation (snow), water equivalent | kg m⁻² (attribute says rate) | accumulated from run start | 1–72 |
| `PRECIP_TYPE` | Precipitation type (code) | code, DOPLNIT code table | per hour (DOPLNIT confirm) | 1–72 |
| `PRECIP_TYPESEV` | Most severe precipitation type (code) | code, DOPLNIT code table | per hour (DOPLNIT confirm) | 1–72 |
| `SURFDIAG_FLASH` | Lightning flash diagnostic | DOPLNIT | accumulated from run start | 1–72 |
| `SUNSHINE_DUR` | Sunshine duration | s | accumulated from run start | 1–72 |
| `SURFRF_SHORT_DO` | Downward short-wave (solar) radiation at surface | J m⁻² | accumulated from run start | 1–72 |
| `SURFRF_LONG_DO` | Downward long-wave radiation at surface | J m⁻² | accumulated from run start | 1–72 |
| `SURF_RAYT_DIR` | Direct solar radiation at surface | J m⁻² | accumulated from run start | 1–72 |

To convert accumulated radiation to average flux, compute hourly differences (J m⁻²) and divide by 3600 s to get W m⁻².

**Precipitation type codes.** In a February 2026 sample the values were 0, 1, 3, 5, 6, 7, 11, 12, 193, 201, 205, 206, 207
and 213. Other codes can occur in other seasons.
Round to the nearest integer first, because GRIB packing gives values like 4.996. DOPLNIT: add the meaning of each code
from ČHMÚ documentation.

> The ALADIN configuration on ČHMÚ also lists `CLSU_RAF_MOD_XFU`, `CLSV_RAF_MOD_XFU`, `SURFCIEN_POS_F00` and
> `MAXSIM_REFLECTI`. These are **not** available for the CZ_1km domain, so they are not in the archive.

## Data gaps and known issues

- **Missing days per parameter.** Coverage is tracked **separately for each parameter**. When the daily ingest
  failed for a parameter, all four runs of that day are missing from that parameter's store, while other parameters
  can have that day. A typical day lacks 1–5 of the 24 parameters, so in most months a store holds roughly 70–100 % of
  the expected 4 × days runs. Example: in February 2026 every one of the 112 runs is present for at least one
  parameter, but only 8 runs are present for all 24.
- Missing runs are **absent from the `time` axis**. They are not filled with NaN, so the axis is irregular, and
  `time` differs between the stores of one month. To combine parameters, align them on `time`
  (`xr.merge(..., join="inner")` keeps only the runs common to all), and check coverage per store:

  ```python
  expected = pd.date_range("2026-02-01", periods=28 * 4, freq="6h")   # all runs of February 2026
  missing = expected.difference(ds.indexes["time"])
  ```

- DOPLNIT: list of stores repaired before upload. A few stores have a `valid_time` coordinate whose length differs
  from `time`, or lack consolidated metadata. These will be fixed before publication.
- A coverage inventory (runs per parameter and month) will be published with this documentation (DOPLNIT).

## Processing

1. Each day, download the previous day's four runs (`.grb.bz2`) from opendata.chmi.cz and decompress them.
2. Convert GRIB → NetCDF with `cfgrib`. The run time is taken from the file name.
3. Keep the first 72 forecast steps and append to the monthly Zarr store of each parameter (`xarray.to_zarr(append_dim="time")`).
   Duplicate run times are skipped.

The processing code is at <https://github.com/dandamasek/Meteo-data-on-S3>.
