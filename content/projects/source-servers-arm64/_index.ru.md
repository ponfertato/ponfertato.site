---
title: "Source-серверы на ARM64: Docker-образы для игр"
description: "Оптимизированные контейнеры для Garry's Mod, TF2, CSS, L4D на ARM-платформах. box86, авто-обновления, health checks."
tags: ["docker", "arm64", "source-engine", "gmod", "tf2", "css", "game-servers"]
---

Серия Docker-образов для запуска серверов на базе Source Engine на архитектурах ARM64 (Orange Pi, Raspberry Pi 4/5, AWS Graviton).

| Репозиторий                                                          | Игра                    | Теги     | Размер образа |
| -------------------------------------------------------------------- | ----------------------- | -------- | ------------- |
| [garrysmod-docker](https://github.com/potatoenergy/garrysmod-docker) | Garry's Mod             | `latest` | ~2.1 GB       |
| [tf2-docker](https://github.com/potatoenergy/tf2-docker)             | Team Fortress 2         | `latest` | ~1.8 GB       |
| [css-docker](https://github.com/potatoenergy/css-docker)             | Counter-Strike: Source  | `latest` | ~1.6 GB       |
| [l4d2-docker](https://github.com/potatoenergy/l4d2-docker)           | Left 4 Dead 2           | `latest` | ~2.3 GB       |
| [hl2dm-docker](https://github.com/potatoenergy/hl2dm-docker)         | Half-Life 2: Deathmatch | `latest` | ~1.5 GB       |
| [dods-docker](https://github.com/potatoenergy/dods-docker)           | Day of Defeat: Source   | `latest` | ~1.6 GB       |
| [l4d-docker](https://github.com/potatoenergy/l4d-docker)             | Left 4 Dead             | `latest` | ~2.0 GB       |

Все образы используют единую архитектуру сборки и конфигурации.

---

## Архитектура: запуск x86-бинарников на ARM64

Source Dedicated Server (srcds) не имеет нативной сборки под ARM64. Решение — эмуляция через `box86`.

### Ключевые компоненты Dockerfile

```dockerfile
FROM sonroyaalmerol/steamcmd-arm64:root

# Установка box86 для эмуляции x86
RUN apt-get update && apt-get install -y \
    box86 \
    lib32gcc-s1 \
    lib32stdc++6 \
    && rm -rf /var/lib/apt/lists/*

# Копирование скриптов запуска
COPY etc/entry.sh /home/steam/entry.sh
COPY etc/srcds_run-arm64 /home/steam/srcds_run-arm64
RUN chmod +x /home/steam/entry.sh /home/steam/srcds_run-arm64

# Health check для мониторинга доступности
HEALTHCHECK --interval=30s --timeout=10s --start-period=120s --retries=3 \
  CMD /home/steam/healthcheck.sh || exit 1

ENTRYPOINT ["/home/steam/entry.sh"]
```

### Почему box86, а не qemu

| Критерий            | box86                        | qemu-user-static            |
| ------------------- | ---------------------------- | --------------------------- |
| Производительность  | ~70-85% от нативной          | ~30-50% от нативной         |
| Потребление памяти  | Низкое                       | Высокое                     |
| Поддержка библиотек | Хорошая для glibc-приложений | Универсальная, но медленнее |
| Размер образа       | ~150 MB доп.                 | ~300 MB доп.                |

---

## Конфигурация через переменные окружения

Все параметры сервера вынесены в environment variables для гибкости деплоя.

### Общие переменные

| Переменная       | Описание                        | Пример                     |
| ---------------- | ------------------------------- | -------------------------- |
| `*_PORT`         | Порт игры (TCP/UDP)             | `27015`                    |
| `*_CLIENTPORT`   | Порт клиентов (UDP)             | `27005`                    |
| `*_SOURCETVPORT` | Порт SourceTV                   | `27020`                    |
| `*_MAXPLAYERS`   | Максимум игроков                | `12`                       |
| `*_MAP`          | Стартовая карта                 | `gm_construct`, `de_dust2` |
| `*_TICKRATE`     | Частота тиков сервера           | `66`                       |
| `*_IP`           | Привязка к интерфейсу           | `""` (все интерфейсы)      |
| `*_LAN`          | Режим только для локальной сети | `0` или `1`                |
| `*_ARGS`         | Дополнительные аргументы srcds  | `+sv_cheats 1`             |
| `*_STEAMTOKEN`   | Steam Game Server Account Token | `A1B2C3D4E5F6...`          |

### Пример: Garry's Mod

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

### Пример: Counter-Strike: Source

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

## Авто-обновления и health checks

### Авто-обновление при старте

`entry.sh` выполняет `steamcmd +app_update` при каждом запуске контейнера:

```bash
#!/bin/bash
mkdir -p "${STEAMAPPDIR}"
bash "${STEAMCMDDIR}/steamcmd.sh" +force_install_dir "${STEAMAPPDIR}" \
  +login anonymous +app_update "${STEAMAPPID}" +quit
cd "${STEAMAPPDIR}"
# Запуск srcds с box86 для ARM64
```

**Преимущества**:

- ✅ Сервер всегда на актуальной версии
- ✅ Не требует отдельного cron-задания
- ✅ Работает при рестарте контейнера

### Health check

Каждый образ включает проверку доступности:

```bash
# healthcheck.sh
#!/bin/bash
# Проверка, слушает ли srcds порт
if netstat -tuln | grep -q ":${GAME_PORT} "; then
  exit 0
else
  exit 1
fi
```

**Просмотр статуса**:

```bash
docker ps --format "table {{.Names}}\t{{.Status}}"
# Пример вывода:
# NAMES           STATUS
# gmod-server     Up 2 hours (healthy)
```

---

## Оптимизация для ARM-платформ

### Память и CPU

Source-серверы чувствительны к ресурсам. Рекомендации:

| Платформа            | Рекомендации           | Макс. игроков |
| -------------------- | ---------------------- | ------------- |
| Raspberry Pi 4 (4GB) | `--cpus=2 --memory=2g` | 8-12          |
| Orange Pi 5 (8GB)    | `--cpus=4 --memory=4g` | 16-24         |
| AWS Graviton2        | `--cpus=4 --memory=8g` | 32+           |

### Дисковая подсистема

Используйте `tmpfs` для временных файлов и кэша:

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

## Деплой через Docker Compose

### Пример: несколько серверов на одном хосте

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

### Запуск

```bash
# Загрузить переменные окружения
export $(cat .env | xargs)

# Запустить все сервисы
docker compose up -d

# Проверить статус
docker compose ps
docker compose logs -f gmod
```

---

## Мониторинг и логирование

### Логи в реальном времени

```bash
# Просмотр логов конкретного сервера
docker logs -f gmod-server --tail 100

# Фильтрация по уровню (srcds выводит в stdout)
docker logs gmod-server 2>&1 | grep -E "L [0-9/]+|Error|Warning"
```

### Интеграция с Prometheus (опционально)

Добавьте `prometheus-node-exporter` для сбора метрик хоста:

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

## Устранение неполадок

| Проблема                     | Диагностика                                 | Решение                                                         |
| ---------------------------- | ------------------------------------------- | --------------------------------------------------------------- |
| Сервер не запускается        | `docker logs <container> \| grep -i error`  | Проверить `*_STEAMTOKEN`, наличие места на диске                |
| Низкий FPS на сервере        | `docker stats <container>`                  | Уменьшить `*_MAXPLAYERS`, добавить `--cpus`                     |
| Игроки не могут подключиться | `netstat -tuln \| grep 27015`               | Проверить firewall: `ufw allow 27015:27020/udp`                 |
| box86 ошибка при старте      | `docker exec <container> box86 --version`   | Переустановить образ: `docker pull ponfertato/garrysmod:latest` |
| Health check fails           | `docker inspect <container> \| grep Health` | Увеличить `--start-period` в HEALTHCHECK                        |

---

## Лицензия и вклад

Все образы распространяются под лицензией **MIT**.

**Вклад**: принимаются через Pull Request. Перед отправкой:

- Протестируйте сборку на целевой архитектуре
- Обновите README с описанием изменений
- Убедитесь, что health check проходит

---

## Ссылки

- 🐙 [Репозитории](https://github.com/potatoenergy)
- 🐳 [Docker Hub](https://hub.docker.com/u/ponfertato)
- 📦 [box86 Documentation](https://github.com/ptitSeb/box86)
- 🔧 [SteamCMD Guide](https://developer.valvesoftware.com/wiki/SteamCMD)
- 🎮 [Source Dedicated Server](https://developer.valvesoftware.com/wiki/Dedicated_Server)
