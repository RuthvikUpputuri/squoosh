# Squoosh Docker Deployment Guide

This guide covers deploying Squoosh using Docker with various reverse proxy setups.

## Quick Start (Standalone)

```bash
docker run -d -p 8080:80 --name squoosh ruthvikupputuri/squoosh:latest-arm64
```

Access at: `http://localhost:8080`

## Docker Compose (Standalone)

```yaml
version: '3.8'

services:
  squoosh:
    image: ruthvikupputuri/squoosh:latest-arm64
    container_name: squoosh
    restart: unless-stopped
    ports:
      - '8080:80'
```

Run: `docker compose up -d`

---

## Reverse Proxy Configurations

### Important: CORS Headers

Squoosh requires these headers for WebAssembly/SharedArrayBuffer to work:

- `Cross-Origin-Embedder-Policy: require-corp`
- `Cross-Origin-Opener-Policy: same-origin`

These are already set in the container. Your reverse proxy should **preserve** them.

---

### 1. Traefik

#### Root Path (e.g., `squoosh.example.com`)

**With HTTPS/TLS (using Let's Encrypt):**

```yaml
version: '3.8'

services:
  squoosh:
    image: ruthvikupputuri/squoosh:latest-arm64
    container_name: squoosh
    restart: unless-stopped
    networks:
      - traefik
    labels:
      - 'traefik.enable=true'
      - 'traefik.http.routers.squoosh.rule=Host(`squoosh.example.com`)'
      - 'traefik.http.routers.squoosh.entrypoints=websecure'
      - 'traefik.http.routers.squoosh.tls=true'
      - 'traefik.http.routers.squoosh.tls.certresolver=letsencrypt'
      - 'traefik.http.services.squoosh.loadbalancer.server.port=80'

networks:
  traefik:
    external: true
```

**HTTP only (or with Cloudflare Tunnel/external SSL):**

```yaml
version: '3.8'

services:
  squoosh:
    image: ruthvikupputuri/squoosh:latest-arm64
    container_name: squoosh
    restart: unless-stopped
    networks:
      - traefik
    labels:
      - 'traefik.enable=true'
      - 'traefik.http.routers.squoosh.rule=Host(`squoosh.example.com`)'
      - 'traefik.http.routers.squoosh.entrypoints=web'
      - 'traefik.http.services.squoosh.loadbalancer.server.port=80'

networks:
  traefik:
    external: true
```

#### Subpath (e.g., `example.com/squoosh`)

**With HTTPS/TLS (using Let's Encrypt):**

```yaml
version: '3.8'

services:
  squoosh:
    image: ruthvikupputuri/squoosh:latest-arm64
    container_name: squoosh
    restart: unless-stopped
    networks:
      - traefik
    labels:
      - 'traefik.enable=true'
      - 'traefik.http.routers.squoosh.rule=Host(`example.com`) && PathPrefix(`/squoosh`)'
      - 'traefik.http.routers.squoosh.entrypoints=websecure'
      - 'traefik.http.routers.squoosh.tls=true'
      - 'traefik.http.routers.squoosh.tls.certresolver=letsencrypt'
      - 'traefik.http.routers.squoosh.middlewares=squoosh-stripprefix'
      - 'traefik.http.middlewares.squoosh-stripprefix.stripprefix.prefixes=/squoosh'
      - 'traefik.http.services.squoosh.loadbalancer.server.port=80'

networks:
  traefik:
    external: true
```

**HTTP only (or with Cloudflare Tunnel/external SSL):**

```yaml
version: '3.8'

services:
  squoosh:
    image: ruthvikupputuri/squoosh:latest-arm64
    container_name: squoosh
    restart: unless-stopped
    networks:
      - traefik
    labels:
      - 'traefik.enable=true'
      - 'traefik.http.routers.squoosh.rule=Host(`example.com`) && PathPrefix(`/squoosh`)'
      - 'traefik.http.routers.squoosh.entrypoints=web'
      - 'traefik.http.routers.squoosh.middlewares=squoosh-stripprefix'
      - 'traefik.http.middlewares.squoosh-stripprefix.stripprefix.prefixes=/squoosh'
      - 'traefik.http.services.squoosh.loadbalancer.server.port=80'

networks:
  traefik:
    external: true
```

---

### 2. Nginx Proxy Manager

1. Add a new Proxy Host
2. **Domain Names**: `squoosh.example.com`
3. **Forward Hostname/IP**: `squoosh` (container name)
4. **Forward Port**: `80`
5. **Enable SSL** if needed
6. No custom headers needed (container headers are preserved by default)

For subpath deployment, use **Custom Locations**:

- **Location**: `/squoosh`
- **Forward Hostname/IP**: `squoosh`
- **Forward Port**: `80`

---

### 3. Caddy

#### Caddyfile (Root Path)

```
squoosh.example.com {
    reverse_proxy squoosh:80
}
```

#### Caddyfile (Subpath)

```
example.com {
    handle_path /squoosh* {
        reverse_proxy squoosh:80
    }
}
```

#### docker-compose.yml with Caddy

```yaml
version: '3.8'

services:
  squoosh:
    image: ruthvikupputuri/squoosh:latest-arm64
    container_name: squoosh
    restart: unless-stopped
    networks:
      - caddy

  caddy:
    image: caddy:latest
    restart: unless-stopped
    ports:
      - '80:80'
      - '443:443'
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
      - caddy_data:/data
      - caddy_config:/config
    networks:
      - caddy

networks:
  caddy:
    driver: bridge

volumes:
  caddy_data:
  caddy_config:
```

---

### 4. Nginx (Manual Config)

#### Root Path

```nginx
server {
    listen 80;
    server_name squoosh.example.com;

    location / {
        proxy_pass http://squoosh:80;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

#### Subpath

```nginx
server {
    listen 80;
    server_name example.com;

    location /squoosh/ {
        proxy_pass http://squoosh:80/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

#### docker-compose.yml with Nginx

```yaml
version: '3.8'

services:
  squoosh:
    image: ruthvikupputuri/squoosh:latest-arm64
    container_name: squoosh
    restart: unless-stopped
    networks:
      - nginx

  nginx:
    image: nginx:alpine
    restart: unless-stopped
    ports:
      - '80:80'
      - '443:443'
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf
      - ./ssl:/etc/nginx/ssl # Optional: for SSL certificates
    networks:
      - nginx
    depends_on:
      - squoosh

networks:
  nginx:
    driver: bridge
```

---

### 5. Apache (with mod_proxy)

#### Root Path

```apache
<VirtualHost *:80>
    ServerName squoosh.example.com

    ProxyPreserveHost On
    ProxyPass / http://squoosh:80/
    ProxyPassReverse / http://squoosh:80/
</VirtualHost>
```

#### Subpath

```apache
<VirtualHost *:80>
    ServerName example.com

    ProxyPreserveHost On
    ProxyPass /squoosh http://squoosh:80/
    ProxyPassReverse /squoosh http://squoosh:80/
</VirtualHost>
```

---

## Environment Variables

Create a `.env` file for easier configuration:

```env
# Your domain
HOST=squoosh.example.com

# Container port (internal, always 80)
PORT=80

# External port (if running standalone)
EXTERNAL_PORT=8080
```

Use in docker-compose.yml:

```yaml
ports:
  - '${EXTERNAL_PORT:-8080}:80'
```

---

## Architecture Support

- **ARM64**: `ruthvikupputuri/squoosh:latest-arm64` or `ruthvikupputuri/squoosh:v1.0.2-arm64`
- **AMD64/x86_64**: Build your own using the included Dockerfile

To build for your architecture:

```bash
docker build -t ruthvikupputuri/squoosh:latest .
```

---

## Troubleshooting

### CORS Issues

If you see CORS errors in the browser console:

1. Check that your reverse proxy is **preserving** headers from the container
2. Don't add conflicting CORS headers at the proxy level
3. The app needs: `Cross-Origin-Embedder-Policy: require-corp` and `Cross-Origin-Opener-Policy: same-origin`

### SharedArrayBuffer Not Available

- Ensure the CORS headers are present (check browser DevTools → Network tab)
- Some older browsers don't support SharedArrayBuffer

### 404 Errors on Refresh

- Make sure your reverse proxy handles SPA routing (try_files or equivalent)
- The nginx config in the container already handles this

### Container Won't Start

```bash
# Check logs
docker logs squoosh

# Verify image
docker images | grep squoosh

# Test standalone
docker run --rm -p 8080:80 ruthvikupputuri/squoosh:latest-arm64
```

---

## Security Recommendations

1. **Always use HTTPS in production** (via reverse proxy)
2. **Set up rate limiting** at the reverse proxy level
3. **Use firewall rules** to restrict access if needed
4. **Keep the image updated** by pulling latest regularly:
   ```bash
   docker compose pull
   docker compose up -d
   ```

---

## Support

For issues related to:

- **Squoosh app**: https://github.com/GoogleChromeLabs/squoosh
- **Docker image**: https://github.com/RuthvikUpputuri
- **Image registry**: https://hub.docker.com/r/ruthvikupputuri/squoosh
