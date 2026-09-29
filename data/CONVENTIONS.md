# Data conventions

See `../CLAUDE.md` for the rules. This file records the freight-only audit and how to repeat it.

## Freight-only audit — 2026-08-24

**Rule enforced: the map shows passenger rail only.** A feature's segments must end when its
passenger service ends. See `../CLAUDE.md` Rule 1.

### Removed

Segments dropped (the feature survives, now ending at its last passenger year):

| File | Feature | Segment | Why |
|---|---|---|---|
| canada_southern_railway | caso-main-amherstburg-fort-erie | 1968–1985 | "freight only" under Penn Central/Conrail |
| canada_southern_railway | caso-main-amherstburg-fort-erie | 1985–2001 | CN/CP rundown; no passenger service claimed |
| canada_southern_railway | caso-niagara-branch | 1968–2001 | "Penn Central, Conrail then CN/CP freight only" |
| canada_southern_railway | caso-detroit-river-tunnel | 1968–1985 | freight; tunnel is freight-only today |
| nyc_adirondack_division | nyc-adirondack-division-utica-lake-placid | 1965–1972 | "Freight only" |
| nyo_and_w_railway | nyow-main-line | 1953–1957 | "Freight only"; last passenger train 10 Sep 1953 |
| np_moscow_lewiston | np-moscow-lewiston | 1970–1997 | "freight-only decline" under BN |
| rochester_lockport_niagara_falls | nyc-falls-road-branch | 1957–1996 | "Freight only after passenger service ended" |
| snake_river_valley_wallula_riparia | snake-river-valley-wallula-riparia | 1958–2026 | "Freight-only Union Pacific main line" |

Whole features dropped:

| File | Feature | Why |
|---|---|---|
| adirondack_railway_north_creek | dh-tahawus-extension | sole segment labelled "FREIGHT LINE — no scheduled passenger service is documented" |
| adirondack_railway_north_creek | dh-warrensburg-branch | sole segment labelled "FREIGHT BRANCH — no passenger service is documented for it" |
| rochester_lockport_niagara_falls | falls-road-railroad | freight short line, no scheduled passengers |

Whole layer pulled from the map:

- `toledo_ohio_central_kanawha_michigan.geojson` → moved to `_excluded_freight_only/` and removed
  from `manifest.json`. Its own metadata calls it "The New York Central's coal railroad", and its
  segment notes say the headway "stands in for coal-train density" — a freight layer by its author's
  own description, with no passenger service documented in any of its three features.

Every file touched carries a `freight_only_audit` metadata field naming what was removed.

### Deliberately NOT removed

- **Segments that merely mention freight.** Most freight mentions are context at the end of a
  passenger era ("freight continued until 1983"). Those are correct and were left alone — 67 segments
  mention freight; only 9 asserted freight-only service.
- **Mixed trains.** Lines whose last service was a mixed freight/passenger train still carried
  passengers: `lehigh_valley_railroad` (Cortland gas-electric mixed train), `myrtle_beach_rail`,
  `utica_chenango_susquehanna_valley_railway` (the "Milk Train"). Kept.
- **Tourist and excursion operations.** `nyc_adirondack_division` (Adirondack Scenic/Adirondack
  Railroad) and `rochester_lockport_niagara_falls` (`medina-museum-excursions`) carry passengers.
  Kept, tagged `mode: "tourist_rail"` where added.
- **Freight-heavy common carriers that also carried passengers.** A vocabulary scan flagged 11
  features that use freight words and never say "passenger" — BR&P, Detroit Bay City & Alpena, DSS&A,
  Wyoming Central, NYO&W Scranton Division, Walla Walla & Columbia River and others. These were all
  common-carrier railroads with passenger service; the notes simply emphasise freight traffic.
  **Vocabulary alone is not evidence of freight-only operation** — do not bulk-remove on that basis.

### How to re-run the check

Flag any segment whose note asserts freight-only service:

```
grep -oih "freight[- ]only\|only freight\|freight line\|freight branch\|no scheduled passenger\|no passenger service is documented" data/*.geojson data/*/*.geojson
```

**Use `-i`.** The original version of this command was case-sensitive, which let `canandaigua_niagara_falls_peanut_line.geojson` through the 2026-08-24 audit: its note reads `Freight only.` with a capital F, so `freight[- ]only` never matched, and the map went on drawing a Canandaigua-Niagara Falls passenger line into 1978, 45 years after its last passenger train. Caught and fixed the same day by a case-insensitive re-scan, which found that one feature and no others.

Then read each hit **in the context of its whole feature**. The question is not "does it mention
freight" but "do the drawn years assert passenger service that did not exist". Also check any
feature whose final segment ends at `2026` — a line still drawn today should be one you can name a
current passenger train for.

## `_excluded_freight_only/`

Layers removed from the map but kept on disk. They are not in `manifest.json` and never load.
Repo-wide checks that compare `manifest.json` against every `.geojson` on disk will report these as
"unregistered" — that is expected, not a bug.

