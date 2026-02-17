# Snapcast LXC Server Configuration Guide

This document covers the configuration and management of the Snapcast multiroom audio server running on Proxmox LXC (10.7.7.221).

## Architecture Overview

```
┌─────────────────────────────────────────────────────┐
│  Proxmox LXC Container (10.7.7.221)                 │
│  Debian 13 (trixie)                                 │
│                                                     │
│  ┌─────────────────┐                                │
│  │  shairport-sync │───┐                            │
│  │  (Airplay 2)    │   │                            │
│  └─────────────────┘   │    ┌──────────────────┐    │
│                        ├───►│ /tmp/snapfifo    │    │
│  ┌─────────────────┐   │    │ (named pipe)     │    │
│  │   raspotify     │───┘    └────────┬─────────┘    │
│  │ (Spotify Connect)                 │              │
│  └─────────────────┘                 ▼              │
│                          ┌─────────────────┐        │
│                          │   snapserver    │        │
│                          │  :1704 :1705    │        │
│                          └────────┬────────┘        │
└───────────────────────────────────┼─────────────────┘
                                    │ TCP
                    ┌───────────────┼───────────────┐
                    ▼               ▼               ▼
              Raspberry Pi    Raspberry Pi    Raspberry Pi
               IoTSound        IoTSound        IoTSound
```

## Services

| Service | Port | Config File | Purpose |
|---------|------|-------------|---------|
| snapserver | 1704, 1705, 1780 | `/etc/snapserver.conf` | Audio distribution server |
| shairport-sync | 5000 | `/etc/shairport-sync.conf` | AirPlay 2 receiver |
| raspotify | - | `/etc/raspotify/conf` | Spotify Connect receiver |
| avahi-daemon | 5353 | `/etc/avahi/avahi-daemon.conf` | mDNS/Bonjour discovery |

## Configuration Files

### Raspotify (Spotify Connect)

**Location:** `/etc/raspotify/conf`

```bash
# Device name shown in Spotify
LIBRESPOT_NAME="IoTSound"

# Use pipe backend to output to snapfifo
LIBRESPOT_BACKEND=pipe
LIBRESPOT_DEVICE=/tmp/snapfifo

# Audio quality (96, 160, or 320 kbps)
LIBRESPOT_BITRATE=320
LIBRESPOT_FORMAT=S16

# Volume settings - let Snapcast handle volume
LIBRESPOT_INITIAL_VOLUME=100
LIBRESPOT_VOLUME_CTRL=linear

# Quiet logging
LIBRESPOT_QUIET=

# Disable caching
LIBRESPOT_DISABLE_AUDIO_CACHE=
LIBRESPOT_DISABLE_CREDENTIAL_CACHE=

TMPDIR=/tmp
```

**Common Configuration Options:**

| Option | Values | Description |
|--------|--------|-------------|
| `LIBRESPOT_NAME` | string | Device name shown in Spotify app |
| `LIBRESPOT_BITRATE` | 96, 160, 320 | Audio quality in kbps |
| `LIBRESPOT_BACKEND` | alsa, pipe, pulseaudio | Audio output backend |
| `LIBRESPOT_DEVICE` | path | Output device/pipe path |
| `LIBRESPOT_INITIAL_VOLUME` | 0-100 | Starting volume percentage |
| `LIBRESPOT_VOLUME_CTRL` | linear, log, fixed | Volume control curve |
| `LIBRESPOT_ENABLE_VOLUME_NORMALISATION` | (flag) | Normalize track volumes |
| `LIBRESPOT_AUTOPLAY` | (flag) | Auto-play similar songs |

**Systemd Override:** `/etc/systemd/system/raspotify.service.d/override.conf`
```ini
[Service]
# Allow access to the shared FIFO for Snapcast
PrivateTmp=false
PrivateUsers=false
ProtectSystem=full
ReadWritePaths=/tmp/snapfifo
```

### Shairport-Sync (AirPlay)

**Location:** `/etc/shairport-sync.conf`

```
general = {
    name = "IoTSound";
    output_backend = "pipe";
    ignore_volume_control = "yes";
};

pipe = {
    name = "/tmp/snapfifo";
};
```

**Common Configuration Options:**

| Section | Option | Description |
|---------|--------|-------------|
| `general` | `name` | Device name shown in AirPlay |
| `general` | `output_backend` | Output type: alsa, pipe, pa, stdout |
| `general` | `ignore_volume_control` | Ignore source volume (yes/no) |
| `general` | `volume_range_db` | Volume range in dB (30-150) |
| `pipe` | `name` | Path to output pipe |
| `alsa` | `output_device` | ALSA device name |
| `alsa` | `mixer_control_name` | ALSA mixer control |

