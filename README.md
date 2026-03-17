# geo-agent-template

A GitHub template for deploying an AI-powered interactive map app.
Users describe in plain language what datasets to show; the app uses an LLM agent with map tools and SQL access to visualize and analyze cloud-native geospatial data.

**No JavaScript to write.** The core modules (map, chat, agent, tools) are loaded from CDN. You configure which data to show via three small files.

## Repository structure

```
index.html          ← HTML shell — loads core JS/CSS from CDN
layers-input.json   ← which STAC collections to show + LLM settings
system-prompt.md    ← LLM system prompt (customize per app)
k8s/                ← Kubernetes deployment manifests (optional)
```

## Quick start

### 1. Create your repo from this template

Click **"Use this template"** on GitHub → **"Create a new repository"**.

### 2. Choose your datasets

Browse the available STAC catalog:

```
https://radiantearth.github.io/stac-browser/#/external/s3-west.nrp-nautilus.io/public-data/stac/catalog.json
```

Edit `layers-input.json` — set your collections and adjust the default map view.

### 3. Edit `system-prompt.md`

Describe the domain, what users are likely to ask, and include SQL examples relevant to your datasets.

### 4. Choose a deployment method

#### Option A: GitHub Pages (no server needed)

The `llm` block in `layers-input.json` is already enabled. Each visitor enters their own API key (e.g. from [OpenRouter](https://openrouter.ai)) in the in-app settings panel — keys are stored in the browser only, never on the server.

1. Enable GitHub Pages in your repo: Settings → Pages → Source → **GitHub Actions**
2. Add `.github/workflows/gh-pages.yml`:

```yaml
name: Deploy to GitHub Pages
on:
  push:
    branches: [main]
permissions:
  contents: read
  pages: write
  id-token: write
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/configure-pages@v5
      - uses: actions/upload-pages-artifact@v3
        with:
          path: .
      - id: deployment
        uses: actions/deploy-pages@v4
```

3. Push — the workflow deploys automatically on changes to `main`.

#### Option B: Kubernetes

API keys are injected server-side via a ConfigMap + Kubernetes secrets — no user-facing key entry.

1. Delete the `llm` block from `layers-input.json` (the server-injected `config.json` takes precedence anyway)
2. Replace the git clone URL in `k8s/deployment.yaml` with your repo URL
3. Replace the slug `calenviroscreen` throughout `k8s/` with your app name
4. Set your hostname in `k8s/ingress.yaml`
5. Create the required secret:

```bash
kubectl create secret generic llm-proxy-secrets \
  --from-literal=proxy-key=YOUR_PROXY_KEY
```

6. Deploy:

```bash
kubectl apply -f k8s/
kubectl rollout status deployment/my-app
```

After pushing changes, redeploy: `kubectl rollout restart deployment/my-app`

## `layers-input.json` reference

### Top-level fields

| Field | Required | Description |
|---|---|---|
| `catalog` | Yes | STAC catalog root URL. The app traverses child links to find collection metadata. |
| `titiler_url` | No | TiTiler server for COG/raster tile rendering. Defaults to `https://titiler.nrp-nautilus.io`. |
| `mcp_url` | No | MCP server URL for DuckDB SQL queries. Omit to disable analytics. |
| `view` | No | Initial map view: `{ "center": [lon, lat], "zoom": z }` |
| `llm` | No | LLM configuration (see below). Omit for server-provided mode. |
| `collections` | Yes | Array of collection specs (see below). |
| `welcome` | No | Welcome message: `{ "message": "...", "examples": ["...", "..."] }` |

### Collection-level fields

Each entry in `collections` is either a **bare string** (loads all visual assets from that collection) or an **object**:

| Field | Type | Description |
|---|---|---|
| `collection_id` | string | STAC collection ID to load. |
| `collection_url` | string | Direct URL to the STAC collection JSON. Bypasses root catalog traversal — useful for private or external catalogs. |
| `group` | string | Group label shown in the layer toggle panel. |
| `assets` | array | Asset selector: bare strings load a named asset with no extra config; objects configure a specific asset (see below). Omit to load all visual assets. |
| `display_name` | string | Override the collection title shown in the UI. |

### Asset config fields — vector (PMTiles)

Each entry in `assets` may be a **bare string** (the STAC asset key, loaded with defaults) or a **config object**:

| Field | Type | Description |
|---|---|---|
| `id` | string | **Required.** STAC asset key in the collection JSON (e.g., `"pmtiles"`). |
| `alias` | string | Alternative layer ID when you need two logical layers from one STAC asset (e.g., two `default_filter` views of the same file). |
| `display_name` | string | Label in the layer toggle UI. Falls back to the STAC asset title. |
| `visible` | boolean | Default visibility on load. Default: `false`. |
| `default_style` | object | MapLibre **fill** paint properties for polygon layers (e.g., `fill-color`, `fill-opacity`). |
| `outline_style` | object | MapLibre **line** paint for an auto-added outline on top of the fill (e.g., `{"line-color": "#333", "line-width": 1.5}`). Use this — not `layer_type` — to draw polygon borders. |
| `layer_type` | `"line"` | Set **only** when tile features are true LineString/MultiLineString geometries. |
| `default_filter` | array | MapLibre filter expression applied at load time. |
| `tooltip_fields` | array | Feature property names shown in the hover tooltip. |
| `group` | string | Overrides the collection-level `group` for this specific layer. |

### Asset config fields — raster (COG)

| Field | Type | Description |
|---|---|---|
| `id` | string | **Required.** STAC asset key. |
| `display_name` | string | Label in the layer toggle UI. |
| `visible` | boolean | Default visibility. Default: `false`. |
| `colormap` | string | TiTiler colormap name (e.g., `"reds"`, `"blues"`, `"viridis"`). Default: `"reds"`. |
| `rescale` | string | TiTiler min,max range for color scaling (e.g., `"0,150"`). |
| `legend_label` | string | Label shown next to the color legend. |
| `legend_type` | string | `"categorical"` to use STAC `classification:classes` color codes for a discrete legend. |

### How to find STAC asset IDs

Browse the catalog in STAC Browser:

```
https://radiantearth.github.io/stac-browser/#/external/s3-west.nrp-nautilus.io/public-data/stac/catalog.json
```

Open a collection → click the **Assets** tab. The keys listed there (e.g., `"pmtiles"`, `"v2-total-2024-cog"`) are the `id` values to use. For PMTiles vector layers, the asset's `vector:layers` field gives the internal layer name used by MapLibre (the app reads this automatically — no manual config needed).

### Worked example: polygon fill with categorical coloring

```json
{
    "id": "pmtiles",
    "display_name": "Fee Lands",
    "visible": true,
    "default_style": {
        "fill-color": ["match", ["get", "GAP_Sts"],
            "1", "#26633A",
            "2", "#3E9C47",
            "3", "#7EB3D3",
            "4", "#BDBDBD",
            "#888888"
        ],
        "fill-opacity": 0.7
    },
    "default_filter": ["match", ["get", "GAP_Sts"], ["1", "2"], true, false],
    "tooltip_fields": ["Unit_Nm", "GAP_Sts", "Mang_Type"]
}
```

### Worked example: boundary-only (outline) layer for polygon features

To render polygon features as outlines only (e.g., census tracts, admin boundaries), keep the fill type but make the fill transparent and set `outline_style`:

```json
{
    "id": "pmtiles",
    "display_name": "Congressional Districts",
    "visible": true,
    "default_style": {
        "fill-color": "#000000",
        "fill-opacity": 0
    },
    "outline_style": {
        "line-color": "#1565C0",
        "line-width": 1.5
    },
    "tooltip_fields": ["DISTRICTID", "STATE"]
}
```

> **Common mistake:** do not use `"layer_type": "line"` for polygon outline layers. That flag tells the renderer the tile features are LineString geometries — on a polygon layer it causes features to silently not render. `outline_style` is the correct approach.

**Filter syntax note:** use `["match", ["get", "col"], ["val1", "val2"], true, false]` for list membership. Do **not** use the legacy `["in", "col", val1, val2]` form — it is silently ignored in current MapLibre.

## Local development

```bash
python -m http.server 8000
# Open http://localhost:8000 — enter your API key in the settings panel
```

See [boettiger-lab/geo-agent](https://github.com/boettiger-lab/geo-agent) for the core library source and live examples.
