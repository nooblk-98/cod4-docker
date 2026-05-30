# COD4 Docker dedicated server

[![Docker Cod4](images/title.png)](https://github.com/qdm12/cod4-docker)

[![Build status](https://github.com/qdm12/cod4-docker/actions/workflows/release.yml/badge.svg)](https://github.com/qdm12/cod4-docker/actions/workflows/release.yml)
[![Docker Pulls](https://img.shields.io/docker/pulls/qmcgaw/cod4.svg)](https://hub.docker.com/r/qmcgaw/cod4)
[![GitHub release](https://img.shields.io/github/v/release/qdm12/cod4-docker)](https://github.com/qdm12/cod4-docker/releases)
[![Image size](https://img.shields.io/docker/image-size/qmcgaw/cod4/latest)](https://hub.docker.com/r/qmcgaw/cod4)

Call of Duty 4: Modern Warfare dedicated server in a ~125 MB Docker image using the [Cod4x](https://github.com/callofduty4x/CoD4x_Server) community server fork.

> **Note:** This image is also available on [GitHub Container Registry](ghcr.io/qdm12/cod4-docker).

## Requirements

- COD4 game client (version 1.8+)
- Original COD4 `main/` and `zone/` files from your client installation

## Features

- **Cod4x server** — latest Cod4x build downloaded from [cod4x.me](https://cod4x.me), auto-updates supported
- **Custom mods & maps** — drop `.ff` and `.iwd` files into the appropriate volumes
- **Built-in HTTP file server** — serves mods and usermaps on port `8000` so clients download custom content automatically
- **Non-root runtime** — container runs as an unprivileged user for better security
- **Default server config** — production-ready [`server.cfg`](server.cfg) with sensible defaults
- **Works with the Cod4x masterlist** — server is visible to the public by default
- **Multi-arch** — built via Docker Buildx; currently published for `linux/amd64`
- **Minimal size** — ~125 MB compressed

## Quick start

### 1. Prepare directories

```bash
mkdir -p main zone mods usermaps logs
```

### 2. Copy game files

From your COD4 installation directory:

```bash
cp /path/to/cod4/main/*.iwd ./main/
cp /path/to/cod4/zone/*    ./zone/
```

Optional: copy mods and custom maps

```bash
cp -r /path/to/cod4/mods/*    ./mods/
cp -r /path/to/cod4/usermaps/* ./usermaps/
```

### 3. Fix ownership

The container runs as user ID `1000` by default:

```bash
sudo chown -R 1000:1000 main zone mods usermaps logs
chmod -R 700 main zone mods usermaps
```

To use a different UID/GID, rebuild with `--build-arg UID=yourid --build-arg GID=yourgid`.

### 4. Run

```bash
docker run -d --name=cod4 \
  -p 28960:28960/tcp -p 28960:28960/udp -p 8000:8000/tcp \
  -v $(pwd)/main:/home/user/cod4/main \
  -v $(pwd)/zone:/home/user/cod4/zone \
  -v $(pwd)/mods:/home/user/cod4/mods \
  -v $(pwd)/usermaps:/home/user/cod4/usermaps:ro \
  -v $(pwd)/logs:/home/user/.callofduty4 \
  qmcgaw/cod4 +map mp_shipment
```

Or with Docker Compose:

```yaml
# docker-compose.yml
services:
  cod4:
    image: qmcgaw/cod4
    container_name: cod4
    ports:
      - 28960:28960/udp
      - 28960:28960/tcp
      - 8000:8000/tcp
    volumes:
      - ./main:/home/user/cod4/main
      - ./zone:/home/user/cod4/zone
      - ./mods:/home/user/cod4/mods
      - ./usermaps:/home/user/cod4/usermaps:ro
      - ./logs:/home/user/.callofduty4
    environment:
      - HTTP_SERVER=on
      - ROOT_URL=/
    command: +set dedicated 2 +set sv_cheats 1 +set sv_maxclients 64 +set ui_maxclients 64 +exec server.cfg +map_rotate
    restart: always
```

```bash
docker compose up -d
```

> The default command is `+set dedicated 2 +set sv_cheats "1" +set sv_maxclients "64" +exec server.cfg +map_rotate`.

## HTTP file server

The built-in HTTP server on port `8000` serves files from the `mods/` and `usermaps/` directories. Clients download custom content automatically when connecting.

- **Disable:** `-e HTTP_SERVER=off`
- **Change port:** `-p 9000:8000/tcp`
- **Change root URL (e.g. behind a reverse proxy):** `-e ROOT_URL=/cod4`
- **Security:** only `.ff` and `.iwd` file extensions are served; everything else returns HTTP 403

Configure `sv_wwwBaseURL` in `server.cfg` to point clients to your server:

```c
set sv_allowdownload "1"
set sv_wwwDownload "1"
set sv_wwwBaseURL "http://youraddress:8000"
set sv_wwwDlDisconnected "0"
```

## Mods

Place your mod in `./mods/mymod/` and override the container command:

```bash
+set dedicated 2 +set sv_cheats "1" +set sv_maxclients "64" +set fs_game mods/mymod +exec server.cfg +map_rotate
```

## Write-protected arguments

These parameters **cannot** be set in `server.cfg` — they must be passed in the command line:

| Argument | Description |
|---|---|
| `+set dedicated 2` | 2 = public, 1 = LAN, 0 = local |
| `+set sv_cheats "1"` | Enable cheats |
| `+set sv_maxclients "64"` | Max player count |
| `+exec server.cfg` | Load config file |
| `+set fs_game mods/mymod` | Use a custom mod |
| `+map_rotate` / `+map <name>` | Must be the **last** argument |

## Environment variables

| Variable | Default | Description |
|---|---|---|
| `HTTP_SERVER` | `on` | Enable/disable the built-in HTTP file server |
| `ROOT_URL` | `/` | Root URL prefix for the HTTP server (useful behind a reverse proxy) |

## Architecture

```
Entrypoint (Go binary)
    │
    ├─ 1. Verify file permissions (read/write/exec)
    ├─ 2. Copy default Cod4x files if missing
    ├─ 3. Parse environment config
    ├─ 4. Launch cod4x18_dedrun subprocess
    └─ 5. Start HTTP file server (if enabled)
         └─ Serves mods/*.ff and usermaps/*.iwd
```

The Go entrypoint handles graceful shutdown, log streaming, and crash recovery. The Cod4x server binary runs in the foreground.

## Build from source

```bash
docker build -t cod4 .
docker run cod4 +map mp_shipment
```

### Build arguments

| Argument | Default | Description |
|---|---|---|
| `UID` | `1000` | Container user ID |
| `GID` | `1000` | Container group ID |
| `COD4X_VERSION` | `21.2` | Cod4x server release version |

## Updating the client

1. Update your COD4 game to version 1.7
2. Download the [Cod4x client ZIP](https://cod4x.me/downloads/cod4x_client_20_1.zip)
3. Extract to your game directory
4. Run `install.cmd`
5. Version `20.1` should appear in the bottom-right corner of the multiplayer menu

## Testing

1. Launch COD4 multiplayer
2. **Join Game** → **Source** → *Favourites*
3. **New Favourite** → enter your host LAN IP
4. **Refresh** and connect

![COD4 screenshot](images/test.png)

## Use GitHub Container Registry

Pull from `ghcr.io` instead of Docker Hub:

```bash
docker pull ghcr.io/qdm12/cod4-docker:latest
```

## Acknowledgements

- [Cod4x server](https://github.com/callofduty4x/CoD4x_Server) developers
- [Cod4x.me](https://cod4x.me) community
