# Pirates, Vikings, and Knights II Server (Docker)

This repository provides a Dockerized **Pirates, Vikings, and Knights II** game server suitable for running a PVKII server in a clean, reproducible way.  
The image is designed for **headless operation**, automatically downloads and updates the game server via SteamCMD (anonymous login, app `17575`), supports file overlaying using OverlayFS, and includes optional SourceMod/MetaMod support.

---

## Features

- Pirates, Vikings, and Knights II game server support (Booty, Trinket Wars, Territory, Last Team Standing, Team Deathmatch)
- Runs the 64-bit Source dedicated server (`srcds_linux64`), the game's default
- Automatically downloads and updates the game server via SteamCMD, including the first install on a fresh SteamCMD
- Optional SourceMod/MetaMod installation (64-bit PVKII builds)
- Supports file overlaying using OverlayFS (no file copying required)
- Updates the game server on every start, with optional file validation

## Docker Compose Example

```yaml
services:
  pvkii-server:
    image: lancommander/piratesvikingsandknightsii:latest
    container_name: my-pvkii-server

    # Expose game server ports
    ports:
      - "27015:27015/udp"  # Game server port
      - "27015:27015/tcp"  # RCON
      - "27020:27020/udp"  # SourceTV port (optional)

    # Bind mounts
    volumes:
      - ./config:/config      # Game server configuration and overlay directory

    environment:
      START_ARGS: "-game pvkii -console +maxplayers 24 +map bt_island"
      INSTALL_SOURCEMOD: "true"              # SourceMod/MetaMod (default: true)

    user: root

    # For overlayfs support
    cap_add:
      - SYS_ADMIN
    security_opt:
      - apparmor:unconfined

    restart: unless-stopped
```

---

## Directory Layout (Host)

```text
.
└── config/
    ├── Server/            # Game files downloaded by SteamCMD (auto-created, about 4.2 GB)
    │   └── pvkii/         # Pirates, Vikings, and Knights II game directory
    ├── Overlay/           # Files to overlay on game directory (optional)
    │   └── pvkii/         # Pirates, Vikings, and Knights II overlay directory
    │       ├── addons/    # SourceMod/MetaMod installation location
    │       │   ├── sourcemod/
    │       │   │   ├── plugins/     # Place your .smx plugin files here
    │       │   │   ├── configs/     # SourceMod configuration files
    │       │   │   └── ...
    │       │   └── metamod/
    │       ├── maps/      # Custom maps
    │       ├── cfg/       # Server configuration files (server.cfg, etc.)
    │       ├── rulesets/  # Custom rulesets
    │       └── ...        # Any other files you want to overlay
    ├── Merged/            # Overlayfs merged view (auto-created, if overlayfs enabled)
    ├── .overlay-work/     # Overlayfs work directory (auto-created)
    ├── Scripts/
    │   └── Hooks/         # Script files in this directory get automatically executed if registered to a hook
    └── ...                # Your game server configuration files
```

The `config` directory **must be writable** by Docker. The `Server` directory is automatically created and populated by SteamCMD on first startup.

---

## Configuration

### Custom Scripts

You can create custom scripts in the `/config/Scripts/Hooks/{HookName}` directory to execute at various points in the container's startup. This is useful for:
- Installing additional dependencies
- Setting up environment variables
- Downloading additional files
- Modifying configuration before the game server starts

All scripts should be PowerShell scripts.

### File Overlaying

The overlay directory (`/config/Overlay`) allows you to overlay files on top of the game directory without copying files. This is useful for:
- Replacing game files (maps, scripts, assets)
- Adding custom content
- Modifying game files without touching the base installation

**How it works:**
- Files in `/config/Overlay` will appear in the merged view at `/config/Merged`
- If a file exists in both `/config/Server` and `/config/Overlay`, the version in `/config/Overlay` takes precedence
- The game server runs from the merged directory

**Requirements:**
- Container must run with `--cap-add SYS_ADMIN` or `--privileged` flag

**Example overlay structure:**
```text
/config/Overlay/
└── pvkii/
    ├── cfg/
    │   └── server.cfg
    ├── maps/
    │   └── bt_custom.bsp
    └── mapcycle.txt
```

### Game modes and maps

The map prefix selects the game mode: `bt_` Booty, `tw_` Trinket Wars, `te_` Territory, `lts_` Last Team Standing and `dm_` Team Deathmatch. The stock rotation is `pvkii/mapcycle.txt`; overlay your own `pvkii/mapcycle.txt` to change it. Rulesets in `pvkii/rulesets/` (`classic` by default, `competitive`, `deathmatch` for Team Deathmatch on any map, `randomizer`, or your own) are loaded with `mp_ruleset <name>`. See [Documentation/Docker.md](Documentation/Docker.md) for the full map list.

---

## SourceMod Installation

This image includes support for automatically installing SourceMod and MetaMod:Source. SourceMod is a powerful administration plugin framework for Source engine games.

### Enabling SourceMod

SourceMod installation is enabled by default (`INSTALL_SOURCEMOD="true"`). Set it to `"false"` to skip it:

