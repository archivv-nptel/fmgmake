# Standalone Formbricks snapshot

Adapted on 2026-11-26 from `formbricks/formbricks` commit
`1104f2c6a8ebf706e27ae5aa69ceca1884d6d95d` (default branch: `main`).
Changes cover this guide, README clone instructions, a local source-build Compose file,
and GitHub Actions configuration. Upstream application source, test assertions, licenses,
attribution, dependency declarations, lockfiles, and tracked assets remain intact.
The existing `apps/web/vendor` SheetJS tarball is
an upstream tracked dependency; no new vendoring was introduced.

## Get started in Windows / WSL 2

Use the Linux filesystem inside WSL 2, with Docker Desktop configured for Linux containers and
WSL integration. Docker Desktop is not pinned by upstream: use a supported release whose Compose
supports `env_file.required` and BuildKit supports Dockerfile frontend `1.20`. Validation here used
Docker Engine/CLI 29.9.2 and Compose 2.40.3; these are not Windows Docker Desktop release numbers.

```sh
git clone https://github.com/archivv-nptel/fmgmake.git
cd fmgmake
nvm install 26.13.1
nvm use 26.13.1
npm install -g pnpm@11.7.0
pnpm install --frozen-lockfile
pnpm dev:setup
```

A native addon may compile: install a C/C++ build toolchain, make, and Python. The locked
`node-gyp` 8.4.1 needs Python `distutils`; Python 3.11 includes it. Point `npm_config_python`
at Python 3.11, or use a Python environment with working setuptools/distutils compatibility.
Do not change the lockfile or bypass package checksums to resolve an installation issue.
The app Dockerfile uses upstream's frozen `--ignore-scripts` install and builds its required
packages explicitly.

## Local source-built Docker stack

After `pnpm dev:setup`, edit the ignored `.env` with your own local settings and secrets.
Generate each secret independently with `openssl rand -hex 32`; never commit `.env`.
Supply `POSTGRES_PASSWORD`, `AUTHZED_DATABASE_PASSWORD` (URL-safe), `AUTHZED_TOKEN`,
`BETTER_AUTH_SECRET` (or legacy `NEXTAUTH_SECRET`), `ENCRYPTION_KEY`, `CRON_SECRET`,
`CUBEJS_API_SECRET`, and a separately provisioned `HUB_API_URL` / `HUB_API_KEY`.
For a password containing URL-special characters also set `POSTGRES_PASSWORD_URL_ENCODED`.

**External prerequisite:** this snapshot has no Hub or taxonomy service source. The web app
unconditionally validates Hub URL/key and Cube URL/secret. There is no supported Hub disable flag.
`docker-compose.local.yml` therefore requires an external Hub rather than pulling a prebuilt
upstream Formbricks image. Configure `CUBEJS_DB_HOST/PORT/NAME/USER/PASS` to match the database
of your separately provisioned Hub for analytics. The `.env.example` dev Hub URL/key are examples,
not a provisioned Hub. A URL resolving in configuration does not verify a working Hub service.

```sh
docker compose -f docker-compose.local.yml config --quiet
docker compose -f docker-compose.local.yml up --build -d
docker compose -f docker-compose.local.yml ps
docker compose -f docker-compose.local.yml logs formbricks
```

App and Cube images build from the local Dockerfiles. PostgreSQL18/pgvector, digest-pinned Valkey,
and SpiceDB1.57.1 provide persistence, cache, and authorization. The stack includes database
migrations and SpiceDB schema initialization. The app binds to `127.0.0.1:3000` by default; use
`LOCAL_BIND_ADDRESS` / `LOCAL_PORT` to change this. Database/cache ports are private.
Use `DOCKER_BUILD_NODE_OPTIONS=--max-old-space-size=4096` on smaller builders, although upstream's
web package scripts set their own 8192 MB heap limit; allow sufficient total memory for full builds.

For host development, upstream `pnpm go` / `pnpm db:up` use the original development Compose file,
which includes the upstream prebuilt Hub service. Use that workflow only if you permit that image.
For the no-upstream-image path use the new local Compose stack and supply Hub separately.

## Checks and CI

Run checks from the repository root so Turbo builds workspace dependencies first:

```sh
pnpm format:check
pnpm lint
pnpm typecheck --concurrency=1
pnpm test --concurrency=1 -- --maxWorkers=2
```

Core lint (including formatting), typecheck, unit tests, and Docker build validation run on pushes
to `main` and support `workflow_dispatch`. Standard GitHub-hosted runners replace custom builders.
Optional remote Turbo cache credentials and coverage/Sonar/preview/publish credentials are not
required by standalone core CI. Publishing, tag/release creation, deployment, translation sync,
and organization-only bots/notifications are gated to `formbricks/formbricks`, including manual
runs. Enterprise-license-dependent e2e tests are also upstream-only; they are not core unit tests.
All original workflow files remain. See the final setup report for the exact active/gated list
and measured validation results; configuration validation alone does not prove a green remote run.

