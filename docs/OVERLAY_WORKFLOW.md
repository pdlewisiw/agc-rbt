# Custom Vector Overlay Workflow (No Full Basemap Rebuild)

This guide is for adding your own overlay data (region boundaries, timezone polygons, AOI masks, etc.) to the existing RBT TileServer + MapProxy stack without rebuilding core basemap MBTiles.

Target use case: small/medium overlay, rendered on top of existing styles, zoom range `z0-z8`.

## 1. What You Are Doing

- Keep existing basemap datasets (`CULTURAL`, `PHYSICAL`, `HILLSHADE`) as-is.
- Build a separate overlay MBTiles from your raw vector data.
- Register that MBTiles in TileServer `config.json` as a new data source.
- Add a style (or update an existing style) that draws the overlay source.
- Optional: expose through MapProxy as a cached WMS/WMTS layer.

This avoids full global tile rebuilds.

## 2. Folder Paths in This Repo

- TileServer config:
  - `tileserver/config/config.json`
- TileServer styles:
  - `tileserver/styles/`
- TileServer MBTiles runtime mount (docker-compose):
  - host path `tileserver/data`
  - container path `/data`
- MapProxy config:
  - `mapproxy/config/mapproxy.yaml`

## 3. Raw Data Prep (Before Building Tiles)

Supported inputs are usually GeoJSON / GPKG / Shapefile.

Minimum prep checklist:

1. Confirm geometry type(s): polygon/line/point.
2. Keep only fields needed for styling/labels (reduce payload size).
3. Ensure valid geometries (no self-intersections if possible).
4. Keep one clear layer name for output, e.g. `custom_overlay`.
5. Define expected zoom visibility (default here: `0-8`).

## 4. Build Overlay MBTiles (Tippecanoe Example)

Example command:

```bash
tippecanoe \
  -o custom-overlay-3395.mbtiles \
  -n custom_overlay \
  -l custom_overlay \
  -Z 0 -z 8 \
  --drop-densest-as-needed \
  --detect-shared-borders \
  your_overlay.geojson
```

Notes:

- `-l custom_overlay` becomes your `source-layer` in style JSON.
- For line/polygon administrative overlays, `z0-z8` is usually enough.
- If detail is missing, bump max zoom to `z9` or `z10`.

## 5. Install the Overlay MBTiles

Copy file to:

- `tileserver/data/custom-overlay-3395.mbtiles`

## 6. Register Overlay Dataset in TileServer

Edit `tileserver/config/config.json` under `data`:

```json
"CUSTOM_OVERLAY": {
  "mbtiles": "custom-overlay-3395.mbtiles"
}
```

Use stable uppercase key names in `data`.

## 7. Add Overlay to a Style

Option A (recommended): create a new style folder, e.g. `tileserver/styles/RBT-CUSTOM-3395/` by copying an existing style.

Option B: patch existing style (TOPO/DARK/OVERLAY).

In style JSON:

1. Add source:

```json
"CUSTOM_OVERLAY": {
  "type": "vector",
  "url": "mbtiles://{CUSTOM_OVERLAY}"
}
```

2. Add one or more layers using:

```json
"source": "CUSTOM_OVERLAY",
"source-layer": "custom_overlay"
```

3. For polygon overlays, common style properties:
- fill with transparency (`fill-opacity: 0.1-0.35`)
- boundary line (`line-width` by zoom)

4. For labels, ensure label field exists and use symbol layer.

## 8. Restart and Validate

From repo root:

```bash
docker compose restart tileservergl
```

Validation checklist:

1. Open TileServer style endpoint and visually confirm overlay.
2. Verify overlay appears at expected zoom range (`0-8`).
3. Check no style errors in container logs.

## 9. Optional: Expose Overlay via MapProxy

If you need WMS/WMTS service + caching for overlay:

1. Add new TileServer source URL in `mapproxy/config/mapproxy.yaml`.
2. Add corresponding cache block (`geopackage` cache like existing overlay caches).
3. Add layer entry to publish via WMS/WMTS.

Use your existing `rbt_overlay_*` blocks as template.

## 10. Naming Convention (Recommended)

- TileServer `data` key: `CUSTOM_OVERLAY`
- MBTiles filename: `custom-overlay-3395.mbtiles`
- Style source name: `CUSTOM_OVERLAY`
- Vector source-layer: `custom_overlay`

Keep naming consistent to reduce debug time.

## 11. Lift / Sizing Guidance

- Light lift:
  - Data already clean and small
  - One overlay layer
  - `z0-z8`
- Moderate lift:
  - Multiple feature classes + labels
  - Several style rules / filters
- Heavy lift:
  - Very large/global raw data
  - High zooms (`z12+`) with dense features
  - Complex preprocessing/generalization requirements

Build server needs grow mainly with data volume and max zoom.

## 12. Troubleshooting Quick Checks

1. Overlay not visible:
- Wrong `source-layer` name in style.
- Layer filtered out by zoom/minzoom.
- Data key mismatch between style and `config.json`.

2. TileServer starts but style fails:
- JSON syntax error.
- Missing source key in `data` section.

3. Overlay too heavy/slow:
- Reduce attributes before tippecanoe.
- Use lower max zoom.
- Use simplification/generalization.

---

If you are unsure about raw data quality, do a small pilot build first:

- small AOI extract
- one layer
- `z0-z8`

Then scale once visual output is correct.
