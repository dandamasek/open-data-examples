# Open Data Examples

Documentation and tutorials for datasets published through the
[Registry of Open Data on AWS](https://registry.opendata.aws/).

| Dataset | Documentation | Tutorial | Registry |
|---|---|---|---|
| ALADIN Czech Republic 1 km Weather Forecasts | [README](aladin-cz-forecast/README.md) | [get-to-know-a-dataset.ipynb](aladin-cz-forecast/get-to-know-a-dataset.ipynb) | [registry.opendata.aws/aladin-cz-forecast](https://registry.opendata.aws/aladin-cz-forecast) (pending) |

## ALADIN Czech Republic 1 km Weather Forecasts

An archive of hourly 72-hour forecasts from the ALADIN model of the Czech Hydrometeorological Institute
(ČHMÚ), on its 1 km CZ_1km domain. The archive starts in June 2025 and is stored as cloud-optimized Zarr v3 in a
public S3 bucket, so you can read it anonymously, without an AWS account:

```python
import xarray as xr

ds = xr.open_zarr("s3://DOPLNIT-bucket-name/meteo_data/202602/CLSTEMPERATURE.zarr",
                  consolidated=True, storage_options={"anon": True})
```

- Data source: Czech Hydrometeorological Institute (ČHMÚ), <https://opendata.chmi.cz/>
- Data license: [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/)
- Processing code: <https://github.com/dandamasek/Meteo-data-on-S3>

## Running the notebooks

```bash
pip install "xarray>=2025.1" "zarr>=3.0" "s3fs>=2025.1" "dask>=2025.1" "pandas>=2.2" "matplotlib>=3.9" jupyterlab
jupyter lab
```

## License

Data: see each dataset's README. Code and documentation in this repository: DOPLNIT (e.g. MIT for code, CC BY 4.0 for text).

## Contact

Questions and feedback: <https://github.com/dandamasek/open-data-examples/issues>
