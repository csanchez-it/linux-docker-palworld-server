# Palworld Dedicated Server on Docker

## Overview

This repository documents the deployment, configuration, and administration of a self-hosted **Palworld dedicated server** ("Servidor de prueba"), run on a mini PC using **Ubuntu Server 24.04 LTS** and **Docker Compose**.

The project was undertaken primarily as a hands-on exercise to build practical experience with **Linux server administration** and **containerization with Docker** — the game server itself was the vehicle for learning, not the end goal. Over roughly three months of operation, the project involved real troubleshooting of environment configuration, Linux file permissions, container lifecycle management, and REST API integration.

## Tech Stack

- **OS:** Ubuntu Server 24.04 LTS
- **Containerization:** Docker, Docker Compose
- **Shell / Scripting:** Bash
- **Integration:** REST API (Basic Auth), PowerShell, curl

## Timeline

| Date | Milestone |
|---|---|
| Jun 1 | Initial deployment — Palworld Beta 0.7.3 |
| Jul 9 | Updated to official release v1.0 |
| Jul 30 | Minor update and configuration tuning — v1.0.2 |
| Sep 1 | Server decommissioned after ~3 months of continuous operation |

## Architecture

The server ran as a single Docker container on Ubuntu Server, built on the [`thijsvanloef/palworld-server-docker`](https://github.com/thijsvanloef/palworld-server-docker) image. Game data and save files were persisted via a mounted volume (the exact original path was not preserved — the value in `docker-compose.yml` is a placeholder, not the confirmed original), and configuration was managed entirely through environment variables loaded from a `.env` file — avoiding the need to modify game configuration files directly. SteamCMD (bundled inside the image) handled dependency downloads and version updates.

Three ports were exposed:
- **8211/udp** — main game port
- **27015/udp** — Steam query port (server browser visibility)
- **8212/tcp** — REST API (player queries, admin actions)

See [`docker-compose.yml`](docker-compose.yml) for the full service definition.

## Technical Problems Solved

### 1. Environment file conflict (`.env` vs `env`)

During a routine configuration update, the container stopped picking up changes (player count, server name). Investigation revealed **two coexisting environment files**: an outdated `.env` (274 bytes) and a newer `env` (1667 bytes). Docker was silently defaulting to the hidden, outdated file.

**Resolution:**
```bash
sudo rm .env
sudo mv env .env
```
Residual cached settings in `PalWorldSettings.ini` were also purged to guarantee a clean state, followed by a forced container recreation:
```bash
docker compose down && docker compose up -d --force-recreate
```

### 2. Linux file permission conflict (root vs. standard user)

The outdated `.env` file was owned by `root` and could not be removed by the standard operating user (`usuario`). This required privilege escalation via `sudo` to safely remove and replace the file — a practical exercise in Linux file ownership and permission management.

### 3. Log auditing — distinguishing false positives from real failures

After reviewing container logs (`docker logs -f palworld-server`) following a SteamCMD-driven update, several warnings referencing `SteamAPI_Init` and `IPC` latency appeared. Through log analysis and cross-referencing with the game engine's known behavior (Unreal Engine), these were determined to be **expected, non-critical warnings** rather than service failures — avoiding unnecessary intervention on a healthy server.

## Server Configuration

Server behavior was tuned entirely through environment variables (see [`.env.example`](.env.example)), including player capacity, egg incubation time, hunger/stamina decay rates, and item durability. This approach kept all configuration centralized, version-controllable, and free of manual edits to game config files.

## REST API Integration

The server exposed Palworld's REST API on port `8212`, used to query connected players remotely with Basic Authentication (Base64-encoded credentials).

**PowerShell:**
```powershell
$headers = @{ Authorization = "Basic <base64-encoded-credentials>" }
Invoke-RestMethod -Uri "http://<server-ip>:8212/v1/api/players" -Headers $headers
```

**Linux (curl):**
```bash
curl -u admin:<password> http://<server-ip>:8212/v1/api/players
```

## In-Game Engine Limitations

While tuning building mechanics, an in-game "insufficient supports" error was reported when constructing roofs and floors. After investigation, this was determined to be a **hard-coded engine constraint** (structures require a physical support connection every 2 tiles) rather than a configurable server parameter — resolved with an in-game architectural workaround instead of a server-side fix.

## Lessons Learned / Planned Improvements

The following items were researched and planned during the project's lifetime; some were not completed before the server was decommissioned:

- [ ] Automated backups via CRON *(researched, not implemented)*
- [ ] Adjust resource respawn multiplier (`CollectionObjectRespawnSpeedRate`) *(researched, not implemented)*
- [ ] Fine-tune stamina drain rate *(researched, not implemented)*
- [ ] One-click Windows script for querying the player API *(planned, not implemented)*
- [x] Resolve `.env` / `env` file conflict
- [x] Resolve root-owned file permission issue
- [x] Establish standard Docker Compose operational commands (`down`, `up -d`, `stop`, `start`, `restart`)
- [x] Validate REST API integration (PowerShell and curl)

## Disclaimer

All identifying details (server name, IP addresses, credentials, and usernames) have been removed or replaced with generic placeholders. This repository is shared for educational and portfolio purposes, documenting real Linux and Docker administration work performed on a personal test environment.
