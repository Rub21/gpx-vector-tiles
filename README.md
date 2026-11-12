# gpx-vector-tiles

Tilekiln config for the OpenStreetMap GPS traces as vector tiles (MVT), read from the PostGIS tables that [gpxtiler](https://github.com/Rub21/gpxtiler) fills. It has the same role as [spirit](https://github.com/pnorman/spirit) for vector.openstreetmap.org.

- `config.yaml`: the tileset `gps`, one layer `traces`.
- `traces.sql.jinja2`: the SQL of the layer.

## The `traces` layer

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
