# Deploy and Host Manyfold on Railway

Manyfold is a self-hosted digital asset manager for 3D print files. Browse models
in interactive 3D, organise by tags, creators, and collections, share privately
or on the Fediverse, and keep STL/3MF libraries tidy on disk.

## About Hosting Manyfold

This template runs the standard Manyfold stack: the app (web UI plus background
workers), PostgreSQL for metadata, and Redis 8 for cache and Sidekiq jobs. Only
the app is public; the database and Redis stay on private networking. Persist a
single volume at `/models` for your library (Railway allows one volume per
service). First boot runs database prepare and may take a minute before
`/health` is ready. After deploy, create an admin account, point a library at
`/models`, and upload a small model to confirm storage and workers.

## Common Use Cases

- Personal or makerspace library for STL, 3MF, and related 3D print assets
- Shared team catalogue with tags, collections, and creator metadata
- Federated sharing of models across Manyfold instances / the Fediverse

## Dependencies for Manyfold Hosting

- Manyfold app (`manyfold3d/manyfold`)
- PostgreSQL 15
- Redis 8 (password required in `REDIS_URL` on Railway)

### Deployment Dependencies

- Upstream project: https://github.com/manyfold3d/manyfold
- Product site & docs: https://manyfold.app/
- Docker image: https://hub.docker.com/r/manyfold3d/manyfold
- Self-host configuration: https://manyfold.app/sysadmin/configuration.html

### Implementation Details

- Public entry: app on port `3214`, healthcheck `/health`
- `REDIS_URL` must include Redis auth (Railway Redis 8 rejects unauthenticated clients)
- One volume on `app` at `/models`; upstream’s plugins path is ephemeral on Railway

## Why Deploy Manyfold on Railway?

<!-- Recommended: Keep this section as shown below -->
Railway is a singular platform to deploy your infrastructure stack. Railway will host your infrastructure so you don't have to deal with configuration, while allowing you to vertically and horizontally scale it.

By deploying Manyfold on Railway, you are one step closer to supporting a complete full-stack application with minimal burden. Host your servers, databases, AI agents, and more on Railway.
<!-- End recommended section -->
