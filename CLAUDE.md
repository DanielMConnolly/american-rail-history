# American Rail — Historical Network

The dataset behind a map of historical US passenger rail. `data/manifest.json` lists every GeoJSON
layer, and each line is drawn only during the years its `segments[]` cover (1830–2026). The map
application itself is not part of this repository; these are the rules the data follows.

## Rule 1: passenger rail only — no freight-only trains

**This map shows passenger service. A line must not appear on the map for years when it carried
no passengers.**

When a railroad loses its passenger service but keeps hauling freight, its segments **end there**.
Do not add a trailing "freight only" segment, and do not add a feature for a freight-only railroad,
branch, or short line — however historically interesting it is.

- ✅ `1898–1957` ending because passenger service ended in 1957.
- ❌ `1957–1996` noted as "freight only after passenger service ended".
- ❌ a feature for a coal branch or a modern short line that hauls only freight.

It is fine — encouraged — for a segment note to *mention* what happened after passengers stopped
("freight continued until 1983"). That is context, not a claim of service. The test is whether the
**drawn years** assert passenger service that did not exist.

**These do count as passenger service and belong on the map:**
- tourist and heritage excursions (use `mode: "tourist_rail"`)
- mixed trains that carried passengers along with freight
- charters and one-off passenger specials, if that is genuinely what ran

A repo-wide audit on 2026-08-24 removed the freight-only content that had accumulated; each file
touched carries a `freight_only_audit` metadata field saying what was removed. `data/_excluded_freight_only/`
holds a layer pulled from the map in full. See `data/CONVENTIONS.md` for the audit method and for how
to re-check this rule.

## Rule 2: say how good the geometry is

Route geometry should be real-traced from OpenStreetMap (Overpass) wherever the alignment survives.
When it doesn't, a straight-line approximation is acceptable **only if it is labelled**:

- put a `geometry_quality` string on the feature, naming every approximated leg and its mileage
- record traced length against the documented mileage, so the reader can judge the fit
- state when a feature draws a **later alignment than its own era** (e.g. a pre-tunnel era drawn on
  the post-tunnel route) — this is a common and easy-to-miss error

## Rule 3: don't assert what isn't sourced

`headway_min` and `avg_speed_kmh` are almost always estimates. Say so in the segment note or in
`speed_source`. Put unresolved questions in a `not_yet_sourced` metadata field, and record source
conflicts rather than silently picking one.

**Every segment carries its own `sources` array.** A repo-wide pass on 2026-08-24 added one to all
2,300 segments. The first entry is a status line and the rest are reference URLs:

- `Segment-level: …` — the segment's own note names the source for its dates and service.
- `NOT SOURCED AT SEGMENT LEVEL: …` — the layer has references, but nobody has checked them against
  *this* segment's start year, end year or headway.
- `NOT SOURCED: …` — neither the note nor the layer metadata cites anything. Treat the dates as
  unverified.

When you add or edit a segment, set its `sources` honestly: cite what you actually checked, and
leave the NOT SOURCED marker in place if you checked nothing. Do not copy a layer's reference list
into a segment to make it look sourced — the whole point of the field is to show where the gaps are.

## Rule 4: one company, one color

**Every feature belonging to the same real-world company uses the same `color` value** — across
every file, not just within one, and **with no exception for modern transit agencies**: WMATA,
MARTA, NYC Subway, Metra, LA Metro, RTD, etc. each get one color across all their lines, even
though the real agencies brand each line differently. If a line is later absorbed into another
company (e.g. leased, merged, or acquired), give the post-absorption segments/features the *new*
owner's color, not the original company's — the color should track who actually ran the trains in
that era, matching how `segments[]` already splits eras by ownership. Different name strings for the
same real company (e.g. a trailing `"(RW&O)"` on some features and not others) still count as one
company — normalize before comparing, don't match on the literal string.

