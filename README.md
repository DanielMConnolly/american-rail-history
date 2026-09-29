# American Rail — Historical Network (data)

A GeoJSON dataset of US passenger rail from 1830 to today: intercity and commuter railroads,
interurbans, streetcars, rapid transit, light rail and tourist lines. Every line records the years
it carried passengers, who operated it, and where those facts come from.

This repository contains the data only.

## Layout

- `data/manifest.json` lists every layer, under three keys:
  - `layers`: line features
  - `stations`: station gazetteers
  - `timetables`: per-route departure files
- `data/<name>.geojson` is one FeatureCollection per railroad or system. Each feature carries:
  - `company`, `line`, `id`, `color`, `mode`, `avg_speed_kmh`
  - a `segments[]` array of dated service eras (`start`, `end`, `headway_min`, `note`, `sources`)
- `data/stations/` holds station Point features, each defined once, with the years each company
  served it.
- `data/timetables/` holds per-route departure times, following GTFS conventions.
- `data/_excluded_freight_only/` holds layers deliberately kept off the map. They aren't listed
  in the manifest.

## What the data claims, and what it doesn't

The rules are in [CLAUDE.md](CLAUDE.md) and [data/CONVENTIONS.md](data/CONVENTIONS.md). In short:

- **Passenger service only.** A line's drawn years stop when passengers stopped, even if freight
  continued.
- **No animal power.** Street railways start at electrification, not at the horsecar era.
- **Geometry quality is labelled.** Most routes are traced from OpenStreetMap. Any straight-line
  approximation is named in a `geometry_quality` field.
- **Sourcing is honest.** The first entry of every segment's `sources` array says whether its dates
  were checked. A `NOT SOURCED` marker means exactly that. `headway_min` and `avg_speed_kmh` are
  usually estimates.
- **One company, one color** across every file.

Corrections with citations are welcome as pull requests.

## License

The data is released under the [Open Database License (ODbL) 1.0](DATA-LICENSE.md). Route
geometry © OpenStreetMap contributors.
