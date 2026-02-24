# geo-agent-template

A GitHub template for deploying an AI-powered interactive map app.
Users describe in plain language what datasets to show; the app uses an LLM agent with map tools and SQL access to visualize and analyze cloud-native geospatial data.

**No JavaScript to write.** The core modules (map, chat, agent, tools) are loaded from CDN. You configure which data to show via three small files.

## Choose a deployment option

This template includes two deployment options. **Pick one** and delete the other folder.

| Option | Folder | LLM key | Best for |
|---|---|---|---|
| **GitHub Pages** | `ghpages/` | User-provided (browser) | Public demos, no server needed |
| **Kubernetes** | `k8s/` | Server-managed (secrets) | Production, private LLM proxy |

---

## Option A: GitHub Pages

The simplest option. No server-side secrets needed — each visitor enters their own LLM API key (e.g. from [OpenRouter](https://openrouter.ai)) via an in-app settings panel. Keys are stored in the browser's `localStorage` only.

### Setup

1. **Create your repo** from this template (click "Use this template" on GitHub)
2. **Delete the `k8s/` folder** — you won't need it
3. **Edit `ghpages/layers-input.json`** — set your STAC collections and preferred model list
4. **Edit `ghpages/system-prompt.md`** — customize the AI assistant persona
5. **Enable GitHub Pages**: Settings → Pages → Source → **GitHub Actions**
6. Add the workflow file at `.github/workflows/gh-pages.yml`:

```yaml
name: Deploy to GitHub Pages
on:
  push:
    branches: [main]
    paths: [ghpages/**]
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
          path: ghpages/
      - id: deployment
        uses: actions/deploy-pages@v4
```

### Local development

```bash
cd ghpages
python -m http.server 8000
# Open http://localhost:8000 — enter your API key in the settings panel
```

---

## Option B: Kubernetes

For production deployments with a managed LLM proxy and server-injected API keys (no user-facing key entry).

### Setup

1. **Create your repo** from this template
2. **Delete the `ghpages/` folder** — you won't need it
3. **Edit `k8s/layers-input.json`** — set your STAC collections
4. **Edit `k8s/system-prompt.md`** — customize the AI assistant
5. **Update `k8s/k8s/deployment.yaml`** — replace the git clone URL with your repo:

```yaml
git clone --depth 1 https://github.com/YOUR-ORG/YOUR-REPO.git /tmp/repo
```

6. **Replace the slug** `calenviroscreen` everywhere in `k8s/k8s/` with your app name:

| File | What to change |
|---|---|
| `deployment.yaml` | `metadata.name`, all `app:` labels |
| `service.yaml` | `metadata.name`, `app:` label and selector |
| `ingress.yaml` | `metadata.name`, TLS host, rules host, backend `service.name` |
| `configmap.yaml` | Both ConfigMap `metadata.name` values |

7. **Set your hostname** in `k8s/k8s/ingress.yaml`:

```yaml
- host: my-app.nrp-nautilus.io
```

8. **Create the required Kubernetes secret**:

```bash
kubectl create secret generic llm-proxy-secrets \
  --from-literal=proxy-key=YOUR_PROXY_KEY
```

9. **Deploy**:

```bash
kubectl apply -f k8s/k8s/configmap.yaml
kubectl apply -f k8s/k8s/deployment.yaml
kubectl apply -f k8s/k8s/service.yaml
kubectl apply -f k8s/k8s/ingress.yaml
```

After pushing changes, redeploy (the init container re-clones your repo):

```bash
kubectl rollout restart deployment/my-app
```

### Local development

```bash
cd k8s
python -m http.server 8000

# Create a minimal config.json so the LLM works locally:
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

---

## Configuring your datasets

Browse available data in the STAC catalog:

```
https://radiantearth.github.io/stac-browser/#/external/s3-west.nrp-nautilus.io/public-data/stac/catalog.json
```

### `layers-input.json` reference

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
                {
                    "id": "asset-id-2",
                    "display_name": "Friendly Name",
                    "visible": true,
                    "default_style": {
                        "fill-color": ["match", ["get", "MyColumn"],
                            "A", "#ff0000", "B", "#00ff00", "#888888"],
                        "fill-opacity": 0.7
                    },
                    "default_filter": ["match", ["get", "MyColumn"], ["A", "B"], true, false],
                    "tooltip_fields": ["Name", "MyColumn"]
                }
            ]
        }
    ]
}
```

- A bare **string** collection entry loads all visual assets
- An **object** entry with `assets` cherry-picks specific STAC asset IDs
- `visible`: show layer on by default (default: false)
- `default_style`: MapLibre paint expression applied at load
- `default_filter`: MapLibre filter expression applied at load — use `["match", ["get", "col"], ["val1", "val2"], true, false]` for list membership (do **not** use the legacy `["in", ...]` form)
- `tooltip_fields`: property names shown on hover

## How it works

`index.html` loads the core library from jsDelivr CDN. To pin to a stable release:

```html
<script type="module" src="https://cdn.jsdelivr.net/gh/boettiger-lab/geo-agent@v1.0.0/app/main.js"></script>
```

At runtime `main.js` fetches `layers-input.json`, `system-prompt.md`, and `config.json` from the same origin as the page — so each deployment provides its own configuration while sharing the same application code.

See [boettiger-lab/geo-agent](https://github.com/boettiger-lab/geo-agent) for the core library source and live examples.
