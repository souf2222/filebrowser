# Deployment Guide

This guide covers deployment of FileBrowser using Docker Compose with a self-built image.

---

## Quick Start with Docker Compose

### Prerequisites

- Docker and Docker Compose installed
- Git to clone the repository

### Step 1: Clone the Repository

```bash
git clone https://github.com/your-username/filebrowser.git
cd filebrowser
```

### Step 2: Build and Deploy

```bash
docker-compose up -d --build
```

This will:
1. Build the Docker image from the local source
2. Start the container
3. Mount the necessary volumes

### Step 3: Access FileBrowser

Access FileBrowser at http://localhost:8080

Default credentials: `admin` / `admin`

---

## Docker Compose Configuration

Create a `docker-compose.yml` file in the repository root:

```yaml
version: '3.8'

services:
  filebrowser:
    build:
      context: .
      dockerfile: _docker/Dockerfile.slim
    container_name: filebrowser
    restart: unless-stopped
    ports:
      - "8080:80"
    volumes:
      - files:/srv
      - database:/database
      - config:/config
    environment:
      - FB_DATABASE=/database/filebrowser.db
    healthcheck:
      test: ["CMD", "wget", "--spider", "-q", "http://localhost/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 10s

volumes:
  files:
    driver: local
  database:
    driver: local
  config:
    driver: local
```

### Configuration Options

| Setting | Description |
|---------|-------------|
| `ports` | Map host port 8080 to container port 80 |
| `volumes.files` | Your file storage location (add path mapping) |
| `volumes.database` | Persistent database storage |
| `volumes.config` | Persistent configuration storage |
| `restart` | Container restart policy |

### Customizing File Storage

To use a specific host directory for files, modify the volumes section:

```yaml
volumes:
  - /path/to/your/files:/srv
  - database:/database
  - config:/config
```

---

## Unraid Deployment

### Prerequisites

- Unraid 6.9.0 or later
- Docker plugin enabled
- Git plugin (optional, for cloning)

### Method 1: Using Docker Compose in Unraid (Recommended)

#### Step 1: Where to Clone

Clone to your **appdata** folder (recommended):

```bash
mkdir -p /mnt/user/appdata/filebrowser
cd /mnt/user/appdata/filebrowser
git clone https://github.com/your-username/filebrowser.git .
```

**Recommended location:** `/mnt/user/appdata/filebrowser`

**Why this location?**
- Persists across reboots
- Easy to backup
- Separated from your file storage
- On your array or cache pool

#### Step 2: Configure File Storage

Edit `docker-compose.yml` and set your file path:

```bash
cd /mnt/user/appdata/filebrowser
nano docker-compose.yml
```

Change the volume mapping:
```yaml
volumes:
  # Replace 'files:/srv' with your actual file location:
  - /mnt/user/your_media:/srv  # Your movies, documents, etc.
  - filebrowser_db:/database
  - filebrowser_config:/config
```

#### Step 3: Deploy

```bash
cd /mnt/user/appdata/filebrowser
docker-compose up -d --build
```

#### Step 4: Access FileBrowser

Open browser to: `http://<unraid-ip>:8080`

Default login: `admin` / `admin`

#### Updating (Future)

To update to the latest version:

```bash
cd /mnt/user/appdata/filebrowser
git pull
docker-compose down
docker-compose up -d --build
```

### Method 2: Using Unraid's Docker Tab (Manual Template)

If you prefer not to use docker-compose:

1. Go to **Docker** tab → **Add Container**
2. Configure:

| Setting | Value |
|---------|-------|
| **Name** | `filebrowser` |
| **Repository** | `filebrowser` (change to your image name) |
| **Registry Tag** | `latest` |

3. **Network**: Host or Bridge
4. **Volumes**:

| Container Path | Host Path |
|---------------|-----------|
| `/srv` | `/mnt/user/your_files` |
| `/database` | `/mnt/user/appdata/filebrowser/database` |
| `/config` | `/mnt/user/appdata/filebrowser/config` |

5. **Ports** (if Bridge):
   - `80` → `8080`

6. **Extra Parameters**: `--restart=unless-stopped`

---

### Method 3: Unraid with Traefik (Reverse Proxy)

For HTTPS with automatic SSL certificates:

#### Step 1: Clone Repository

```bash
mkdir -p /mnt/user/appdata/traefik
mkdir -p /mnt/user/appdata/filebrowser
cd /mnt/user/appdata/filebrowser
git clone https://github.com/your-username/filebrowser.git .
```

#### Step 2: Create Traefik Directories

```bash
mkdir -p /mnt/user/appdata/traefik/letsencrypt
mkdir -p /mnt/user/appdata/traefik/certs
```

#### Step 3: Create Traefik Configuration

```bash
nano /mnt/user/appdata/traefik/traefik.yml
```