## Licenses (preserve verbatim)

- `LICENSE`: copyright (c) 2024 Formbricks GmbH; GNU Affero General Public License version 3, 19 November 2007 (AGPLv3), with explicit carveouts for `apps/web/modules/ee` under its own license, specified SDK/survey package directories under MIT where their manifests say so, and third-party components under their original licenses.
- `apps/web/modules/ee/LICENSE`: **The Formbricks Enterprise Edition (EE) license (the “EE License”)**, copyright (c) 2020-present Formbricks GmbH. Production use requires compliance with Formbricks Enterprise Terms and a valid license for the correct number of seats; development/testing is permitted without subscription. This remains proprietary EE code, not converted to AGPL by copying the snapshot.
- `charts/spicedb-operator/LICENSE`: Apache License, Version 2.0, January 2004.
- `.agents/skills/pr/LICENSE.md`: MIT License; copyright (c) 2026 Matt Pocock.
- MIT package manifest declarations: `packages/config-prettier`, `packages/i18n-utils`, `packages/js-core`, `packages/logger`, `packages/survey-ui`, `packages/surveys`, `packages/vite-plugins`.

## Dependency audit

26 source `package.json` manifests scanned (all dependency/devDependency/optionalDependency/peerDependency/override/resolution maps). **No direct non-registry GitHub URL dependencies** (`github:`, `git+https://`, `https://github.com/`, `git@github.com`) found. No source Python (`pyproject.toml`, requirements, Pipfile), Rust (`Cargo.toml`), or Go (`go.mod`) manifests found. Repository/homepage/bugs/source links are attribution metadata, not fetched dependency URLs. Preserve `pnpm-lock.yaml` and `pnpm-workspace.yaml` unchanged.

## Exact toolchain evidence for Windows + WSL 2

- Use Linux checkout and commands inside WSL 2; Docker Desktop Linux-container engine with WSL integration.
- `.nvmrc`: **Node.js 26.13.1**; root engines: `>=26.0.0 <27`.
- root `packageManager`: **pnpm 11.7.0**.
- `apps/web/Dockerfile`: `node:26.13.1-trixie-slim` pinned to digest.
- PostgreSQL base: **pgvector/pgvector:pg18** (PostgreSQL major 18 + vector extension). Patch version is not pinned by upstream tag; cannot claim an exact unobserved patch version.
- Redis-compatible service: **valkey/valkey@sha256:418652cfb58ef879d4978c33553735d7147016032d5aefaa14c828e611eb9dfd**. Upstream uses Valkey rather than a `redis` image.
- Required authorization: **authzed/spicedb:v1.57.1**; dev CLI **authzed/zed:v1.14.0**.
- Upstream dev Hub: **ghcr.io/formbricks/hub:0.14.3** (independent service image; Hub source is not in this repository).
- Upstream dev Cube: **cubejs/cube:v1.8.25**; production custom Cube Dockerfile retained in `docker/cube-image/`.
- Docker Desktop version is **not pinned by upstream**. Do not equate the cloud's Docker Engine/Compose versions with an installed Windows Docker Desktop release. Cloud CLI/daemon versions should be reported separately if tested.

## Runtime .env requirements

Authoritative complete template is root `.env.example`; server validator is `apps/web/lib/env.ts` and client validator `apps/web/lib/env-client.ts`. Current v6 has additional mandatory services beyond PostgreSQL and Redis.

Required for basic server operation:

- `DATABASE_URL`: PostgreSQL connection URL.
- `WEBAPP_URL`: externally accessible application origin; `BETTER_AUTH_URL` should match (including custom auth path when applicable).
- `BETTER_AUTH_SECRET`: generate with `openssl rand -hex 32` (fresh installs need >=32 characters). Legacy `NEXTAUTH_SECRET` and `NEXTAUTH_URL` remain supported aliases; avoid replacing an existing auth secret because it invalidates sessions/tokens.
- `ENCRYPTION_KEY`: generate with `openssl rand -hex 32`.
- `CRON_SECRET`: generate with `openssl rand -hex 32`; documented mandatory for cron endpoints (schema allows omission but cron operations require it).
- `REDIS_URL`: Redis/Valkey URL, required outside tests; optional `REDIS_TLS_CA` when TLS trust requires it.
- SpiceDB is mandatory in v6: `AUTHZED_ENABLED=true`, `AUTHZED_ENDPOINT=host:50051`, `AUTHZED_TOKEN`, `AUTHZED_SYSTEM_KEY=formbricks`, `AUTHZED_INSECURE=true` only on trusted local plaintext network, `AUTHZED_CONSISTENCY=fully_consistent`. `AUTHZED_DATABASE_PASSWORD` is needed by Compose to provision the persistent SpiceDB datastore.
- `HUB_API_URL`, nonempty `HUB_API_KEY`; `HUB_DATABASE_URL` configures Hub DB. Hub API/worker/migrations are separate runtime services.
- `CUBEJS_API_URL`, nonempty `CUBEJS_API_SECRET`; optional `CUBEJS_JWT_ISSUER=formbricks-web` and `CUBEJS_JWT_AUDIENCE=formbricks-cube`. Cube DB connection uses `CUBEJS_DB_HOST/PORT/NAME/USER/PASS`.
- Compose requires `POSTGRES_PASSWORD` in the production stack (URL-encoded password variant is supported); dev stack uses local dev credentials and `pnpm dev:setup` generates/preserves secrets.

