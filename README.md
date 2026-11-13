# gpx-vector-tiles

Tilekiln config for the OpenStreetMap GPS traces as vector tiles (MVT), read from the PostGIS tables that [gpxtiler](https://github.com/Rub21/gpxtiler) fills. It has the same role as [spirit](https://github.com/pnorman/spirit) for vector.openstreetmap.org.

- `config.yaml`: the tileset `gps`. Zoom 0 to 9: the layers `traces` (every mode but plane) and `plane`. Zoom 10 and up: one layer per transport mode (`walk`, `bike`, `vehicle`, `plane`, `unknown`).
- `traces.sql.jinja2`: the SQL of the layers. It is the only place to edit.
- `gen-layers.sh`: writes the files in `layers/` from `traces.sql.jinja2`.
- `layers/`: generated, do not edit.

## Layers per mode

Tilekiln renders each layer file alone, with no include and no variables from `config.yaml`. So `gen-layers.sh` copies `traces.sql.jinja2` once per layer into `layers/traces-<name>.sql.jinja2` and sets the mode condition on the first line. Run `./gen-layers.sh` after every change to `traces.sql.jinja2` and commit `layers/`.

| Zoom | Layers in the tile | Why |
|---|---|---|
| 0 to 9 | `traces` (walk, bike, vehicle, unknown together), `plane` | A line is one pixel: the modes would repeat the same edges. Planes are long straight lines across the map, the first thing a style hides |
| 10 to 20 | `walk`, `bike`, `vehicle`, `plane`, `unknown` | One layer per mode, so a client shows or hides each one |

The low zooms also keep only the edges that several traces go through (`min_n` in the template: 3 traces up to z4, 2 up to z6, every edge from z7). A busy road shows from far away, a lone trace shows when you zoom in, like the brightness of the PNG tiles. A z4 tile of Europe went from 239 KB to 127 KB.

The snap grid is also coarser at the low zooms (1 px up to z9, half a pixel from z10; 2 px made the lines look like a circuit board). With 20 years of traces a z4 tile of Europe went from 1.9 MB (five layers, half a pixel) to about 270 KB. The low zooms are still slow to make (a few seconds of PostGIS per tile), so they are generated in advance and stored (`tilekiln generate zooms`, see gpx-tiles-stack), like the PNG tiles.

Only this year and the 2 before it are served (`recent_years` at the top of `traces.sql.jinja2`). The tables are partitioned by year, so a tile reads only those partitions.

## The layers

Lines of the traces, cut by direction of travel so a style can color them like the PNG tiles of gps-tile.openstreetmap.org.

The tile does not say which trace a line is from. Many traces follow the same road, and one line per trace made the tiles big (a z9 tile of a busy area was 12 MB). Every edge is snapped to a grid of half a pixel, and the edges on the same place are kept once with a count, like a pixel of the PNG tiles that many traces go through.

| Field | Value |
|---|---|
| `dir` | direction of travel in 32 steps of 11.25 degrees: 0 = east, 8 = north, 16 = west, 24 = south |
| `fade` | 1 to 10. Long edges (GPS jumps) get a low value, so the style can dim them |
| `n` | 1 to 10. How many traces go through the line (10 = 10 or more), so the style can draw busy roads stronger |

Zooms 0 to 12 read the simplified copies `trace_segments_z5`, `_z8`, `_z11` and `_z14` (made by `gpxtiler import -simplified-zooms 5,8,11,14`), two zooms finer than the tile so the lines stay smooth. Zooms 13 and up read `trace_segments`.

## Run

With a database loaded by gpxtiler:

```
pip install "tilekiln==0.8.4" "psycopg[binary]"
tilekiln serve dev --config config.yaml --source-dbname gpxtiler
```

The TileJSON is on `http://localhost:8000/gps/tilejson.json`. [gpx-tiles-stack](https://github.com/Rub21/gpx-tiles-stack) runs it with Docker next to gpxtiler.
