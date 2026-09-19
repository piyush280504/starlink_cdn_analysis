# Learning guide: `main.py`

## What this program does

`main.py` is an exported Jupyter notebook rather than a conventional command-line application. It analyses Starlink and terrestrial CDN measurements, compares their latency, and writes seven PDF figures. It also analyses a satellite-network simulation and produces two additional latency plots.

Run it from the repository directory with:

```powershell
python .\main.py
```

The script expects the downloaded `m1756495` dataset and reads two optional environment variables:

- `DATA_PATH`: where extracted data should live. Its default is `..\d`, a deliberately short Windows path.
- `OUTPUT_PATH`: where the output directory is created. Its default is `..\Figs`. Note that the plotting calls themselves currently save to `..\Figs`.

## Entry points

There is **no `main()` function** and no `if __name__ == "__main__":` guard. Consequently, there are two practical entry points:

1. **Command-line entry:** `python main.py` starts at line 1 and runs every top-level statement in file order.
2. **Import entry:** `import main` also runs every top-level statement. This is usually surprising; importing the module triggers archive extraction, data processing, figure creation, and `plt.show()` calls.

The module begins by importing libraries and defining its first two helper functions. After that, execution immediately starts configuring plots, locating data, extracting archives, processing measurements, and generating figures.

## High-level execution flow

```text
Start script
  -> configure Matplotlib and city-name helpers
  -> locate/link/extract the four dataset ZIP archives
  -> aggregate Starlink NetMet browser measurements by country and URL
  -> aggregate terrestrial NetMet browser measurements by country and URL
  -> create FCP and HTTP-response comparison plots
  -> load Starlink and terrestrial Cloudflare measurements
  -> create Maputo-to-CDN maps and a global RTT-difference map
  -> load satellite simulation files
  -> calculate path latencies, build a DataFrame, create CDF and box plots
  -> finish naturally
```

## Setup and data loading

### City-name cleanup — `city_lookup_key(value)` (lines 31–42)

Some records use incorrectly decoded text such as `GÃ¶ttingen`. This function makes names comparable:

1. It returns a non-string value unchanged.
2. It tries to reinterpret Latin-1 bytes as UTF-8.
3. It decomposes accented characters with Unicode NFKD normalization.
4. It keeps only letters and digits, then lowercases the result.

For example, visually different spellings can become a comparable lookup key. It returns a normalized string.

### City lookup — `find_city(city_name, normalized_cities)` (lines 45–50)

This function calls `city_lookup_key`, then looks for an exact key in `normalized_cities`. If there is no exact match, `difflib.get_close_matches` performs a conservative fuzzy match with a similarity threshold of `0.86`. It returns the matching city metadata dictionary or `None`.

### Archive/data setup (lines 57–119)

The program:

1. Finds the project directory from `__file__` and switches the working directory there.
2. Treats `m1756495` as the source archive directory.
3. Uses `..\d` as the default extracted-data directory, keeping paths below the Windows length limit.
4. Creates hard links to the four ZIP files when possible; otherwise it copies them.
5. Extracts each ZIP only when its `.extraction_complete` marker is absent.

The four archives are NetMet measurements, helper files, Cloudflare measurements, and simulation results.

## NetMet aggregation and first figures

### Starlink aggregation (lines 126–207)

The first loop reads files whose names contain AS14593, Starlink's ASN. It ignores speed-test records and retains web-browsing records only when:

- a client city can be resolved to a country;
- HTTP status is `200`;
- the CDN cache result contains `hit`.

The resulting nested dictionary is:

```text
country -> URL -> metric -> list of observed values
```

Metrics include HTTP response time, DNS lookup time, TCP connection time, page load time, and first contentful paint (FCP).

### Terrestrial aggregation (lines 214–300)

This repeats the same process for non-Starlink files. It ignores speed tests, Starlink records, and clients marked as using a VPN in `all_netmet_user_ip_details.json`.

### FCP box plot and HTTP-response CDF (lines 312–437)

The first plot compares Starlink and terrestrial FCP for Germany and Great Britain. The second plot compares the per-URL median HTTP response-time difference for all countries found in both datasets. A positive difference means Starlink took longer.

## Cloudflare analysis and geographic plots

Lines 445–867 load two Cloudflare datasets: global Starlink client results and terrestrial client-facing results. They build dictionaries that group RTT samples by country, city, and CDN airport/location.

The code then:

- builds a Maputo client-to-CDN map for Starlink;
- builds the equivalent terrestrial map;
- selects the best CDN per user city, requiring at least 100 RTT samples;
- aggregates best-CDN RTTs by country;
- produces a global map of median Starlink RTT minus median terrestrial RTT.

Negative values on the global map mean Starlink was faster; positive values mean terrestrial access was faster.

### `plot_gsdata(gsdata_df, ax, label, marker, color, size=10, lw=0.6)` (lines 874–882)

Adds point-of-presence locations to an existing map axis. It reads `lat` and `long` columns from a DataFrame and calls `ax.scatter`. It returns nothing; Matplotlib keeps the plotted artist on the axis.

## Simulation-analysis functions

The final section turns satellite path/simulation JSON files into latency distributions.

### `find_files(directory, prefix)` (lines 962–963)

Returns filenames in `directory` whose names start with `prefix`.

### `parse_timestamp(filename)` (lines 966–969)

