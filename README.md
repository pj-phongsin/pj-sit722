# CI/CD for a Microservices Application on Azure

A continuous integration and continuous deployment pipeline built with **GitHub Actions**, deploying a four-component containerised application to **Azure App Service** through separate staging and production environments.

Built for SIT722 (Software Deployment and Operation) at Deakin University — Task 10.2D.

---

## What's here

The application is three Python microservices plus a frontend, each independently containerised:

| Component | Role |
|---|---|
| `backend/customer_service` | Customer records |
| `backend/order_service` | Order processing |
| `backend/product_service` | Product catalogue |
| `frontend` | Web client |

Each service has its own `Dockerfile`, its own `requirements-dev.txt`, and its own `pytest` suite. `docker-compose.yml` runs the whole stack locally.

---

## Pipeline design

Two workflows, deliberately split by trigger so that reaching production requires an explicit human step:

```mermaid
graph LR
    A[push to testing] --> B[CI: test]
    B --> C[CI: build + push images]
    C --> D[CI: deploy to staging]
    D --> E[open PR to main]
    E --> F[PR merged]
    F --> G[CD: deploy to production]
```

### `ci.yml` — Build, Test, and Deploy to Staging
Triggered on every push to the `testing` branch. Three sequential jobs, each gated on the one before it:

1. **Build and Test** — spins up a `postgres:13` service container with a health check, installs each service's dev dependencies under Python 3.11, and runs the `pytest` suite for all three backend services. Nothing proceeds if a test fails.
2. **Build & Push Images** — logs in to Azure Container Registry and builds all four images, tagging each with `${{ github.sha }}` so every image traces back to the exact commit that produced it.
3. **Deploy to Staging** — deploys the four SHA-tagged images to their staging App Services (`customer-stg`, `order-stg`, `product-stg`, `frontend-stg`).

### `cd.yml` — Deploy to Production
Triggered only when a pull request into `main` is **closed and merged** — guarded by `if: github.event.pull_request.merged == true`, so closing a PR without merging deploys nothing. It checks out the merge commit and deploys the same four images to the production App Services (`customer-prod`, `order-prod`, `product-prod`, `frontend-prod`).

---

## Why it's split this way

Staging deploys automatically because feedback should be fast and cheap. Production deploys only behind a merged pull request, so promotion is a reviewed decision rather than a side effect of pushing code.

Tagging images by commit SHA rather than `latest` means staging and production can be verified to be running the same artefact, and a rollback is a redeploy of a known tag rather than a rebuild.

---

## Required secrets

Configured under **Settings → Secrets and variables → Actions**:

| Secret | Used for |
|---|---|
| `ACR_USERNAME` | Azure Container Registry login |
| `ACR_PASSWORD` | Azure Container Registry login |
| `AZURE_CREDENTIALS` | Service principal JSON for `azure/login` |

`AZURE_CREDENTIALS` is generated from the Azure CLI:

```bash
az ad sp create-for-rbac --name "github-actions" \
  --role contributor \
  --scopes /subscriptions/<subscription-id>/resourceGroups/<resource-group> \
  --sdk-auth
```

---

## Running locally

```bash
docker compose up --build
```

Running the backend test suites directly requires a reachable Postgres instance:

```bash
pip install -r backend/customer_service/requirements-dev.txt
pytest backend/customer_service/tests/
```

---

## Notes from building it

The first CI run failed because the workflow assumed a `Dockerfile` in each service directory and none existed yet — the build step can't be written before the thing it builds. Adding a `Dockerfile` per service fixed it.

The Postgres service container also needed `POSTGRES_DB: postgres` rather than an application-specific database name, because the tests connect to the default database first and create what they need from there. A health check (`pg_isready`) on the service container prevents the test job from racing the database's startup.
