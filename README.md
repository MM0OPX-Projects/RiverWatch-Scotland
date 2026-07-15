# RiverWatch Scotland

RiverWatch Scotland is a lightweight river-fishing conditions helper for Scottish rivers. It combines SEPA river level readings, recent level trends, catchment weather, rainfall forecasts, pressure, and simple fishing-method scoring into one mobile-friendly dashboard.

Created by Colin Summers.

## What The App Does

- Lists Scottish SEPA river level stations by river/catchment.
- Lets you search for a river, then choose one of the available measuring stations.
- Shows the latest river level in metres.
- Calculates 3h, 6h, 12h, and 24h river-level change.
- Estimates likely water clarity from current level and trend.
- Shows a 5-day or 31-day river-level graph with readable date labels.
- Shows weather for the selected river/catchment where a weather point can be resolved.
- Shows estimated rain for the next 24h and 48h.
- Gives separate trout/grayling and salmon/sea trout scores.
- Gives method guidance for fly fishing, spinning, and bait fishing for now, 24h, and 48h.
- Keeps weather loading independent from SEPA level loading where possible.
- Includes an Android APK and a standalone HTML version.

## How It Works

The app is a single-page HTML/JavaScript dashboard wrapped in a simple Android WebView APK.

At startup it loads a bundled SEPA station catalogue cache containing Scottish river/station names. This means the river list remains available even if SEPA rate-limits the live catalogue API.

When you select a river/station, the app asks a Cloudflare Worker proxy for recent SEPA level readings for that station. The Worker holds the SEPA API key as a Cloudflare secret, requests a short-lived SEPA access token server-side, and returns the SEPA data to the app. The API key is not stored in the public HTML or APK.

The app then calculates level trend, estimated clarity, and fishing condition scores. Successful station readings are cached locally on the device. A station is reused from the device cache for 15 minutes, reducing repeat API use when switching between stations or reopening the app. Older cached readings can still be shown if SEPA temporarily returns an error or the proxy is unavailable. The Refresh button deliberately requests new data.

The recommendation model adapts to each selected gauging station. It compares the current level with that station's own recent range, measures the relative speed and size of a rise, finds the latest peak, and estimates how long the water has remained settled. Faster-changing stations use about 48 hours as their settling period, while slower stations can require 60 or 72 hours.

Trout and grayling recommendations are deliberately cautious: they need locally normal or slightly low water, improving clarity, limited forecast rainfall, and the station-specific settled period before the app rates conditions as good. Salmon and sea trout recommendations favour a meaningful fresh rise followed by falling water and slight colour. Rapidly rising water, full spate, and locally very low stale conditions are penalised.

If SEPA live level data is unavailable, the app still lets you select a catchment and station and open the official SEPA station page from the app.

Weather uses Open-Meteo. Known catchment points are built in for several common rivers, and the app can fall back to Open-Meteo geocoding for other station/river names. No Open-Meteo API key is required.

## Important Limitations

This app is a guide, not an official safety or fishing authority.

SEPA API access depends on the configured Cloudflare Worker proxy and the SEPA access key limits. If the proxy, SEPA service, or access limit is unavailable, live level updates may not load. The river list should still work because it is bundled in the app.

Water colour/clarity is not directly measured by SEPA. The app estimates it from the station's recent rise, time since peak, level trend, and catchment rainfall, so local observation remains more reliable.

Fishing recommendations are estimates based on station-relative level, trend, recent rise and settling time, plus catchment rainfall. Local knowledge, regulations, conditions, and safety should always come first.

## Files

- `index.html` - browser-based HTML version of RiverWatch Scotland.
- `apk/RiverWatch-Scotland-v0.18-debug.apk` - current Android debug APK build.
- Older APK builds are kept in `apk/` for reference.

## Install On Android

This APK is not Play Store verified. Android will warn you because it is a manually installed debug APK.

1. Download `apk/RiverWatch-Scotland-v0.18-debug.apk` from this repository.
2. Open the downloaded APK on your Android device.
3. If Android blocks the install, choose the option to allow installs from that source, usually your browser or file manager.
4. Confirm the install.
5. Open `RiverWatch Scotland` from your app drawer.

If Android says the app is unsafe or unknown, that is because it is not signed by Google Play or distributed through the Play Store. Only install it if you trust this repository/source.

If you already installed an older build, install the latest APK over it. If Android refuses, uninstall the old RiverWatch Scotland app first, then install the new APK.

## Use The HTML Version

You can also use the app directly in a browser by opening `index.html`.

For local testing, serve the folder with a small local web server, for example:

```powershell
python -m http.server 8897
```

Then open:

```text
http://127.0.0.1:8897/
```

## Data Sources

- [SEPA Water Levels](https://waterlevels.sepa.org.uk/)
- SEPA Time Series API
- [Open-Meteo](https://open-meteo.com/) forecast and geocoding data

## Credits

Created by Colin Summers.
