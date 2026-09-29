# Excluded: freight-only layers

Layers kept on disk but **removed from the map** because they represent freight-only operation.
This map shows passenger rail only — see `../../CLAUDE.md` Rule 1.

Nothing here is listed in `data/manifest.json`, so `app.js` never loads it. Do not re-register a
layer from this folder without first establishing that it carried passengers, and rewriting its
segments to cover only the passenger years.

## Contents

### `toledo_ohio_central_kanawha_michigan.geojson` — excluded 2026-08-24

The Toledo & Ohio Central and its Kanawha & Michigan subsidiary, three features covering the Western
Branch, Eastern Branch and Southern Branch. Its own metadata opens "The New York Central's coal
railroad", the layer exists to move Kanawha Valley coal to the Toledo lakefront docks, and its
segment notes state the headway "stands in for coal-train density and is estimated" — i.e. the
frequency figures were never passenger frequencies.

The real T&OC did run passenger trains, so this corridor could legitimately return to the map — but
only after someone researches the actual passenger service and dates the segments to it. The layer
as written documents coal traffic and cannot just be re-registered.