```yaml
environment:
  INSTALL_SOURCEMOD: "false"
```

When enabled, the container will automatically:
- Install MetaMod:Source (required dependency for SourceMod)
- Install SourceMod
- Place all files in the overlay directory (`/config/Overlay/pvkii/addons/`)

The server is 64-bit, so the image pins MetaMod:Source `1.12.0-git1229` and SourceMod `1.12.0-git7259`, the first builds that ship 64-bit PVKII binaries.

### Installing Plugins

Once SourceMod is installed, you can add custom plugins by placing `.smx` files in:

```
/config/Overlay/pvkii/addons/sourcemod/plugins/
```

**Example:**
```bash
# Place your plugin file
cp my_plugin.smx ./config/Overlay/pvkii/addons/sourcemod/plugins/

# Restart the container to load the plugin
docker restart my-pvkii-server
```

### SourceMod Configuration

SourceMod configuration files should be placed in:

```
/config/Overlay/pvkii/addons/sourcemod/configs/
```

Common configuration files:
- `admins_simple.ini` - Admin user configuration
- `core.cfg` - Core SourceMod settings
- `database.cfg` - Database configuration (if using database features)

## Match Statistics

The server can report the matches played on it to LANCommander, with the
[LANCommander match statistics plugin](https://github.com/LANCommander/LANCommander.MatchStatistics.SourceMod).
The plugin is built into the image and needs SourceMod (`INSTALL_SOURCEMOD`). It is turned on when
`MATCH_STATS_SERVER_URL` and `MATCH_STATS_TOKEN` are set, and is then copied into `pvkii/addons/sourcemod/` on every
start. The core plugin loads the Pirates, Vikings and Knights II module itself. Turned off, the plugin's files are removed again.

| Variable | Description | Default |
|----------|-------------|---------|
| `MATCH_STATS_ENABLED` | `auto`: on when `MATCH_STATS_SERVER_URL` and `MATCH_STATS_TOKEN` are set. `true` or `false` | `auto` |
| `MATCH_STATS_SERVER_URL` | The LANCommander server, e.g. `http://lancommander.lan:1337`. Only `http://` works on Linux | *(empty)* |
| `MATCH_STATS_TOKEN` | A LANCommander access token with the **Report matches** scope (Profile → Access Tokens). It isn't written to any file | *(empty)* |
| `MATCH_STATS_GAME_ID` | The game's id in LANCommander | *(empty)* |
| `MATCH_STATS_SERVER_ID` | This server's id in LANCommander, if LANCommander manages it | *(empty)* |
| `MATCH_STATS_REPORTER_NAME` | How the server is named in LANCommander | *(hostname)* |
| `MATCH_STATS_MIN_PLAYERS` | Human players needed before a match is recorded | *(1)* |
| `MATCH_STATS_MODE` | The mode reported. Empty lets the game module decide | *(empty)* |
| `MATCH_STATS_RECORD_BOTS` | Record bots as players (`true`/`false`) | *(true)* |
| `MATCH_STATS_RECORD_CHAT` | Record chat messages (`true`/`false`) | *(false)* |
| `MATCH_STATS_ATTACH_DEMOS` | Upload SourceTV demos recorded with `tv_record` (`true`/`false`) | *(true)* |

The settings are written to `pvkii/cfg/sourcemod/lancommander.cfg` as `lc_*` cvars on every start. Settings left
empty aren't written, so they and the plugin's other `lc_*` cvars (see its README) can be set in
`pvkii/cfg/sourcemod/sourcemod.cfg` instead. A `lancommander.cfg` you put in `Overlay/pvkii/cfg/sourcemod/` yourself
is left alone.

`sm_lc_status` on the server console or over RCON shows the match being recorded and how delivery is going. Reports
that couldn't be delivered yet are kept in `Overlay/pvkii/addons/sourcemod/extensions/.interposer/Matches/` and sent
once LANCommander can be reached.

## Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `STEAM_APP_ID` | Steam app id of the dedicated server | `17575` |
| `STEAM_VALIDATE` | Validate the game files when SteamCMD updates on startup (`true`/`false`) | `true` |
| `START_EXE` | Server executable (relative to the game directory) | `srcds_run` |
| `START_ARGS` | Arguments used to start the game server | `"-game pvkii -console +maxplayers 24 +map bt_island"` |
| `GAME_MOD` | Game directory that SourceMod/MetaMod are installed into | `pvkii` |
| `INSTALL_SOURCEMOD` | Enable SourceMod/MetaMod installation (`true`/`false`) | `true` |
| `SOURCEMOD_MAJOR_VERSION` | SourceMod release branch | `1.12` |
| `SOURCEMOD_VERSION` | SourceMod version to install | `1.12.0-git7259` |
| `SOURCEMOD_URL` | Override the SourceMod download URL | *(unset)* |
| `METAMOD_MAJOR_VERSION` | MetaMod:Source release branch | `1.12` |
| `METAMOD_VERSION` | MetaMod version to install | `1.12.0-git1229` |
| `METAMOD_URL` | Override the MetaMod download URL | *(unset)* |
| `HTTP_FILESERVER_ENABLED` | Serve custom content over HTTP (FastDL) on port 80 | `true` |
| `HTTP_FILESERVER_WEB_ROOT` | Directory served by the HTTP file server | `/config/Merged/pvkii` |
| `HTTP_FILESERVER_FILE_PATTERN` | Regex of request paths the file server allows | maps, materials, models and sound files |

### `STEAM_VALIDATE`

SteamCMD updates the game server on every container start. With `STEAM_VALIDATE` set to `true` (the default) it also checks every game file, which takes longer; set it to `false` for faster restarts.

### `START_ARGS`

Arguments passed to the game server executable at startup. Default example:

```bash
START_ARGS="-game pvkii -console +maxplayers 24 +map bt_island"
```

You can customize this with additional server parameters:

```bash
START_ARGS="-game pvkii -console +maxplayers 32 +map te_tortuga +sv_lan 1 +rcon_password mypassword"
```

---

## Running the Server

### Basic run (with automatic updates)

```bash
mkdir -p config

docker run --rm -it \
  --user root \
  --cap-add SYS_ADMIN \
  --security-opt apparmor:unconfined \
  -p 27015:27015/udp \
  -p 27015:27015/tcp \
  -v "$(pwd)/config:/config" \
  -e START_ARGS="-game pvkii -console +maxplayers 24 +map bt_island" \
  lancommander/piratesvikingsandknightsii:latest
```

### Without SourceMod

```bash
docker run --rm -it \
  --user root \
  --cap-add SYS_ADMIN \
  --security-opt apparmor:unconfined \
  -p 27015:27015/udp \
  -v "$(pwd)/config:/config" \
  -e INSTALL_SOURCEMOD="false" \
  lancommander/piratesvikingsandknightsii:latest
```

### Skip file validation on startup

```bash
docker run --rm -it \
  --user root \
  --cap-add SYS_ADMIN \
  --security-opt apparmor:unconfined \
  -p 27015:27015/udp \
  -v "$(pwd)/config:/config" \
  -e STEAM_VALIDATE="false" \
  lancommander/piratesvikingsandknightsii:latest
```

---

## OverlayFS Details

The container uses Linux OverlayFS to merge the server directory (`/config/Server`) with the overlay directory (`/config/Overlay`) into a merged view (`/config/Merged`). This allows you to:

1. **Replace files** without modifying the base game installation
2. **Add files** that don't exist in the base game
3. **Avoid copying** large files - OverlayFS is a union filesystem

**Technical details:**
- **Lower layer**: `/config/Server` (base game files from SteamCMD)
- **Upper layer**: `/config/Overlay` (your custom files)
- **Merged view**: `/config/Merged` (where the game server runs from)
- **Work directory**: `/config/.overlay-work` (required by OverlayFS)

If OverlayFS cannot be mounted (e.g., missing privileges), the container will fall back to using `/config/Server` directly and log a warning.

---

## Troubleshooting

### OverlayFS not working

If you see warnings about OverlayFS, ensure your container has the required privileges:

```bash
# Option 1: Add SYS_ADMIN capability (recommended)
docker run --cap-add SYS_ADMIN --security-opt apparmor:unconfined ...

# Option 2: Use privileged mode (less secure)
docker run --privileged ...
```

### Game server not starting

1. Verify `START_ARGS` contains the correct arguments for Pirates, Vikings, and Knights II
2. Check container logs: `docker logs <container-name>`
3. Ensure the server directory was downloaded: check `/config/Server/pvkii` in the container
4. Verify the server configuration files are correct (e.g., `server.cfg`)

### SourceMod not loading

1. Ensure `INSTALL_SOURCEMOD` is set to `"true"` (not just `true` without quotes in YAML)
2. Check that SourceMod was installed: verify `/config/Overlay/pvkii/addons/sourcemod` exists
3. Check the server console for `[META] Loaded 1 plugin.`
4. Verify MetaMod is installed: check for `/config/Overlay/pvkii/addons/metamod_x64.vdf`
5. Don't set `METAMOD_VERSION`/`SOURCEMOD_VERSION` older than `1.12.0-git1229`/`1.12.0-git7259`: earlier builds have no 64-bit PVKII binaries, and the server then starts without any plugins
6. Ensure plugins are compiled `.smx` files (not `.sp` source files)

### `ERROR! Failed to install app '17575' (Missing configuration)`

A fresh SteamCMD fails the first install with this error until it has refreshed the app info. The `PreSteamInstallGame/10-BootstrapInstall.ps1` hook runs `app_info_update 1` and the install in one SteamCMD session when `Server/steamapps/appmanifest_17575.acf` doesn't exist yet. If the first install was interrupted, delete that file and restart the container.

### `Out of memory or address space` on map load

This happens with the 32-bit server (`-binary ./srcds_linux32`). Use the default 64-bit server.

### Permission errors

If you encounter permission errors:

1. Ensure mounted volumes are writable
2. Check file ownership in the container
3. Review logs for specific permission error messages

---

## License

SteamCMD and game servers are distributed under their respective licenses.
This repository contains only Docker build logic and helper scripts licensed under MIT.
