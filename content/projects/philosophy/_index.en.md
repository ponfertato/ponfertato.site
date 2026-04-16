---
title: "Many Projects. One Structure. No Chaos."
description: "How to manage dozens of services without pain: unified approach to configuration, deployment, and documentation."
tags: ["philosophy", "infrastructure", "docker", "git", "workflow"]
---

When you have more than ten projects, questions arise:

- Where is the config for this service?
- Which environment variable to set on deploy?
- Why does one container consume all memory while another stays silent on crash?
- How to quickly deploy all this on a new machine?

Answer: **unified structure, unified rules, unified approach**.

---

## 🗂 Unified Repository Structure

Every active project follows one template:

```
project-name/
├── compose.yaml          # Unified format: services, networks, volumes, limits
├── stack.env             # Environment variables (in .gitignore)
├── README.md             # Brief: what it does, how to run, which ports
├── .github/
│   └── workflows/        # CI: build, test, deploy (if needed)
└── (optional)
    ├── Dockerfile        # If custom image needed
    ├── config/           # App configs (not secrets)
    └── scripts/          # Utilities: backup, migration, check
```

**Why it works**:

- ✅ New project - copy template, change name, configure
- ✅ Familiar format - no guessing where to find config
- ✅ Documentation next to code - always up to date

---

## ⚙️ Unified Rules for All Containers

Doesn't matter if you're running a Minecraft server or Grafana - these parameters are everywhere:

### Resource Limits (Preventing "Starvation")

```yaml
services:
  my-service:
    cpus: "1.0"
    mem_limit: 1g
    pids_limit: 100
```

### Security by Default

```yaml
security_opt:
  - no-new-privileges:true
  - seccomp:unconfined # only if truly needed
```

### Logging with Rotation

```yaml
logging:
  driver: json-file
  options:
    max-file: "3"
    max-size: 10m
```

### Health Check for Observability

```yaml
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost:80/health"]
  interval: 30s
  timeout: 10s
  retries: 3
  start_period: 40s
```

### Graceful Shutdown

```yaml
stop_signal: SIGTERM
stop_grace_period: 30s
```

**Result**: any service behaves predictably. Crashed - restarted. Hit limit - immediately visible. Logs don't fill disk.

---

## 🔐 Configuration via `.env`, Not Hardcoding

```yaml
# compose.yaml
services:
  app:
    env_file: [stack.env]
    image: ${REGISTRY:-ghcr.io}/potatoenergy/app:${TAG:-latest}
```

```dotenv
# stack.env (in .gitignore)
REGISTRY=ghcr.io/potatoenergy
TAG=2026.04.15
APP_SECRET=${APP_SECRET}  # injected from host environment
```

**Benefits**:

- ✅ Secrets not in repository
- ✅ One file - easy to rotate, easy to audit
- ✅ Deploy to dev/stage/prod - different `.env`, same `compose.yaml`

---

## 🌐 Networks: Explicit Contracts Instead of `default`

```yaml
networks:
  prometheus:
    external: true
    name: prometheus
  traefik:
    external: true
    name: traefik

services:
  grafana:
    networks: [prometheus, traefik] # sees metrics + web traffic
  loki:
    networks: [prometheus] # metrics only, no public access
```

**Why**:

- Services discover each other by name (`http://loki:3100`)
- Public access only through services explicitly connected to `traefik`
- Easy to add new service to monitoring: connect to `prometheus` - it's already visible

---

## 🔄 Deployment: One Command for Everything

```bash
# Clone stack
git clone https://github.com/potatoenergy/grafana-stack.git
cd grafana-stack

# Fill variables
cp stack.env.example stack.env
# (edit stack.env)

# Start
docker compose up -d

# Check status
docker compose ps
```

**For updates**:

```bash
git pull
docker compose pull
docker compose up -d
```

**For rollback**:

```bash
git checkout <previous-commit>
docker compose up -d
```

---

## 📊 What This Delivers in Practice

| Metric                  | Before (scattered configs)           | After (unified approach)            |
| ----------------------- | ------------------------------------ | ----------------------------------- |
| New service deploy time | 30–60 min (searching configs, setup) | 5–10 min (clone + `.env` + `up -d`) |
| Problem diagnosis       | Dig through 5+ repos, guess          | One `compose.yaml`, standard logs   |
| Scaling                 | Manual edit of each service          | Template + variables                |
| Onboarding new member   | Week to learn "how we do it"         | One README, one approach            |

---

## 🎯 Philosophy in Three Principles

1. **Reproducibility**  
   Config that works for me should work for you. No "it works on my machine".

2. **Transparency**  
   Code is open, documentation is nearby, variables are extracted. No magic - just clear steps.

3. **Pragmatism**  
   Tools are chosen by task, not by hype. If it works - don't touch. If it doesn't - fix or replace.

---

## 🗂 Archive Is Not Failure, But a Step

In `.old/` live projects that:

- ✅ Were useful experiments but became architecturally outdated
- ✅ Were replaced by a more advanced solution
- ✅ Were forks for hypothesis testing

**Archiving ≠ deletion**. Code remains accessible, commit history is preserved, license still applies. The only difference: "not actively maintained".

> Don't fear archiving. Fear standing still.

---

## 🔮 What's Ahead

- **Template unification**: one `compose.template.yaml` for all new projects
- **Auto-generated README**: from `compose.yaml` + `.env.example`
- **CI audit**: automatic config checks for security and completeness
- **"Potato Index"**: aggregator of self-hosting solutions with reproducibility checks

---

## 🗂 Checklist for a New Project

- [ ] Created repo from template: `compose.yaml`, `stack.env`, `README.md`
- [ ] Set resource limits: `cpus`, `mem_limit`, `pids_limit`
- [ ] Added `security_opt: [no-new-privileges:true]`
- [ ] Configured log rotation: `max-file`, `max-size`
- [ ] Defined health check with reasonable intervals
- [ ] Extracted secrets to `.env`, added it to `.gitignore`
- [ ] Connected service to required external networks (`prometheus`, `traefik`)
- [ ] Wrote README: what it does, how to run, which ports

---

## Links

- 🐙 [Active Repositories](https://github.com/orgs/potatoenergy/repositories)
- 🗄️ [Archived Projects](https://github.com/orgs/potatoenergy/repositories?q=archived%3Atrue)
- 🐳 [Docker Compose Reference](https://docs.docker.com/compose/compose-file/)
- 🔐 [Docker Security Best Practices](https://docs.docker.com/engine/security/)
