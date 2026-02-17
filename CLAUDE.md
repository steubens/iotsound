# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

IoTSound (formerly balenaSound) is a multi-room audio streaming platform for Raspberry Pi that supports Bluetooth, Airplay2, Spotify Connect, and UPnP. It runs as a multi-container Docker application on balenaOS.

## Build Commands

### sound-supervisor (TypeScript service in `core/sound-supervisor/`)
```bash
cd core/sound-supervisor
npm install
npm run build      # Clean build: compiles TS to build/ and copies UI assets
npm run watch      # TypeScript watch mode for development
npm run livepush   # Development mode with nodemon and ts-node
```

### Full deployment
Deploy to the balenaCloud fleet from the `scott/customizations` branch:
```bash
balena push g_scott_langer/sound
```
Local development uses `docker-compose.yml`.

## Architecture

### Service Groups

**Sound Core:**
- `audio` - PulseAudio-based audio router (balena block). Handles all audio routing between sources and outputs.
- `sound-supervisor` - TypeScript orchestration service. Provides REST API (port 80), manages multi-room coordination via cote.js pub/sub, and interfaces with the audio block.

**Multi-room:**
- `multiroom-server` - Snapcast server broadcasting synchronized audio (ports 1704, 1705, 1780)
- `multiroom-client` - Snapcast client receiving and playing synchronized audio

**Plugins (audio sources):**
- `bluetooth`, `airplay`, `spotify`, `upnp` - Each sends audio to the audio block's `balena-sound.input` sink

### Audio Routing

Two virtual PulseAudio sinks handle all routing:
- `balena-sound.input` - Input multiplexer; all plugins send audio here by default
- `balena-sound.output` - Output multiplexer; always wired to the selected output device

**Standalone mode:** `balena-sound.input` → `balena-sound.output`

**Multi-room mode:** `balena-sound.input` → Snapcast server → network → Snapcast clients → `balena-sound.output`

Configuration scripts: `core/audio/balena-sound.pa` and `core/audio/start.sh`

### Key Files

- `core/sound-supervisor/src/index.ts` - Main entry point; initializes audio block, API, and fleet communication
- `core/sound-supervisor/src/SoundConfig.ts` - Configuration management
- `core/sound-supervisor/src/SoundAPI.ts` - REST API endpoints
- `docker-compose.yml` - Service definitions and networking

### Adding New Plugins

Plugins must send audio to PulseAudio at `tcp:localhost:4317`. Two methods:
1. **PulseAudio backend:** Set `ENV PULSE_SERVER=tcp:localhost:4317` in Dockerfile
2. **ALSA bridge:** For apps without PulseAudio support, use the alsa-bridge setup script from the audio block

## Environment Variables

- `SOUND_VOLUME` - Default volume (0-100, default: 75)
- `AUDIO_OUTPUT` - Output selection (AUTO, DAC, etc.)
- `BALENA_API_KEY` - Provided by balenaOS for SDK access

## Commit Guidelines

Follow semantic commit conventions for auto-generated changelog. PRs should be squashed before merging using `git rebase -i master`.

## Git Workflow

This repo is a fork of `iotsound/iotsound`. All custom changes live on the `scott/customizations` branch; `master` stays clean and in sync with upstream.

```
upstream → iotsound/iotsound (original project)
origin   → steubens/iotsound (our fork on GitHub)

Branches:
  master               — tracks upstream/master exactly, never commit here
  scott/customizations — all our changes; deploy from this branch
```

**Pulling upstream updates:**
```bash
git fetch upstream
git checkout master && git merge upstream/master && git push
git checkout scott/customizations && git rebase master
```

**Deploy after changes:**
```bash
git add <files> && git commit -m "..."
git push
balena push g_scott_langer/sound
```

## Infrastructure

### Snapcast LXC Server (10.7.7.221)

A Proxmox LXC container running Debian 13 serves as the central Snapcast server for whole-house audio distribution.

**Services:**
| Service | Config | Purpose |
|---------|--------|---------|
| snapserver | `/etc/snapserver.conf` | Audio distribution (ports 1704, 1705, 1780) |
| shairport-sync | `/etc/shairport-sync.conf` | AirPlay 2 receiver |
| raspotify | `/etc/raspotify/conf` | Spotify Connect receiver |

**Access:**
```bash
ssh root@10.7.7.61 "pct exec 106 -- <command>"
```

**Web UI:** `http://10.7.7.221:1780` (Snapweb for volume control)

**Volume API:**
```bash
curl http://10.7.7.221:1780/jsonrpc -d '{"method":"Group.SetVolume","params":{"id":"default","volume":{"percent":75}}}'
```

See `docs/SNAPCAST-LXC-SERVER.md` for full configuration guide.

### balenaCloud Fleet

**Fleet:** `g_scott_langer/sound`
**Fleet type:** `raspberry-pi` (supports Pi 1/Zero through Pi 4; kept broad to allow Pi Zero devices if added)

**Devices:**
| Device | UUID | Type | OS Version | Supervisor | Role |
|--------|------|------|------------|------------|------|
| Bedroom | `835dcd5` | Pi 4 (`raspberrypi4-64`) | balenaOS 6.10.24+rev1 | 17.5.3 | Client |
| Kitchen | `74a8ccd` | Pi 4 (`raspberrypi4-64`) | balenaOS 6.10.24+rev1 | 17.5.3 | Client |
| Living Room | `98d060a` | Pi 3 (`raspberrypi3`) | balenaOS 6.10.24+rev1 | 17.5.3 | Client |

**Client Configuration:**
```
SOUND_MODE=MULTI_ROOM_CLIENT
SOUND_MULTIROOM_MASTER=10.7.7.221
```

**balena CLI Access:**
```bash
balena device ssh <uuid>                  # Via cloud
balena device ssh <ip> -p 22222           # Direct local
balena device list --fleet g_scott_langer/sound  # List all devices
```

### WiFi Configuration (balenaOS)

To add WiFi on a Pi:
```bash
echo 'nmcli con add type wifi con-name "ssid" ifname wlan0 ssid "ssid" wifi-sec.key-mgmt wpa-psk wifi-sec.psk "password"; exit' | balena device ssh <ip> -p 22222
```
