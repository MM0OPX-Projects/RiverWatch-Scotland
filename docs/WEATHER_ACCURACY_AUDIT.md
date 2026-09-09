# RiverWatch Scotland Weather Accuracy Audit

Date: 2026-09-09

This is the investigation phase only. Production HTML, Android assets, scoring layout, and APK have not been changed.

## Findings

### 1. The app does not currently obtain a true upstream watershed rainfall estimate

`CATALOGUE_CACHE` contains 394 stations but the bundled station records have no latitude or longitude. `loadCatalogue()` uses that cache directly and does not call `stationCatalogueUrl()` to refresh station metadata (`index.html:172-177`). As a result, the app cannot locate most selected SEPA stations from the bundled data.

Only the hand-maintained `WEATHER_POINTS` table supplies coordinates for a limited set of rivers (`index.html:80-82`). For other rivers, `geocodeWeatherPoint()` may geocode a station, river, or catchment name and use the first Scottish result (`index.html:214-220`). This is a place lookup, not a hydrological boundary.

### 2. A single coordinate is used for each weather request

`weatherTargetForCurrentRiver()` averages any available station coordinates into one latitude/longitude (`index.html:222-231`). It does not use the selected station's upstream area, elevation distribution, or rainfall-gauge locations. This is especially weak for long rivers and Highland catchments where rainfall changes rapidly over short distances.

Open-Meteo also reports the centre of the forecast grid cell actually used, which may be several kilometres from the requested coordinate, and applies elevation downscaling based on its digital elevation model. That makes the chosen coordinate and elevation materially important.

### 3. The forecast model is implicit

`weatherUrl()` requests the default Open-Meteo forecast without an explicit model (`index.html:80`). Open-Meteo's documentation says Best Match selects the best model for the location; it is not a fixed, auditable model choice. Open-Meteo provides a UK Met Office UKV 2 km model for the UK and Ireland, but the open data has an approximately four-hour delay. That model should be benchmarked for 0-48 hour rainfall rather than adopted automatically.

### 4. The rainfall total has a time-boundary problem

`sumHours()` compares each forecast timestamp with `Date.now()` and adds the full value when the timestamp is between now and the target horizon (`index.html:257`). Open-Meteo describes hourly precipitation as a preceding-hour sum. Therefore the current partially completed hour can be counted as if it were a complete future hour, and the result depends on exactly when the app is opened. This can materially distort a 24-hour total during a heavy shower.

The calculation should use the API's hourly interval semantics, exclude the already elapsed portion of the current interval, and use the returned timezone/timestamp rather than relying on implicit device parsing.

### 5. The rainfall-to-river thresholds are universal

The UI labels all catchments using the same thresholds: over 5 mm in 24 hours means `May rise`, and over 12 mm means `Rise likely` (`index.html:263`). The fishing forecast adds further fixed rainfall increments at 3, 8, 18, and 22 mm (`index.html:268-272`). A small steep burn, a regulated river, a large lowland river, and a Highland catchment do not respond to the same rainfall total on the same timescale.

### 6. Weather and SEPA are technically independent, but the fishing interpretation is coupled

Weather is fetched separately and can render when SEPA level loading fails (`index.html:241-251`). That separation should be preserved. However, the fishing scores directly consume the point rainfall totals (`index.html:278-292`), so a misplaced or poorly timed forecast point can make an otherwise sensible level-based prediction look wrong.

## Data-source assessment

| Source | Useful resolution | Strength | Limitation | Recommended use |
|---|---:|---|---|---|
| SEPA station metadata | Station coordinates, river/catchment fields | Authoritative mapping for selected station | Does not by itself provide an upstream polygon | Replace bundled missing coordinates; keep cached |
| SEPA rainfall gauges | 15-minute raw observations; hourly endpoint | Ground truth for Scottish rainfall and bias checks | Around 275 gauges; transfer is typically once or twice daily | Validation and optional observed-rain context |
| Open-Meteo Best Match | Location-dependent model | Simple and low-friction | Model choice is implicit and point-based | Baseline only |
| Open-Meteo UKMO UKV | UK-wide 2 km; hourly; about 2-day horizon | Promising for short-range Scottish rainfall | Open-data delay of about four hours | Benchmark for 0-48 hour forecasts |
| Open-Meteo ensemble | Multiple model members and distributions | Gives uncertainty for convective/localised rainfall | More calls/data and not automatically catchment-weighted | Use only if backtesting shows value |
| Catchment sample points | Area/elevation-stratified | Practical approximation without hosting polygons | Requires a maintained point set and weights | Recommended first improvement |

