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

### Method 1: Using Docker Compose in Unraid

#### Step 1: Upload Files to Unraid

1. Enable SMB sharing on your Unraid server
2. Copy the filebrowser repository to a share (e.g., `/mnt/user/appdata/filebrowser`)
3. Or clone directly via terminal:

```bash
mkdir -p /mnt/user/appdata/filebrowser
cd /mnt/user/appdata/filebrowser
git clone https://github.com/your-username/filebrowser.git .
```

#### Step 2: Create docker-compose.yml

Create `docker-compose.yml` in the appdata folder:

```yaml
version: '3.8'

services:
  filebrowser:
    build:
      context: /mnt/user/appdata/filebrowser
      dockerfile: _docker/Dockerfile.slim
    container_name: filebrowser
    restart: unless-stopped
    ports:
      - "8080:80"
    volumes:
      - /mnt/user/your_files:/srv
      - filebrowser_db:/database
      - filebrowser_config:/config
    environment:
      - FB_DATABASE=/database/filebrowser.db

volumes:
  filebrowser_db:
    driver: local
  filebrowser_config:
    driver: local
```

#### Step 3: Deploy Using Command Line

1. SSH into your Unraid server
2. Run:

```bash
cd /mnt/user/appdata/filebrowser
docker-compose up -d --build
```

#### Step 4: Verify in Unraid UI

1. Go to **Docker** tab
2. You should see the `filebrowser` container running
3. Check logs if needed: `docker logs filebrowser`

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
