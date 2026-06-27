# Phase 08: Docker & Deployment

Containerize the app and configure deployment. By the end of this phase, `docker compose up --build` runs the full stack.

**Prerequisite:** Phases 04 and 06 complete (backend and frontend both working locally).

---

## Multi-Stage Dockerfile

Create `Dockerfile` in the project root with three stages:

```dockerfile
# --- Backend ---
FROM python:3.12-slim AS backend

COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /bin/

WORKDIR /app
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev
COPY api/ api/

EXPOSE 8020
CMD ["uv", "run", "uvicorn", "api.main:app", "--host", "0.0.0.0", "--port", "8020"]

# --- Frontend Build ---
FROM node:20-slim AS frontend-build

WORKDIR /app
COPY frontend/package*.json ./
RUN npm ci
COPY frontend/ .
RUN npm run build

# --- Frontend Serve ---
FROM nginx:alpine AS frontend

COPY --from=frontend-build /app/dist /usr/share/nginx/html
COPY frontend/nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

Key points:
- `python:3.12-slim` as the base — stable and small
- uv is copied from its official container image
- `uv sync --frozen --no-dev` installs only production dependencies from the lockfile
- Frontend is built with node then served with nginx — no node runtime in production

---

## docker-compose.yml

Create `docker-compose.yml`:

```yaml
services:
  api:
    build:
      context: .
      target: backend
    container_name: myapp-api
    ports:
      - "8020:8020"
    extra_hosts:
      - "host.docker.internal:host-gateway"
    environment:
      - MONGODB_URL=mongodb://host.docker.internal:27017
      - MONGODB_DB_NAME=myapp
      - JWT_SECRET=${JWT_SECRET:-change-me-in-production}
      - SMTP_EMAIL=${SMTP_EMAIL:-}
      - SMTP_APP_PASSWORD=${SMTP_APP_PASSWORD:-}
      - SMTP_HOST=${SMTP_HOST:-smtp.gmail.com}
      - SMTP_PORT=${SMTP_PORT:-587}
      - PASSWORD_RESET_EXPIRE_MINUTES=${PASSWORD_RESET_EXPIRE_MINUTES:-60}
      - FRONTEND_BASE_URL=${FRONTEND_BASE_URL:-https://localhost:8095}
    restart: unless-stopped

  frontend:
    build:
      context: .
      target: frontend
    container_name: myapp-frontend
    ports:
      - "8095:80"
    depends_on:
      - api
    restart: unless-stopped
```

Key points:
- `extra_hosts` maps `host.docker.internal` so the API container can reach MongoDB running on the host
- `restart: unless-stopped` keeps services running after reboots
- Secrets like `JWT_SECRET` come from the shell environment or a `.env` file

---

## Nginx Config (Frontend)

Create `frontend/nginx.conf`:

```nginx
server {
    listen 80;
    root /usr/share/nginx/html;
    index index.html;

    # SPA: serve index.html for all routes
    location / {
        try_files $uri $uri/ /index.html;
    }

    # Proxy API calls to the backend container
    location /api/ {
        proxy_pass http://api:8020;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Cache static assets
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }
}
```

The `proxy_pass http://api:8020` works because Docker Compose puts both containers on the same network and `api` resolves to the backend container.

---

## Local Development

### Without Docker (recommended during development)

```bash
# Terminal 1: Backend
uv run uvicorn api.main:app --reload --port 8020

# Terminal 2: Frontend
cd frontend && npm run dev
```

The Vite dev server proxies `/api` requests to the backend automatically.

### With Docker

```bash
docker compose up --build
```

Access the app at `http://localhost:8095`.

---

## Docker Log Rotation

To prevent logs from filling disk, configure Docker's log rotation. Add to `/etc/docker/daemon.json`:

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

This caps each container at 30MB of logs (3 files x 10MB). Restart Docker after changing: `sudo systemctl restart docker`.

---

## Choosing a Deployment Path

Three paths, in increasing order of managed-infrastructure and cost-at-idle. Pick one — they're mutually exclusive in production, though local `docker compose` is identical for all three.

| Path | Best for | Hosting | Idle cost | CI |
|------|----------|---------|-----------|----|
| **A — Raspberry Pi** | Hobby / homelab, full control, free | Self-hosted Docker + Cloudflare Tunnel | Hardware only | Push-to-`main` auto-deploy script |
| **B — Azure** | Managed containers, always-warm option | Azure Container Apps (scale-to-zero) | ~$35–50/mo (always-on Redis) | Terraform + GitHub Actions |
| **C — AWS** | Cheapest managed, true scale-to-zero | Lambda (container images) + API Gateway + CloudFront | ~$0–5/mo (pay-per-request) | Terraform + GitHub Actions (OIDC) |

The big structural difference between B and C is the async/event model: Azure runs an **always-on Redis** queue with a KEDA-scaled worker; AWS uses **SNS fan-out to per-handler Lambdas** with nothing always-on. The app abstracts this behind an events backend switch (`EVENTS_BACKEND=local|sns`), so the same code runs locally and on either cloud.

---

## Path A: Raspberry Pi Deployment

The Pi runs Docker, MongoDB (on host), nginx (reverse proxy), Cloudflare Tunnel, and an auto-deploy script.

### Deploy a new app

1. Build and start containers on the Pi:
   ```bash
   cd /path/to/appname
   docker compose up -d --build
   ```

2. Add nginx config and Cloudflare DNS record (see `house/adding-new-app.md`)

3. Register the app's ports in `house/applications.md`

4. Add the app to the auto-deploy script's `APPS` array for automatic deployment on push to `main`

### Pi-specific notes

- The Pi's MongoDB instance is shared across apps. Each app uses a separate database (set via `MONGODB_DB_NAME`)
- Apps never access each other's databases directly — cross-app communication goes through REST APIs
- No extra files needed beyond the standard `Dockerfile` and `docker-compose.yml`

---

## Path B: Azure Deployment (Azure Container Apps)

Three environments managed by Terraform + GitHub Actions, on **Azure Container Apps** with **scale-to-zero**. Prod is fronted by Cloudflare on a custom domain. Budget **~$35–50/mo** for a low-volume app — the always-on Redis container is the main line item; the frontend/api/worker all scale to zero when idle.

> This path used to target Azure Container Instances (ACI). Container Apps is cheaper at low volume (ACI bills 24/7) and supports scale-to-zero + KEDA. The gotchas below are all Container-Apps-specific and were hit on a real build — read them before you start.

### Resources per component

| Component | Azure resource | Notes |
|---|---|---|
| **Frontend** | `azurerm_container_app` | nginx; **external** ingress :80; `min_replicas = 0`. Public front door — serves the SPA and proxies `/api` to the api app. |
| **API** | `azurerm_container_app` | FastAPI; **internal** ingress :8020 (`external_enabled = false`); `min_replicas = 0`. Never publicly exposed; reached only via the frontend. |
| **Worker** | `azurerm_container_app` | no ingress; `command = ["python","-m","worker"]`; `min_replicas = 0`; **KEDA `redis-streams` scaler** (wake-from-zero). |
| **Redis** | `azurerm_container_app` | `redis:7-alpine`; **internal TCP** ingress :6379; **always-on** (`min = max = 1`). NOT managed — see gotcha #2. |
| **Environment** | `azurerm_container_app_environment` | VNet-injected (`infrastructure_subnet_id`); Consumption (keeps scale-to-zero). |
| **Network** | `azurerm_virtual_network` + `azurerm_subnet` | dedicated /23 subnet delegated to `Microsoft.App/environments`, one per env. Free. |
| **Egress IP** (optional) | `azurerm_nat_gateway` + `azurerm_public_ip` | deferred behind a flag; gives one static egress IP to whitelist in Mongo Atlas. ~$36/mo when enabled. |
| **Registry** | ACR (`data` source) | images pushed by CI. |
| **Photo storage** | Azure Blob (`data` source) | `STORAGE_BACKEND=azure`; one container per env (`photos` / `photos-dev`). |
| **TF state** | Azure Blob | `backend "azurerm"`. |

### Environments

| Tier | Trigger | URL |
|------|---------|-----|
| **Local** | `docker compose up --build` | `localhost:<port>` |
| **Dev** | PR opened/updated | generated `<frontend-app>-dev.<envid>.<region>.azurecontainerapps.io` |
| **Production** | push to `main` | custom domain via Cloudflare (e.g. `yourapp.com`) |

Dev uses the auto-generated Container Apps FQDN (no custom domain). It's stable for the life of the environment but changes if the app is **renamed** (see gotcha #3) — relevant for the Google OAuth origin.

### Additional Files for Azure

**`Dockerfile.api`** — standalone API image (also runs the worker via a different command):
```dockerfile
FROM python:3.12-slim
COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /bin/
WORKDIR /app
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev
COPY api/ api/
COPY worker/ worker/
EXPOSE 8020
CMD ["uv", "run", "uvicorn", "api.main:app", "--host", "0.0.0.0", "--port", "8020"]
```

**`Dockerfile.frontend`** — nginx image using an **envsubst template** (the api host is per-environment, so it can't be hardcoded):
```dockerfile
FROM node:20-slim AS build
WORKDIR /app
COPY frontend/package*.json ./
RUN npm ci
COPY frontend/ .
RUN npm run build

FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
# nginx:alpine runs envsubst on /etc/nginx/templates/*.template at startup,
# substituting ${API_HOST} (injected by the Container App) into the config.
COPY frontend/default.conf.template /etc/nginx/templates/default.conf.template
EXPOSE 80
```

**`frontend/default.conf.template`** — proxies `/api` to the internal api app by name:
```nginx
server {
    listen 80;
    root /usr/share/nginx/html;
    index index.html;

    location /api/ {
        # ${API_HOST} -> e.g. myapp-api-dev / myapp-api-prod (set on the container)
        proxy_pass http://${API_HOST}/api/;
        # Container Apps internal ingress routes by Host — send the api app name,
        # NOT the public frontend host, or envoy returns a 404 page (gotcha #4).
        proxy_set_header Host ${API_HOST};
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host $host;
        client_max_body_size 20M;
    }

    location / {
        try_files $uri $uri/ /index.html;
    }
}
```
On the frontend Container App, set `API_HOST` to the api app name and limit substitution so nginx's own `$host`/`$uri`/etc. survive:
```hcl
env { name = "API_HOST" value = local.api_app_name }
env { name = "NGINX_ENVSUBST_FILTER" value = "API_HOST" }
```
(Local compose still uses `frontend/nginx.conf` with `proxy_pass http://api:8020` — unchanged.)

### Terraform Structure

```
infra/
├── providers.tf       # azurerm provider + azurerm backend (state in Blob)
├── variables.tf       # inputs (secrets, sizing, enable_nat_gateway, custom_domain)
├── main.tf            # locals + data sources + (dev) blob container
├── network.tf         # VNet + delegated subnet + NAT module (flagged off)
├── redis.tf           # internal Redis Container App
├── containerapps.tf   # environment + frontend / api / worker apps
└── outputs.tf         # frontend FQDN, api internal FQDN, NAT IP
```

Do **NOT** commit a `terraform.tfvars` — it is auto-loaded into *every* workspace, so prod-only values (e.g. the prod invite URL) leak into dev. Pass per-env values via `-var` / `TF_VAR_` in the workflow instead.

Key patterns:
- **Env-scoped app names** via locals: `myapp-api-${var.environment}` etc. (gotcha #3).
- `data` sources for shared pre-existing resources (resource group, ACR, storage account).
- `min_replicas = 0` everywhere except Redis (`1`).
- cpu/memory must be a **valid pair**: 0.25/0.5Gi, 0.5/1Gi, 0.75/1.5Gi, 1/2Gi…
- Distinct VNet CIDRs per env (e.g. prod `10.10.0.0/16`, dev `10.20.0.0/16`) so they can be peered later.
- State in Azure Blob; environment selected per workspace (`terraform workspace select prod|dev`).

**Worker — KEDA scaler** (in `containerapps.tf`):
```hcl
custom_scale_rule {
  name             = "redis-streams-scaler"
  custom_rule_type = "redis-streams"
  metadata = {
    address             = local.redis_addr        # myapp-redis-<env>:6379
    stream              = "item_photo_added"       # your stream name
    consumerGroup       = "workers"
    pendingEntriesCount = "5"
    enableTLS           = "false"
    databaseIndex       = "0"
  }
}
```

**Worker — required code change (gotcha #5):** the worker reaches Redis through the environment's ingress proxy, so a blocking `XREADGROUP BLOCK` that returns no messages can raise a client-side `TimeoutError`. Catch it (= "no new messages", retry) and enable keepalive:
```python
from redis.exceptions import ConnectionError as RedisConnectionError, TimeoutError as RedisTimeoutError

redis = Redis.from_url(url, decode_responses=True, socket_keepalive=True)
while True:
    try:
        results = await redis.xreadgroup(group, consumer, streams, count=10, block=5000)
    except RedisTimeoutError:
        continue
    except RedisConnectionError:
        await asyncio.sleep(1); continue
    ...
```

### Secret Management

Same multi-layer flow, but the Terraform layer is Container App `secret {}` + `env { secret_name = ... }` (not `secure_environment_variables`):

```
GitHub Environment Secret (e.g. MONGODB_URL on "production")
  → Workflow env: TF_VAR_mongodb_url: ${{ secrets.MONGODB_URL }}
  → Terraform variable "mongodb_url" (sensitive)
  → azurerm_container_app {
       secret { name = "mongodb-url" value = var.mongodb_url }
       template.container.env { name = "MONGODB_URL" secret_name = "mongodb-url" }
     }
  → Container env var MONGODB_URL
  → pydantic-settings: Settings.mongodb_url
```
Non-secret config uses plain `env { name = ... value = ... }`. All layers must be wired for a secret to reach the container.

Standard secrets for an app with email + Google + AI:

| GitHub Secret | Terraform Variable | Container Env Var |
|---|---|---|
| `MONGODB_URL` | `mongodb_url` | `MONGODB_URL` |
| `JWT_SECRET` | `jwt_secret` | `JWT_SECRET` |
| `AZURE_STORAGE_CONNECTION_STRING` | `azure_storage_connection_string` | `AZURE_STORAGE_CONNECTION_STRING` |
| `SMTP_EMAIL` | `smtp_email` | `SMTP_EMAIL` |
| `SMTP_APP_PASSWORD` | `smtp_app_password` | `SMTP_APP_PASSWORD` |
| `GOOGLE_CLIENT_ID` | `google_client_id` | `GOOGLE_CLIENT_ID` (+ `VITE_GOOGLE_CLIENT_ID` build-arg on the frontend image) |
| `ANTHROPIC_API_KEY` | `anthropic_api_key` | `ANTHROPIC_API_KEY` (worker) |
| `ACR_PASSWORD` | `acr_password` | _(used in the app `registry` block)_ |

`REDIS_URL` is **not** a secret here — it's a plain env var `redis://myapp-redis-<env>:6379` (internal, no auth).

### GitHub Actions Workflows

**`deploy.yml`** (production):
- Trigger: push to `main`
- Job 1 (`build`): build + push **api and frontend** images to ACR (`:$SHA` + `:latest`)
- Job 2 (`deploy`): `environment: production` → `terraform init` → `workspace select prod` → `plan` → `apply`. Passes `TF_VAR_custom_domain` + `TF_VAR_invite_base_url`.

**`deploy-dev.yml`** (development):
- Trigger: `pull_request: [opened, synchronize, reopened]`, concurrency group with cancel-in-progress
- Job 1 (`build`): build + push api + frontend images (`:dev-$SHA`)
- Job 2 (`deploy`): `environment: development` → `workspace select dev` → `plan` → `apply`
- Post-deploy: comment the dev URL from `terraform output -raw frontend_url` (don't hardcode it)

### Azure Gotchas (all hit on a real build)

1. **Register resource providers (one-time, needs an owner).** First apply fails with `MissingSubscriptionRegistration: Microsoft.App`. The CI service principal can't self-register:
   ```bash
   az provider register --namespace Microsoft.App --wait
   az provider register --namespace Microsoft.OperationalInsights --wait
   ```
2. **Classic Azure Cache for Redis is blocked for new creates** ("create Azure Managed Redis instead"; AMR is ~$50–75/mo each). Run Redis as an **internal Container App** (`redis:7-alpine`, TCP :6379, always-on, no auth/TLS). Fine because the queue is an ephemeral event bus — a restart just drops un-processed events.
3. **Container App names must be unique within the resource group** (dev + prod share one RG). Hardcoded `myapp-api` etc. collide on the prod apply (`already exists`). **Env-scope every app name** (`myapp-api-${env}`). This renames existing apps on the next apply (destroy/recreate) and changes the generated FQDN → re-add the dev origin to Google.
4. **nginx → internal api needs the right `Host`.** Internal ingress routes by Host; sending the public frontend host returns the ingress's **HTML 404** for *every* `/api/*` call (looks like "all API calls 404"). Send `Host: <api-app-name>` via the `${API_HOST}` envsubst.
5. **Blocking Redis reads time out through the ingress proxy** → worker crash-loops and never drains events, *even though publish + KEDA scaling work*. Catch `TimeoutError` + set `socket_keepalive=True` (code above).
6. **cpu/memory must be a valid pair** (0.5 cpu → 1Gi, etc.). ACI's "0.1 GB increments / use 0.3" rule does NOT apply here.
7. **Scale-to-zero cold start** — ~1–3s on the first request after idle (api and frontend). Set `min_replicas = 1` if that matters for users or agents; `0` is cheapest.
8. **MongoDB Atlas** still needs `0.0.0.0/0` unless you enable the NAT gateway (`enable_nat_gateway = true`), then whitelist `terraform output -raw nat_gateway_ip` and drop `0.0.0.0/0`. Enabling/disabling NAT is in-place — no environment rebuild.
9. **The custom-domain binding is manual** (not in Terraform). If a future apply recreates the frontend app, re-run the bind (the `asuid` TXT + verification ID stay valid).
10. Setting explicit `permissions` on a GitHub Actions job removes all defaults — include `contents: read` for checkout (and `pull-requests: write` for the dev-URL comment).

### Cloudflare + Custom Domain (prod)

The apps get generated `*.azurecontainerapps.io` FQDNs; prod is served on your real domain via Cloudflare. Binding a custom domain to a Container App is a **manual, two-step** flow.

1. **Get the values:**
   ```bash
   az containerapp show -n <frontend-app>-prod -g <rg> --query "properties.configuration.ingress.fqdn" -o tsv
   az containerapp show -n <frontend-app>-prod -g <rg> --query "properties.customDomainVerificationId" -o tsv
   ```
2. **Cloudflare DNS** — set CNAMEs to **DNS-only / grey cloud** first:
   - `CNAME yourapp.com` (and `www`) → the frontend FQDN
   - `TXT asuid` and `TXT asuid.www` → the verification ID (ownership proof)
3. **Attach then bind** (`bind` does NOT auto-add the hostname):
   ```bash
   az containerapp hostname add  -n <frontend-app>-prod -g <rg> --hostname yourapp.com
   az containerapp hostname bind -n <frontend-app>-prod -g <rg> --hostname yourapp.com \
       --environment <env-name> --validation-method HTTP
   ```
   - The **apex can't use `--validation-method CNAME`** — Cloudflare flattens the apex CNAME into A records, so Azure says "Supported validation method(s): HTTP, TXT". **Use `HTTP`**: it validates through the app and avoids the manual DCV-token TXT that the `TXT` method makes you chase.
   - If a bind errors `Certificate '…' is not in succeeded provisioning state`, it **reused a stuck cert** from a prior failed attempt. Delete it and re-bind:
     ```bash
     az containerapp env certificate list   -n <env> -g <rg> -o table
     az containerapp env certificate delete -n <env> -g <rg> --certificate <mc-…> --yes
     ```
   - The hostname must be `add`ed before a managed cert can be created (`RequireCustomHostnameInEnvironment`), and the `asuid` TXT must be live before `add` (`InvalidCustomHostNameValidation`). Order: DNS records → `add` → `bind`.
4. Wait for the hostname to reach `SniEnabled` (cert issued, up to ~20 min — `az containerapp hostname list …`), then flip the CNAMEs to **Proxied / orange** and set Cloudflare **SSL/TLS → Full (strict)** (Flexible causes redirect loops).
5. **Keep the `asuid` TXT records** — they're reused on any future re-bind (the verification ID is stable per app).
6. **Google OAuth (if used):** add the exact serving origin to **Authorized JavaScript origins** — `https://yourapp.com` for prod, and the generated `https://<frontend-app>-dev.<envid>….azurecontainerapps.io` for dev. The ID-token flow validates JS origins only (no redirect URI). Renaming apps changes the dev FQDN → re-add it.
7. **Managed-cert renewal** behind the Cloudflare proxy can be flaky (~6-month renewal re-validates through DNS/HTTP). If it ever fails to renew, upload a **Cloudflare Origin Certificate** as a custom cert (15-yr, no renewal dance).

### One-Time Bootstrap for Azure

1. **Register `Microsoft.App` + `Microsoft.OperationalInsights`** resource providers (gotcha #1).
2. Create the Terraform state storage account + container.
3. Create a service principal for GitHub Actions (Contributor on the resource group).
4. Create the shared resources the `data` sources expect: resource group, ACR, storage account (+ blob container).
5. Create GitHub Environments (`production`, `development`) with secrets (including `SMTP_EMAIL` / `SMTP_APP_PASSWORD` for password-reset emails — see Phase 01 for Gmail App Password setup).
6. Create `infra/` Terraform files + `deploy.yml` / `deploy-dev.yml`.
7. After the first prod apply, bind the custom domain via Cloudflare (above).

---

## Path C: AWS Deployment (Lambda + API Gateway + CloudFront)

Fully serverless and **scale-to-zero with nothing always-on** — the whole stack idles at ~$0–5/mo (Lambda + API Gateway + CloudFront are pay-per-request; the only standing cost is MongoDB Atlas, which has a free tier). Two environments (dev + prod) managed by Terraform + GitHub Actions, deployed from container images. This path was built and is live for a real app (`recall`: dev `dev.sharedrecall.com`, prod `sharedrecall.com`) — the gotchas below were all hit on that build.

> Versus Azure (Path B): there is **no always-on Redis and no KEDA**. Background work is event-driven via **SNS → one Lambda per handler**; when idle, zero compute runs. The frontend is **S3 + CloudFront same-origin** (no per-env nginx envsubst — the SPA calls a relative `/api` that CloudFront proxies to API Gateway).

### Resources per component

| Component | AWS resource | Notes |
|---|---|---|
| **API** | `aws_lambda_function` (`package_type = "Image"`) | FastAPI container fronted by the **Lambda Web Adapter** (LWA) — runs plain uvicorn, no code change. Exposed via **API Gateway HTTP API**, not a Function URL (gotcha #1). 1024 MB / 30 s defaults. |
| **Processors** | `aws_lambda_function` × N (one per handler) | All off **one worker image**; each overrides the handler via `image_config.command`. Entry point is `awslambdaric` (Lambda Runtime Interface Client). SNS-subscribed. 512 MB / 120 s defaults. |
| **HTTP front door** | `aws_apigatewayv2_api` (HTTP API) | `$default` route → `AWS_PROXY` integration → API Lambda, payload format **2.0**, `auto_deploy` stage. LWA consumes the v2 payload natively. |
| **Event bus** | `aws_sns_topic` | One topic per env (e.g. `myapp-<env>-memory-created`). API publishes; each processor Lambda is an `aws_sns_topic_subscription` (protocol `lambda`) with an `aws_lambda_permission` for SNS invoke. Fire-and-forget, no DLQ. |
| **Frontend** | `aws_s3_bucket` (private) + `aws_cloudfront_distribution` | S3 locked down with **Origin Access Control** (only CloudFront reads it). CloudFront has **two origins**: S3 (SPA) and the API Gateway host (`/api/*`). A **CloudFront Function** rewrites extension-less paths to `/index.html` for SPA routing. |
| **TLS cert** | `aws_acm_certificate` (+ `_validation`) | DNS-validated in **us-east-1** (CloudFront requires us-east-1). Validation CNAME added manually in Cloudflare. |
| **IAM** | one shared `aws_iam_role` (`lambda_exec`) | All four functions share it: `AWSLambdaBasicExecutionRole` (CloudWatch Logs) + an inline `sns:Publish` policy. No VPC, so no networking perms. |
| **Logs** | `aws_cloudwatch_log_group` per function | `retention_in_days` configurable (14 default). |
| **Registry** | ECR (`data` source) | `myapp-api` / `myapp-worker`; images pushed by CI, pulled by Lambda (needs an ECR repo policy — created in bootstrap). |
| **Database** | MongoDB Atlas (external) | Lambda runs **outside any VPC** and reaches Atlas over the public internet; Atlas Network Access must allow it (`0.0.0.0/0` for dev, or PrivateLink for prod). |
| **TF state** | S3 bucket + DynamoDB lock | Created by the bootstrap stack. |

### Environments

| Tier | Trigger | URL | State key |
|------|---------|-----|-----------|
| **Local** | `docker compose up --build` | `localhost:<port>` (`EVENTS_BACKEND=local`, in-process events) | — |
| **Dev** | PR opened/updated → `main` | `https://dev.yourapp.com` (`deploy-dev.yml`, GitHub env `development`) | `myapp/dev.tfstate` |
| **Production** | push/merge to `main` | `https://yourapp.com` (`deploy-prod.yml`, GitHub env `production`) | `myapp/prod.tfstate` |

Both clouds share one ECR/state/bootstrap; the per-env stack is selected entirely by the `environment` variable and the state **key** (no Terraform workspaces — the key carries the separation).

### Additional Files for AWS

**`docker/Dockerfile.api`** — FastAPI behind the Lambda Web Adapter (the adapter bridges the Lambda Runtime API to a normal HTTP server, so **no app code change** is needed):
```dockerfile
FROM ghcr.io/astral-sh/uv:python3.12-bookworm-slim

# Lambda Web Adapter as an external extension — bridges Lambda/API GW <-> uvicorn.
COPY --from=public.ecr.aws/awsguru/aws-lambda-adapter:0.8.4 /lambda-adapter /opt/extensions/lambda-adapter

WORKDIR /app
ENV UV_COMPILE_BYTECODE=1 UV_LINK_MODE=copy \
    PORT=8000 AWS_LWA_PORT=8000 AWS_LWA_READINESS_CHECK_PATH=/api/health

COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev --no-install-project
COPY api/ api/
COPY core/ core/
COPY handlers/ handlers/
RUN uv sync --frozen --no-dev
ENV PATH="/app/.venv/bin:${PATH}"
CMD ["uvicorn", "api.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

**`docker/Dockerfile.worker`** — one image for all processors; the Lambda Runtime Interface Client (`awslambdaric`) is the entry point and each Lambda overrides the handler via `image_config.command`:
```dockerfile
FROM ghcr.io/astral-sh/uv:python3.12-bookworm-slim
WORKDIR /app
ENV UV_COMPILE_BYTECODE=1 UV_LINK_MODE=copy

COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev --no-install-project
COPY api/ api/
COPY core/ core/
COPY handlers/ handlers/
RUN uv sync --frozen --no-dev
RUN uv pip install awslambdaric
ENV PATH="/app/.venv/bin:${PATH}"

ENTRYPOINT ["python", "-m", "awslambdaric"]
CMD ["handlers.tagging.handler"]   # overridden per-Lambda in Terraform
```

(Local `docker compose` still uses the standard multi-stage `Dockerfile` + `frontend/nginx.conf` from the top of this phase — these two files are AWS-only.)

**SNS handler contract** — each processor is a thin Lambda handler that unwraps the SNS envelope and calls shared `core/` logic:
```python
# handlers/tagging.py
import json
from core.tagging import process_memory_tagging

def handler(event, _context):
    for record in event["Records"]:
        msg = json.loads(record["Sns"]["Message"])
        process_memory_tagging(msg["memory_id"], msg["user_id"])
```

The API publishes the same envelope via boto3 when `EVENTS_BACKEND=sns`; locally (`=local`) the same `events.publish()` dispatches in-process so no SNS/queue is needed for dev or tests.

### Frontend serving (no envsubst needed)

Unlike Azure, the API host isn't baked into the frontend. CloudFront serves the SPA from S3 and proxies `/api/*` to the API Gateway origin, so the SPA just calls a **relative `/api`** — same-origin, no CORS, no build-time API URL, no per-env nginx template. The only build-time frontend input is `VITE_GOOGLE_CLIENT_ID` (public).

### Terraform Structure

```
infra/
├── versions.tf      # aws provider (~> 5.40) + partial S3 backend (init-time config)
├── variables.tf     # project, environment, image_tag, secrets, sizing, custom_domain
├── data.tf          # ECR data sources + locals (image URIs, base_env / api_env, processors map)
├── lambda.tf        # API Lambda + processor Lambdas (for_each) + SNS subscriptions + log groups
├── apigateway.tf    # HTTP API, AWS_PROXY integration, $default route+stage, invoke permission
├── sns.tf           # the memory_created topic
├── iam.tf           # one shared lambda_exec role (logs + sns:Publish)
├── acm.tf           # us-east-1 cert, DNS-validated
├── frontend.tf      # S3 (private) + OAC + CloudFront (2 origins) + SPA-router function
├── outputs.tf       # app_url, cloudfront_distribution_id, frontend_bucket, api_endpoint, acm_validation_record
└── bootstrap/       # run ONCE, locally, with admin creds (local state) — see below
```

Key patterns:
- **Env-scoped names** via a local: `name = "${var.project}-${var.environment}"` → `myapp-dev-api`, `myapp-prod-api`, etc.
- The processors are a **map local** (`{ tagging = "handlers.tagging.handler", ... }`) driven through `for_each` — one Lambda, one subscription, one log group, one invoke permission per entry. Add a handler by adding a map key.
- **Container-image Lambdas** (`package_type = "Image"`, `image_uri`) — no zip packaging; CI builds and pushes to ECR and passes the `image_tag` (git SHA).
- Secrets arrive as **`sensitive` variables** (`TF_VAR_*` from CI), land in `base_env` / `api_env` locals, and become Lambda `environment.variables`.
- **Partial S3 backend** (`backend "s3" {}`) — bucket/key/region/lock table supplied at `terraform init` time by the workflow, so the same code serves both envs.

### Secret Management

Same multi-layer flow as Azure, but the cloud layer is Lambda `environment.variables` (sourced from `sensitive` TF vars). GitHub holds env-scoped **secrets** and repo-level **variables**:

| GitHub (kind) | Terraform variable | Lands as |
|---|---|---|
| `MONGODB_URL` (secret) | `mongodb_url` (sensitive) | Lambda env `MONGODB_URL` |
| `JWT_SECRET` (secret) | `jwt_secret` (sensitive) | API Lambda env `JWT_SECRET` |
| `ANTHROPIC_API_KEY` (secret) | `anthropic_api_key` (sensitive) | Lambda env `ANTHROPIC_API_KEY` |
| `ANTHROPIC_MODEL` (variable) | `anthropic_model` (default `claude-sonnet-4-6`) | Lambda env `ANTHROPIC_MODEL` |
| `GOOGLE_CLIENT_ID` (variable) | `google_client_id` | API Lambda env + `VITE_GOOGLE_CLIENT_ID` frontend build |
| `AWS_DEPLOY_ROLE_ARN` (variable) | — | `role-to-assume` in the OIDC step |
| `TF_STATE_BUCKET` / `TF_LOCK_TABLE` (variables) | — | `terraform init -backend-config` |

`EVENTS_BACKEND` and `SNS_TOPIC_ARN_MEMORY_CREATED` are plain (non-secret) Lambda env values set by Terraform. **Keep the model id configurable** (`ANTHROPIC_MODEL`) — a hardcoded id 404s when that model is retired.

### GitHub Actions Workflows

Auth is **GitHub OIDC** — no long-lived AWS keys; the workflow assumes the `myapp-deploy` role (gotcha #5). `deploy-dev.yml` and `deploy-prod.yml` are near-duplicates differing only in trigger, GitHub environment, `ENVIRONMENT`, and the state key:

- Trigger — **dev**: `pull_request` to `main`; **prod**: `push` to `main` (both `+ workflow_dispatch`). `concurrency` serialized (shared env).
- `permissions: { id-token: write, contents: read, pull-requests: write }` — `id-token` for OIDC, `pull-requests` for the dev-URL comment (setting any `permissions` block drops the defaults — include `contents: read` for checkout).
- Steps: checkout → resolve image tag (PR head SHA / push SHA) → `configure-aws-credentials@v4` (OIDC) → `amazon-ecr-login@v2` → **build & push** `api` + `worker` images to ECR → `terraform init` (backend-config from vars) + `apply` (`-var image_tag=$SHA …`, secrets via `TF_VAR_*`) → **build frontend** (Node 20, `VITE_GOOGLE_CLIENT_ID`) → `aws s3 sync frontend/dist` + `cloudfront create-invalidation --paths "/*"`.
- **dev** also comments the deployed URL (`terraform output -raw app_url`) on the PR.

### AWS Gotchas (all hit on a real build)

1. **Public Lambda Function URLs are blocked on new accounts.** A `FunctionUrlAuthType = NONE` URL **403s account-wide** (proven against a throwaway function), so you can't expose the API that way. Front the Lambda with an **API Gateway HTTP API** instead — the Lambda Web Adapter consumes the same v2 payload, so it's a pure infra swap, no image change.
2. **Container-image Lambdas need an ECR repository policy** even within the same account. Without it the function can't pull and the create fails. Add an `aws_ecr_repository_policy` granting `lambda.amazonaws.com` `ecr:BatchGetImage` + `ecr:GetDownloadUrlForLayer` (scoped to `function:myapp-*`) — it lives in the **bootstrap** stack.
3. **The ACM cert must be in us-east-1** regardless of where the rest lives, because CloudFront only reads certs from us-east-1. This whole stack is us-east-1, so it's automatic — but don't move it.
4. **CloudFront needs the right cache + origin-request policies for `/api/*`.** Use the managed `CachingDisabled` (never cache the API) **and** `AllViewerExceptHostHeader` (forward everything but `Host`, so API Gateway routes correctly). The SPA-router CloudFront Function must be attached **only** to the default (S3) behavior, never to `/api/*`.
5. **Setting explicit `permissions` on a job removes all defaults** — include `contents: read` (checkout) and, for the dev workflow, `pull-requests: write` (URL comment) alongside `id-token: write`.
6. **Lambda runs outside a VPC by default — that's wanted here** (it gets internet egress for free to reach Atlas + Anthropic). The price is that you can't put it behind a private Atlas endpoint without adding a VPC + NAT. For dev, allow Atlas from `0.0.0.0/0`; for prod, prefer PrivateLink.
7. **Cold starts on container-image Lambdas** are larger than zip (first request after idle ~1–3 s). Fine for low-volume apps; set Provisioned Concurrency if you need consistent latency.
8. **Secrets currently live in TF state + Lambda env** (state is encrypted at rest). Acceptable for dev; before prod move them to **SSM Parameter Store / Secrets Manager** (or the Lambda parameters-and-secrets extension) and tighten the deploy role's OIDC `sub` from `repo:<owner>/<repo>:*` to a specific branch/environment.

### Custom Domain (ACM + Cloudflare)

DNS lives in **Cloudflare (DNS-only / grey cloud)**; AWS terminates TLS at CloudFront. No Route 53.

1. First apply requests the ACM cert (pending validation). Read the validation CNAME:
   ```bash
   terraform output -json acm_validation_record
   ```
2. Add that **CNAME** in Cloudflare (DNS-only). ACM flips to **ISSUED** (minutes); the `aws_acm_certificate_validation` resource then resolves and CloudFront picks up the cert.
3. Point the hostname at CloudFront: add a **CNAME** (`dev.yourapp.com` → the distribution's `*.cloudfront.net` domain). For an **apex** (`yourapp.com`), use Cloudflare's CNAME flattening.
4. **Google OAuth (if used):** add each serving origin (`https://yourapp.com`, `https://dev.yourapp.com`) to the client's **Authorized JavaScript origins** (the ID-token flow validates JS origins, no redirect URI).

### One-Time Bootstrap for AWS

`infra/bootstrap/` runs **once, locally, with admin credentials** (local state, committed nowhere). It creates everything the per-env stack and CI can't create for themselves:

1. **TF remote state** — S3 bucket `myapp-tfstate-<account_id>` (versioned, encrypted, public-access-blocked) + DynamoDB lock table `myapp-tflock`.
2. **ECR repos** `myapp-api` / `myapp-worker` (with lifecycle policies + the Lambda-pull repo policy from gotcha #2).
3. **GitHub OIDC provider** + the `myapp-deploy` role CI assumes, with a pragmatic deploy policy scoped to the `myapp-*` name prefix (trust `repo:<owner>/<repo>:*`).
4. Wire the bootstrap outputs into **GitHub**: `AWS_DEPLOY_ROLE_ARN`, `TF_STATE_BUCKET`, `TF_LOCK_TABLE` as repo variables; create the `development` / `production` Environments and add `MONGODB_URL`, `JWT_SECRET`, `ANTHROPIC_API_KEY` secrets (+ `ANTHROPIC_MODEL`, `GOOGLE_CLIENT_ID` variables).
5. Create `infra/` (the per-env stack) + `deploy-dev.yml` / `deploy-prod.yml`.
6. After the first apply per env, add the ACM validation CNAME and the hostname CNAME in Cloudflare (above).

---

## Checklist

### All apps
- [ ] `Dockerfile` created (multi-stage: backend, frontend-build, frontend)
- [ ] `docker-compose.yml` created with correct ports and environment
- [ ] `frontend/nginx.conf` created with SPA fallback and API proxy
- [ ] `docker compose up --build` starts both services successfully
- [ ] App is accessible at `http://localhost:<frontend-port>`
- [ ] API proxy works (frontend can call `/api/*` endpoints)

### Path A only (Pi)
- [ ] nginx reverse proxy config added on Pi
- [ ] Cloudflare DNS record created
- [ ] App added to auto-deploy script

### Path B only (Azure — Container Apps)
- [ ] `Microsoft.App` + `Microsoft.OperationalInsights` providers registered
- [ ] `Dockerfile.api` + `Dockerfile.frontend` created; frontend uses the envsubst template
- [ ] `frontend/default.conf.template` created (`${API_HOST}`); `API_HOST` + `NGINX_ENVSUBST_FILTER` set on the frontend app
- [ ] `infra/` Terraform: env-scoped app names, VNet + delegated subnet, internal Redis app, frontend/api/worker apps, NAT module (flagged off)
- [ ] Worker has blocking-read resilience (catch `TimeoutError` + `socket_keepalive=True`) and a KEDA `redis-streams` scaler
- [ ] No committed `terraform.tfvars`; per-env values passed via `TF_VAR_`/`-var`
- [ ] GitHub Environments, secrets, and workflows (`deploy.yml`, `deploy-dev.yml`) created
- [ ] Test: PR deploys to dev, merge deploys to prod
- [ ] Prod custom domain bound via Cloudflare (CNAME + `asuid` TXT + `hostname add`→`bind --validation-method HTTP`, grey→orange, SSL Full (strict))
- [ ] Google OAuth Authorized JavaScript origins updated for the prod (and dev) URLs

### Path C only (AWS — Lambda + API Gateway + CloudFront)
- [ ] Bootstrap stack applied once (state bucket + DynamoDB lock, ECR repos + Lambda-pull policy, GitHub OIDC provider + `myapp-deploy` role)
- [ ] `docker/Dockerfile.api` (Lambda Web Adapter) + `docker/Dockerfile.worker` (`awslambdaric`, handler overridden per-Lambda) created
- [ ] SNS handlers (`handlers/*.py`) unwrap the SNS envelope and call `core/`; `EVENTS_BACKEND` switches `local` (dev/test) vs `sns` (cloud)
- [ ] `infra/` Terraform: env-scoped names, API Lambda behind **API Gateway HTTP API** (not a Function URL), processors via `for_each`, SNS topic + subscriptions, S3 + CloudFront (2 origins, OAC, SPA-router function), us-east-1 ACM cert
- [ ] GitHub OIDC role wired (`AWS_DEPLOY_ROLE_ARN`); state/lock vars (`TF_STATE_BUCKET`, `TF_LOCK_TABLE`); per-env secrets set
- [ ] `deploy-dev.yml` (PR→main) + `deploy-prod.yml` (push→main): build+push images to ECR, `terraform apply`, `s3 sync` + CloudFront invalidation
- [ ] Frontend calls relative `/api` (CloudFront proxies to API Gateway — no envsubst, no CORS)
- [ ] `ANTHROPIC_MODEL` configurable (not hardcoded); MongoDB Atlas Network Access allows Lambda egress
- [ ] Custom domain: ACM validation CNAME + host CNAME added in Cloudflare (DNS-only); Google OAuth origins updated
- [ ] Test: PR deploys to dev, merge deploys to prod
