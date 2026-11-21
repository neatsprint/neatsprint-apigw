# neatsprint-apigw

Traefik v3 edge configuration for NeatSprint (ADR-003). The repository contains configuration
only; the orchestrator runs the official `traefik:v3` image with these files mounted.

| File | Purpose |
| :--- | :--- |
| `traefik.yml` | Static config: entrypoints, file provider, JSON logs, Prometheus, dashboard |
| `dynamic/routers.yml` | Path-prefix routing to services |
| `dynamic/services.yml` | Upstream service URLs |
| `dynamic/middlewares.yml` | Security headers, CORS, compression, rate limits, ForwardAuth (M1) |

## Routing

| Prefix | Service | Host port (dev) |
| :--- | :--- | :--- |
| `/api/v1/auth`, `/api/v1/iam` | `neatsprint-iam` | 8001 |
| `/api/v1/core` | `neatsprint-core` | 8002 |
| `/api/v1/tasks` | `neatsprint-task` | 8003 |
| `/api/v1/teams` | `neatsprint-team` | 8004 |
| `/api/v1/work` | `neatsprint-worklog` | 8005 |
| `/api/v1/monitor` | `neatsprint-monitoring` | 8006 |
| `/api/v1/chat` | `neatsprint-chat` | 8007 |
| `/api/v1/reports` | `neatsprint-reports` | 8008 |
| `/api/v1/ai` | `neatsprint-ai` | 8009 |
| `/` | `neatsprint-web` | 3000 |

## Operating notes

- **Auth**: the `forwardauth-iam` middleware is defined but intentionally not attached in M0,
  because IAM's `/api/v1/iam/auth/verify` endpoint lands in M1. Services already enforce their
  own checks in code (ADR-007); the gateway is an additional choke point, never the only one.
- **Rate limits**: Traefik OSS counters are per instance. When more than one gateway replica is
  deployed, enable the Redis-backed rate-limit plugin so counters are shared.
- **Headers**: inbound `X-User-ID`, `X-Tenant-ID` and `X-Roles` are never trusted. ForwardAuth
  overwrites them; services also ignore them unless the JWT validates.
- **TLS**: local uses HTTP. Production terminates TLS at the managed ingress with ACME or a
  provisioned certificate; HSTS is already emitted by `security-headers`.
- **Dashboard**: exposed on `:8080` in development only (`api.insecure: true`). Disable in
  staging/production.

## Change process

Routing or middleware changes require a platform `docker compose config` check and a smoke test
through the gateway. Add the new service to `services.yml`, `routers.yml`, the compose file, and
`docs/ARCHITECTURE.md` §3.2 in the same change.