Email (only when enabled): `MAIL_FROM`, `MAIL_FROM_NAME`, `SMTP_HOST`, `SMTP_PORT`, `SMTP_USER`, `SMTP_PASSWORD`, `SMTP_AUTHENTICATED`, `SMTP_SECURE_ENABLED`, `SMTP_REQUIRE_TLS`, `SMTP_REJECT_UNAUTHORIZED_TLS`; keep certificate verification enabled. Without email configure `EMAIL_VERIFICATION_DISABLED=1`, `PASSWORD_RESET_DISABLED=1`. Optional SES: `SES_CONFIGURATION_SET`, `SES_EMAIL_ENVIRONMENT`; audit storage: `EMAIL_SEND_AUDIT_ENABLED`, `EMAIL_SEND_AUDIT_BUCKET`, `EMAIL_SEND_AUDIT_KMS_KEY_ARN`, `EMAIL_SEND_AUDIT_HMAC_KEY_ARN`.

Storage (conditional): `S3_ACCESS_KEY`, `S3_SECRET_KEY`, `S3_REGION`, `S3_BUCKET_NAME`, optional `S3_ENDPOINT_URL`, `S3_FORCE_PATH_STYLE=1` for compatible stores. `PUBLIC_URL` is for allowed external public storage URLs. S3-compatible storage is required for uploads in serverless deployments.

OAuth/SSO (conditional): `GITHUB_ID/GITHUB_SECRET`, `GOOGLE_CLIENT_ID/GOOGLE_CLIENT_SECRET`, `AZUREAD_CLIENT_ID/AZUREAD_CLIENT_SECRET/AZUREAD_TENANT_ID`, `OIDC_CLIENT_ID/OIDC_CLIENT_SECRET/OIDC_ISSUER/OIDC_DISPLAY_NAME/OIDC_SIGNING_ALGORITHM`, `SAML_DATABASE_URL`; enterprise features additionally require `ENTERPRISE_LICENSE_KEY`. Current server supports `AUTH_DEFAULT_ORGANIZATION_ID`, `AUTH_DEFAULT_ORGANIZATION_ROLE`, `AUTH_DEFAULT_TEAM_ID`, `AUTH_SKIP_INVITE_FOR_SSO`; `.env.example` still documents `AUTH_SSO_DEFAULT_*` variants, so consult validator/constants before using provisioning knobs.

Optional integrations stay intact: Notion (`NOTION_OAUTH_CLIENT_ID/SECRET`), Google Sheets (`GOOGLE_SHEETS_CLIENT_ID/SECRET/REDIRECT_URL`), Airtable (`AIRTABLE_CLIENT_ID`), Slack (`SLACK_CLIENT_ID/SECRET`), Stripe (`STRIPE_PUBLISHABLE_KEY/SECRET_KEY/WEBHOOK_SECRET`), AI (`AI_PROVIDER`, `AI_MODEL`, provider-specific credentials), Hub embedding/sentiment/emotions/translation provider settings, opt-in taxonomy and Qwen model services. None should be filled with invented production credentials.

Telemetry/error tracking controls:

- `TELEMETRY_DISABLED=1` disables Formbricks usage telemetry when no active EE license; upstream explicitly ignores this disable flag for licensed EE compliance telemetry.
- Leave `POSTHOG_KEY` unset to disable PostHog clients and PostHog Next rewrites; no separate `POSTHOG_DISABLED` flag found.
- Leave `SENTRY_DSN`, `SENTRY_AUTH_TOKEN`, `SENTRY_ENVIRONMENT` unset to disable runtime Sentry and authenticated build uploads; no separate app `SENTRY_DISABLED` flag found.
- Leave `OTEL_EXPORTER_OTLP_ENDPOINT` unset to avoid exporting to an external OpenTelemetry collector; `OTEL_LOGS_ENABLED`, tracing sampler flags are optional.
- Set `SPICEDB_TELEMETRY_ENDPOINT` empty in SpiceDB container (upstream dev compose already does).
- Clear `FORMBRICKS_WORKSPACE_ID` and `FORMBRICKS_APP_URL` in private local `.env` to avoid mounting upstream dogfood survey widget; `.env.example` currently ships an upstream workspace ID. Integration hooks are left intact.
