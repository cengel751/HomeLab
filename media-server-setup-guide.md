# Home Lab Media Server Setup Guide
## Automated Downloads with Plex, Radarr, Sonarr on Zorin OS

### Table of Contents
1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Prerequisites](#prerequisites)
4. [Directory Structure Setup](#directory-structure-setup)
5. [Docker Compose Configuration](#docker-compose-configuration)
6. [Initial Setup](#initial-setup)
7. [Service Configuration](#service-configuration)
8. [Integration Steps](#integration-steps)
9. [Testing](#testing)
10. [Maintenance](#maintenance)
11. [Troubleshooting](#troubleshooting)

---

## Overview

This guide will help you set up a fully automated media server using Docker on Zorin OS. The stack includes:

- **Plex Media Server**: Stream your media library
- **Radarr**: Automated movie management and downloads
- **Sonarr**: Automated TV show management and downloads
- **Jackett**: Torrent indexer proxy/aggregator
- **qBittorrent**: Torrent download client
- **Overseerr**: Media request management (optional but recommended)

### How It Works
1. You (or users via Overseerr) request a movie or TV show
2. Radarr/Sonarr searches for the content via Jackett
3. Download is sent to qBittorrent
4. Once complete, Radarr/Sonarr moves/renames the file to your media library
5. Plex automatically detects and adds the new content
6. Stream and enjoy!

---

## Architecture

```
User Request (Overseerr)
        |
        v
Radarr/Sonarr (Management)
        |
        v
Jackett (Indexer Search)
        |
        v
qBittorrent (Downloads)
        |
        v
Media Library (Movies/TV Shows)
        |
        v
Plex (Streaming)
```

---

## Prerequisites

### System Requirements
- **OS**: Zorin OS (Ubuntu-based)
- **Docker**: Installed and running (Version 28.5.1+)
- **Docker Compose**: Installed (Version 2.40.0+)
- **Disk Space**: Minimum 100GB free (more recommended for media storage)
- **RAM**: Minimum 4GB (8GB+ recommended)
- **Network**: Stable internet connection

### Current System Status
Your system has:
- Docker: 28.5.1 ✓
- Docker Compose: v2.40.0-desktop.1 ✓
- Available Space: 181GB ✓
- User: root (UID: 0, GID: 0)

**Important Note**: Running as root is detected. For better security, consider creating a dedicated user for this setup.

### Optional Requirements
- **Plex Pass**: Required for hardware transcoding (highly recommended)
- **VPN Service**: Recommended for qBittorrent (like NordVPN, ExpressVPN, Private Internet Access)

---

## Directory Structure Setup

The most critical aspect of this setup is the directory structure. Using a unified structure allows for **hard links** instead of copying files, saving massive amounts of disk space and time.

### Recommended Structure

```
/opt/
├── mediaserver/
│   ├── config/           # All service configurations
│   │   ├── plex/
│   │   ├── radarr/
│   │   ├── sonarr/
│   │   ├── jackett/
│   │   ├── qbittorrent/
│   │   └── overseerr/
│   └── data/            # All media and downloads
│       ├── downloads/   # qBittorrent downloads here
│       ├── movies/      # Radarr manages this
│       └── tv/          # Sonarr manages this
```

### Create the Directory Structure

Run these commands to create the necessary directories:

```bash
# Create main directory structure
sudo mkdir -p /opt/mediaserver/{config,data}/{plex,radarr,sonarr,jackett,qbittorrent,overseerr}
sudo mkdir -p /opt/mediaserver/data/{downloads,movies,tv}

# Create subdirectories for better organization
sudo mkdir -p /opt/mediaserver/data/downloads/{movies,tv,incomplete}

# Set proper permissions
# Note: Using 1000:1000 as standard user. Adjust if needed.
sudo chown -R 1000:1000 /opt/mediaserver
sudo chmod -R 775 /opt/mediaserver

# Verify structure
tree -L 3 /opt/mediaserver
```

**Why This Structure?**
- **Same Filesystem**: Downloads and media in `/opt/mediaserver/data` enables instant moves
- **Hard Links**: Radarr/Sonarr can create hard links instead of copies (saves space)
- **Atomic Moves**: File operations are instant (no copying time)
- **Organized**: Clear separation between config and data

---

## Docker Compose Configuration

Create a `docker-compose.yml` file in `/opt/mediaserver/`:

```bash
cd /opt/mediaserver
nano docker-compose.yml
```

### Complete docker-compose.yml

```yaml
version: "3.8"

services:
  # qBittorrent - Torrent Client
  qbittorrent:
    image: lscr.io/linuxserver/qbittorrent:latest
    container_name: qbittorrent
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/New_York
      - WEBUI_PORT=8080
    volumes:
      - /opt/mediaserver/config/qbittorrent:/config
      - /opt/mediaserver/data/downloads:/downloads
    ports:
      - 8080:8080
      - 6881:6881
      - 6881:6881/udp
    restart: unless-stopped
    networks:
      - mediaserver

  # Jackett - Indexer Proxy
  jackett:
    image: lscr.io/linuxserver/jackett:latest
    container_name: jackett
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/New_York
      - AUTO_UPDATE=true
    volumes:
      - /opt/mediaserver/config/jackett:/config
      - /opt/mediaserver/data/downloads:/downloads
    ports:
      - 9117:9117
    restart: unless-stopped
    networks:
      - mediaserver

  # Radarr - Movie Management
  radarr:
    image: lscr.io/linuxserver/radarr:latest
    container_name: radarr
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/New_York
    volumes:
      - /opt/mediaserver/config/radarr:/config
      - /opt/mediaserver/data:/data
    ports:
      - 7878:7878
    restart: unless-stopped
    depends_on:
      - qbittorrent
      - jackett
    networks:
      - mediaserver

  # Sonarr - TV Show Management
  sonarr:
    image: lscr.io/linuxserver/sonarr:latest
    container_name: sonarr
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/New_York
    volumes:
      - /opt/mediaserver/config/sonarr:/config
      - /opt/mediaserver/data:/data
    ports:
      - 8989:8989
    restart: unless-stopped
    depends_on:
      - qbittorrent
      - jackett
    networks:
      - mediaserver

  # Plex Media Server
  plex:
    image: lscr.io/linuxserver/plex:latest
    container_name: plex
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/New_York
      - VERSION=docker
      - PLEX_CLAIM=${PLEX_CLAIM}
    volumes:
      - /opt/mediaserver/config/plex:/config
      - /opt/mediaserver/data/movies:/movies
      - /opt/mediaserver/data/tv:/tv
    ports:
      - 32400:32400
      - 1900:1900/udp
      - 3005:3005
      - 5353:5353/udp
      - 8324:8324
      - 32410:32410/udp
      - 32412:32412/udp
      - 32413:32413/udp
      - 32414:32414/udp
      - 32469:32469
    restart: unless-stopped
    devices:
      - /dev/dri:/dev/dri  # For hardware transcoding (Intel/AMD)
    networks:
      - mediaserver

  # Overseerr - Request Management (Optional)
  overseerr:
    image: lscr.io/linuxserver/overseerr:latest
    container_name: overseerr
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/New_York
    volumes:
      - /opt/mediaserver/config/overseerr:/config
    ports:
      - 5055:5055
    restart: unless-stopped
    depends_on:
      - plex
      - radarr
      - sonarr
    networks:
      - mediaserver

networks:
  mediaserver:
    driver: bridge
```

### Important Configuration Notes

1. **PUID/PGID**: Set to `1000` (standard user). To find yours: `id`
   - For root (current): Use `0` for both, but create a dedicated user instead

2. **Timezone (TZ)**: Change to your timezone. Find yours: `timedatectl`
   - Examples: `America/New_York`, `Europe/London`, `Australia/Sydney`

3. **PLEX_CLAIM**: Required for initial Plex setup
   - Get your claim token: https://www.plex.tv/claim/
   - Valid for 4 minutes, use immediately

4. **Hardware Transcoding**: The `/dev/dri:/dev/dri` device mapping enables Intel/AMD GPU transcoding
   - For NVIDIA: Add `runtime: nvidia` and different device mapping
   - Requires Plex Pass subscription

### Create Environment File

```bash
cd /opt/mediaserver
nano .env
```

Add this content (replace with your actual claim token):

```env
PLEX_CLAIM=claim-xxxxxxxxxxxxxxxxxxxx
```

---

## Initial Setup

### Step 1: Verify Configuration

```bash
# Navigate to the directory
cd /opt/mediaserver

# Validate docker-compose.yml syntax
docker compose config

# Check if all required directories exist
ls -la /opt/mediaserver/config/
ls -la /opt/mediaserver/data/
```

### Step 2: Get Plex Claim Token

1. Visit: https://www.plex.tv/claim/
2. Sign in to your Plex account
3. Copy the claim token (starts with `claim-`)
4. Update `/opt/mediaserver/.env` with your token

**Note**: The token expires in 4 minutes. Get it right before starting containers.

### Step 3: Start the Services

```bash
# Start all services in detached mode
docker compose up -d

# View logs (optional)
docker compose logs -f

# Press Ctrl+C to exit logs
```

### Step 4: Verify All Services Are Running

```bash
# Check container status
docker compose ps

# All containers should show "running" state
```

Expected output:
```
NAME           STATUS          PORTS
jackett        Up 30 seconds   0.0.0.0:9117->9117/tcp
overseerr      Up 30 seconds   0.0.0.0:5055->5055/tcp
plex           Up 30 seconds   0.0.0.0:32400->32400/tcp, ...
qbittorrent    Up 30 seconds   0.0.0.0:8080->8080/tcp, ...
radarr         Up 30 seconds   0.0.0.0:7878->7878/tcp
sonarr         Up 30 seconds   0.0.0.0:8989->8989/tcp
```

### Step 5: Access Web Interfaces

All services are now accessible via your web browser:

- **Plex**: http://localhost:32400/web
- **Radarr**: http://localhost:7878
- **Sonarr**: http://localhost:8989
- **qBittorrent**: http://localhost:8080
- **Jackett**: http://localhost:9117
- **Overseerr**: http://localhost:5055

**Note**: Replace `localhost` with your server's IP if accessing remotely.

---

## Service Configuration

Now we'll configure each service in the correct order to ensure proper integration.

### 1. qBittorrent Configuration

**Access**: http://localhost:8080

**Default Credentials**:
- Username: `admin`
- Password: Check logs with `docker logs qbittorrent | grep "temporary password"`

**Configuration Steps**:

1. **Login** with default credentials
2. **Change Password**: Tools → Options → Web UI → Authentication
3. **Configure Downloads**:
   - Tools → Options → Downloads
   - Default Save Path: `/downloads/incomplete`
   - Keep incomplete torrents in: `/downloads/incomplete` (checked)
   - Copy .torrent files to: `/downloads` (optional)

4. **Configure BitTorrent Settings**:
   - Tools → Options → BitTorrent
   - Privacy: Enable Anonymous Mode (optional)
   - Torrent Queueing: Max active downloads: 3-5

5. **Connection Settings**:
   - Tools → Options → Connection
   - Listening Port: 6881 (or any preferred port)
   - Use UPnP/NAT-PMP: Enabled

6. **Advanced Settings**:
   - Tools → Options → Advanced
   - Network Interface: eth0 (or your primary interface)
   - Optional: Set up VPN if using one

**Recommended**: Set up categories for better organization:
- Right-click in torrents list → Add category
- Create categories: `movies`, `tv`, `music`, etc.

---

### 2. Jackett Configuration

**Access**: http://localhost:9117

**Configuration Steps**:

1. **Initial Setup**:
   - Set admin password (recommended)
   - Note down the API Key (top right corner)

2. **Add Indexers**:
   - Click "Add indexer"
   - Search for your preferred torrent sites
   - Popular public indexers:
     - The Pirate Bay
     - RARBG
     - 1337x
     - YTS (movies)
     - EZTV (TV shows)

3. **Configure Each Indexer**:
   - Click wrench icon next to indexer
   - Add credentials if it's a private tracker
   - Test the indexer (green checkmark should appear)
   - Save

4. **Test All Indexers**:
   - Click "Test All" at the top
   - Remove any non-working indexers

**Copy API Key**: You'll need this for Radarr and Sonarr configuration.

---

### 3. Radarr Configuration

**Access**: http://localhost:7878

**Initial Setup Wizard**:

1. **Authentication** (recommended):
   - Settings → General → Security
   - Authentication: Forms (Login page)
   - Username/Password: Set your credentials

2. **Media Management**:
   - Settings → Media Management
   - **Root Folders**:
     - Add `/data/movies`
   - **File Management**:
     - Rename Movies: Yes
     - Replace Illegal Characters: Yes
   - **Importing**:
     - Use Hardlinks instead of Copy: Yes (CRITICAL)
     - Import Extra Files: Yes (for subtitles, etc.)

3. **Add Download Client** (qBittorrent):
   - Settings → Download Clients → Add ('+' icon)
   - Select "qBittorrent"
   - Configuration:
     - Name: qBittorrent
     - Host: `qbittorrent` (container name)
     - Port: `8080`
     - Username: `admin`
     - Password: [your qBittorrent password]
     - Category: `movies`
   - Test → Save

4. **Add Indexers** (Jackett):
   - Settings → Indexers → Add ('+' icon)
   - Select "Torznab" → "Custom"
   - For each Jackett indexer:
     - Name: [Indexer Name]
     - URL: `http://jackett:9117/api/v2.0/indexers/[INDEXER-ID]/results/torznab/`
     - API Key: [Your Jackett API Key]
     - Categories: 2000,2010,2020,2030,2040,2050 (Movies)
   - Test → Save

   **Tip**: In Jackett, click "Copy Torznab Feed" for each indexer to get the exact URL

5. **Quality Profiles**:
   - Settings → Profiles
   - Review/edit the default profiles
   - Recommended: Use "HD-1080p" or create custom profile

6. **Connect List** (Optional):
   - Settings → Lists
   - Add lists like IMDb, Trakt, TMDb for automatic movie additions

---

### 4. Sonarr Configuration

**Access**: http://localhost:8989

**Initial Setup Wizard** (very similar to Radarr):

1. **Authentication** (recommended):
   - Settings → General → Security
   - Authentication: Forms (Login page)
   - Username/Password: Set your credentials

2. **Media Management**:
   - Settings → Media Management
   - **Root Folders**:
     - Add `/data/tv`
   - **Episode Naming**:
     - Rename Episodes: Yes
     - Replace Illegal Characters: Yes
     - Standard Episode Format: Keep default or customize
   - **Importing**:
     - Use Hardlinks instead of Copy: Yes (CRITICAL)
     - Import Extra Files: Yes

3. **Add Download Client** (qBittorrent):
   - Settings → Download Clients → Add ('+' icon)
   - Select "qBittorrent"
   - Configuration:
     - Name: qBittorrent
     - Host: `qbittorrent` (container name)
     - Port: `8080`
     - Username: `admin`
     - Password: [your qBittorrent password]
     - Category: `tv`
   - Test → Save

4. **Add Indexers** (Jackett):
   - Settings → Indexers → Add ('+' icon)
   - Select "Torznab" → "Custom"
   - For each Jackett indexer:
     - Name: [Indexer Name]
     - URL: `http://jackett:9117/api/v2.0/indexers/[INDEXER-ID]/results/torznab/`
     - API Key: [Your Jackett API Key]
     - Categories: 5000,5010,5020,5030,5040,5050 (TV)
   - Test → Save

5. **Quality Profiles**:
   - Settings → Profiles
   - Review/edit the default profiles
   - Recommended: Use "HD-1080p" or create custom profile

6. **Series Type**:
   - Understand the difference:
     - **Standard**: Normal TV shows (aired date based)
     - **Daily**: Daily shows (date-based naming)
     - **Anime**: Japanese animation (absolute numbering)

---

### 5. Plex Configuration

**Access**: http://localhost:32400/web

**Initial Setup Wizard**:

1. **Sign In**: Use your Plex account (the one you used for claim token)

2. **Server Setup**:
   - Name your server (e.g., "Home Media Server")
   - Allow access outside your home: Yes (if you want remote access)

3. **Add Libraries**:

   **Movies Library**:
   - Add Library → Movies
   - Add folder: `/movies`
   - Advanced:
     - Scanner: Plex Movie Scanner
     - Agent: Plex Movie
     - Enable video preview thumbnails: Yes

   **TV Shows Library**:
   - Add Library → TV Shows
   - Add folder: `/tv`
   - Advanced:
     - Scanner: Plex Series Scanner
     - Agent: Plex Series
     - Enable video preview thumbnails: Yes

4. **Server Settings**:
   - Settings → Server → General
   - Check "Enable Plex Media Server debug logging" (optional, for troubleshooting)

5. **Transcoder Settings** (Plex Pass Required):
   - Settings → Server → Transcoder
   - Show Advanced → Enable
   - Use hardware acceleration when available: Yes
   - Use hardware-accelerated video encoding: Yes
   - Transcoder temporary directory: `/transcode` (optional, see note below)

**Hardware Transcoding Note**:
- Verify it's working by checking Dashboard during playback
- Look for "(hw)" next to the video format when transcoding
- If not working, check `/dev/dri` permissions: `ls -l /dev/dri`

6. **Network Settings**:
   - Settings → Server → Network
   - Custom server access URLs: Add your external URL/IP if needed
   - Enable Relay: Yes (for fallback remote access)

7. **Library Settings**:
   - Settings → Server → Library
   - Scan my library automatically: Yes
   - Run a partial scan when changes are detected: Yes
   - Empty trash automatically after every scan: No (keep for 7 days)

---

### 6. Overseerr Configuration

**Access**: http://localhost:5055

**Initial Setup Wizard**:

1. **Sign In with Plex**:
   - Click "Sign in with Plex"
   - Authorize Overseerr to access your Plex account

2. **Plex Server Setup**:
   - Select your Plex server
   - Sync libraries: Select your Movies and TV Shows libraries
   - Click "Continue"

3. **Radarr Configuration**:
   - Enable: Yes
   - Server Name: Radarr
   - Hostname/IP: `radarr`
   - Port: `7878`
   - API Key: Get from Radarr → Settings → General → Security → API Key
   - URL Base: Leave blank
   - Quality Profile: Select default (e.g., HD-1080p)
   - Root Folder: `/data/movies`
   - Minimum Availability: Released
   - Tags: Leave blank (optional)
   - Test → Save

4. **Sonarr Configuration**:
   - Enable: Yes
   - Server Name: Sonarr
   - Hostname/IP: `sonarr`
   - Port: `8989`
   - API Key: Get from Sonarr → Settings → General → Security → API Key
   - URL Base: Leave blank
   - Quality Profile: Select default (e.g., HD-1080p)
   - Root Folder: `/data/tv`
   - Language Profile: Deprecated (skip)
   - Tags: Leave blank (optional)
   - Anime Quality Profile: Same as regular or create separate
   - Anime Root Folder: Same or separate (e.g., `/data/anime`)
   - Season Folders: Yes
   - Test → Save

5. **Finish Setup**:
   - Set up admin account (if not already configured)
   - Configure notification settings (optional)
   - Set up user permissions

---

## Integration Steps

Now that all services are configured, let's verify the integration between them.

### Verify Download Chain

```
Overseerr/Manual → Radarr/Sonarr → Jackett → qBittorrent → Media Library → Plex
```

### Test Movie Download (via Radarr)

1. **Go to Radarr**: http://localhost:7878
2. **Add a Movie**:
   - Click "Add New" (or "Add Movie")
   - Search for a movie (try an older, smaller movie for testing)
   - Select the movie
   - Configure:
     - Root Folder: `/data/movies`
     - Monitor: Yes
     - Quality Profile: HD-1080p
     - Search on Add: Yes (important for immediate testing)
   - Click "Add Movie"

3. **Monitor Progress**:
   - Go to Activity → Queue
   - You should see the movie being searched
   - If found, it will be sent to qBittorrent
   - Go to qBittorrent: http://localhost:8080
   - Verify the torrent is downloading in the "movies" category

4. **Wait for Completion**:
   - Once download completes in qBittorrent
   - Radarr will automatically import it to `/data/movies`
   - File will be renamed according to your naming scheme
   - Check Radarr → Movies → [Your Movie] for status

5. **Verify in Plex**:
   - Go to Plex: http://localhost:32400/web
   - Navigate to Movies library
   - The movie should appear (may take a few minutes)
   - If not, manually scan library: Library → ... → Scan Library Files

### Test TV Show Download (via Sonarr)

1. **Go to Sonarr**: http://localhost:8989
2. **Add a TV Show**:
   - Click "Add New" (or "Add Series")
   - Search for a show (try a popular show with recent episodes)
   - Select the show
   - Configure:
     - Root Folder: `/data/tv`
     - Monitor: All Episodes (or choose specific seasons)
     - Quality Profile: HD-1080p
     - Series Type: Standard (usually)
     - Season Folder: Yes
     - Search on Add: Yes
   - Click "Add Series"

3. **Monitor Progress**:
   - Go to Activity → Queue
   - Episodes will be searched via Jackett
   - Downloads sent to qBittorrent
   - Verify in qBittorrent under "tv" category

4. **Wait for Completion**:
   - After download, Sonarr imports to `/data/tv/[Show Name]/Season X/`
   - Files renamed according to your format
   - Check Sonarr → Series → [Your Show] for episode status

5. **Verify in Plex**:
   - Go to Plex → TV Shows library
   - The show should appear with available episodes
   - Metadata and posters should be automatically fetched

### Test Request via Overseerr

1. **Go to Overseerr**: http://localhost:5055
2. **Search for Content**:
   - Use the search bar
   - Search for a movie or TV show
3. **Request**:
   - Click on the content
   - Click "Request"
   - Choose quality profile (if multiple configured)
   - Submit request
4. **Automatic Processing**:
   - Request automatically sent to Radarr/Sonarr
   - Follow steps above to monitor progress
5. **Notification**:
   - Once available, you'll get notification (if configured)
   - Content appears in Plex

---

## Testing

### Quick Test Checklist

Run through this checklist to ensure everything is working:

- [ ] All containers are running: `docker compose ps`
- [ ] qBittorrent is accessible and configured
- [ ] Jackett has working indexers
- [ ] Radarr can connect to qBittorrent and Jackett
- [ ] Sonarr can connect to qBittorrent and Jackett
- [ ] Plex libraries are configured for `/movies` and `/tv`
- [ ] Overseerr is connected to Plex, Radarr, and Sonarr
- [ ] Test movie download completes successfully
- [ ] Test TV show episode download completes successfully
- [ ] Downloaded media appears in Plex
- [ ] Plex can stream the media without issues

### Verify Hard Links Are Working

Hard links save disk space. Verify they're working:

```bash
# After a successful download and import
# Check the inode numbers - they should match for hardlinked files

# Find a movie file
find /opt/mediaserver/data/movies -type f -name "*.mkv" | head -1

# Check its inode
ls -li /opt/mediaserver/data/movies/[Movie Name]/*.mkv

# Check if the same file exists in downloads
ls -li /opt/mediaserver/data/downloads/movies/*.mkv

# If inode numbers match, hard linking is working!
```

If inode numbers are different, files are being copied. Review:
1. Both paths are on the same filesystem
2. Radarr/Sonarr have "Use Hardlinks instead of Copy" enabled
3. File permissions are correct

### Performance Test

1. **Stream a file in Plex**
2. **Open Plex Dashboard** (top right → More → Dashboard)
3. **Check transcoding status**:
   - Direct Play: Best (no transcoding)
   - Transcode (hw): Good (hardware acceleration working)
   - Transcode: Acceptable (CPU transcoding)

---

## Maintenance

### Regular Maintenance Tasks

#### Daily
- Check download queue in qBittorrent
- Remove completed/stale torrents

#### Weekly
- Review Radarr/Sonarr queues for failures
- Check disk space: `df -h /opt/mediaserver`
- Review Plex streaming activity

#### Monthly
- Update all containers:
  ```bash
  cd /opt/mediaserver
  docker compose pull
  docker compose up -d
  ```
- Clean up old logs
- Review and adjust quality profiles
- Update Jackett indexers

### Backup Strategy

**What to Backup**:
1. **Configuration files** (critical):
   ```bash
   tar -czf mediaserver-config-backup-$(date +%Y%m%d).tar.gz /opt/mediaserver/config/
   ```

2. **Docker Compose file**:
   ```bash
   cp /opt/mediaserver/docker-compose.yml ~/docker-compose.yml.backup
   ```

3. **Plex Database** (most important):
   ```bash
   tar -czf plex-db-backup-$(date +%Y%m%d).tar.gz /opt/mediaserver/config/plex/Library/Application\ Support/Plex\ Media\ Server/Plug-in\ Support/Databases/
   ```

**What NOT to Backup**:
- Media files (too large, can be re-downloaded)
- Download queue (temporary)
- Cache files

**Recommended Backup Schedule**:
- Configuration: Weekly
- Plex Database: Daily (automated)
- Docker Compose: After any changes

### Log Management

**View logs for a specific service**:
```bash
# Real-time logs
docker logs -f [container_name]

# Last 100 lines
docker logs --tail 100 [container_name]

# Logs since specific time
docker logs --since 24h [container_name]
```

**Clear logs** (if they get too large):
```bash
# Check log sizes
docker ps -q | xargs -I {} docker inspect --format='{{.Name}} {{.LogPath}}' {} | xargs -I {} sh -c 'echo {}; ls -lh {}'

# Truncate logs (careful!)
truncate -s 0 $(docker inspect --format='{{.LogPath}}' [container_name])
```

### Updating Containers

```bash
cd /opt/mediaserver

# Pull latest images
docker compose pull

# Recreate containers with new images
docker compose up -d

# Remove old images
docker image prune -a
```

### Managing Disk Space

**Check space usage**:
```bash
# Overall usage
df -h /opt/mediaserver

# Per directory
du -sh /opt/mediaserver/data/*

# Largest files
find /opt/mediaserver/data/downloads -type f -exec du -h {} + | sort -rh | head -20
```

**Clean up downloads**:
```bash
# Remove completed torrents older than 14 days
find /opt/mediaserver/data/downloads -type f -mtime +14 -delete

# Clean incomplete downloads
rm -rf /opt/mediaserver/data/downloads/incomplete/*
```

---

## Troubleshooting

### Container Issues

**Container won't start**:
```bash
# Check container logs
docker logs [container_name]

# Check all container statuses
docker compose ps

# Restart specific container
docker compose restart [container_name]

# Restart all containers
docker compose restart
```

**Permission issues**:
```bash
# Fix ownership (use your actual PUID/PGID)
sudo chown -R 1000:1000 /opt/mediaserver

# Fix permissions
sudo chmod -R 775 /opt/mediaserver/data
sudo chmod -R 755 /opt/mediaserver/config
```

### Network Issues

**Containers can't communicate**:
```bash
# Check network
docker network ls
docker network inspect mediaserver_mediaserver

# Verify containers are on the same network
docker inspect [container_name] | grep -A 10 Networks
```

**Can't access web interface**:
```bash
# Check if port is listening
netstat -tuln | grep [PORT]

# Check firewall (if enabled)
sudo ufw status
sudo ufw allow [PORT]/tcp
```

### Radarr/Sonarr Issues

**Movies/shows not downloading**:
1. Check Radarr/Sonarr logs: Activity → History
2. Verify Jackett indexers are working
3. Test download client connection
4. Check search terms and quality requirements
5. Verify file size limits aren't too restrictive

**Import failures**:
1. Check file permissions on `/data`
2. Verify paths in download client match Radarr/Sonarr
3. Check logs for specific error messages
4. Ensure files are complete in qBittorrent

**Downloads stuck in queue**:
```bash
# Check qBittorrent logs
docker logs qbittorrent | tail -100

# Check Radarr/Sonarr connection
# Settings → Download Clients → Test
```

### Jackett Issues

**Indexers failing**:
1. Test individual indexers (wrench icon → Test)
2. Update indexer definitions (top right → Update All)
3. Check if sites are down (test in browser)
4. For private trackers, verify credentials

**No results in Radarr/Sonarr**:
1. Verify API key is correct
2. Check Torznab URL format
3. Test manually in Jackett
4. Verify category IDs are correct

### Plex Issues

**Media not appearing**:
1. Manually scan library: Library → ... → Scan Library Files
2. Check file permissions: `ls -la /opt/mediaserver/data/movies`
3. Verify file naming matches Plex standards
4. Check Plex logs: `/opt/mediaserver/config/plex/Library/Application Support/Plex Media Server/Logs/`

**Transcoding not working**:
```bash
# Check GPU access
ls -l /dev/dri

# Verify GPU is available to container
docker exec plex ls -l /dev/dri

# Check Plex logs during playback
docker logs plex | grep -i transcode
```

**Buffering issues**:
1. Check network bandwidth
2. Verify transcoding settings
3. Consider Direct Play/Direct Stream instead
4. Check server CPU/RAM usage: `docker stats`

### qBittorrent Issues

**Downloads not starting**:
1. Check connection status in qBittorrent
2. Verify port forwarding (if behind router)
3. Check VPN connection (if using)
4. Test with a known-working torrent

**Slow speeds**:
1. Check global speed limits: Tools → Options → Speed
2. Verify connection limits: Tools → Options → BitTorrent
3. Try different indexers/torrents
4. Check ISP throttling
5. Verify VPN isn't limiting speeds

### Overseerr Issues

**Can't connect to Radarr/Sonarr**:
1. Verify API keys are correct
2. Check hostnames (use container names, not localhost)
3. Test connection in Overseerr settings
4. Check Docker network connectivity

**Requests not processing**:
1. Check Overseerr logs: Settings → Logs
2. Verify Radarr/Sonarr are receiving requests
3. Check Radarr/Sonarr activity queues
4. Ensure root folders are correctly configured

### General Debugging

**Check all service statuses**:
```bash
# Container status
docker compose ps

# Resource usage
docker stats

# Recent logs from all services
docker compose logs --tail=50

# Follow logs for specific service
docker compose logs -f [service_name]
```

**Restart everything**:
```bash
cd /opt/mediaserver

# Stop all
docker compose down

# Start all
docker compose up -d

# Or restart without stopping
docker compose restart
```

**Complete reset** (nuclear option):
```bash
# WARNING: This will delete all configurations!
# Backup first if needed

cd /opt/mediaserver
docker compose down -v
sudo rm -rf config/*
docker compose up -d

# Then reconfigure everything
```

### Getting Help

If you're still stuck:

1. **Check logs**: Most issues show up in container logs
2. **Search documentation**:
   - LinuxServer.io: https://docs.linuxserver.io/
   - Servarr Wiki: https://wiki.servarr.com/
   - Plex Support: https://support.plex.tv/
3. **Community forums**:
   - r/Plex, r/Radarr, r/Sonarr on Reddit
   - LinuxServer.io Discord
   - Plex Forums
4. **Provide details when asking for help**:
   - Docker version
   - Docker Compose file
   - Relevant logs
   - What you've already tried

---

## Advanced Topics

### Adding VPN to qBittorrent

For enhanced privacy, you can route qBittorrent through a VPN:

Replace the qBittorrent service in docker-compose.yml with:

```yaml
qbittorrent:
  image: dyonr/qbittorrentvpn
  container_name: qbittorrent
  privileged: true
  environment:
    - PUID=1000
    - PGID=1000
    - TZ=America/New_York
    - VPN_ENABLED=yes
    - VPN_TYPE=openvpn
    - LAN_NETWORK=192.168.1.0/24  # Adjust to your network
    - NAME_SERVERS=1.1.1.1,1.0.0.1
  volumes:
    - /opt/mediaserver/config/qbittorrent:/config
    - /opt/mediaserver/data/downloads:/downloads
  ports:
    - 8080:8080
  restart: unless-stopped
  networks:
    - mediaserver
```

Place your VPN configuration (.ovpn file) in:
`/opt/mediaserver/config/qbittorrent/openvpn/`

### Hardware Transcoding for NVIDIA GPU

If you have an NVIDIA GPU:

1. Install NVIDIA Container Toolkit
2. Update Plex service in docker-compose.yml:

```yaml
plex:
  image: lscr.io/linuxserver/plex:latest
  runtime: nvidia
  environment:
    - NVIDIA_VISIBLE_DEVICES=all
    - NVIDIA_DRIVER_CAPABILITIES=compute,video,utility
  # ... rest of configuration
```

### Adding More Services

Popular additions:

- **Prowlarr**: Better indexer management (replaces Jackett)
- **Bazarr**: Automatic subtitle downloads
- **Tautulli**: Plex statistics and monitoring
- **Portainer**: Docker GUI management
- **Watchtower**: Automatic container updates
- **Organizr**: Unified dashboard for all services

### Remote Access

**Plex Remote Access**:
- Easiest: Use Plex's built-in relay
- Better performance: Set up port forwarding (32400)
- Most secure: Set up reverse proxy (Nginx/Caddy)

**For other services**:
- Use a reverse proxy (Nginx Proxy Manager recommended)
- Set up Authelia for authentication
- Use Cloudflare Tunnel for zero-trust access

### Automation Scripts

**Automatic cleanup script** (`/opt/mediaserver/cleanup.sh`):

```bash
#!/bin/bash
# Clean downloads older than 14 days
find /opt/mediaserver/data/downloads -type f -mtime +14 -delete

# Remove empty directories
find /opt/mediaserver/data/downloads -type d -empty -delete

# Update containers
cd /opt/mediaserver
docker compose pull
docker compose up -d

# Cleanup Docker
docker image prune -af
docker volume prune -f
```

Add to crontab:
```bash
# Weekly on Sunday at 3 AM
0 3 * * 0 /opt/mediaserver/cleanup.sh
```

---

## Quick Reference

### Service URLs

| Service | URL | Default Port |
|---------|-----|--------------|
| Plex | http://localhost:32400/web | 32400 |
| Radarr | http://localhost:7878 | 7878 |
| Sonarr | http://localhost:8989 | 8989 |
| qBittorrent | http://localhost:8080 | 8080 |
| Jackett | http://localhost:9117 | 9117 |
| Overseerr | http://localhost:5055 | 5055 |

### Common Commands

```bash
# Navigate to directory
cd /opt/mediaserver

# Start all services
docker compose up -d

# Stop all services
docker compose down

# Restart all services
docker compose restart

# View status
docker compose ps

# View logs
docker compose logs -f

# Update containers
docker compose pull && docker compose up -d

# Clean up
docker system prune -a
```

### Directory Paths

| Purpose | Host Path | Container Path |
|---------|-----------|----------------|
| Movies | /opt/mediaserver/data/movies | /movies (Plex), /data/movies (Radarr) |
| TV Shows | /opt/mediaserver/data/tv | /tv (Plex), /data/tv (Sonarr) |
| Downloads | /opt/mediaserver/data/downloads | /downloads |
| Configs | /opt/mediaserver/config/* | /config |

### Category IDs for Jackett

| Media Type | Categories |
|------------|------------|
| Movies | 2000,2010,2020,2030,2040,2050 |
| TV Shows | 5000,5010,5020,5030,5040,5050 |
| Music | 3000,3010,3020,3030,3040 |
| Books | 7000,7010,7020 |

---

## Conclusion

You now have a fully automated media server setup! The system will:

1. Accept requests via Overseerr (or manual additions)
2. Automatically search for and download content
3. Organize and rename files properly
4. Make content available in Plex immediately
5. Stream to any device, anywhere

**Next Steps**:
- Set up user accounts in Overseerr for family/friends
- Configure quality profiles to match your preferences
- Set up notifications (Discord, email, etc.)
- Explore additional services (Bazarr, Tautulli)
- Set up automated backups
- Configure remote access properly

**Important Reminders**:
- Regularly update containers for security
- Monitor disk space
- Use VPN for downloads (recommended)
- Respect copyright laws in your jurisdiction
- Keep regular backups of configurations

Enjoy your automated media server!

---

## Additional Resources

- **LinuxServer.io Documentation**: https://docs.linuxserver.io/
- **Servarr Wiki**: https://wiki.servarr.com/
- **Plex Support**: https://support.plex.tv/
- **TRaSH Guides**: https://trash-guides.info/ (highly recommended for optimization)
- **r/Plex**: https://reddit.com/r/Plex
- **r/Sonarr**: https://reddit.com/r/Sonarr
- **r/Radarr**: https://reddit.com/r/Radarr

---

**Guide Version**: 1.0
**Last Updated**: October 2025
**System Tested**: Zorin OS (Ubuntu-based) with Docker Desktop
**Author**: Claude
