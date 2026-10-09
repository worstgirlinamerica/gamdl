# gamdl-pr

Fork of [glomatico/gamdl](https://github.com/glomatico/gamdl) that adds a PlayReady decryption path for music videos and a `/webplayback` fallback for songs. Works alongside [worstgirlinamerica/wrappr](https://github.com/worstgirlinamerica/wrappr/tree/playready) — the companion daemon that handles Apple authentication and decryption.

> **Coming from glomatico/gamdl?** The CLI command is still `gamdl` and all the same flags work. The difference is this fork requires two extra pieces at install time: a Rust build step (handled automatically by pip) and a small Go binary for PlayReady. See [Installation](#installation) below.

---

## What's different from upstream

| | glomatico/gamdl | gamdl-pr (this fork) |
|-|-----------------|----------------------|
| Songs | ✅ | ✅ |
| Music videos (WVD) | 1080 | 1080 |
| Music videos (PRD) | ❌ | 4k |
| Song webplayback fallback | ❌ | ✅ |
| Requires wrappr | optional | required for 2160p Music Videos (PlayReady) and ALAC |
| Extra install step | none | Go binary (`gamdl-playready`) |

---

## Prerequisites

You need all three of these before installing:

### 1. Python 3.10+

Check with `python3 --version`. If you need to install it:
- **macOS:** `brew install python` or download from [python.org](https://www.python.org/downloads/)
- **Linux:** `sudo apt install python3 python3-pip` (Debian/Ubuntu) or your distro's equivalent
- **Windows:** Download from [python.org](https://www.python.org/downloads/) — check "Add to PATH" during install

### 2. Go 1.21+

Required to build the `gamdl-playready` helper binary. Check with `go version`. If you need to install it:
- **macOS:** `brew install go`
- **Linux:** `sudo apt install golang-go` or follow [go.dev/doc/install](https://go.dev/doc/install) for the latest version
- **Windows:** Download the installer from [go.dev/dl](https://go.dev/dl/)

### 3. Rust (via rustup) + a C compiler

Required because pip builds a native Rust extension (`gamdl._ammuxer`) during install. This happens automatically — you just need Rust present.
- **All platforms:** `curl -sSf https://sh.rustup.rs | sh` then restart your terminal
- **Windows:** Download rustup from [rustup.rs](https://rustup.rs/) — also requires [Microsoft C++ Build Tools](https://visualstudio.microsoft.com/visual-cpp-build-tools/)

### 4. Apple Music subscription + cookies

Export your browser cookies in Netscape format while logged into [music.apple.com](https://music.apple.com):
- **Firefox:** [Export Cookies](https://addons.mozilla.org/addon/export-cookies-txt)
- **Chromium/Chrome:** [Get cookies.txt LOCALLY](https://chromewebstore.google.com/detail/get-cookiestxt-locally/cclelndahbckbenkjhflpdbgdldlbecc)

### 5. wrappr (for PlayReady and ALAC)

The companion daemon — handles Apple account auth, playback dispatch, and the `/license` endpoint this fork uses for PlayReady. See [worstgirlinamerica/wrappr](https://github.com/worstgirlinamerica/wrappr/tree/playready) for setup. Once it's running, `curl http://127.0.0.1/health` should return `"status":"ok"`.

---

## Installation

### Step 1 — Clone and install the Python package

```bash
git clone -b playready https://github.com/worstgirlinamerica/gamdl.git
cd gamdl
pip install .
```

This will compile the native Rust extension. It takes a minute or two. If it fails, the most common cause is a missing C compiler — see [Troubleshooting](#troubleshooting).

### Step 2 — Build the PlayReady helper

This is a small Go binary that handles the PlayReady CDM challenge/license exchange. It needs to be built once and placed somewhere on your PATH.

```bash
cd tools/playready-helper
go mod tidy
go build -o "$HOME/.local/bin/gamdl-playready" .
cd ../..
```

Make sure `$HOME/.local/bin` is on your PATH. On macOS/Linux, add this to your `~/.zshrc` or `~/.bashrc` if it isn't already:

```bash
export PATH="$HOME/.local/bin:$PATH"
```

On **Windows**, build to a folder that's on your PATH:

```powershell
go build -o "$env:USERPROFILE\AppData\Local\Microsoft\WindowsApps\gamdl-playready.exe" .
```

Or build it anywhere and set the env variable (see [Configuration](#configuration)):

```bash
GAMDL_PLAYREADY_HELPER=/path/to/gamdl-playready gamdl ...
```

### Step 3 — Place your cookies file

Put your `cookies.txt` in the directory you'll run `gamdl` from, or pass the path explicitly:

```bash
gamdl --cookies-path /path/to/cookies.txt "https://music.apple.com/..."
```

### Step 4 — Start wrappr

Follow the [wrappr setup guide](https://github.com/worstgirlinamerica/wrappr/tree/playready). Once it's running:

```bash
curl http://127.0.0.1/health
# should return: {"status":"ok","runtime":{"playback_ready":true},...}
```

---

## Usage

```bash
gamdl [OPTIONS] URLS...
```

### Basic examples

```bash
# Download a song
gamdl "https://music.apple.com/us/album/song-name/1234567890?i=1234567891"

# Download an album
gamdl "https://music.apple.com/us/album/album-name/1234567890"

# Download a music video with PlayReady (requires wrappr running)
gamdl --use-wrapper "https://music.apple.com/us/music-video/title/1234567890"

# Download ALAC (lossless), requires wrappr
gamdl --use-wrapper --song-codec-priority alac "https://music.apple.com/..."
```

### Supported URL types

Songs, albums, playlists, music videos, artists, post videos — catalog and library. Apple Music Classical included.

### Interactive prompt controls

| Key | Action |
|-----|--------|
| Arrow keys | Move selection |
| Space | Toggle selection |
| Ctrl + A | Select all |
| Enter | Confirm |

---

## Configuration

Config file is created automatically on first run:
- **macOS / Linux:** `~/.gamdl/config.ini`
- **Windows:** `%USERPROFILE%\.gamdl\config.ini`

Command-line arguments override config values.

### Key options

| Option | Description | Default |
|--------|-------------|---------|
| `--cookies-path`, `-c` | Cookies file path | `./cookies.txt` |
| `--use-wrapper` | Enable wrappr for account, playback, and decrypt | `false` |
| `--wrapper-url` | wrappr HTTP base URL | `http://127.0.0.1` |
| `--wrapper-decrypt-host` | wrappr TCP decrypt host | `127.0.0.1` |
| `--wrapper-decrypt-port` | wrappr TCP decrypt port | `10020` |
| `--song-codec-priority` | Comma-separated codec priority | `aac-web` |
| `--music-video-resolution` | Max music video resolution | `1080p` |
| `--output-path`, `-o` | Output directory | `./Apple Music` |
| `--download-mode` | `ytdlp` or `nm3u8dlre` | `ytdlp` |
| `--log-level` | `DEBUG`, `INFO`, `WARNING`, `ERROR` | `INFO` |

Full option reference is in the [upstream gamdl docs](https://github.com/glomatico/gamdl#readme) — all flags are the same.

### Song codecs

**Web (no wrapper needed):**
- `aac-web` — AAC 256kbps 44.1kHz
- `aac-he-web` — AAC-HE 64kbps

**Non-web (wrapper recommended):**
- `alac` — Lossless up to 24-bit/192kHz
- `atmos` — Dolby Atmos 768kbps
- `aac`, `aac-he`, `ac3`, and spatial variants

### Music video codecs / resolutions

- H.264: up to 1080p
- H.265: up to 2160p (4K)
- Remux format: `m4v` (default) or `mp4`

---

## Troubleshooting

### `pip install .` fails with a compiler error

The install compiles a Rust extension. Make sure:
1. Rust is installed: `rustup --version`
2. You have a C compiler:
   - macOS: `xcode-select --install`
   - Linux: `sudo apt install build-essential`
   - Windows: Install [Microsoft C++ Build Tools](https://visualstudio.microsoft.com/visual-cpp-build-tools/)

Then retry `pip install .`

### `gamdl-playready: command not found`

The helper binary isn't on your PATH. Either:
- Move it to a directory that's on your PATH (`$HOME/.local/bin`, `/usr/local/bin`, etc.)
- Or set the env variable when running gamdl: `GAMDL_PLAYREADY_HELPER=/full/path/to/gamdl-playready gamdl ...`

### `go: command not found` during helper build

Go isn't installed. See [Prerequisites → Go](#2-go-121).

### `wrapper_api is required for PlayReady decrypt`

You ran a music video download without `--use-wrapper`, or wrappr isn't running. Start wrappr first and pass `--use-wrapper`.

### `PlayReady helper returned an invalid content key`

The helper ran but got a bad response from wrappr's `/license` endpoint. Check:
1. `curl http://127.0.0.1/health` — is wrappr running and `playback_ready: true`?
2. Is your Apple session still valid? Try `curl http://127.0.0.1/me` — if `state` isn't `"authenticated"`, log in again via wrappr.

### `Wrapper is not authenticated`

wrappr is running but hasn't been logged into. Either pass credentials via `--wrapper-url` pointed at a logged-in instance, or log in manually:

```bash
curl -X POST http://127.0.0.1/login \
     -H 'content-type: application/json' \
     -d '{"username":"you@example.com","password":"your-app-specific-password"}'
```

### Songs download fine but music videos fail

Music video PlayReady decrypt requires both `--use-wrapper` and the `gamdl-playready` binary. Songs (non-ALAC) use Widevine and work without either.

---

## Embedding

```python
import asyncio
from gamdl.api import AppleMusicApi
from gamdl.api.wrapper import WrapperApi
from gamdl.downloader import AppleMusicDownloader, AppleMusicSongDownloader, AppleMusicMusicVideoDownloader, AppleMusicUploadedVideoDownloader, AppleMusicBaseDownloader
from gamdl.interface import AppleMusicBaseInterface, AppleMusicInterface, AppleMusicSongInterface, AppleMusicMusicVideoInterface, AppleMusicUploadedVideoInterface

async def main():
    apple_music_api = await AppleMusicApi.create_from_netscape_cookies(cookies_path="cookies.txt")

    # Optional: pass wrapper_api for PlayReady / ALAC
    wrapper_api = await WrapperApi.create(base_url="http://127.0.0.1")

    base_interface = await AppleMusicBaseInterface.create(
        apple_music_api=apple_music_api,
        wrapper_api=wrapper_api,
    )

    interface = AppleMusicInterface(
        song=AppleMusicSongInterface(base=base_interface),
        music_video=AppleMusicMusicVideoInterface(base=base_interface),
        uploaded_video=AppleMusicUploadedVideoInterface(base=base_interface),
    )
    base_downloader = AppleMusicBaseDownloader(interface=interface)
    downloader = AppleMusicDownloader(
        song=AppleMusicSongDownloader(base=base_downloader),
        music_video=AppleMusicMusicVideoDownloader(base=base_downloader),
        uploaded_video=AppleMusicUploadedVideoDownloader(base=base_downloader),
    )

    url = "https://music.apple.com/us/album/example/1234567890?i=1234567891"
    async for media in downloader.get_download_item_from_url(url):
        await downloader.download(media)

asyncio.run(main())
```

---

## License

MIT — see [LICENSE](LICENSE).

Upstream: [glomatico/gamdl](https://github.com/glomatico/gamdl). This fork is not affiliated with Apple Inc.
