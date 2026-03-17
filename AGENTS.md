# AI Agent Guide — geo-agent-template

This file provides machine-readable guidance for AI agents creating or modifying a geo-agent app.

## What you configure (and what you don't)

**You configure:** `layers-input.json` (which datasets to show and how), `system-prompt.md` (LLM persona and guidelines), and `k8s/` manifests if deploying to Kubernetes.

**You do not write JavaScript.** The core map, chat, agent, and tool modules are loaded from the CDN. Do not create or modify JS files in a client app repo.

**Do not duplicate STAC descriptions in `system-prompt.md`.** Dataset titles, descriptions, column schemas, and parquet paths are automatically injected into the LLM system prompt from the STAC catalog at startup. Only add domain-specific guidance, SQL examples, or interaction style notes to `system-prompt.md`.

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

### Collection-level fields

Each `collections` entry is a bare string (loads all visual assets) or an object:

| Field | Type | Description |
|---|---|---|
| `collection_id` | string | STAC collection ID |
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
| `layer_type` | `"line"` | **Only for true LineString features** — see warning below |
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

Only use `"layer_type": "line"` when the STAC asset explicitly contains LineString or MultiLineString features (e.g., road networks, rivers).

## Finding STAC asset IDs

Browse the catalog:

```
https://radiantearth.github.io/stac-browser/#/external/s3-west.nrp-nautilus.io/public-data/stac/catalog.json
```

Open a collection → **Assets** tab. The keys there (e.g., `"pmtiles"`, `"v2-total-2024-cog"`) are the `id` values. For PMTiles, the asset's `vector:layers` field lists internal layer names — the app reads this automatically, no manual config needed.

## MapLibre filter syntax

Use the modern `match` form for list membership:

```json
["match", ["get", "ColumnName"], ["value1", "value2"], true, false]
```

Do **not** use the legacy `["in", "ColumnName", "value1", "value2"]` form — it is silently ignored by current MapLibre.

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
