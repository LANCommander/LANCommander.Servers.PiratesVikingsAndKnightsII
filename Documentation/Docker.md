# Pirates, Vikings, and Knights II Docker Container

This Docker container provides a fully configured Pirates, Vikings, and Knights II game server that automatically downloads and updates the game via SteamCMD, supports file overlaying using OverlayFS, and includes optional SourceMod/MetaMod support.

## Quick Start

```yaml
services:
  pvkii-booty:
    image: lancommander/piratesvikingsandknightsii:latest
    container_name: pvkii-booty

    ports:
      - 27015:27015/udp
      - 27015:27015/tcp
      - 27080:80 # For FastDL support

    volumes:
      - "/data/Servers/Pirates, Vikings, and Knights II/Booty:/config"

    environment:
      START_ARGS: "-game pvkii -console +maxplayers 24 +map bt_island"
      INSTALL_SOURCEMOD: "true"

    user: root

    cap_add:
      - SYS_ADMIN

    security_opt:
      - apparmor:unconfined

    restart: unless-stopped
```

## Configuration Options

### Ports

The container exposes the following ports:

- **27015/udp** - Main game server port (default). Clients connect to this port to join the server, and server browser queries (A2S) are answered on it.
- **27015/tcp** - RCON (remote console).
- **27020/udp** - SourceTV port (optional). Used for game broadcasting and recording.
- **80/tcp** - HTTP file server port (optional). Used for serving game files to clients (See [FastDL](https://developer.valvesoftware.com/wiki/FastDL)).

**Port Mapping:**
In the example configuration, ports are mapped as:
- `27015:27015/udp` - Maps host port 27015 to container port 27015
- `27015:27015/tcp` - Maps host port 27015 to container port 27015 (RCON)
- `27080:80` - Maps host port 27080 to container port 80

You can customize these mappings based on your network requirements. If you're running multiple servers, use different host ports for each instance.

### Volumes

The container requires a volume mount for the `/config` directory, which stores:

- **Server/** - Game files downloaded by SteamCMD (auto-created, about 4.2 GB)
- **Overlay/** - Custom files that overlay on top of the game directory
- **Merged/** - OverlayFS merged view (auto-created)
- **Scripts/** - Custom PowerShell scripts for hooks

**Example:**
```yaml
volumes:
  - "/data/Servers/Pirates, Vikings, and Knights II/Booty:/config"
```

The host path can be:
- An absolute path (Windows: `C:\data\...`, Linux: `/data/...`)
- A relative path (e.g., `./config:/config`)
- A named volume (e.g., `pvkii-server-data:/config`)

**Important:** The mounted directory must be writable by the container.

### Environment Variables

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `STEAM_APP_ID` | Steam app id of the dedicated server | `17575` | No |
| `STEAM_VALIDATE` | Validate the game files when SteamCMD updates on startup (`true`/`false`) | `true` | No |
| `START_EXE` | Server executable (relative to the game directory) | `srcds_run` | No |
| `START_ARGS` | Arguments used to start the game server | `"-game pvkii -console +maxplayers 24 +map bt_island"` | No |
| `GAME_MOD` | Game directory that SourceMod/MetaMod are installed into | `pvkii` | No |
| `INSTALL_SOURCEMOD` | Enable SourceMod/MetaMod installation (`true`/`false`) | `true` | No |
| `SOURCEMOD_MAJOR_VERSION` | SourceMod release branch | `1.12` | No |
| `SOURCEMOD_VERSION` | SourceMod version to install | `1.12.0-git7259` | No |
| `SOURCEMOD_URL` | Override the SourceMod download URL | *(unset)* | No |
| `METAMOD_MAJOR_VERSION` | MetaMod:Source release branch | `1.12` | No |
| `METAMOD_VERSION` | MetaMod version to install | `1.12.0-git1229` | No |
| `METAMOD_URL` | Override the MetaMod download URL | *(unset)* | No |
| `HTTP_FILESERVER_ENABLED` | Serve custom content over HTTP (FastDL) on port 80 | `true` | No |
| `HTTP_FILESERVER_WEB_ROOT` | Directory served by the HTTP file server | `/config/Merged/pvkii` | No |
| `HTTP_FILESERVER_FILE_PATTERN` | Regex of request paths the file server allows | maps, materials, models and sound files | No |

#### `STEAM_VALIDATE`

SteamCMD updates the game server on every container start. With `STEAM_VALIDATE` set to `"true"` (the default) it also checks every game file, which takes longer; set it to `"false"` for faster restarts.

**Example:**
```yaml
environment:
  STEAM_VALIDATE: "false"
```

#### `START_ARGS`

Arguments passed to the game server executable at startup.

**Default:**
```yaml
START_ARGS: "-game pvkii -console +maxplayers 24 +map bt_island"
```

**Customization Examples:**

Territory server:
```yaml
START_ARGS: "-game pvkii -console +maxplayers 24 +map te_tortuga"
```

LAN-only server with an RCON password:
```yaml
START_ARGS: "-game pvkii -console +maxplayers 24 +map bt_island +sv_lan 1 +rcon_password mypassword"
```

Deathmatch ruleset:
```yaml
START_ARGS: "-game pvkii -console +maxplayers 24 +map dm_fort +mp_ruleset deathmatch"
```

**Common Server Parameters:**
- `+map <mapname>` - Set the starting map. The map prefix selects the game mode (see [Game modes and maps](#game-modes-and-maps))
- `+maxplayers <n>` - Player slots
- `+mp_ruleset <name>` - Load a ruleset from `pvkii/rulesets/` (see [Rulesets](#rulesets))
- `+sv_lan 1` - LAN-only server (0 = internet, 1 = LAN only)
- `+ip <address>` - Server IP address (use `0.0.0.0` for all interfaces)
- `+port <port>` - Server port (default: 27015)
- `+rcon_password <password>` - RCON password for remote administration
- `+hostname "<name>"` - Server name displayed in server browser
- `-binary ./srcds_linux32` - Run the 32-bit server instead of the 64-bit one (not recommended, see [Troubleshooting](#out-of-memory-or-address-space-with-the-32-bit-server))

#### `INSTALL_SOURCEMOD`

Enables automatic installation of SourceMod and MetaMod:Source, which provide plugin support and enhanced server administration.

- `"true"` - Install SourceMod and MetaMod
- `"false"` - Skip SourceMod installation

Default: `"true"`.

When enabled, the container will:
1. Download and install MetaMod:Source (required dependency)
2. Download and install SourceMod
3. Place all files in `/config/Overlay/pvkii/addons/`
4. Remove the 32-bit `metamod.vdf` loader (the server is 64-bit; `metamod_x64.vdf` loads MetaMod)

**Example:**
```yaml
environment:
  INSTALL_SOURCEMOD: "true"
```

### Security Options

The container requires elevated privileges to use OverlayFS for file overlaying.

#### `cap_add: SYS_ADMIN`

Adds the `SYS_ADMIN` capability, which is required for mounting OverlayFS. This is the recommended approach as it provides minimal necessary privileges.

```yaml
cap_add:
  - SYS_ADMIN
```

#### `security_opt: apparmor:unconfined`

On Ubuntu hosts with AppArmor enabled, you may need to disable AppArmor restrictions for the container. This is often necessary for OverlayFS to function properly.

```yaml
security_opt:
  - apparmor:unconfined
```

**Alternative Options:**

If you prefer less security but simpler configuration, you can use privileged mode:

```yaml
privileged: true
```

**Note:** Privileged mode grants the container extensive access to the host system and is less secure than using `cap_add: SYS_ADMIN`.

### Restart Policy

```yaml
restart: unless-stopped
```

This ensures the container automatically restarts if it stops unexpectedly, but won't restart if you manually stop it.

**Other options:**
- `no` - Never restart
- `always` - Always restart, even after manual stop
- `on-failure` - Restart only on failure

## Directory Structure

The `/config` directory contains the following structure:

```
/config/
├── Server/              # Game files from SteamCMD (auto-created)
│   └── pvkii/           # Pirates, Vikings, and Knights II game directory
├── Overlay/             # Custom files overlay (your modifications)
│   └── pvkii/
│       ├── addons/      # SourceMod/MetaMod installation location
│       │   ├── sourcemod/
│       │   │   ├── plugins/     # Place .smx plugin files here
│       │   │   ├── configs/     # SourceMod configuration files
│       │   │   └── ...
│       │   └── metamod/
│       ├── maps/        # Custom maps
│       ├── cfg/         # Server configuration files (server.cfg, etc.)
│       ├── rulesets/    # Custom rulesets
│       ├── mapcycle.txt # Map rotation
│       └── ...
├── Merged/              # OverlayFS merged view (auto-created)
├── .overlay-work/       # OverlayFS work directory (auto-created)
└── Scripts/
    └── Hooks/           # Custom PowerShell scripts for hooks
```

## Game Configuration

### Server Configuration

Place server configuration files in:

```
/config/Overlay/pvkii/cfg/
```

An `Overlay/pvkii/cfg/server.cfg` replaces the stock one, which only sets `hostname`, `mp_roundtime` and `sv_region`.

**Example `server.cfg`:**
```
hostname "My PVKII Server"
sv_lan 1
mp_timelimit 30
rcon_password "your_rcon_password"
```

### Game modes and maps

The game mode comes from the map prefix:

| Prefix | Game mode | Stock maps |
|--------|-----------|------------|
| `bt_` | Booty | `bt_forgotten`, `bt_glacier`, `bt_island`, `bt_pinegrove`, `bt_townsquare` |
| `tw_` | Trinket Wars | `tw_desertruin`, `tw_fort`, `tw_frostbite`, `tw_temple`, `tw_tortuga` |
| `te_` | Territory | `te_cara`, `te_mine`, `te_sandstorm`, `te_tortuga` |
| `lts_` | Last Team Standing | `lts_arena`, `lts_cathedral`, `lts_gravedanger`, `lts_town` |
| `dm_` | Team Deathmatch | `dm_fort` |

The game also ships `duel_*` and `comp_*` maps. The stock rotation is `pvkii/mapcycle.txt`; to change it, put your own `mapcycle.txt` in `/config/Overlay/pvkii/`.

### Rulesets

Rulesets are files in `pvkii/rulesets/` that set cvars for the whole server and per game mode, and are reverted when another ruleset is loaded. Load one with `mp_ruleset <name>` (for example `+mp_ruleset deathmatch` in `START_ARGS`). The game ships `classic` (the default), `competitive`, `deathmatch` and `randomizer`; `deathmatch` sets `mp_dm_force 1`, which plays Team Deathmatch on any map; `pvkii/rulesets/templates/` has templates and a readme for writing your own. Put custom rulesets in `/config/Overlay/pvkii/rulesets/`.

## SourceMod Configuration

### Installing Plugins

Once SourceMod is installed, add plugins by placing compiled `.smx` files in:

```
/config/Overlay/pvkii/addons/sourcemod/plugins/
```

**Example:**
```bash
# Copy plugin to the overlay directory
cp my_plugin.smx "/data/Servers/Pirates, Vikings, and Knights II/Booty/Overlay/pvkii/addons/sourcemod/plugins/"

# Restart the container to load the plugin
docker restart pvkii-booty
```

### SourceMod Configuration Files

Place SourceMod configuration files in:

```
/config/Overlay/pvkii/addons/sourcemod/configs/
```

**Common configuration files:**
- `admins_simple.ini` - Admin user configuration
- `core.cfg` - Core SourceMod settings
- `database.cfg` - Database configuration (if using database features)

## OverlayFS

The container uses Linux OverlayFS to merge the base game files with your custom files:

- **Lower layer**: `/config/Server` (base game files from SteamCMD)
- **Upper layer**: `/config/Overlay` (your custom files)
- **Merged view**: `/config/Merged` (where the game server runs from)

**Benefits:**
- Replace files without modifying the base installation
- Add custom content (maps, plugins, configs)
- No file copying required - OverlayFS is a union filesystem
- Easy updates - base game files can be updated without losing customizations

If OverlayFS cannot be mounted (e.g., missing privileges), the container will fall back to using `/config/Server` directly and log a warning.

## Troubleshooting

### Container Won't Start

1. **Check logs:**
   ```bash
   docker logs pvkii-booty
   ```

2. **Verify permissions:**
   Ensure the mounted volume is writable:
   ```bash
   # Linux
   chmod -R 755 "/data/Servers/Pirates, Vikings, and Knights II/Booty"

   # Windows
   # Ensure the directory has proper permissions in Windows
   ```

3. **Check security options:**
   Ensure `cap_add: SYS_ADMIN` is set, or use `privileged: true`

### Game Server Not Starting

1. **Verify START_ARGS:**
   Check that `START_ARGS` contains valid server arguments:
   ```yaml
   START_ARGS: "-game pvkii -console +maxplayers 24 +map bt_island"
   ```

2. **Check server directory:**
   Verify that game files were downloaded:
   ```bash
   docker exec pvkii-booty ls -la /config/Server/pvkii
   ```

3. **Review server logs:**
   Check container logs for server startup messages and errors

### SourceMod Not Loading

1. **Verify installation:**
   Ensure `INSTALL_SOURCEMOD: "true"` is set (with quotes in YAML)

2. **Check installation directory:**
   ```bash
   docker exec pvkii-booty ls -la /config/Overlay/pvkii/addons/sourcemod
   ```

3. **Verify MetaMod:**
   Check that MetaMod's 64-bit loader is installed:
   ```bash
   docker exec pvkii-booty ls -la /config/Overlay/pvkii/addons/metamod_x64.vdf
   ```

4. **Check the versions:**
   Pirates, Vikings, and Knights II runs a 64-bit server. MetaMod:Source and SourceMod only ship 64-bit PVKII binaries (`metamod/bin/linux64/metamod.2.pvkii.so`, `sourcemod/bin/x64/sourcemod.2.pvkii.so`) from MetaMod 1.12.0-git1229 and SourceMod 1.12.0-git7259 on. With older versions the server starts with no plugins loaded and `meta` is an unknown command.

5. **Check server console:**
   Look for `[META] Loaded 1 plugin.` in the container logs. The `Failed to load GAMEaddons/metamod/bin/linux64/server` line before it is the engine trying search paths and is harmless.

### `ERROR! Failed to install app '17575' (Missing configuration)`

A fresh SteamCMD (no cached app info) fails to install the Pirates, Vikings, and Knights II Dedicated Server with this error. The image handles this in the `PreSteamInstallGame/10-BootstrapInstall.ps1` hook: when `Server/steamapps/appmanifest_17575.acf` doesn't exist yet, it refreshes the app info (`app_info_update 1`) and installs in the same SteamCMD session. Later updates work as usual. If the first install was interrupted, delete `Server/steamapps/appmanifest_17575.acf` and restart the container to run the bootstrap again.

### Out of memory or address space with the 32-bit server

The game defaults to the 64-bit server (`DefaultArchitecture "64bit"` in `launch_settings.txt`; `srcds_run` starts `srcds_linux64`). Forcing the 32-bit server with `-binary ./srcds_linux32` makes the map load fail with `Out of memory or address space.  Texture quality setting may be too high.` and the server restarts in a loop. Keep the 64-bit default.

### Harmless startup messages

`Failed to load ... stdshader_dx6.so`/`game_shader_dx6.so` (and dx7/dx8), `CSoundEmitterSystem::AddSoundsFromFile: No such file ...`, `missmatched parent bones` and `SOLID_VPHYSICS static prop with no vphysics model!` come from the stock game files on every start and don't affect the server.

### Port Already in Use

If you get port binding errors:

1. **Check for existing containers:**
   ```bash
   docker ps -a
   ```

2. **Use different ports:**
   Change the port mapping in docker-compose.yml:
   ```yaml
   ports:
     - 27016:27015/udp  # Use a different host port
   ```

3. **Stop conflicting containers:**
   ```bash
   docker stop <container-name>
   ```

### Permission Errors

The container runs as a non-root user. If you encounter permission errors:

1. **Check file ownership:**
   ```bash
   docker exec pvkii-booty ls -la /config
   ```

2. **Verify volume mount:**
   Ensure the host directory is accessible and writable

3. **Review logs:**
   Check container logs for specific permission error messages

## Advanced Usage

### Custom Hooks

You can create custom PowerShell scripts that execute at various points in the container's lifecycle. Place scripts in:

```
/config/Scripts/Hooks/{HookName}/
```

**Available hooks:**
- `PreSteamInstallGame` - Before SteamCMD runs
- `PostSteamInstallGame` - After SteamCMD completes
- `PreInstallSourceMod` - Before SourceMod installation
- `PostInstallSourceMod` - After SourceMod installation

**Example hook script** (`/config/Scripts/Hooks/PostSteamInstallGame/10-CustomSetup.ps1`):
```powershell
Write-Host "Running custom setup..."
# Your custom commands here
```

### HTTP File Server

The container includes an optional HTTP file server (port 80) for serving game files to clients. This is useful for fast downloads of custom maps and content.

The file server is configured via environment variables:
- `HTTP_FILESERVER_ENABLED` - Enable/disable the file server
- `HTTP_FILESERVER_WEB_ROOT` - Root directory for file serving
- `HTTP_FILESERVER_FILE_PATTERN` - Pattern for files to serve

### Multiple Server Instances

To run multiple Pirates, Vikings, and Knights II servers on the same host:

1. **Use different container names:**
   ```yaml
   container_name: pvkii-server1
   ```

2. **Use different ports:**
   ```yaml
   ports:
     - 27016:27015/udp
     - 27016:27015/tcp
   ```

3. **Use different volume mounts:**
   ```yaml
   volumes:
     - "/data/Servers/Pirates, Vikings, and Knights II/Server1:/config"
   ```

4. **Use different maps/names:**
   ```yaml
   environment:
     START_ARGS: "-game pvkii -console +maxplayers 24 +map te_tortuga +hostname 'Server 1'"
   ```

## Additional Resources

- [Source Dedicated Server Documentation](https://developer.valvesoftware.com/wiki/Source_Dedicated_Server)
- [SourceMod Documentation](https://wiki.alliedmods.net/Category:SourceMod_Documentation)
- [MetaMod:Source Documentation](https://wiki.alliedmods.net/Metamod:Source)
