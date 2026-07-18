# not-cumplo-api

GCP API Gateway config repo — a single Swagger 2.0 YAML defines the public API surface, routing, `x-api-key` auth, and rate limits (5 req/min) for all Cloud Run backends in the not-cumplo ecosystem.

## Deploy

Requires a `.env` file (gitignored) that the Makefile `include`s. Variables needed:

```
CONFIGURATION_ID=api-configuration
API_ID=<gateway api id>
PROJECT_ID=cumplo-scraper
SERVICE_ACCOUNT=<backend auth SA email>
GATEWAY_ID=<gateway id>
LOCATION=us-central1
```

```sh
make login    # activate gcloud config + application-default login
make publish  # create new timestamped config + update gateway to it
```

There is no CI deployment — deploys are always manual.

## Validate

No local lint tooling. To catch YAML or OpenAPI errors before deploying:

```sh
gcloud api-gateway api-configs create <test-id> \
  --api=$API_ID \
  --openapi-spec=api-configuration.yml \
  --project=$PROJECT_ID \
  --backend-auth-service-account=$SERVICE_ACCOUNT \
  --dry-run
```

## Architecture

`api-configuration.yml` is the sole artifact — Swagger 2.0 with GCP extensions:

- `x-google-backend`: routes each path to a Cloud Run service via `APPEND_PATH_TO_ADDRESS`
- `x-google-quota`: rate-limits at the metric level (5 req/min per project)
- Auth: `x-api-key` header required on all paths

Backend Cloud Run services (region `us-central1`, project `cumplo-scraper`):
- `cumplo-spotter` — funding-request endpoints
- `cumplo-tailor` — user, channel, and filter endpoints

## Gotchas

- **Configs are immutable.** API Gateway never overwrites an existing config. `make publish`
  appends a timestamp (`CONFIGURATION_ID-YYYYMMDDHHMMSS`) on every deploy — the old config
  stays in the registry until manually deleted. The bulk-delete loop in the Makefile is
  commented out (broken) — clean up old configs manually via the GCP Console or `gcloud`.
- **`.env` is required and not committed.** The Makefile fails with confusing errors if it is
  missing. Keep a local copy; never commit it.
- **No CI deployment.** Only PR-title validation runs in GitHub Actions. There is no Cloud
  Build trigger here — all deploys go through `make publish` by hand.

## Git workflow

Branch prefixes: `feat/`, `fix/`, `chore/`, `ci/`. Conventional-commit subjects; subject must
start with an uppercase letter (enforced by CI PR-title check).

## Before committing

After making changes, validate the YAML (`--dry-run` above or manual deploy review); check no
hardcoded secrets (API keys, SA emails, tokens) beyond the `.env` variables — before committing.