Before picking a color for a new feature, check whether the company already has one elsewhere in
`data/` (grep other files for the `company` string) and reuse it. When you add a branch, extension,
or previously-missing piece of an existing railroad, add it as a new feature but do not use `Read`
plus manual hex-copying alone — pull the color programmatically from the existing feature (so a typo
can't silently create a near-duplicate shade) and confirm with a quick script that every feature
sharing that `company` string has an identical `color`.

A repo-wide audit on 2026-08-24 unified color across 33 companies with pre-existing mismatches
(historical railroads and transit agencies alike, including collapsing a deliberately
multi-colored-per-line watertown_rome_railroad.geojson down to one color per company) — 36 files
touched, canonical color chosen per company by most-used-hex. If you find a new mismatch, fix it the
same way: pick the most-common existing color for that company and apply it everywhere.

A second pass the same day fixed a related but distinct bug: several features whose `segments[]`
covered *multiple owners over time* (e.g. an independent railroad later leased or merged into the
New York Central) were a single feature colored entirely in the *original* company's color, even for
the years NYC actually ran the trains. The fix there isn't recoloring — it's **splitting the feature
at the ownership-change date** into two features sharing identical geometry: one keeping the
original company/color for the pre-change segments, one recolored to the new owner for the
post-change segments (see `ulster_delaware_railroad.geojson`, `watertown_rome_railroad.geojson`,
`utica_black_river_railroad.geojson`, `west_shore_railroad.geojson`, `canada_southern_railway.geojson`
for examples). Split at the date a segment's own note names the new operator (a lease counts, not
just a formal merger) — don't wait for outright corporate dissolution. When auditing a file, check
both things: does every feature for a company share its color (recolor to fix), and does any single
feature's `segments[]` quietly span an ownership change without splitting (split to fix)?

## Rule 5: no animal-powered rail — no horsecars

**This map shows mechanically powered rail. Horsecars and mule cars do not belong on it**, however
early or historically important the line.

A great many street railways and a fair number of early steam roads began under animal traction and
were later electrified or converted to steam. Draw them **from the year they stopped using animals**,
not from the year the company or the corridor began:

- ✅ a street railway drawn `1894–1941`, because it electrified in 1894.
- ❌ the same line drawn `1866–1941`, with the first decades noted as "horsecar era".
- ❌ a feature whose whole point is the horsecar operation, e.g. `line: "… (horsecar): A – B"`.

As with Rule 1, the test is what the **drawn years** assert. A note may freely *mention* the animal
era as context ("horsecars ran this street from 1866; electrified 1894") — that is history, not a
claim of service. What must not happen is a segment whose years cover animal haulage.

This does **not** exclude cable cars, inclined planes, or other mechanical traction; `mode:
"cable_car"` remains valid. It is animal power specifically that is out of scope.

An audit on 2026-08-24 removed the horsecar content then in the repo; each file touched carries a
`horsecar_audit` metadata field saying what was removed.

## Rule 6: name every line "<company>: <line>"

**A feature's `line` label is what the map shows on hover, so it must say who ran the train and what
the passenger called it** — in that order, separated by a colon:

```
<operating company, common display name>: <line name as advertised to passengers>
```

- ✅ `Delaware and Hudson: Adirondack Branch`
- ✅ `Amtrak: Carolinian`
- ✅ `New York State Railways: Westcott`
- ❌ `Adirondack Branch` — no company, so the label doesn't say whose train it is
- ❌ `A&S: Albany - Oneonta - Sidney - Afton - Harpursville - Binghamton` — an itinerary, not a name

**The company half** is the common display name, not the legal one: drop `Railroad`, `Railway`,
`Company`, `Corporation` where the name still reads correctly without them (`New York Central`, not
`New York Central Railroad`), but keep them where they are part of how the company is known
(`Long Island Rail Road`, `New York State Railways`, `MBTA Commuter Rail`). Use a well-known
nickname if there is one (`Big Four`). Where a feature spans several operators, name the one that
ran it for **most of its drawn years** — the same company whose color the feature carries under
Rule 4.