```yaml
global:
  checkNewVersion: false
  sendAnonymousUsage: false

api:
  dashboard: true
  insecure: true

providers:
  docker:
    endpoint: "unix:///var/run/docker.sock"
    network: fb_network
    exposedByDefault: false

entryPoints:
  http:
    address: ":80"
    http:
      redirections:
        entryPoint:
          to: https
          scheme: https
          permanent: true
  https:
    address: ":443"

certificatesResolvers:
  letsencrypt:
    acme:
      email: "your-email@example.com"
      storage: "/letsencrypt/acme.json"
      httpChallenge:
        entryPoint: http

log:
  level: INFO
```

#### Step 4: Create docker-compose.yml with Traefik

Create at `/mnt/user/appdata/filebrowser/docker-compose.yml`:

```yaml
version: '3.8'

services:
  traefik:
    image: traefik:v3.0
    container_name: traefik
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
      - "8080:8080"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /mnt/user/appdata/traefik/traefik.yml:/traefik.yml:ro
      - /mnt/user/appdata/traefik/letsencrypt:/letsencrypt
    networks:
      - fb_network
    labels:
      - "traefik.enable=true"

  filebrowser:
    build:
      context: /mnt/user/appdata/filebrowser
      dockerfile: _docker/Dockerfile.slim
    container_name: filebrowser
    restart: unless-stopped
    networks:
      - fb_network
    volumes:
      - /mnt/user/your_files:/srv
      - filebrowser_db:/database
      - filebrowser_config:/config
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.filebrowser.rule=Host(`files.your-domain.com`)"
      - "traefik.http.routers.filebrowser.entrypoints=https"
      - "traefik.http.routers.filebrowser.tls.certresolver=letsencrypt"
      - "traefik.http.services.filebrowser.loadbalancer.server.port=80"

networks:
  fb_network:
    driver: bridge

volumes:
  filebrowser_db:
    driver: local
  filebrowser_config:
    driver: local
```

#### Step 5: Start the Stack

```bash
cd /mnt/user/appdata/filebrowser
docker-compose up -d --build
```

#### Step 6: Access

- FileBrowser: `https://files.your-domain.com`
- Traefik Dashboard: `https://your-unraid-ip:8080`

**Note:** Replace `your-domain.com` and `your-email@example.com` with your actual values.

---

## Updating FileBrowser

### Using Docker Compose

```bash
cd /path/to/filebrowser
git pull
docker-compose up -d --build
```

### Manual Update (Unraid CLI)

```bash
cd /mnt/user/appdata/filebrowser
git pull
docker-compose down
docker-compose up -d --build
```

---

## Custom Configuration

### Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `FB_DATABASE` | `./filebrowser.db` | Database file path |
| `FB_CONFIG` | `./config.json` | Configuration file path |
| `FB_LOG_DIR` | `` | Log directory |

### Creating Custom Config

1. Create a config file:

```bash
mkdir -p /path/to/config
cat > /path/to/config/settings.json <<EOF
{
  "port": 80,
  "baseURL": "/",
  "logging": true,
  "theme": "dark"
}
EOF
```

2. Update docker-compose.yml:

```yaml
volumes:
  - /path/to/your/files:/srv
  - /path/to/config/settings.json:/config/settings.json
  - database:/database
```

---

## Troubleshooting

### Container Won't Start

```bash
# Check logs
docker logs filebrowser

# Common issues:
# - Port already in use (change host port in docker-compose.yml)
# - Invalid volume paths (verify paths exist)
```

### Can't Access Web Interface

1. Verify container is running:
   ```bash
   docker ps | grep filebrowser
   ```

2. Check firewall settings
3. Verify correct IP address

### Permission Issues

```bash
# Fix permissions on file storage
chmod -R 755 /path/to/your/files
```

---

## Production Deployment

### Reverse Proxy with Nginx

```nginx
server {
    listen 80;
    server_name files.example.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl;
    server_name files.example.com;

    ssl_certificate /etc/letsencrypt/live/files.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/files.example.com/privkey.pem;

    location / {
        proxy_pass http://localhost:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### Docker Compose with Production Settings

```yaml
version: '3.8'

services:
  filebrowser:
    build:
      context: .
      dockerfile: _docker/Dockerfile.slim
    container_name: filebrowser
    restart: always
    ports:
      - "127.0.0.1:8080:80"  # Listen only on localhost
    volumes:
      - /mnt/user/your_files:/srv
      - filebrowser_db:/database
      - filebrowser_config:/config
    environment:
      - FB_DATABASE=/database/filebrowser.db
    healthcheck:
      test: ["CMD", "wget", "--spider", "-q", "http://localhost/health"]
      interval: 30s
      timeout: 10s
      retries: 3

volumes:
  filebrowser_db:
    driver: local
  filebrowser_config:
    driver: local
```

---

## Support

- **Issues**: https://github.com/filebrowser/filebrowser/issues
- **Documentation**: https://filebrowser.com/docs
