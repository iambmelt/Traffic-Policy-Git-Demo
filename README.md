# Traffic Policy via GitHub Actions (ngrok)

This repo demonstrates a GitHub Action that reads config from versioned files and creates an **ngrok Endpoint with a Traffic Policy** via the ngrok API—automatically on pushes to the `deployment` branch.

## What it does

* Triggers on `deployment` branch pushes
* Validates `config.yml` (requires `url`, `type`; supports `description`, `metadata`, `pooling_enabled`)
* Reads `policy.yml`, converts it to JSON, and sends it as `traffic_policy`
* Posts a form-encoded request to `https://api.ngrok.com/endpoints`

## Quick start

1. **Fork/clone** this repo.
2. In **Settings → Secrets and variables → Actions**, add:

   * `NGROK_API_TOKEN` — your ngrok API Bearer token.
3. Create a branch named **`deployment`** and add these files at the repo root:

**config.yml** (may contain either JSON or YAML)

```yaml
url: "https://example.ngrok.dev:443"
type: "cloud"
description: "Created by a GitHub Action"
metadata: "Some metadata"
pooling_enabled: false
```

**policy.yml** (example)

```yaml
on_http_request:
  - actions:
      - type: deny
        config:
          status_code: 404
```

4. **Commit & push** to `deployment`.
   The workflow will run and create the endpoint. Check the **Actions** tab for logs and API responses.

## How it works

* Uses `yq` to validate and extract fields from `config.yml`
* Converts `policy.yml` (YAML) → compact JSON
* Builds a form-encoded request mirroring the cURL shape:
  * `url`, `type`, `traffic_policy`, plus any of `description`, `metadata`, `pooling_enabled`
* Sends the request with `Authorization: Bearer ${{ secrets.NGROK_API_TOKEN }}`

## Repo layout

```
.
├── .github/workflows/deploy.yml   # The GitHub Action
├── config.yml                     # Required: endpoint config
└── policy.yml                     # Required: traffic policy (YAML)
```

## Notes

* `policy.yml` must be valid YAML that converts to the structure ngrok expects for `traffic_policy`.
* For demos that **update** an existing endpoint instead of creating a new one each push, you can adapt the workflow to `PATCH` by ID or look up by `metadata`.
