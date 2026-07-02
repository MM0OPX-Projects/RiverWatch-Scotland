# RiverWatch Scotland

Your guide to Scotlands river fishing conditions.

## Files

- `index.html` - browser-based HTML version of RiverWatch Scotland.
- `apk/RiverWatch-Scotland-v0.5-debug.apk` - current Android debug APK build with bundled river catalogue, station-level caching, and Open-Meteo weather fallback.
- `apk/RiverWatch-Scotland-v0.4-debug.apk` - station-level caching build.
- `apk/RiverWatch-Scotland-v0.3-debug.apk` - Android build with bundled river catalogue.
- `apk/RiverWatch-Scotland-v0.2-debug.apk` - previous Android debug APK build.
- `apk/RiverWatch-Scotland-v0.1-debug.apk` - earlier Android debug APK build.

## Data Sources

RiverWatch Scotland uses SEPA Water Levels, the SEPA Time Series API, and Open-Meteo forecast/geocoding data. The app includes a bundled SEPA station catalogue cache so the river list remains available if SEPA rate-limits the catalogue API. Station level readings are cached after a successful read so recent cached data can be shown if SEPA temporarily returns a 429 credit-limit response.

Created by Colin Summers.
