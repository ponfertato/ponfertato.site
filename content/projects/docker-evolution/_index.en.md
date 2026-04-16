---
title: "Docker Configuration Evolution: From Scattered Containers to Managed Stacks"
description: "How deployment approach changed: service grouping, resource limits, security, observability."
tags: ["docker", "compose", "security", "monitoring", "infrastructure"]
---

Previously, each service lived in a separate repository with minimal `docker-compose.yml`:

```yaml
# Old approach: garrysmod-server/
services:
  garrysmod-server:
    image: ceifa/garrysmod:latest
    ports:
      - "27015:27015"
    volumes:
      - ./garrysmod:/data
    restart: unless-stopped
```

**Problems**:

- ❌ No resource control - one service could consume all memory
- ❌ No health checks - crashed services went undetected
- ❌ No log rotation - logs grew until disk full
- ❌ No network isolation - all services in `default` network
- ❌ Manual per-service updates - high risk of human error

---

## New Approach: Stacks with Explicit Contracts

Now related services are grouped into a single repository with unified configuration:

```
grafana-stack/
├── compose.yaml          # Unified stack: grafana, loki, oncall-*
├── stack.env             # Shared variables (extracted from code)
├── loki/
│   ├── Dockerfile        # Custom build if needed
│   └── loki.yml          # Application config
└── README.md             # Docs: ports, dependencies, deployment
```

### Key Changes

#### 1. Resource Limits (Preventing "Starvation")

```yaml
services:
  minecraft-server:
    cpus: "2.0" # Hard CPU limit
    mem_limit: 3g # Hard memory limit
    pids_limit: 300 # Protection against fork bombs
```

**Why**:

- Prevents one service from blocking others
- Enables precise load planning on the host
- Simplifies diagnostics: if a service hits a limit, it's immediately visible

#### 2. Security by Default

```yaml
services:
  grafana:
    security_opt:
      - no-new-privileges:true # Prevent privilege escalation
      - seccomp:unconfined # Only where truly needed (e.g., systemd inside)
```

**Why `no-new-privileges`**:

- Container cannot gain more privileges than it started with
- Protects against vulnerabilities exploiting setuid binaries
- Near-zero overhead

#### 3. Observability: Logging and Health Checks

```yaml
services:
  grafana:
    logging:
      driver: json-file
      options:
        max-file: "3" # Keep only 3 rotated files
        max-size: 10m # Each file max 10 MB
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:3000/api/health || exit 1"]
      interval: 15s # Check every 15 seconds
      timeout: 5s # Response timeout
      retries: 3 # 3 failures = unhealthy
      start_period: 30s # Ignore failures during startup
```

**Result**:

- Logs don't fill disk (rotation + limits)
- Orchestrator sees service state (`docker ps` shows `(healthy)`)
- Can alert on `unhealthy` status

#### 4. Network Isolation and Service Discovery

```yaml
networks:
  prometheus:
    external: true # Shared network for all metric-exporting services
    name: prometheus
  traefik:
    external: true # Shared network for public services behind proxy
    name: traefik

services:
  grafana:
    networks: [prometheus, traefik] # Sees both metrics and web traffic
  loki:
    networks: [prometheus] # Metrics only, no public access
```

**Benefits**:

- Services discover each other by name (`http://loki:3100`)
- Public access only through services explicitly connected to `traefik`
- Easy to add new service to monitoring: connect to `prometheus` network - it's already visible

#### 5. Graceful Shutdown and Signals

```yaml
services:
  minecraft-server:
    stop_signal: SIGTERM # First, polite request to stop
    stop_grace_period: 60s # Wait up to 60s before SIGKILL
    stdin_open: true # For interactive consoles
    tty: true
```

**Why**:

- Prevents data loss on restart (server saves world)
- Allows apps to close connections, flush caches properly
- `SIGTERM` + grace period - production deployment standard

#### 6. Configuration via `.env`, Not Hardcoding

```yaml
# compose.yaml
services:
  grafana:
    env_file: [stack.env]  # Extract sensitive data

# stack.env (in .gitignore)
GF_SECURITY_ADMIN_USER=admin
GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_PASS}
DOMAIN=potatoenergy.ru
```

**Benefits**:

- One secrets file - easier to rotate, easier to audit
- No passwords in repository
- Easy to deploy to different environments (dev/stage/prod) with different `.env`

---

## Comparison Table

| Criterion     | Old Approach         | New Approach                                         |
| ------------- | -------------------- | ---------------------------------------------------- |
| **Grouping**  | 1 service = 1 repo   | Related services = 1 stack                           |
| **Resources** | No limits            | `cpus`, `mem_limit`, `pids_limit`                    |
| **Security**  | Docker defaults      | `no-new-privileges`, explicit `seccomp`              |
| **Logging**   | Grow until disk full | Rotation: `max-file`, `max-size`                     |
| **Health**    | None                 | Health check with interval/timeout                   |
| **Networks**  | All in `default`     | Explicit external networks (`prometheus`, `traefik`) |
| **Shutdown**  | Instant `SIGKILL`    | `SIGTERM` + `stop_grace_period`                      |
| **Config**    | Hardcoded in compose | `.env` files, excluded from repo                     |

---

## Evolution in Numbers

| Metric                        | Before                                | After                                     |
| ----------------------------- | ------------------------------------- | ----------------------------------------- |
| Average stack deployment time | ~15 min (manual update of 5 services) | ~3 min (single `docker compose up -d`)    |
| Host memory consumption       | Unpredictable, frequent OOM           | Stable, limits guarantee isolation        |
| Recovery time after failure   | Depends on manual intervention        | Auto-restart + health check detects issue |
| Configuration audit           | Need to check 10+ repos               | One `compose.yaml` per stack              |

---

## Practical Recommendations

### When Migrating from Old Approach

1. **Start with one stack** (e.g., monitoring: grafana + loki + prometheus)
2. **Add limits gradually**: first `mem_limit`, then `cpus`, then `pids_limit`
3. **Test health checks locally** before deploy: `docker compose up --abort-on-container-exit`
4. **Extract secrets to `.env`** and add it to `.gitignore` before first commit
5. **Document external networks**: what each network is for, which services connect

### Checklist for New Service

- [ ] Resource limits: `cpus`, `mem_limit`, `pids_limit`
- [ ] Security: `no-new-privileges:true`, explicit `seccomp` where needed
- [ ] Logging: `json-file` driver with `max-file`/`max-size`
- [ ] Health check: meaningful `test`, reasonable `interval`/`timeout`
- [ ] Networks: explicit connection to external networks (`prometheus`, `traefik`)
- [ ] Shutdown: `stop_signal: SIGTERM`, `stop_grace_period` for long-running services
- [ ] Config: sensitive data in `env_file`, not in code

---

## Links

- 🐳 [Docker Compose Reference](https://docs.docker.com/compose/compose-file/)
- 🔐 [Docker Security Best Practices](https://docs.docker.com/engine/security/)
- 📊 [Healthcheck Documentation](https://docs.docker.com/engine/reference/builder/#healthcheck)
- 🧰 [itzg Docker Images](https://github.com/itzg/docker-minecraft-server) - examples of well-built images
- 🐙 [Stack Repositories](https://github.com/potatoenergy)