**The line half** is the name a passenger would have used: a branch, division, route or train name
(`Hudson Division`, `Oyster Bay Branch`, `Crescent`, `8-Main`). Many lines never had one — for those
give the two terminals joined by an en dash (`Albany – Binghamton`). Do **not** list intermediate
stops; a short parenthetical qualifier is fine (`Baltimore – Gettysburg (via Emory Grove)`). Aim for
under about 70 characters overall.

The full station list, the corporate succession and the operating detail all belong in the segment
notes and the metadata, not in the label.

An audit on 2026-08-24 renamed 967 labels to this pattern; each file touched carries a
`line_label_audit` metadata field.

## Rule 7: inter-city trolleys are `interurban`, and they draw dashed

**An electric railway built to link separate communities is `mode: "interurban"`, not `streetcar`
and not `intercity_rail`.** The map draws those dashed; everything else stays
solid. Colour is spoken for by Rule 4, so the stroke pattern is the only channel left for mode —
don't reach for colour to distinguish a mode.

The test is what the line was *for*, not how long it is. Length is a bad proxy: Rochester's city
radials run 10–15 mi while the Dunkirk–Fredonia interurban is 3.

- ✅ `interurban` — Rochester – Geneva, Denver – Boulder, Buffalo – Niagara Falls, Dunkirk – Fredonia
- ❌ `streetcar` routes inside one built-up area, however long — Buffalo's numbered IRC routes,
  Rochester's named radials, the Triple Cities car lines
- ❌ modern light rail — San Diego, RTD Denver, LA Metro, Muni Metro stay `light_rail`

An audit on 2026-08-24 retagged 26 features to `interurban`; each file touched carries an
`interurban_audit` metadata field.

## Rule 8: a station claims only the years its source proves

**Stations are Point features in `data/stations/`, and each one is defined exactly once.** The
gazetteer is the authority on where a place is and what it is called; everything else refers to it by
`id`. Ids are place-based (`albany-rensselaer-amtrak-station`), not operator-based, because one
platform will eventually carry several companies and a century of history — the same "one station,
one record" discipline Rule 4 applies to colour.

A station's `services[]` says who called there, in which years, how often and where to. Each entry
names a line feature by `line_id`, and that reference is load-bearing:

- the popup pulls the **company's colour from the referenced feature**, so Rule 4 holds across the
  station layer without a hex string ever being copied
- `stationcheck.py` **fails** if a service claims years outside that line's own `segments[]`, which
  is how Rule 1 is enforced on station data

