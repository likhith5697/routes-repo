# routes-repo

The list of public gateway routes, kept in one file: [`routes.yaml`](routes.yaml).
This repo holds **data only**. Validation, rendering and apply logic live in [platform-templates](https://github.com/likhith5697/platform-templates) and [gateway-infra](https://github.com/likhith5697/gateway-infra).

## How to add a route

**1. Branch**

```bash
git checkout -b add-payments-route
```

**2. Add an entry to `routes.yaml`**

```yaml
  - name: payments-create      # unique; lowercase letters, digits, dashes
    path: /payments            # must start with "/"; use {param} for variables, e.g. /payments/{id}
    method: POST               # GET POST PUT PATCH DELETE HEAD OPTIONS
    namespace: payments        # Kubernetes namespace of your Service
    service: payments          # Kubernetes Service name
    port: 8080                 # the Service port (not the container port)
    auth: custom               # always "custom"
```

**Rules** (CI rejects the PR if any is broken):
- **All 7 fields are required.** No other fields are allowed.
- **`method` + `path` must be unique.** `GET /orders` and `POST /orders` together are fine; two `GET /orders` entries are not.
- **`path` must start with `/`.**
- **`auth` must be `custom`.** `none` is rejected, so every route goes through the Lambda authorizer.
- **You don't choose the Gateway.** `parentRefs` is set by the platform (`public-gateway` in `gateway-system`), and adding it here is an error.

**3. (Optional) Check locally before pushing**

```bash
git clone --branch v0.1.0 https://github.com/likhith5697/platform-templates.git ../platform-templates
pip install -r ../platform-templates/requirements.txt
python ../platform-templates/validate_routes.py routes.yaml
python ../platform-templates/render_httproutes.py routes.yaml   # preview the generated HTTPRoutes
```

**4. Open a pull request**

CI runs the `validate` job (platform-templates `validate.yml`). Fix any errors it reports and push again.

**5. Merge**

On merge to `main`, the `trigger-apply` job sends a `trigger-apply` event to gateway-infra with this commit's SHA. gateway-infra does the apply; nothing is deployed from this repo.

## What happens where

```
PR opened  → ci.yml: validate       → platform-templates/validate.yml@v0.1.0
merge main → ci.yml: trigger-apply  → platform-templates/trigger-apply.yml@v0.1.0 → repository_dispatch → gateway-infra
```

## One-time repo setup (admins)

- **Secret `GATEWAY_INFRA_DISPATCH_TOKEN`:** a fine-grained PAT with access to `likhith5697/gateway-infra` only and **Contents: Read and write**. See the platform-templates README.
- **Branch protection on `main`:** require pull requests and require the `validate` check to pass, so nothing unvalidated reaches `main`.
