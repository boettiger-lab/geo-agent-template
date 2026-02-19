# geo-agent-template

A GitHub template for deploying an AI-powered interactive map app on Kubernetes. 
Users describe in plain language what datasets to show; the app uses an LLM agent with map tools and SQL access to visualize and analyze cloud-native geospatial data.

**No JavaScript to write.** The core modules (map, chat, agent, tools) are loaded from the CDN. You configure which data to show via three small files.

## Quick start

### 1. Create your repo from this template

Click **"Use this template"** on GitHub → **"Create a new repository"**.  
Clone your new repo:

```bash
git clone https://github.com/YOUR-ORG/YOUR-REPO.git
cd YOUR-REPO
```

### 2. Choose your datasets

Browse the available STAC catalog:

```
https://s3-west.nrp-nautilus.io/public-data/stac/catalog.json
```

You can explore it interactively in the STAC Browser:

```
https://radiantearth.github.io/stac-browser/#/external/s3-west.nrp-nautilus.io/public-data/stac/catalog.json
```

Each collection has a `"id"` field (used in `layers-input.json`) and `"assets"` (the specific layers to display). Collections provide:
- **PMTiles** — vector tile layers rendered directly on the map
- **COG** — cloud-optimized raster layers rendered via TiTiler
- **Parquet / H3 Parquet** — tabular/hex data available to the AI for SQL queries

### 3. Edit the three config files

#### `layers-input.json` — which datasets to load

```json
{
    "catalog": "https://s3-west.nrp-nautilus.io/public-data/stac/catalog.json",
    "titiler_url": "https://titiler.nrp-nautilus.io",
    "mcp_url": "https://duckdb-mcp.nrp-nautilus.io/mcp",
    "view": { "center": [-119.4, 36.8], "zoom": 6 },
    "collections": [
        "some-collection",
        {
            "collection_id": "another-collection",
            "assets": [
                "asset-id-1",
                { "id": "asset-id-2", "display_name": "Friendly Name" }
            ]
        }
    ]
}
```

- A bare **string** collection entry loads all visual assets from that collection
- An **object** entry with `assets` cherry-picks specific STAC asset IDs
- Asset filtering only affects map layer toggles — all parquet/H3 data remains available to the AI for SQL queries

#### `system-prompt.md` — the AI assistant's persona and guidelines

Describe the domain, what the user is likely to ask, and how the AI should respond.  
Include SQL examples relevant to your datasets so the LLM has a template to follow.  
The available dataset schemas are injected automatically at runtime via `list_datasets` / `get_dataset_details` MCP tools.

#### `index.html` — page title and CDN version

Update the `<title>` tag. To pin to a specific release of the core library:

```html
<script type="module" src="https://cdn.jsdelivr.net/gh/boettiger-lab/geo-agent@v1.0.0/app/main.js"></script>
```

To track the latest development version:

```html
<script type="module" src="https://cdn.jsdelivr.net/gh/boettiger-lab/geo-agent@main/app/main.js"></script>
```

### 4. Update the Kubernetes manifests

All four files in `k8s/` contain a name slug that must be updated to match your app. Replace every occurrence of the current slug (e.g. `calenviroscreen`) with your own (e.g. `my-app`):

| File | What to change |
|---|---|
| `deployment.yaml` | `metadata.name`, all `app:` labels, and the **git clone URL** (point to your repo) |
| `service.yaml` | `metadata.name`, `app:` label and selector |
| `ingress.yaml` | `metadata.name`, TLS host, rules host, backend `service.name` |
| `configmap.yaml` | Both ConfigMap `metadata.name` values (`*-config` and `*-nginx`) |

The critical line in `deployment.yaml` to update is the git clone URL — the init container clones your repo at deploy time to serve the static files:

```yaml
git clone --depth 1 https://github.com/YOUR-ORG/YOUR-REPO.git /tmp/repo
```

Set the hostname in `ingress.yaml` to your desired subdomain:

```yaml
- host: my-app.nrp-nautilus.io
```

### 5. Create the required Kubernetes secrets

The deployment reads LLM API keys from three secrets. Create them in your namespace before deploying:

```bash
# NRP LLM proxy key
kubectl create secret generic llm-proxy-secrets \
  --from-literal=proxy-key=YOUR_PROXY_KEY
```

### 6. Deploy

```bash
kubectl apply -f k8s/configmap.yaml
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
kubectl apply -f k8s/ingress.yaml
```

Check rollout status:

```bash
kubectl rollout status deployment/my-app
kubectl get pods
kubectl get ingress
```

After pushing changes to your repo, redeploy by restarting the deployment (the init container re-clones the repo):

```bash
kubectl rollout restart deployment/my-app
```

## Local development

```bash
# Serve the app locally
python -m http.server 8000

# Create a minimal config.json so the LLM works (no secret injection locally):
cat > config.json <<'EOF'
{
  "llm_models": [
    {
      "value": "glm-4.7",
      "label": "NRP GLM-4.7",
      "endpoint": "https://llm-proxy.nrp-nautilus.io/v1",
      "api_key": "YOUR_PROXY_KEY"
    }
  ]
}
EOF
```

Then open `http://localhost:8000`.

## Repository structure

```
index.html          ← HTML shell — loads core JS/CSS from CDN
layers-input.json   ← which STAC collections + assets this app shows
system-prompt.md    ← LLM system prompt (customize per app)
k8s/
  configmap.yaml    ← LLM config template + nginx config
  deployment.yaml   ← init container clones this repo; nginx serves it
  service.yaml      ← ClusterIP service
  ingress.yaml      ← hostname + TLS
```

## How the deployment works

At deploy time the Kubernetes Pod runs an **init container** that clones your GitHub repo into a shared volume. The main `nginx` container then serves those static files. The `config.json` (containing API keys) is rendered at startup via `envsubst` from a ConfigMap template + Kubernetes secrets — it is never stored in the repo.

`main.js` (loaded from CDN) fetches `layers-input.json`, `system-prompt.md`, and `config.json` from the same origin as the page. This means every configuration change just requires a `kubectl rollout restart` — no image rebuilds.