**The years must come from the source, never from the route.** A GTFS feed describes current service
and proves nothing about which stops existed twenty years ago. Inheriting a route's start year down to
its stops would assert trains at infill stations that had not been built — exactly what Rules 1 and 3
forbid. A station's years move earlier only when a published timetable names that stop, which is what
`timetablepull.py` reads out of Amtrak's system timetables (2008–2018, born-digital PDFs from
[juckins.net](https://juckins.net/amtrak_timetables/archive/home.php)) into
`data/stations/_timetable_evidence.json`.

Three things follow, and they are the whole discipline of the backfill:

- **Only positive evidence counts.** A station not found on a route's page is left un-backfilled, never
  recorded as absent. A miss is ambiguous — New York's Moynihan Train Hall is missing from a 2018
  timetable because it opened in 2021, while Syracuse is missing because it was then printed under a
  different name.
- **Evidence is split along the line's own spans.** A service may not run through years its line has no
  segment for. The Adirondack's segments stop in 2020 and resume in 2023, so its stations get an entry
  each side of the suspension rather than one claiming straight through. Evidence landing in *no* span
  is a misattribution and is discarded with a report — that is how six Valley Flyer stations, seen on
  2008 Springfield Line pages for a route that began in 2019, were caught.
- **A frequency belongs to the year it was counted in.** The older timetables establish that a train
  stopped, not how often, so `departures_per_day`, `destinations` and the linked `timetable` are
  attached only to the entry covering `frequency_year`. The popup withholds all three in any other year
  rather than back-dating them.

Not yet covered: 2019–2025 (Amtrak stopped publishing system timetables after June 2018) and anything
before 2008, which survives only as scanned images at [timetables.org](http://www.timetables.org/) and
would need OCR.

Other things that hold:

- `services[].sources` uses the same three Rule 3 status markers as segments.
- A station dot takes **no company colour** — it belongs to no one company.
- Timetable files under `data/timetables/` keep the GTFS convention of hours past 24 for a stop after
  midnight, so times stay ordered along a train. Only the display normalises to wall clock; a
  departure board is not the place for "39:27".
- Rules 1 and 5 apply to station-years exactly as they do to drawn line-years.

The pipeline runs in this order, and `stationpull.py` is the only writer of the gazetteer — so
regenerating from GTFS never loses the backfill, it just re-reads the evidence file:

1. `python3 stationpull.py` — GTFS → gazetteer + per-route timetables (`--refresh` re-downloads).
2. `python3 timetablepull.py` — published timetables → `_timetable_evidence.json`.
3. `python3 stationpull.py` again — folds the evidence in as start years.
4. `python3 stationcheck.py` — must pass. Declared geometry gaps are warnings; contradictions between
   a station, its timetable and its line are errors.

## Rule 9: the contiguous United States only — no Hawaii, no Puerto Rico, no Alaska

**This map covers the Lower 48.** Do not add layers for Hawaii, Puerto Rico, Alaska, Guam or any
other non-contiguous US territory, however well documented the railway or however cleanly it traces.

The map will not pan outside the rectangle `[23.5, -126.5]` to `[51.0, -66.0]`. **That rectangle is
the scope decision, not an accident of the current data.** A layer outside it would count toward the
map's mileage and could never be looked at.

Southern Canada is the one thing beyond the border that stays, because a handful of US lines genuinely
ran there (Winnipeg, Ottawa, Montreal, the Canada Southern) — those are US railroads reaching across a
land border, not a separate network.

An attempt on 2026-08-29 added Honolulu's Skyline and San Juan's Tren Urbano, both traced cleanly from
OSM route relations, and widened `US_BOUNDS` to make them visible. Both layers were removed and the
bounds restored. If a coverage audit reports Hawaii or Puerto Rico as a gap, that is the audit being
wrong about scope, not a gap to fill.

## Adding a layer

1. Write `data/<name>.geojson` — a FeatureCollection whose features carry `company`, `line`, `id`,
   `color`, `mode`, `avg_speed_kmh`, `segments[]` (`start`, `end`, `headway_min`, `note`).
   `line` must follow Rule 6: `"<company>: <line name>"`.
2. Add the filename to `data/manifest.json` → `layers`.
3. Check that the file parses and that every segment's years are what its sources support.

`manifest.json` has three keys. `layers` is line features only — their geometry is read as a
LineString, so a Point in one is a bug. Station gazetteers
go in `stations`, and `timetables` lists per-route departure files, which are **not** fetched at load
— the map requests one the first time a station needs it.

`mode` values in use: `intercity_rail`, `commuter_rail`, `rapid_transit`, `light_rail`, `interurban`,
`streetcar`, `elevated_rail`, `cable_car`, `tourist_rail`. The map reads `mode` to pick the stroke
pattern (see Rule 7); everything else about it is metadata.

## Version control

This is the public repository. Changes come in as pull requests against `main`; commit before any
bulk edit so `git diff` shows exactly what changed.

The data pipeline scripts referenced above (`stationpull.py`, `timetablepull.py`,
`stationcheck.py`, and the OSM tracing tools) are not yet published in this repository.
