# Deployment Guide

This guide covers various deployment methods for FileBrowser.

## Quick Start with Docker

The easiest way to deploy FileBrowser is using Docker:

```bash
docker run -d \
  --name filebrowser \
  -p 8080:80 \
  -v /path/to/files:/srv \
  -v filebrowser_data:/database \
  -v filebrowser_config:/config \
  filebrowser/filebrowser
```

Then access FileBrowser at http://localhost:8080

Default credentials: `admin` / `admin`

---

## Unraid Deployment

This section provides step-by-step instructions for deploying FileBrowser on Unraid.

### Prerequisites

- Unraid 6.9.0 or later
- Access to the Unraid web interface
- A share configured for file storage (or create a new one)

### Method 1: Using Unraid's Docker Tab (Recommended)

#### Step 1: Open Docker Settings

1. Log in to your Unraid web interface
2. Click on the **Docker** tab
3. Click **Add Container**

#### Step 2: Configure Container

Fill in the following settings:

| Setting | Value |
|---------|-------|
| **Name** | `filebrowser` |
| **Repository** | `filebrowser/filebrowser` |
| **Registry Tag** | `latest` (or specific version like `v3.0.0`) |

#### Step 3: Network Settings

| Setting | Value |
|---------|-------|
| **Network Type** | `Host` (or `Bridge` if you need a specific port) |
| **Port** | `8080` (if using Bridge) |

#### Step 4: Volume Settings

Click **Add Path** for each mapping:

| Container Path | Host Path | Description |
|---------------|-----------|-------------|
| `/srv` | `/mnt/user/your_share` | Your file storage location |
| `/database` | `/mnt/user/appdata/filebrowser/database` | Database storage |
| `/config` | `/mnt/user/appdata/filebrowser/config` | Configuration storage |

#### Step 5: Port Settings (if using Bridge)

| Container Port | Host Port | Description |
|---------------|-----------|-------------|
| `80` | `8080` | Web interface port |

#### Step 6: Extra Parameters

Add these under **Extra Parameters**:

```bash
--restart=always
--memory=512M
```

#### Step 7: Advanced Settings (Optional)

For better performance, add these environment variables:

| Variable | Value | Description |
|----------|-------|-------------|
| `FB_DATABASE` | `/database/filebrowser.db` | Database location |
| `FB_CONFIG` | `/config/settings.json` | Config file location |

#### Step 8: Create Container

1. Click **Apply** to create and start the container
2. Wait for the container to initialize (check the logs)

#### Step 9: Initial Login

1. Access FileBrowser at `http://<unraid-ip>:8080`
2. Login with default credentials:
   - **Username**: `admin`
   - **Password**: `admin`

### Method 2: Using Unraid's Community Applications

1. Go to the **Apps** tab in Unraid
2. Search for "FileBrowser"
3. Click **Install** on the official template
4. Configure the settings as described above
5. Click **Apply**

### Method 3: Command Line (Via Terminal)

1. SSH into your Unraid server
2. Run the following command:

```bash
docker run -d \
  --name filebrowser \
  --restart=always \
  -p 8080:80 \
  -v /mnt/user/your_share:/srv \
  -v /mnt/user/appdata/filebrowser/database:/database \
  -v /mnt/user/appdata/filebrowser/config:/config \
  filebrowser/filebrowser
```

### Configuration

#### Default Configuration

FileBrowser uses reasonable defaults. To customize:

1. Create a configuration file at `/mnt/user/appdata/filebrowser/config/filebrowser.json`
2. Restart the container

Example configuration:

```json
{
  "port": 80,
  "baseURL": "/",
  "logging": true,
  "header": true,
  "theme": "dark"
}
```

#### Setting Custom Admin Password

After first login, go to **Settings** > **User** and change the admin password.

### Updating FileBrowser on Unraid

#### Via Docker Tab

1. Go to the **Docker** tab
2. Find the `filebrowser` container
3. Click **Check for Updates** or change the tag to a newer version
4. Click **Update** and restart the container

#### Via Command Line

```bash
# Pull the latest image
docker pull filebrowser/filebrowser:latest

# Stop and remove the old container
docker stop filebrowser
docker rm filebrowser

# Start a new container with the same configuration
docker run -d \
  --name filebrowser \
  --restart=always \
  -p 8080:80 \
  -v /mnt/user/your_share:/srv \
  -v /mnt/user/appdata/filebrowser/database:/database \
  -v /mnt/user/appdata/filebrowser/config:/config \
  filebrowser/filebrowser:latest
```

### Troubleshooting

#### Container Won't Start

1. Check logs:
   ```bash
   docker logs filebrowser
   ```

2. Common issues:
   - Port already in use (change the host port)
   - Invalid volume paths (verify paths exist)

#### Can't Access Web Interface

1. Verify the container is running:
   ```bash
   docker ps | grep filebrowser
   ```

2. Check firewall settings on Unraid
3. Verify you're using the correct IP address

#### Permission Issues

Ensure your file share has appropriate permissions:

```bash
# Fix permissions (if needed)
chmod -R 755 /mnt/user/your_share
```

### Docker Compose (Alternative)

For advanced users, create a `docker-compose.yml`:

```yaml
version: '3'

services:
  filebrowser:
    image: filebrowser/filebrowser:latest
    container_name: filebrowser
    restart: always
    ports:
      - "8080:80"
    volumes:
      - /mnt/user/your_share:/srv
      - ./data/database:/database
      - ./data/config:/config
    environment:
      - FB_DATABASE=/database/filebrowser.db
```

Deploy with:
```bash
docker-compose up -d
```

---

## Synology NAS Deployment

See [Synology Deployment Guide](DEPLOYMENT_SYNOLOGY.md)

---

## Raspberry Pi Deployment

See [Raspberry Pi Deployment Guide](DEPLOYMENT_PI.md)

---

## Production Deployment

For production environments, consider:

1. **Reverse Proxy**: Put FileBrowser behind Nginx or Caddy
2. **HTTPS**: Enable SSL/TLS with Let's Encrypt
3. **Authentication**: Configure proper auth methods
4. **Backup**: Regular backups of database and config

### Example with Nginx Reverse Proxy

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
    }
}
```

---

## Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `FB_DATABASE` | `./filebrowser.db` | Database file path |
| `FB_CONFIG` | `./config.json` | Configuration file path |
| `FB_LOG_DIR` | `` | Log directory |

---

## Support

- **Issues**: https://github.com/filebrowser/filebrowser/issues
- **Documentation**: https://filebrowser.com/docs
- **Discord**: https://discord.gg/filebrowser