### Snapserver

**Location:** `/etc/snapserver.conf`

Key sections:

```ini
[http]
doc_root = /usr/share/snapserver/snapweb

[stream]
source = pipe:///tmp/snapfifo?name=default
```

**Common Configuration Options:**

| Section | Option | Description |
|---------|--------|-------------|
| `[server]` | `threads` | Worker threads (-1 = auto) |
| `[http]` | `port` | Web UI port (default: 1780) |
| `[http]` | `doc_root` | Path to Snapweb UI |
| `[tcp]` | `port` | Control port (default: 1705) |
| `[stream]` | `port` | Audio stream port (default: 1704) |
| `[stream]` | `source` | Audio source URI |
| `[stream]` | `codec` | Transport codec: flac, ogg, opus, pcm |
| `[stream]` | `buffer` | End-to-end latency in ms |
| `[streaming_client]` | `initial_volume` | Default client volume |

**Source URI Formats:**
```
pipe:///tmp/snapfifo?name=default
alsa:///?name=ALSA&device=hw:0
tcp://0.0.0.0:4953?name=TCP&mode=server
```

### FIFO Pipe

**Tmpfiles config:** `/etc/tmpfiles.d/snapfifo.conf`
```
p /tmp/snapfifo 0666 root root -
```

This ensures the FIFO is created on boot with correct permissions.

## Management Commands

### Service Control

```bash
# Check all services
systemctl status snapserver shairport-sync raspotify avahi-daemon

# Restart a service after config change
systemctl restart raspotify
systemctl restart shairport-sync
systemctl restart snapserver

# View logs
journalctl -u raspotify -f
journalctl -u shairport-sync -f
journalctl -u snapserver -f
```

### Snapcast Volume Control (JSON-RPC API)

```bash
# Get server status with all clients
curl -s http://10.7.7.221:1780/jsonrpc \
  -d '{"method":"Server.GetStatus"}' | jq

# Set group volume (all clients) to 75%
curl http://10.7.7.221:1780/jsonrpc \
  -d '{"method":"Group.SetVolume","params":{"id":"default","volume":{"percent":75}}}'

# Mute entire group
curl http://10.7.7.221:1780/jsonrpc \
  -d '{"method":"Group.SetMute","params":{"id":"default","mute":true}}'

# Set individual client volume
curl http://10.7.7.221:1780/jsonrpc \
  -d '{"method":"Client.SetVolume","params":{"id":"<client-id>","volume":{"percent":50}}}'
```

### Web UI

Access Snapweb at: `http://10.7.7.221:1780`

## Raspberry Pi Client Configuration

On each Pi running IoTSound, set these environment variables in balenaCloud:

```
SOUND_MODE=MULTI_ROOM_CLIENT
SOUND_MULTIROOM_MASTER=10.7.7.221
```

## Changing Device Names

### Spotify Device Name
Edit `/etc/raspotify/conf`:
```bash
LIBRESPOT_NAME="New Name Here"
```
Then: `systemctl restart raspotify`

### AirPlay Device Name
Edit `/etc/shairport-sync.conf`:
```
general = {
    name = "New Name Here";
    ...
};
```
Then: `systemctl restart shairport-sync`

## Troubleshooting

### No audio output
1. Check FIFO exists: `ls -la /tmp/snapfifo`
2. Check services: `systemctl status snapserver shairport-sync raspotify`
3. Check clients connected: `curl -s http://localhost:1780/jsonrpc -d '{"method":"Server.GetStatus"}' | jq '.result.groups[].clients'`

### Spotify not appearing
1. Check raspotify: `systemctl status raspotify`
2. Check avahi: `systemctl status avahi-daemon`
3. View logs: `journalctl -u raspotify -n 50`

### AirPlay not appearing
1. Check shairport-sync: `systemctl status shairport-sync`
2. Check avahi: `systemctl status avahi-daemon`
3. Ensure port 5000 is open: `ss -tlnp | grep 5000`

### Clients not syncing
1. Check network connectivity between server and clients
2. Verify ports 1704/1705 are accessible
3. Check client logs on the Pi

## SSH Access

```bash
# Via Proxmox host
ssh root@10.7.7.61
pct exec 106 -- bash

# Direct (if SSH installed)
ssh root@10.7.7.221
```

## Backup Configuration

To backup all configs:
```bash
ssh root@10.7.7.61 "pct exec 106 -- tar czf - \
  /etc/raspotify/conf \
  /etc/shairport-sync.conf \
  /etc/snapserver.conf \
  /etc/systemd/system/raspotify.service.d/override.conf \
  /etc/tmpfiles.d/snapfifo.conf" > snapcast-lxc-backup.tar.gz
```
