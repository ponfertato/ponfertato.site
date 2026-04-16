---
title: "Source Servers on ARM64: Docker Images for Games"
description: "Optimized containers for Garry's Mod, TF2, CSS, L4D on ARM platforms. box86, auto-updates, health checks."
tags: ["docker", "arm64", "source-engine", "gmod", "tf2", "css", "game-servers"]
---

A series of Docker images for running Source Engine servers on ARM64 architectures (Orange Pi, Raspberry Pi 4/5, AWS Graviton).

| Repository                                                           | Game                    | Tags     | Image Size |
| -------------------------------------------------------------------- | ----------------------- | -------- | ---------- |
| [garrysmod-docker](https://github.com/potatoenergy/garrysmod-docker) | Garry's Mod             | `latest` | ~2.1 GB    |
| [tf2-docker](https://github.com/potatoenergy/tf2-docker)             | Team Fortress 2         | `latest` | ~1.8 GB    |
| [css-docker](https://github.com/potatoenergy/css-docker)             | Counter-Strike: Source  | `latest` | ~1.6 GB    |
| [l4d2-docker](https://github.com/potatoenergy/l4d2-docker)           | Left 4 Dead 2           | `latest` | ~2.3 GB    |
| [hl2dm-docker](https://github.com/potatoenergy/hl2dm-docker)         | Half-Life 2: Deathmatch | `latest` | ~1.5 GB    |
| [dods-docker](https://github.com/potatoenergy/dods-docker)           | Day of Defeat: Source   | `latest` | ~1.6 GB    |
| [l4d-docker](https://github.com/potatoenergy/l4d-docker)             | Left 4 Dead             | `latest` | ~2.0 GB    |

All images share a unified build and configuration architecture.

---

## Architecture: Running x86 Binaries on ARM64

Source Dedicated Server (srcds) has no native ARM64 build. Solution: emulation via `box86`.

### Key Dockerfile Components

```dockerfile
FROM sonroyaalmerol/steamcmd-arm64:root

# Install box86 for x86 emulation
RUN apt-get update && apt-get install -y \
    box86 \
    lib32gcc-s1 \
    lib32stdc++6 \
    && rm -rf /var/lib/apt/lists/*

# Copy startup scripts
COPY etc/entry.sh /home/steam/entry.sh
COPY etc/srcds_run-arm64 /home/steam/srcds_run-arm64
RUN chmod +x /home/steam/entry.sh /home/steam/srcds_run-arm64

# Health check for availability monitoring
HEALTHCHECK --interval=30s --timeout=10s --start-period=120s --retries=3 \
  CMD /home/steam/healthcheck.sh || exit 1

ENTRYPOINT ["/home/steam/entry.sh"]
```

### Why box86 over qemu

| Criterion       | box86               | qemu-user-static      |
| --------------- | ------------------- | --------------------- |
| Performance     | ~70-85% of native   | ~30-50% of native     |
| Memory usage    | Low                 | High                  |
| Library support | Good for glibc apps | Universal, but slower |
| Image size      | ~150 MB extra       | ~300 MB extra         |

---

## Configuration via Environment Variables

All server parameters are exposed as environment variables for deployment flexibility.

### Common Variables

| Variable         | Description                     | Example                    |
| ---------------- | ------------------------------- | -------------------------- |
| `*_PORT`         | Game port (TCP/UDP)             | `27015`                    |
| `*_CLIENTPORT`   | Client port (UDP)               | `27005`                    |
| `*_SOURCETVPORT` | SourceTV port                   | `27020`                    |
| `*_MAXPLAYERS`   | Maximum players                 | `12`                       |
| `*_MAP`          | Starting map                    | `gm_construct`, `de_dust2` |
| `*_TICKRATE`     | Server tick rate                | `66`                       |
| `*_IP`           | Interface binding               | `""` (all interfaces)      |
| `*_LAN`          | LAN-only mode                   | `0` or `1`                 |
| `*_ARGS`         | Additional srcds arguments      | `+sv_cheats 1`             |
| `*_STEAMTOKEN`   | Steam Game Server Account Token | `A1B2C3D4E5F6...`          |

### Example: Garry's Mod

```bash
docker run -d --name gmod-server \
  -p 27015:27015/tcp -p 27015:27015/udp -p 27005:27005/udp \
  -e GM_MAP=gm_construct \
  -e GM_MAXPLAYERS=16 \
  -e GM_TICKRATE=66 \
  -e GM_STEAMTOKEN="${STEAM_TOKEN}" \
  -v ./gmod/addons:/home/steam/garrysmod-server/addons \
  -v ./gmod/cfg:/home/steam/garrysmod-server/cfg \
  -v ./gmod/data:/home/steam/garrysmod-server/data \
  ponfertato/garrysmod:latest
```

### Example: Counter-Strike: Source

```bash
docker run -d --name css-server \
  -p 27015:27015/tcp -p 27015:27015/udp \
  -e CSS_MAP=de_dust2 \
  -e CSS_MAXPLAYERS=12 \
  -e CSS_TICKRATE=100 \
  -e CSS_ARGS="+sv_region 3 +mp_autokick 0" \
  -v ./css/cfg:/home/steam/cstrike-server/cstrike/cfg \
  ponfertato/css:latest
```

---

## Auto-Updates and Health Checks

### Auto-update on Startup

`entry.sh` runs `steamcmd +app_update` on every container start:

```bash
#!/bin/bash
mkdir -p "${STEAMAPPDIR}"
bash "${STEAMCMDDIR}/steamcmd.sh" +force_install_dir "${STEAMAPPDIR}" \
  +login anonymous +app_update "${STEAMAPPID}" +quit
cd "${STEAMAPPDIR}"
# Launch srcds with box86 for ARM64
```

**Benefits**:

- ✅ Server always on latest version
- ✅ No separate cron job required
- ✅ Works on container restart

### Health Check

Each image includes an availability check:

```bash
# healthcheck.sh
#!/bin/bash
# Check if srcds is listening on port
if netstat -tuln | grep -q ":${GAME_PORT} "; then
  exit 0
else
  exit 1
fi
```

**Check status**:

```bash
docker ps --format "table {{.Names}}\t{{.Status}}"
# Example output:
# NAMES           STATUS
# gmod-server     Up 2 hours (healthy)
```

---

## ARM Platform Optimization

### Memory and CPU

Source servers are resource-sensitive. Recommendations:

| Platform             | Recommendations        | Max Players |
| -------------------- | ---------------------- | ----------- |
| Raspberry Pi 4 (4GB) | `--cpus=2 --memory=2g` | 8-12        |
| Orange Pi 5 (8GB)    | `--cpus=4 --memory=4g` | 16-24       |
| AWS Graviton2        | `--cpus=4 --memory=8g` | 32+         |

### Disk Subsystem

Use `tmpfs` for temporary files and cache:

```yaml
# docker-compose.yml
services:
  gmod:
    image: ponfertato/garrysmod:latest
    tmpfs:
      - /home/steam/garrysmod-server/cache:size=512m
      - /tmp:size=256m
    volumes:
      - ./persistent:/home/steam/garrysmod-server/data
```

---

## Deployment via Docker Compose

### Example: Multiple Servers on One Host

```yaml
version: "3.8"

services:
  gmod:
    image: ponfertato/garrysmod:latest
    container_name: gmod-server
    restart: unless-stopped
    ports:
      - "27015:27015/tcp"
      - "27015:27015/udp"
      - "27005:27005/udp"
    environment:
      GM_MAP: gm_construct
      GM_MAXPLAYERS: "16"
      GM_TICKRATE: "66"
      GM_STEAMTOKEN: "${GMOD_TOKEN}"
    volumes:
      - ./gmod/data:/home/steam/garrysmod-server/data
      - ./gmod/cfg:/home/steam/garrysmod-server/cfg
    tmpfs:
      - /home/steam/garrysmod-server/cache:size=512m
    healthcheck:
      test: ["CMD", "/home/steam/healthcheck.sh"]
      interval: 30s
      timeout: 10s
      retries: 3

  css:
    image: ponfertato/css:latest
    container_name: css-server
    restart: unless-stopped
    ports:
      - "27016:27015/tcp"
      - "27016:27015/udp"
      - "27006:27005/udp"
    environment:
      CSS_MAP: de_dust2
      CSS_MAXPLAYERS: "12"
      CSS_TICKRATE: "100"
      CSS_STEAMTOKEN: "${CSS_TOKEN}"
    volumes:
      - ./css/cfg:/home/steam/cstrike-server/cstrike/cfg
    depends_on:
      - gmod
```

### Running

```bash
# Load environment variables
export $(cat .env | xargs)

# Start all services
docker compose up -d

# Check status
docker compose ps
docker compose logs -f gmod
```

---

## Monitoring and Logging

### Real-time Logs

```bash
# View logs for a specific server
docker logs -f gmod-server --tail 100

# Filter by level (srcds outputs to stdout)
docker logs gmod-server 2>&1 | grep -E "L [0-9/]+|Error|Warning"
```

### Prometheus Integration (Optional)

Add `prometheus-node-exporter` for host metrics:

```yaml
# docker-compose.monitoring.yml
services:
  node-exporter:
    image: prom/node-exporter:latest
    ports:
      - "9100:9100"
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
    command:
      - "--path.procfs=/host/proc"
      - "--path.sysfs=/host/sys"
```

---

## Troubleshooting

| Issue                  | Diagnosis                                   | Solution                                                 |
| ---------------------- | ------------------------------------------- | -------------------------------------------------------- |
| Server won't start     | `docker logs <container> \| grep -i error`  | Check `*_STEAMTOKEN`, disk space                         |
| Low server FPS         | `docker stats <container>`                  | Reduce `*_MAXPLAYERS`, add `--cpus`                      |
| Players can't connect  | `netstat -tuln \| grep 27015`               | Check firewall: `ufw allow 27015:27020/udp`              |
| box86 error on startup | `docker exec <container> box86 --version`   | Re-pull image: `docker pull ponfertato/garrysmod:latest` |
| Health check fails     | `docker inspect <container> \| grep Health` | Increase `--start-period` in HEALTHCHECK                 |

---

## License and Contributions

All images are distributed under the **MIT License**.

**Contributions**: accepted via Pull Request. Before submitting:

- Test build on target architecture
- Update README with change description
- Ensure health check passes

---

## Links

- 🐙 [Repositories](https://github.com/potatoenergy)
- 🐳 [Docker Hub](https://hub.docker.com/u/ponfertato)
- 📦 [box86 Documentation](https://github.com/ptitSeb/box86)
- 🔧 [SteamCMD Guide](https://developer.valvesoftware.com/wiki/SteamCMD)
- 🎮 [Source Dedicated Server](https://developer.valvesoftware.com/wiki/Dedicated_Server)
