# Zymphony

Docker service that watches a directory for Spotify playlist ZIPs and
automatically converts them into well-tagged music compilations ready for
[Navidrome](https://www.navidrome.org/).

**What it does for each playlist:**

1. Detects when all ZIP parts have finished downloading (size + stability heuristic).
2. Fetches the playlist name and cover art from the Spotify API.
3. Extracts all ZIPs, tags every MP3 (`album`, `albumartist`, `tracknumber`, `compilation`)
   and writes `cover.jpg` to the output folder.
4. Moves the result to your music library and deletes the source ZIPs.

---

## Prerequisites

- Docker (Engine 24+ / Desktop 4.25+) with buildx if you build locally.
- A **Spotify Developer app** — see [Create a Spotify app](#create-a-spotify-app).
- A one-time [bootstrap](#one-time-spotify-authorization) to obtain the OAuth refresh token.

---

## Create a Spotify app

1. Go to <https://developer.spotify.com/dashboard> and log in.
2. Click **Create app**. Name and description can be anything.
3. Under **Redirect URIs**, add `http://localhost:8888/callback` and save.
4. Open the app settings and note the **Client ID** and **Client Secret**.

---

## Quick start

```bash
# 1. Copy the example compose file and fill in your values
cp docker-compose.yml docker-compose.local.yml
#    Edit: SPOTIFY_CLIENT_ID, SPOTIFY_CLIENT_SECRET, volume paths, PUID/PGID

# 2. Run the one-time authorization (on a machine with a browser — see below)
docker compose -f docker-compose.local.yml run --rm \
  -p 8888:8888 zymphony zymphony auth

# 3. Start the service
docker compose -f docker-compose.local.yml up -d
```

---

## One-time Spotify authorization

The service needs a **refresh token** stored in `/config/spotify_token.json`.
This token is permanent — you only do this step once.

### Option A — bootstrap on your Mac (recommended)

Run the auth command locally (not in Docker). The browser opens automatically
and the token is written to the path you specify:

```bash
CONFIG_DIR=/path/to/config \
SPOTIFY_CLIENT_ID=your_id \
SPOTIFY_CLIENT_SECRET=your_secret \
SPOTIFY_REDIRECT_URI=http://localhost:8888/callback \
pip install zymphony && zymphony auth
```

Then copy `spotify_token.json` to the NAS config volume:

```bash
scp /path/to/config/spotify_token.json \
    user@NAS_IP:/volume1/docker/zymphony/config/
```

### Option B — bootstrap in Docker on the NAS

Expose port 8888 temporarily, set `SPOTIFY_REDIRECT_URI` to the NAS IP, and
register that URI in your Spotify app settings first:

```bash
# On the NAS via SSH:
docker run --rm -it \
  -p 8888:8888 \
  -v /volume1/docker/zymphony/config:/config \
  -e SPOTIFY_CLIENT_ID=your_id \
  -e SPOTIFY_CLIENT_SECRET=your_secret \
  -e SPOTIFY_REDIRECT_URI=http://NAS_IP:8888/callback \
  -e CONFIG_DIR=/config \
  jorgejiro/zymphony:latest zymphony auth
```

Open the printed URL in your browser, authorize, and the token is saved.

---

## Environment variables

| Variable | Default | Description |
|---|---|---|
| `SPOTIFY_CLIENT_ID` | *(required)* | Spotify app client ID |
| `SPOTIFY_CLIENT_SECRET` | *(required)* | Spotify app client secret |
| `SPOTIFY_REDIRECT_URI` | `http://localhost:8888/callback` | Must match your Spotify app settings |
| `PUID` | `1000` | UID that will own output files |
| `PGID` | `1000` | GID that will own output files |
| `TZ` | `UTC` | Container timezone |
| `SCAN_INTERVAL_SECONDS` | `60` | Seconds between directory scans |
| `STABLE_MINUTES` | `5` | Minutes a file must be unchanged before processing |
| `PART_SIZE_THRESHOLD_MB` | `1000` | Size threshold (MB) separating full parts from the final part |
| `MIN_TOTAL_SIZE_MB` | `100` | Minimum total group size before considering it |
| `ALBUM_ARTIST` | `Various Artists` | Value written to the `albumartist` tag |
| `DELETE_ZIPS_AFTER` | `true` | Delete source ZIPs after successful processing |
| `LOG_LEVEL` | `INFO` | Python logging level (`DEBUG`, `INFO`, `WARNING`, `ERROR`) |
| `INPUT_DIR` | `/input` | Directory to watch for ZIP files |
| `OUTPUT_DIR` | `/output` | Navidrome music library root |
| `CONFIG_DIR` | `/config` | Persistent config directory (Spotify token) |

---

## Synology DS920+ setup

### 1. Find your PUID and PGID

SSH into your NAS and run:

```bash
id your_username
# uid=1026(your_username) gid=100(users) ...
```

Use `uid` as `PUID` and `gid` as `PGID`. These must match the user that owns
your Navidrome music library, so that Zymphony can write files and Navidrome
can read them.

### 2. Create the required folders

```bash
# Folder where your download tool drops ZIP files
mkdir -p /volume1/downloads

# Navidrome library root (skip if it already exists)
mkdir -p /volume1/music

# Persistent config: Spotify token + processing state
mkdir -p /volume1/docker/zymphony/config
```

Suggested folder structure:

```
/volume1/
├── downloads/              → /input   (source ZIPs)
├── music/                  → /output  (Navidrome library root)
└── docker/
    └── zymphony/
        └── config/         → /config  (Spotify token + state)
```

### 3. Run the one-time Spotify authorization

Before starting the service, generate the refresh token once. The easiest
approach is to do this from your Mac (Option A above) and then copy the
resulting `spotify_token.json` to the NAS:

```bash
scp spotify_token.json user@NAS_IP:/volume1/docker/zymphony/config/
```

### 4a. Deploy via Container Manager (DSM 7.2+)

1. Open **Container Manager** → **Project** → **Create**.
2. Set a project name (e.g. `zymphony`).
3. Under **Source**, choose **Create docker-compose.yml** and paste the
   contents of the `docker-compose.yml` from this repo — or import the file
   directly.
4. Edit the three volume paths under `volumes:` to match step 2:

   ```yaml
   volumes:
     - /volume1/downloads:/input
     - /volume1/music:/output
     - /volume1/docker/zymphony/config:/config
   ```

5. Fill in `SPOTIFY_CLIENT_ID`, `SPOTIFY_CLIENT_SECRET`, `PUID`, and `PGID`.
6. Click **Next → Done**. Container Manager starts the service automatically.

### 4b. Deploy via SSH / CLI

```bash
# SSH into the NAS
ssh user@NAS_IP

mkdir -p /volume1/docker/zymphony
cd /volume1/docker/zymphony

# Upload docker-compose.yml and edit it, then:
docker compose up -d
```

### Check logs

```bash
docker compose logs -f zymphony
```

A healthy start-up looks like:

```
zymphony  | [INFO] Starting Zymphony service
zymphony  | [INFO] Watching /input every 60s
```

### Update the container

```bash
docker compose pull
docker compose up -d
```

---

## Building the image

### Local build (single arch)

```bash
docker build -t zymphony:latest .
```

### Multi-arch build and push (amd64 + arm64)

```bash
# One-time setup of a multi-arch builder
docker buildx create --use --name multiarch --driver docker-container

# Build and push both architectures in one step
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t jorgejiro/zymphony:latest \
  --push \
  .
```

---

## Troubleshooting

**The service processes no files after dropping ZIPs into `/input`.**
Check `LOG_LEVEL=DEBUG` for detail. Common causes:
- The ZIPs were not yet stable for `STABLE_MINUTES` — wait or lower the value.
- The total size is below `MIN_TOTAL_SIZE_MB` — lower the threshold or verify
  the ZIPs are correct.
- The filename does not contain a 22-character Spotify playlist ID.

**`SpotifyException: No token` / auth errors.**
The refresh token is missing or expired. Re-run `zymphony auth` and copy the
new `spotify_token.json` to `/config`.

**Output files are owned by `root` instead of my user.**
`PUID`/`PGID` are probably not set. Check the environment section of your
compose file and verify the values with `id your_username` on the NAS.

**MP3s appear in Navidrome as individual albums instead of one compilation.**
The `compilation` (TCMP=1) and `albumartist` tags were not written correctly.
Enable `LOG_LEVEL=DEBUG`, check for tagging errors, and verify the file
is a valid ID3v2.3/2.4 MP3.

**A playlist fails repeatedly and I want to retry from scratch.**
The ZIPs are not deleted on failure — just restart the service or wait for the
next scan cycle. No manual cleanup is needed.

---

## Development

```bash
# Install in editable mode with dev dependencies
pip install -e ".[dev]"

# Run tests
pytest

# Lint and format
ruff check src tests
black src tests
```

### Running the auth bootstrap locally

```bash
export SPOTIFY_CLIENT_ID=your_id
export SPOTIFY_CLIENT_SECRET=your_secret
export SPOTIFY_REDIRECT_URI=http://localhost:8888/callback
export CONFIG_DIR=./local-config
mkdir -p ./local-config
zymphony auth
```
