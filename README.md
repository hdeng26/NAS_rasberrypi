# 🍿 Raspberry Pi 4B Home Media Automation (1080p H.264 + PS5 Plex Playback) Deployment & Maintenance Guide

## Table of Contents
1. [Hardware List](#hardware-list)
2. [System Installation](#system-installation)
3. [Mounting External Drive](#mounting-external-drive)
4. [Docker Stack Deployment](#docker-stack-deployment)
5. [Service Integration](#service-integration)
6. [Quality Profiles (Avoiding Transcoding)](#quality-profiles-avoiding-transcoding)
7. [Plex Client Settings (PS5)](#plex-client-settings-ps5)
8. [Routine Maintenance](#routine-maintenance)
9. [Common Issues & Troubleshooting](#common-issues--troubleshooting)
10. [One-Click Health Check Script](#one-click-health-check-script)

---

## Hardware List
- Raspberry Pi 4 Model B (4GB or 8GB RAM recommended)
- Official 5V/3A power supply
- Gigabit Ethernet cable (Pi → Router)
- 2.5" external HDD (USB3 connection, high-quality cable recommended)
- microSD card (≥16GB, Class 10/UHS-I)
- Optional: powered USB hub (if HDD under-voltage occurs)

---

## System Installation
1. Download and flash **Raspberry Pi OS 64-bit** to microSD card using [Raspberry Pi Imager](https://www.raspberrypi.com/software/).  
2. On first boot:
   ```bash
   sudo apt update && sudo apt full-upgrade -y
   sudo reboot
   ```

---

## Mounting External Drive
1. Identify device:
   ```bash
   lsblk -f
   sudo blkid
   ```
2. Create mount point:
   ```bash
   sudo mkdir -p /srv/media
   ```
3. Edit `/etc/fstab`:
   ```
   UUID=YOUR_UUID  /srv/media  ext4   defaults,noatime  0 2
   ```
   (Change driver for NTFS/exFAT accordingly)
4. Mount:
   ```bash
   sudo mount -a
   df -h | grep /srv/media
   ```

---

## Docker Stack Deployment
### Install Docker
```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
newgrp docker
sudo apt install -y docker-compose-plugin
```

### Create Directory Structure
```bash
sudo mkdir -p /srv/media/{downloads/{incomplete,complete},library/{Movies,TV}}
sudo mkdir -p /srv/docker
sudo chown -R $USER:$USER /srv/media /srv/docker
```

### `docker-compose.yml`
```yaml
services:
  qbittorrent:
    image: lscr.io/linuxserver/qbittorrent
    ports:
      - "8080:8080"
      - "6881:6881"
      - "6881:6881/udp"
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/Toronto
      - WEBUI_PORT=8080
    volumes:
      - /srv/media/downloads:/downloads
      - /srv/docker/qbittorrent:/config
    restart: unless-stopped

  prowlarr:
    image: lscr.io/linuxserver/prowlarr
    ports: ["9696:9696"]
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/Toronto
    volumes:
      - /srv/docker/prowlarr:/config
    restart: unless-stopped

  sonarr:
    image: lscr.io/linuxserver/sonarr
    ports: ["8989:8989"]
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/Toronto
    volumes:
      - /srv/docker/sonarr:/config
      - /srv/media/library/TV:/tv
      - /srv/media/downloads:/downloads
    restart: unless-stopped

  radarr:
    image: lscr.io/linuxserver/radarr
    ports: ["7878:7878"]
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/Toronto
    volumes:
      - /srv/docker/radarr:/config
      - /srv/media/library/Movies:/movies
      - /srv/media/downloads:/downloads
    restart: unless-stopped

  bazarr:
    image: lscr.io/linuxserver/bazarr
    ports: ["6767:6767"]
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/Toronto
    volumes:
      - /srv/docker/bazarr:/config
      - /srv/media/library/Movies:/movies
      - /srv/media/library/TV:/tv
    restart: unless-stopped

  plex:
    image: lscr.io/linuxserver/plex
    network_mode: host
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/Toronto
      - PLEX_CLAIM=claim-XXXX
    volumes:
      - /srv/docker/plex:/config
      - /srv/media/library:/data
      - /tmp:/transcode
    restart: unless-stopped
```

Start:
```bash
docker compose -f /srv/docker/compose.yml up -d
```

---

## Service Integration
1. **Prowlarr** → Add indexers → Settings → Apps → Add Sonarr/Radarr (sync indexers)  
2. **Sonarr/Radarr** → Settings → Download Clients → Add qBittorrent (set category tv/movies)  
3. **Bazarr** → Link Sonarr/Radarr → Set subtitle providers & languages (SRT external)  
4. **Plex** → Add libraries `/data/Movies` & `/data/TV`

---

## Quality Profiles (Avoiding Transcoding)
Goal: **1080p + H.264 + AAC/AC3, exclude HEVC/AV1**  
- Release Profile (Sonarr/Radarr):
  - Must Not Contain: `HEVC|H265|x265|AV1|10bit`
  - Must Contain: `H264|x264|AVC`
  - Preferred: `WEBRip|WEBDL`

---

## Plex Client Settings (PS5)
- Video Quality: **Original**
- Enable **Allow Direct Play**
- Default subtitles to SRT (external)

---

## Routine Maintenance
- Update containers:
  ```bash
  docker compose -f /srv/docker/compose.yml pull
  docker compose -f /srv/docker/compose.yml up -d
  ```
- Check HDD health:
  ```bash
  sudo apt install smartmontools
  sudo smartctl -a /dev/sda
  ```
- Check undervoltage:
  ```bash
  vcgencmd get_throttled
  ```

---

## Common Issues & Troubleshooting
- **Video stutter**: Check if Direct Play; HEVC or image-based subtitles may trigger transcoding  
- **Drive disconnects**: Use better power supply / powered USB hub  
- **Slow downloads**: Check indexer status / port forwarding

---

## One-Click Health Check Script
```bash
#!/bin/bash
echo "==== Container Status ===="
docker ps --format "table {{.Names}}\t{{.Status}}"

echo -e "\n==== Disk Mount ===="
df -h | grep /srv/media || echo "❌ Media drive not mounted"

echo -e "\n==== Undervoltage Check ===="
vcgencmd get_throttled

echo -e "\n==== HDD Health ===="
sudo smartctl -H /dev/sda || echo "❌ Unable to check (possibly unsupported USB enclosure)"
```
