# AI Agent Guide — geo-agent-template

## Repo relationship

This is a **template repo** for creating geo-agent map applications. The core library lives at [boettiger-lab/geo-agent](https://github.com/boettiger-lab/geo-agent) and is loaded from CDN — you never modify it here.

| Repo | Purpose |
|---|---|
| `geo-agent` | Core library (map, chat, agent, tools). Source of truth for all functionality. |
| `geo-agent-template` | Starter template. Users fork this and configure three files for their dataset. |

**Full docs:** [boettiger-lab.github.io/geo-agent/docs](https://boettiger-lab.github.io/geo-agent/docs/)
— includes the complete configuration reference, deployment guide, and agent loop internals.

The schema below is kept inline so you can work without a network fetch. If it conflicts with the docs, the docs are authoritative.

---

## What you configure (and what you don't)

**You configure:** `layers-input.json` (which datasets to show and how), `system-prompt.md` (LLM persona and guidelines), and `k8s/` manifests if deploying to Kubernetes.

**You do not write JavaScript.** The core map, chat, agent, and tool modules are loaded from the CDN. Do not create or modify JS files in a client app repo.

**Do not duplicate STAC descriptions in `system-prompt.md`.** Dataset titles, descriptions, column schemas, and parquet paths are automatically injected into the LLM system prompt from the STAC catalog at startup. Only add domain-specific guidance, SQL examples, or interaction style notes to `system-prompt.md`.

---

## Full `layers-input.json` schema

### Top-level fields

| Field | Required | Type | Description |
|---|---|---|---|
| `catalog` | Yes | string | STAC catalog root URL |
| `collections` | Yes | array | Collection specs (see below) |
| `view` | No | object | `{ "center": [lon, lat], "zoom": z }` |
| `titiler_url` | No | string | TiTiler server for COG rasters (default: `https://titiler.nrp-nautilus.io`) |
| `mcp_url` | No | string | MCP/DuckDB server URL for SQL analytics |
| `llm` | No | object | LLM config for user-provided key mode (see below) |
| `welcome` | No | object | `{ "message": "...", "examples": ["...", "..."] }` |

> **Security note:** The private MCP server (`https://private-duckdb-mcp.nrp-nautilus.io/mcp`) requires a bearer token (`MCP_AUTH_TOKEN`) and is restricted to authorized apps only. Do **not** set `mcp_url` to the private server or inject `MCP_AUTH_TOKEN` into a new app's k8s deployment without explicit authorization.

### Collection-level fields

Each `collections` entry is a bare string (loads all visual assets) or an object:

| Field | Type | Description |
|---|---|---|
| `collection_id` | string | **Must exactly match the `"id"` field in the STAC collection JSON** — not a label you invent. Verify before use (see below). |
| `collection_url` | string | Direct STAC collection JSON URL — bypasses root catalog traversal |
| `group` | string | Layer toggle group label |
| `assets` | array | Asset selector (see below). Omit to load all visual assets. |
| `display_name` | string | Override collection title in UI |

### Asset config — vector / PMTiles

Each `assets` entry is a bare string (the STAC asset key) or a config object:

| Field | Type | Description |
|---|---|---|
| `id` | string | **Required.** STAC asset key (e.g., `"pmtiles"`) |
| `alias` | string | Alternative layer ID — use to create two logical layers from one STAC asset with different filters |
| `display_name` | string | Layer toggle label |
| `visible` | boolean | Default visibility (default: `false`) |
| `default_style` | object | MapLibre fill paint properties |
| `outline_style` | object | MapLibre line paint for an auto-added outline layer |
| `layer_type` | `"line"` or `"circle"` | `"line"` for LineString features; `"circle"` for Point features — see warning below |
| `default_filter` | array | MapLibre filter expression at load time |
| `tooltip_fields` | array | Property names shown on feature hover |
| `group` | string | Override collection-level group for this layer |

### Asset config — raster / COG

| Field | Type | Description |
|---|---|---|
| `id` | string | **Required.** STAC asset key |
| `display_name` | string | Layer toggle label |
| `visible` | boolean | Default visibility (default: `false`) |
| `colormap` | string | TiTiler colormap name (e.g., `"reds"`, `"viridis"`) |
| `rescale` | string | TiTiler min,max range (e.g., `"0,150"`) |
| `legend_label` | string | Legend label |
| `legend_type` | string | `"categorical"` to use STAC `classification:classes` colors |

---

## Critical: `layer_type` vs `outline_style`

**Never use `"layer_type": "line"` to draw polygon outlines.** This tells the renderer the tile features are LineString geometries. On a polygon-feature PMTiles file, it causes MapLibre to silently render nothing.

**To draw polygon boundaries without a fill**, use `outline_style` and set `fill-opacity: 0`:

```json
{
    "id": "pmtiles",
    "display_name": "District Boundaries",
    "visible": true,
    "default_style": {
        "fill-color": "#000000",
        "fill-opacity": 0
    },
    "outline_style": {
        "line-color": "#1565C0",
        "line-width": 1.5
    }
}
```

Only set `layer_type` when the tile features match the geometry type:
- `"line"` — LineString/MultiLineString features (roads, rivers, transects)
- `"circle"` — Point/MultiPoint features (observations, stations, events)

---

## Finding collection IDs and asset IDs

**Always fetch the STAC collection JSON and verify — never guess.** The `collection_id` must match the STAC `"id"` field exactly; a mismatch causes layers to silently not appear. Run this one-liner when you have the collection URL:

```bash
curl -s <collection_url> | python3 -c "
import json, sys
d = json.load(sys.stdin)
print('collection_id:', d['id'])
for k, v in d.get('assets', {}).items():
    vl = v.get('vector:layers', 'MISSING')
    print(f'  asset: {k}  type: {v.get(\"type\",\"\")}  vector:layers: {vl}')
"
```

This also checks `vector:layers` on each PMTiles asset. If it shows `MISSING`, the STAC collection needs to be patched before the layer will render — the app falls back to the asset key as the source-layer name, which is almost always wrong.

Alternatively, browse the catalog in STAC Browser:

```
https://radiantearth.github.io/stac-browser/#/external/s3-west.nrp-nautilus.io/public-data/stac/catalog.json
```

Open a collection → the collection `id` is shown at the top. Under **Assets**, the keys (e.g., `"pmtiles"`, `"v2-total-2024-cog"`) are the `id` values for asset entries. For PMTiles, the asset's `vector:layers` field lists internal layer names — the app reads this automatically, no manual config needed.

### Verifying PMTiles fields for `tooltip_fields` and `default_filter`

PMTiles tiles contain only a subset of the parquet columns — tippecanoe selects fields at tile-build time. **Do not assume field names from the STAC `table:columns` schema are available in the tiles.** Before setting `tooltip_fields` or `default_filter`, inspect the PMTiles metadata directly:

```bash
python3 -c "
import urllib.request, struct, json
url = '<pmtiles_url>'
req = urllib.request.Request(url, headers={'Range': 'bytes=0-16383'})
data = urllib.request.urlopen(req).read()
off = struct.unpack_from('<Q', data, 24)[0]
ln  = struct.unpack_from('<Q', data, 32)[0]
req2 = urllib.request.Request(url, headers={'Range': f'bytes={off}-{off+ln-1}'})
meta = json.loads(urllib.request.urlopen(req2).read())
for layer in meta.get('vector_layers', []):
    print('layer name:', layer['id'])
    print('fields:', list(layer.get('fields', {}).keys()))
"
```

The `vector_layers[].id` value is the internal layer name (must be present in `vector:layers` in the STAC asset). The `vector_layers[].fields` keys are the only field names valid for `tooltip_fields` and `default_filter`.

---

## Troubleshooting: layer not appearing in the overlay list

Two common causes:

1. **`collection_id` mismatch** — the value in `layers-input.json` does not match the STAC collection's actual `"id"` field. Run the one-liner above and compare. The framework silently drops the collection if the IDs don't match.

2. **Wrong source-layer name** — the `vector:layers` field in the STAC asset is missing or incorrect, so the app uses the asset key as the source-layer name and MapLibre finds no matching layer in the tiles. Check `vector:layers` with the one-liner above, and verify it matches the `vector_layers[].id` value from the PMTiles metadata script.

---

## MapLibre filter syntax

Use the modern `match` form for list membership:

```json
["match", ["get", "ColumnName"], ["value1", "value2"], true, false]
```

Do **not** use the legacy `["in", "ColumnName", "value1", "value2"]` form — it is silently ignored by current MapLibre.

---

## LLM config (user-provided key mode)

```json
"llm": {
    "user_provided": true,
    "default_endpoint": "https://openrouter.ai/api/v1",
    "models": [
        { "value": "anthropic/claude-sonnet-4", "label": "Claude Sonnet" },
        { "value": "google/gemini-2.5-flash",   "label": "Gemini Flash" }
    ]
}
```

Omit the `llm` block entirely for Kubernetes deployments where `config.json` is injected server-side.