## Straight-line geometry audit — 2026-08-28

**Rule enforced: say how good the geometry is (`../CLAUDE.md` Rule 2).** Straight-line approximation
is allowed, but a ruled line drawn between two cities is still a guess about where the railroad went,
and the map should prefer real geometry wherever OpenStreetMap still has it.

### The detector — `straightscan.py`

A *straight run* is a maximal stretch of consecutive vertices in which every intermediate vertex sits
within **150 m** of the chord between the run's two endpoints. Real traced rail bends constantly, so
a run of any length is a ruled line by construction; the scan reports every run of **10 miles or
more**.

```bash
python3 straightscan.py                        # ranked report
python3 straightscan.py --min-mi 5             # lower the floor
python3 straightscan.py --json straight.json   # machine-readable, feeds the repair pass
```

Runs that fall inside an already-repaired stretch are reported separately as **verified** — the real
railroad is straight there (prairie main lines genuinely are), and re-tracing them would change
nothing. Verification comes from the `geometry_repairs` records the repair pass writes, so the
report never re-flags work already done.

### The repair — `railtrace.py` + `straightfix.py`

For each flagged run the repair asks Overpass for every `railway=*` way in a padded bounding box,
keeps only nodes within a **corridor** of the chord, and runs Dijkstra between the two endpoints.
Three details matter:

- **Endpoints are chosen per connected component, not independently.** Snapping each end to its own
  nearest node is the classic failure: the nearest node to B often sits on a disconnected fragment,
  and the search then fails or returns a path along the wrong railroad.
- **Sidings, spurs and yard tracks are penalised** 4× so the path stays on the main track.
- **Every accepted trace is attributed to the OSM ways it followed**, by name and mileage. A corridor
  can hold more than one railroad; the way names are how you tell that the path is the line the
  feature actually claims.

A trace is rejected unless every gate holds: each endpoint snaps within 1500 m, traced length is
between 0.97× and 1.6× the chord, and the result has at least 8 vertices. Rejections are reported,
not silently dropped — a rejected run stays flagged.

```bash
python3 straightfix.py --scan straight.json --file illinois_central   # review, writes nothing
python3 straightfix.py --scan straight.json --file illinois_central --apply
```

Repairs are keyed on the chord, and each one is spliced into **every feature in the repo carrying
that geometry** — layers duplicate routes constantly (a train in `amtrak.geojson` and the same train
in its railroad's own file), and repairing one copy would leave the map disagreeing with itself.

Each repaired feature gains a `geometry_repairs` record — the chord, the traced length, the OSM lines
followed, and the method — and its `geometry_quality` prose is updated to say how much ruled line was
replaced. That is Rule 3 applied to geometry: the field says what was actually checked.

### Backing out — `straightrevert.py`

**A labelled straight line is honest; a confident trace down the wrong railroad is not.** When a
repair cannot be trusted, putting the ruled chord back is the correct outcome, not a lesser one.
Every repair records its chord endpoints, so the revert is exact.

```bash
python3 straightrevert.py           # report what it would revert
python3 straightrevert.py --apply
```

It reverts a repair when any of these hold, and rewrites the `geometry_quality` stamp from whatever
survived so it never claims more than it did:

- the feature's `mode` is `interurban`, `streetcar` or `cable_car` — those alignments were private
  right of way or street trackage and almost never survive as mapped rail, so a *confident* trace
  means the search found a different railroad nearby
- the path used modern urban transit (QLINE, M-1, streetcar, LRT, subway) for a heavy-rail feature
- an endpoint snapped more than 800 m from the traced line
- the traced path is more than 1.45× the chord, or hops four or more named lines

`--apply` rewrites files in place with the repo's compact JSON formatting, so diffs show only real
coordinate changes. Back up `data/` first regardless.

### Result of the first pass — 2026-08-29

| | runs | miles |
|---|---|---|
| flagged at the start | 1,354 | 30,492 |
| traced and kept | 999 | 18,113 ruled → 18,594 traced |
| reverted as untrustworthy | 103 records | 2,641 |
| still ruled | 626 | 20,530 |

Of the reverts, 67 were endpoint-snap failures, 13 detours, 12 traces routed onto modern urban
transit, and 11 interurbans. Two caught by hand first: the **Petaluma & Santa Rosa** interurban had
been traced 13.2 mi onto the NWP main line and only 0.2 mi onto its own track, and the **Grand Trunk
Sarnia – Detroit** had been routed onto Detroit's QLINE streetcar. Both are why the mode and
transit-name rules exist.

Runs over 60 miles are excluded from the automatic pass (61 runs, ~5,500 mi): the OSM graph
fragments over that distance and the corridor has to widen enough to admit a parallel railroad. A
100-mile ruled line stands in for several unknown intermediate points, and wants hand-picked
waypoints rather than a shortest-path search.

**This pass proves alignment, not history.** It says a railroad ran along these rails; it does not
source the dates, the service or which company's trains used them. Rule 3 still applies — the
`sources` array on a segment is untouched by any of this.