Extracts the final two underscore-separated filename components and parses them as a `%Y%m%d_%H%M%S` `datetime`. It returns that timestamp.

### `read_json_file(filepath)` (lines 972–974)

Opens a JSON file and returns the decoded Python object.

### `process_gsl_files(directory)` (lines 977–987)

Finds all `gsl_latency_bw_` files, collects every `latency` field, writes a comma-separated `cdf0` file, and returns the latency list. “GSL” is the ground-station link portion of a satellite path.

### `process_path_files(directory)` (defined twice)

There are two functions with this name:

- The first definition at lines 990–1018 prints diagnostics and writes `other_cdfs`.
- The second definition at lines 1079–1097 replaces the first one before it can be called.

Therefore, the active version only reads each `path_` file, finds the closest timestamped GSL and ISL files, calculates a latency vector, and returns the list of vectors. It does **not** write `other_cdfs` or print the diagnostics from the earlier definition.

### `find_matching_file(directory, prefix, timestamp)` (lines 1021–1023)

Finds all files with the requested prefix and returns the one whose parsed timestamp is closest to `timestamp`.

### `process_http_responses(filename)` (lines 1026–1041)

Loads nested simulation HTTP-response data. It traverses outer group, city, provider, and location layers, keeps responses strictly between 25 and 80 ms, and returns one flat list.

### `calculate_latencies(path_info, gsl_data, isl_data)` (lines 1044–1076)

This is the core simulation calculation.

1. The first list item is a path filename; the rest are satellite identifiers in the route.
2. It reads source and destination station identifiers from the filename.
3. It finds the ground-to-first-satellite latency.
4. It adds an inter-satellite-link (ISL) latency for each subsequent hop, up to ten hops.
5. Missing hops are filled with the last calculated non-zero latency.
6. It returns ten cumulative latency values, one for each hop count.

### `read_terrestrial(filename)` (lines 1100–1102)

Loads a JSON array and returns only values below 80 ms.

### `create_dataframe(cdf0, other_cdfs, cdf7, cdf8)` (lines 1105–1126)

Creates a Pandas DataFrame with satellite-cache scenarios (`1st Sat`, `3 ISLs`, `5 ISLs`, and `10 ISLs`) plus Starlink and terrestrial observations. Shorter series are padded with `NaN` so columns have equal length. `cdf0` is accepted but not used in the current DataFrame.

### `plot_cdf(df)` (lines 1129–1160)

For every DataFrame column, removes missing values, sorts the remaining values, calculates empirical percentiles, and plots a CDF. The function saves `cdn_cdf_plot.pdf` and returns nothing.

### `adjust_box(plot, idx)` (lines 1163–1166)

Changes the first box's colour and median colour. This definition replaces the earlier `adjust_box(plot)` function from lines 315–318. Neither version is called by the current program, so they have no runtime effect.

### `plot_boxplot(df)` (lines 1169–1213)

Selects the `3 ISLs`, `5 ISLs`, and `10 ISLs` columns, makes a horizontal box plot, colours its components, adds a 17 ms terrestrial reference line, saves `satellite_cdn_percentile.pdf`, and returns nothing.

## Final top-level calls

Lines 1219–1234 assemble the simulation inputs and call:

```python
cdf0 = process_gsl_files(directory)
other_cdfs = process_path_files(directory)
cdf7 = process_http_responses(http_response_file)
cdf8 = read_terrestrial(terrestrial)
df = create_dataframe(cdf0, other_cdfs, cdf7, cdf8)
plot_cdf(df)
plot_boxplot(df)
```

This is the closest thing the script has to its final analysis entry point.

## Exit points

The program has no explicit `sys.exit()` or `exit()` call. It ends naturally after `plot_boxplot(df)` returns. It may exit early through an uncaught exception, including:

- `FileNotFoundError` if `m1756495` is absent (line 80) or expected data files are missing.
- `KeyError` when an input record lacks an assumed dictionary key.
- `json.JSONDecodeError` for malformed JSON.
- `zipfile.BadZipFile` for corrupt archives.
- Matplotlib, GeoPandas, Cartopy, or filesystem exceptions while generating figures.

Inside `process_path_files`'s first (overwritten) definition, an exception while writing `other_cdfs` is caught, printed, and causes that function to return `None`; this behavior is irrelevant because the second definition replaces it.

## Files written

The script writes these PDFs to `..\Figs`:

- `fcp_abs_boxplot_horizontal.pdf`
- `starlink_vs_terrestrial_country_wise_http_delta.pdf`
- `starlink_maputo_cdn_latency.pdf`
- `terrestrial_Maputo_cdn_latency.pdf`
- `terrestrial_vs_starlink_rtt_difference_global_heatmap_bestCDN.pdf`
- `cdn_cdf_plot.pdf`
- `satellite_cdn_percentile.pdf`

It can also write intermediate simulation files named `cdf0` and `other_cdfs` under the extracted simulation directory, although the active `process_path_files` definition no longer writes `other_cdfs`.

## Suggested next refactor

To make this easier to learn from and reuse, first move each top-level analysis block into a named function, then add:

```python
def main():
    # setup, load data, build plots
    pass


if __name__ == "__main__":
    main()
```

That change would make importing the helpers safe, provide one clear program entry point, and allow unit tests for the calculation functions.