## Recommended staged design

### Phase 1: fix correctness and observability

1. Refresh the station catalogue from SEPA metadata when possible and cache station latitude, longitude, catchment name, and catchment area.
2. Remove silent geocoding as the primary method. Retain it only as an explicitly labelled fallback.
3. Fix the hourly precipitation window and daylight-saving/timezone handling.
4. Display the actual weather model, coordinates, elevation, forecast issue time, and aggregation method in an auditable source label or detail view.

### Phase 2: use a small weighted catchment approximation

For each station, maintain three to five points upstream or representative of the station's contributing area: lower catchment, middle catchment, upper catchment, and one or two high-elevation points where relevant. Assign weights that sum to 1.0, preferably using area/elevation strata rather than an unweighted average.

Request all points in one Open-Meteo call. For each hour, calculate:

`catchmentRain[t] = sum(weight[i] * rain[i][t])`

Keep the weighted median or interquartile spread as an uncertainty signal. Do not show a false precision such as `3.7 mm` when the points range from dry to a local 15 mm shower.

### Phase 3: calibrate only after backtesting

Use SEPA rain gauges and subsequent SEPA level changes to learn conservative, station-specific response factors and lag windows. Keep a simple default for stations with insufficient history. Do not fit thresholds to the same events used to report accuracy.

## Initial recommendation

The best first production change is not an ensemble or a complicated hydrological model. It is:

1. authoritative SEPA station coordinates;
2. a maintained three-to-five-point upstream approximation;
3. an explicit UKV-versus-Best-Match comparison for the first 48 hours;
4. a correct forecast-hour window; and
5. catchment-specific, conservative rainfall response parameters added only after out-of-sample tests.

This should improve the Almond and other small catchments without imposing a large API or Cloudflare cost. A full polygon/grid solution can follow if the experiment shows that three to five points miss important orographic rainfall patterns.

## Preliminary backtest

The reproducible first-pass experiment is in `tools/weather-backtest.ps1`. It uses the last seven complete days available from the SEPA rainfall service, compares Open-Meteo Best Match with UK Met Office Seamless, and compares the nearest rainfall gauge with the mean of the three nearest gauges. The three-gauge result is a spatial validation proxy, not yet an upstream catchment estimate.

Results for daily rainfall totals:

| Test catchment | Model | One nearest gauge MAE | Three-gauge proxy MAE | One-point bias | Three-gauge bias |
|---|---|---:|---:|---:|---:|
| Almondell, Almond Lothian | Best Match / UKMO Seamless | 1.11 mm | **0.57 mm** | +0.63 mm | **+0.03 mm** |
| Perth, Tay | Best Match / UKMO Seamless | 1.31 mm | 0.99 mm | -0.54 mm | +0.15 mm |
| Norham, Tweed | Best Match / UKMO Seamless | 0.64 mm | 0.58 mm | +0.13 mm | -0.09 mm |
| Aberlour, Spey | Best Match / UKMO Seamless | 1.09 mm | 1.00 mm | +0.57 mm | +0.52 mm |
| Townfoot, Nith area | Best Match / UKMO Seamless | 0.87 mm | 1.26 mm | +0.07 mm | +0.86 mm |
| Kinbuck, Allan Water | Best Match / UKMO Seamless | 2.04 mm | 2.03 mm | -0.21 mm | +0.35 mm |

The two Open-Meteo model labels produced identical values for this short sample at these locations, so there is no evidence yet that switching model alone will improve RiverWatch. The spatial proxy helped Almondell, Perth, Tweed, and Spey, but hurt the Nith and made almost no difference at Kinbuck. That is exactly why the next experiment must use true upstream points and a longer event sample rather than adopting a universal three-point average.

The backtest currently measures rainfall only. It does not yet claim improved river-level prediction because the observations are SEPA rain gauges, not the subsequent SEPA level response at each river station. The next validation stage should add archived SEPA level series and event timing.

## Almondell-focused longer audit

The longer Almondell run is reproducible with `tools/almond-long-backtest.ps1`. It covers 30 complete SEPA reporting days from 2026-08-10 to 2026-09-08 and compares:

- the Almondell weather point (`55.895, -3.485`) against the mean of Harperrig, Gogarbank, and Murray Burn;
- a three-gauge forecast mean against that same three-gauge observation mean;
- fixed 24-hour and 48-hour previous-run forecasts; and
- Best Match versus UK Met Office Seamless model labels.

The complete output is in `tools/ALMOND_LONG_BACKTEST.md` and `tools/almond-long-backtest-results.json`.

### Results

