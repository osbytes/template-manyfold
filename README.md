# Manyfold on Railway

Railway template scaffolding for [Manyfold](https://github.com/manyfold3d/manyfold),
a self-hosted digital asset manager for 3D print files. The published Railway
marketplace name is **manyfold**.

Based on the upstream [Docker Compose](https://manyfold.app/get-started/docker.html)
standard (non-solo) stack.

The template runs Manyfold as three Railway services:

- `app` — Manyfold web UI + workers (public)
- `postgres` — application database
- `redis` — cache and background jobs (Redis 8; Railway deploys need auth in `REDIS_URL`)

## Railway template

Follow [`docs/railway-wiring.md`](docs/railway-wiring.md) when creating or
updating the marketplace template. It contains the exact service names,
reference variables, generated secrets, ports, healthchecks, and volume mounts.

Paste [`docs/description.md`](docs/description.md) into the Railway template
**Description** field when publishing.

Only the **app** URL should be opened in a browser. Postgres and Redis stay on
Railway private networking. Railway allows **one volume per service** — mount
`/models` only; the upstream plugins path is ephemeral on Railway (see wiring
docs).

The wrapper currently tracks Manyfold `0.146.0` (Docker Hub semver tag for
upstream release `v0.146.0`).

## Run locally

1. Copy `.env.example` to `.env`.
2. Replace `SECRET_KEY_BASE` with a long random value (`openssl rand -hex 64`).
3. Start the stack:

```bash
docker compose up --build
```

Open `http://localhost:3214`. Create the administrator account when prompted,
then add a library pointed at `/models` (the mounted volume).

## Updating Manyfold

1. Bump the image tag in `services/app/Dockerfile` to the new Docker Hub semver
   (GitHub release `vX.Y.Z` → image tag `X.Y.Z`).
2. Diff upstream
   [docker-compose.example.yml](https://github.com/manyfold3d/manyfold/blob/main/docker-compose.example.yml)
   and [configuration docs](https://manyfold.app/sysadmin/configuration.html)
   for env/volume changes and port them here.
3. Run `docker compose config` and smoke-test `/health`, first-admin setup,
   library creation against `/models`, and a small model upload.

`upstream-check.yml` opens a version-bump PR weekly. Manyfold's Docker tags omit
the leading `v` from GitHub releases — strip it if a bot PR rewrites `FROM` to a
non-existent `v*` tag before merging.

## License

This repository contains deployment configuration only. Manyfold is distributed
under AGPL-3.0; see the
[upstream license](https://github.com/manyfold3d/manyfold/blob/main/LICENSE.md).

Community-maintained Railway packaging — not an official Manyfold project.
