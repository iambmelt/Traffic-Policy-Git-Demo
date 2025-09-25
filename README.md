# Deploy ngrok Endpoint via GitHub Actions (Reusable Workflow)

Create or update an **ngrok Endpoint** from files in your repo—on every push (or on your schedule). The workflow:

* Reads your **endpoint config** (URL, type, optional metadata).
* Reads a **traffic policy** (YAML or JSON).
* Calls the **ngrok API** to create or update the endpoint.
* On first create, opens a **PR that writes the new endpoint `id`** back to your config file so future runs update the same endpoint.

You can run it inside the same repo or call it from another repo as a reusable workflow.

---

## 1) What this action does

* **Turns config + policy files into a live ngrok endpoint.**
* **First run:** creates the endpoint; records its `id` in your config via an auto-opened PR.
* **Later runs:** sees the `id` and updates that endpoint (no PR).
* **Safe for production:** runs with least-privilege permissions, supports branch protection, and can auto-merge the PR when allowed.

---

## 2) How to add it to your project

### A. If your wrapper is **in the same repo** (simple, no version pin)

1. Create a wrapper workflow in your repo (e.g. `.github/workflows/deploy-ngrok.yml`):

   ```yml
   name: Deploy ngrok endpoint
   on:
     push:
       branches: [ deployment ]

   jobs:
     deploy:
       permissions:
         contents: write
         pull-requests: write
       uses: ./.github/workflows/ngrok-endpoint.yml
       with:
         config_path: ".ngrok/configs/prod.yml"  # your config file
         policy_path: ".ngrok/policies/prod.yaml" # optional; see auto-detect below
         base_branch: "main"                      # PR target when recording new id
         auto_merge: true                         # try auto-merge when allowed
       secrets:
         NGROK_API_TOKEN: ${{ secrets.NGROK_API_TOKEN }}
   ```

### B. If your wrapper is **in another repo** (pin a tag/branch/SHA)

```yml
jobs:
  deploy:
    permissions:
      contents: write
      pull-requests: write
    uses: owner/repo/.github/workflows/ngrok-endpoint.yml@v1
    with:
      config_path: ".ngrok/configs/prod.yml"
      policy_path: ".ngrok/policies/prod.yaml"
      base_branch: "main"
      auto_merge: true
    secrets:
      NGROK_API_TOKEN: ${{ secrets.NGROK_API_TOKEN }}
```

### Requirements

* **Secret:** `NGROK_API_TOKEN` (ngrok API token).
* **Permissions in the *caller* workflow:**

  ```yml
  permissions:
    contents: write
    pull-requests: write
  ```
* **Repo settings (recommended):** enable “Allow auto-merge” if you want the PR to merge itself.

---

## 3) Files you’ll create / modify

1. **Endpoint config** (YAML) — referenced by `with.config_path` (defaults to `config.yml`).

   * Required keys: `url`, `type`
   * Optional: `description`, `metadata`, `pooling_enabled`, `id`
   * The workflow adds `id` for you on first create (via PR).

2. **Traffic policy** — referenced by `with.policy_path` **or** auto-detected at repo root (see allowed names).

   * Format: YAML or JSON.
   * You may store multiple environment variants (e.g. `prod.yaml`, `staging.yaml`).

3. **Wrapper workflow** — the small file shown above that sets your triggers and calls the reusable workflow.

---

## 4) Recommended repository structure

Use a clear, environment-centric layout:

```
.
├─ .github/
│  └─ workflows/
│     ├─ ngrok-endpoint.yml         # the reusable workflow (lives in the template repo)
│     └─ deploy-ngrok.yml           # your wrapper (caller’s repo)
└─ .ngrok/
   ├─ configs/
   │  ├─ prod.yml
   │  └─ staging.yml
   └─ policies/
      ├─ prod.yaml
      └─ staging.yaml
```

* Point `with.config_path` and `with.policy_path` at these files in your wrapper.
* Alternatively, if you prefer **auto-detect**, place exactly one policy file at the repo root with an allowed name (see below).

---

## 5) Sample config & policy files

### Example endpoint config (`.ngrok/configs/prod.yml`)

```yaml
# Required
url: "https://example.ngrok.dev:443"
type: "cloud"

# Optional (recommended)
description: "Production endpoint managed by GitHub Actions"
metadata: "owner=payments,env=prod"
pooling_enabled: false

# Added automatically after first creation:
# id: "ep_123abc..."
```

### Example traffic policy (`.ngrok/policies/prod.yaml`)

```yaml
on_http_request:
  - match:
      path:
        prefix: "/internal"
    actions:
      - type: deny
        config:
          status_code: 403

  - actions:
      - type: allow
```

> JSON is equally valid. If you prefer JSON, set `policy_path` to your `.json` file or name your root file `policy.json` to be auto-detected.

---

## 6) Allowed file names (auto-detect mode)

If you **omit** `with.policy_path`, the workflow looks for **exactly one** of these at the repo root:

* `policy.yml`
* `policy.yaml`
* `policy.json`

If more than one is present, the run fails (please pick one).
If none are present, the run fails unless you set `with.policy_path`.

> You can also pass a **glob** in `policy_path` (e.g., `.ngrok/policies/prod.*`), which must resolve to exactly one file.

---

## Inputs (advanced)

These are supported when calling the reusable workflow:

| Input               | Type    | Default                           | Purpose                                                                |
| ------------------- | ------- | --------------------------------- | ---------------------------------------------------------------------- |
| `config_path`       | string  | `config.yml`                      | Path to your endpoint config file.                                     |
| `policy_path`       | string  | *(empty)*                         | Path **or glob** to your policy file. If omitted, auto-detect is used. |
| `base_branch`       | string  | `main`                            | PR target branch when recording a newly created endpoint `id`.         |
| `ngrok_api_url`     | string  | `https://api.ngrok.com/endpoints` | Override if needed.                                                    |
| `ngrok_api_version` | string  | `2`                               | Sent as the `Ngrok-Version` header.                                    |
| `auto_merge`        | boolean | `true`                            | Enable auto-merge for the PR when allowed.                             |

---

## Behavior summary

* **Create vs Update**

  * No `id` in config → **create** (POST) → capture `id` → PR writes `id` to your config.
  * `id` present → **update** (PATCH) → no PR, endpoint is updated in place.

* **PR/merge**

  * PR opens against `base_branch`.
  * Auto-merge is attempted if `auto_merge: true` and repo policies allow it; otherwise you can merge manually (or the job attempts an immediate squash merge when allowed).

* **Validation**

  * Fails fast if required fields are missing or if the policy file is missing/ambiguous.

---

## Troubleshooting

* **Permissions error** (“requesting contents: write… allowed read”)
  Add the `permissions` block in the **caller** job/workflow (see wrappers above).

* **Policy not found / multiple policies**

  * If using auto-detect, ensure exactly one of: `policy.yml`, `policy.yaml`, `policy.json` at repo root.
  * If using `policy_path`, ensure it resolves to exactly one file (globs allowed).

* **PR didn’t auto-merge**

  * Check branch protection and that “Allow auto-merge” is enabled.
  * The workflow will still open the PR so you can merge manually.
