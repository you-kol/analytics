# Infra handoff: repo -> image -> runtime mapping

- Registry: ghcr.io/you-kol
- Versioning: single vX.Y.Z across all images; develop uses vX.Y.Z-dev.N; latest points to main.
- Platform: linux/arm64; runners: self-hosted

## Images and sources

| Service (runtime) | Image name | Source context | Dockerfile | Build args | Ports | Notes |
|---|---|---|---|---|---|---|
| web/worker/plugins/migrate/temporal-django-worker | analytics-app | . | Dockerfile | - | 8000 (web), 8001 (metrics) | Single image, multiple roles via command; includes Django, built frontend, plugin-server dist. |
| capture | analytics-capture | rust/ | rust/Dockerfile | BIN=capture | 3000 | Same image used for replay-capture with CAPTURE_MODE=recordings. |
| replay-capture | analytics-capture | rust/ | rust/Dockerfile | BIN=capture | 3000 | Different topic + mode. |
| property-defs-rs | analytics-property-defs-rs | rust/ | rust/Dockerfile | BIN=property-defs-rs | - | Kafka/PG/ClickHouse. |
| feature-flags | analytics-feature-flags | rust/ | rust/Dockerfile | BIN=feature-flags | 3001 | Requires Redis; optional cookieless Redis. |
| cyclotron-janitor | analytics-cyclotron-janitor | rust/ | rust/Dockerfile | BIN=cyclotron-janitor | - | Kafka + PG. |
| livestream | analytics-livestream | livestream/ | livestream/Dockerfile | - | 8080 | Mount /configs/configs.yml. |
| mcp | analytics-mcp | products/mcp/ | products/mcp/Dockerfile | - | - | Runs mcp-remote to remote MCP endpoint; needs auth. |

## Runtime configuration

- Shared: DATABASE_URL, CLICKHOUSE_*, KAFKA_HOSTS, REDIS_URL, SECRET_KEY, ENCRYPTION_SALT_KEYS.
- Object storage (recordings): OBJECT_STORAGE_*, SESSION_RECORDING_V2_S3_*.
- capture/replay-capture: ADDRESS, KAFKA_TOPIC, CAPTURE_MODE (events or recordings), REDIS_URL.
- feature-flags: READ_DATABASE_URL, WRITE_DATABASE_URL, REDIS_URL, COOKIELESS_REDIS_*, ADDRESS.
- mcp: POSTHOG_AUTH_HEADER (token), POSTHOG_REMOTE_MCP_URL (optional).

## Tagging & promotion

- develop: vX.Y.Z-dev.N, sha-<commit>
- main: vX.Y.Z, latest, sha-<commit>
- Rollback: redeploy previous immutable tag.

## Deployment mapping

- analytics-app provides multiple roles: web (port 8000), worker (Celery), plugins (plugin-server), migrate (Job), temporal-django-worker.
- Separate Deployments for: capture, replay-capture, property-defs-rs, feature-flags (port 3001), cyclotron-janitor, livestream (port 8080), mcp.
- Health/readiness: web /health/; feature-flags /health; others TCP or service-specific endpoints.

## CI

- Self-hosted GitHub Actions build and push images on develop/dev (prerelease) and main (final + latest).
- Workflows: .github/workflows/build-develop.yml, .github/workflows/release-main.yml.