| Forecast | Comparison | Lead 24 MAE | Lead 48 MAE | Lead 24 bias | Lead 48 bias |
|---|---|---:|---:|---:|---:|
| Almondell point | Three-gauge mean | 2.93 mm | 3.29 mm | +0.40 mm | +0.46 mm |
| Three-gauge mean proxy | Three-gauge mean | **2.86 mm** | **3.12 mm** | **+0.28 mm** | **+0.36 mm** |
| Almondell point | Harperrig only | 2.64 mm | 3.09 mm | +0.50 mm | +0.55 mm |

The short-range lead is better than the 48-hour lead, but the three-gauge improvement is modest. Correlation is near zero for this month, which means the rainfall totals are not reliable enough to drive sharp fishing-condition changes on their own. The two model labels returned identical values in this sample, so changing the model name is not justified as a fix.

### What the Almondell level record shows

The event table in the detailed report shows a more useful pattern than the rainfall total alone. Rainfall around 8-11 mm across the three gauges was followed by sizeable Almondell level rises, often within 12-24 hours: about 0.19 m after the 8.3 mm event on 17 August and the 10.5 mm event on 19 August. Smaller totals also sometimes preceded a rise, including roughly 0.13 m after 5.3 mm on 2 September and 0.15 m after 3.0 mm on 7 September. This indicates that antecedent wetness, event timing, and the current level matter; a universal threshold such as “over 5 mm means rise likely” is too blunt.

Using the 30 daily observations in this run, the correlation between the three-gauge rainfall total and the maximum subsequent Almondell rise was approximately **0.19 at 6 hours, 0.57 at 12 hours, and 0.88 at 24 hours**. This is encouraging evidence for a delayed rainfall-to-level response, but it must be treated cautiously because the sample is short and adjacent rainfall events can overlap. It does, however, explain why the app is currently undercalling the rise: its current display classifies the next 24/48 hours of forecast rain, while the river response commonly develops later and can reflect rainfall that has already fallen.

For the Almond specifically, production logic should therefore require several signals before downgrading trout and grayling: a meaningful recent level rise, evidence that the rise is still active or that water remains coloured, and recent/forecast rainfall over a catchment-specific lag window. A quiet level with only a small 12-hour change should not be labelled poor just because a forecast point contains rain. The next implementation should use the level trend and recent level range as the primary signal, with rainfall as a cautious modifier.

This remains a one-month, single-station audit. The results support a conservative Almondell-specific rule change and a longer historical validation, but not a claim that the weather forecast itself is accurate enough to predict fishing success without the observed SEPA level and trend data.

## v0.24 implementation

Version 0.24 introduces an experimental expected-rise range rather than a single precise figure. At Almondell, it combines the observed 6h, 12h, and 24h level movement with the mean of recent Harperrig, Gogarbank, and Murray Burn rainfall observations, plus the next 24h and 48h Open-Meteo rainfall. The three rainfall requests are cached for 30 minutes. Other stations use a wider conservative fallback based on level movement and the available weather point.

The displayed confidence is High only when fresh Almondell level, SEPA rainfall-gauge, and weather signals are available and agree. Partial or generic inputs produce Medium or Low confidence. The estimate is unavailable when recent level history cannot support it.

Fishing scoring now treats the time horizons separately. Current conditions are driven by observed level state and do not change merely because a forecast changes. The 24h and 48h trout outlooks progressively penalise a forecast rise and allow recovery after settling. Salmon outlooks favour a moderate fresh rise or the subsequent falling phase, while a large rise/full spate remains penalised. Recent scores are locally smoothed, except when an observed rapid rise or full-spate state requires an immediate safety-significant downgrade.

Controlled tests cover stable/dry, modest rise, large rise, actively rising/coloured water, falling fresh water, and missing prediction data. These implementation coefficients remain experimental and should be recalibrated as a longer Almondell event history becomes available.

## Evidence and documentation

- [SEPA rainfall data download and API](https://www2.sepa.org.uk/rainfall/DataDownload)
- [SEPA time-series rainfall queries](https://timeseriesdoc.sepa.org.uk/api-documentation/api-endpoint-examples/time-series-data-queries/)
- [SEPA time-series metadata and quality codes](https://timeseriesdoc.sepa.org.uk/api-documentation/api-function-reference/principal-query-functions/)
- [Open-Meteo UK Met Office model documentation](https://open-meteo.com/en/docs/ukmo-api)
- [Open-Meteo forecast response and grid-cell/elevation behaviour](https://open-meteo.com/en/docs)
- [Open-Meteo ensemble API](https://open-meteo.com/en/docs/ensemble-api)
