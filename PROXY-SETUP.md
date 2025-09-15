# Traefik Proxy Setup Guide

This guide shows how to create HTTPS proxies for local applications using Traefik and Nginx, without modifying the original application.

## Table of Contents

- [Overview](#overview)
- [Quick Setup](#quick-setup)
- [Step-by-Step Guide](#step-by-step-guide)
- [Configuration Files](#configuration-files)
- [Testing](#testing)
- [Multiple Applications](#multiple-applications)
- [Troubleshooting](#troubleshooting)

## Overview

This setup allows you to:
- ✅ Access local applications via custom HTTPS domains
- ✅ Use valid SSL certificates (mkcert)
- ✅ Automatic HTTP to HTTPS redirection
- ✅ No modifications to your original application
- ✅ Easy to replicate for multiple apps

## Quick Setup

### 1. Generate SSL Certificate
```bash
# Generate certificate for your domain
mkcert your-app.example.local

# Move certificates to certs directory
mv your-app.example.local.pem certs/
mv your-app.example.local-key.pem certs/
```

### 2. Update TLS Configuration
Add to `dynamic/tls.yml`:
```yaml
  certificates:
    - certFile: /certs/your-app.example.local.pem
      keyFile: /certs/your-app.example.local-key.pem
```

### 3. Create Nginx Proxy Config
Create `nginx-proxy-your-app.conf`:
```nginx
events {
    worker_connections 1024;
}

http {
    upstream your_app {
        server host.docker.internal:YOUR_PORT;
    }

    server {
        listen 80;
        server_name _;

        # Increase timeouts for long requests
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;

        # Increase buffer sizes
        proxy_buffer_size 4k;
        proxy_buffers 8 4k;
        proxy_busy_buffers_size 8k;

        # Pass all requests to the upstream app
        location / {
            proxy_pass http://your_app;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_set_header X-Forwarded-Host $host;
            proxy_set_header X-Forwarded-Port $server_port;

            # Handle WebSocket connections if needed
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";

            # Disable buffering for streaming responses
            proxy_buffering off;
        }
    }
}
```

### 4. Add to Docker Compose
Add to `docker-compose.yml`:
```yaml
  # Proxy service for your app
  your-app-proxy:
    image: nginx:alpine
    container_name: your-app-proxy
    restart: unless-stopped
    volumes:
      - ./nginx-proxy-your-app.conf:/etc/nginx/nginx.conf:ro
    extra_hosts:
      - "host.docker.internal:host-gateway"
    labels:
      # Enable Traefik for this service
      - "traefik.enable=true"
      - "traefik.docker.network=traefik"

      # HTTP router: redirect to HTTPS
      - "traefik.http.routers.your-app.rule=Host(`your-app.example.local`)"
      - "traefik.http.routers.your-app.entrypoints=web"
      - "traefik.http.routers.your-app.middlewares=your-app-https-redirect"

      # HTTPS router
      - "traefik.http.routers.your-app-secure.rule=Host(`your-app.example.local`)"
      - "traefik.http.routers.your-app-secure.entrypoints=websecure"
      - "traefik.http.routers.your-app-secure.tls=true"

      # Service configuration
      - "traefik.http.services.your-app.loadbalancer.server.port=80"

      # Middleware: HTTP -> HTTPS
      - "traefik.http.middlewares.your-app-https-redirect.redirectscheme.scheme=https"
      - "traefik.http.middlewares.your-app-https-redirect.redirectscheme.permanent=true"

    networks:
      - traefik
```

### 5. Update Hosts File
Add to your hosts file:
```
127.0.0.1 your-app.example.local
```

### 6. Start Services
```bash
docker-compose up -d your-app-proxy
```

## Step-by-Step Guide

### Step 1: Generate SSL Certificate

#### Windows
```powershell
mkcert your-app.example.local
Move-Item "your-app.example.local.pem" "certs/" -Force
Move-Item "your-app.example.local-key.pem" "certs/" -Force
```

#### macOS/Linux
```bash
mkcert your-app.example.local
mv your-app.example.local.pem certs/
mv your-app.example.local-key.pem certs/
```

### Step 2: Update TLS Configuration

Edit `dynamic/tls.yml` and add your certificate:
```yaml
# dynamic/tls.yml
tls:
  stores:
    default:
      defaultCertificate:
        certFile: /certs/_wildcard.example.local.pem
        keyFile: /certs/_wildcard.example.local-key.pem

  certificates:
    - certFile: /certs/_wildcard.example.local.pem
      keyFile: /certs/_wildcard.example.local-key.pem
    - certFile: /certs/your-app.example.local.pem
      keyFile: /certs/your-app.example.local-key.pem
```

### Step 3: Create Nginx Proxy Configuration

Create a new file `nginx-proxy-your-app.conf` with the configuration shown above, replacing:
- `YOUR_PORT` with your application's port
- `your_app` with a unique upstream name
- `your-app` with your application name

### Step 4: Add Service to Docker Compose

Add the proxy service to your `docker-compose.yml`, replacing:
- `your-app-proxy` with your service name
- `your-app.example.local` with your domain
- `nginx-proxy-your-app.conf` with your config file name

### Step 5: Update Hosts File

#### Windows
Edit `C:\Windows\System32\drivers\etc\hosts` (as Administrator):
```
127.0.0.1 your-app.example.local
```

#### macOS/Linux
Edit `/etc/hosts` (with sudo):
```bash
sudo nano /etc/hosts
```
Add:
```
127.0.0.1 your-app.example.local
```

### Step 6: Start and Test

```bash
# Start the proxy
docker-compose up -d your-app-proxy

# Test the connection
docker exec your-app-proxy wget -qO- http://host.docker.internal:YOUR_PORT
```

## Configuration Files

### Nginx Proxy Template

```nginx
events {
    worker_connections 1024;
}

http {
    upstream APP_NAME {
        server host.docker.internal:APP_PORT;
    }

    server {
        listen 80;
        server_name _;

        # Timeouts
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;

        # Buffers
        proxy_buffer_size 4k;
        proxy_buffers 8 4k;
        proxy_busy_buffers_size 8k;

        location / {
            proxy_pass http://APP_NAME;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_set_header X-Forwarded-Host $host;
            proxy_set_header X-Forwarded-Port $server_port;

            # WebSocket support
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";

            # Disable buffering
            proxy_buffering off;
        }
    }
}
```

### Docker Compose Service Template

```yaml
  SERVICE_NAME-proxy:
    image: nginx:alpine
    container_name: SERVICE_NAME-proxy
    restart: unless-stopped
    volumes:
      - ./nginx-proxy-SERVICE_NAME.conf:/etc/nginx/nginx.conf:ro
    extra_hosts:
      - "host.docker.internal:host-gateway"
    labels:
      - "traefik.enable=true"
      - "traefik.docker.network=traefik"
      - "traefik.http.routers.SERVICE_NAME.rule=Host(`DOMAIN_NAME`)"
      - "traefik.http.routers.SERVICE_NAME.entrypoints=web"
      - "traefik.http.routers.SERVICE_NAME.middlewares=SERVICE_NAME-https-redirect"
      - "traefik.http.routers.SERVICE_NAME-secure.rule=Host(`DOMAIN_NAME`)"
      - "traefik.http.routers.SERVICE_NAME-secure.entrypoints=websecure"
      - "traefik.http.routers.SERVICE_NAME-secure.tls=true"
      - "traefik.http.services.SERVICE_NAME.loadbalancer.server.port=80"
      - "traefik.http.middlewares.SERVICE_NAME-https-redirect.redirectscheme.scheme=https"
      - "traefik.http.middlewares.SERVICE_NAME-https-redirect.redirectscheme.permanent=true"
    networks:
      - traefik
```

## Testing

### 1. Verify Certificate
```bash
# Check if certificate exists
ls -la certs/your-app.example.local*

# Test SSL connection
openssl s_client -connect your-app.example.local:443 -servername your-app.example.local
```

### 2. Test Proxy Connection
```bash
# Test from inside the proxy container
docker exec your-app-proxy wget -qO- http://host.docker.internal:YOUR_PORT

# Expected: 401 Unauthorized (if auth required) or 200 OK
```

### 3. Test Full HTTPS Flow
```bash
# Test HTTPS access
curl -k https://your-app.example.local

# Test HTTP redirect
curl -I http://your-app.example.local
# Should return 301 redirect to HTTPS
```

## Multiple Applications

To add multiple applications, repeat the process for each:

### Example: Multiple Apps
```yaml
# docker-compose.yml
services:
  # App 1: React app on port 3000
  react-app-proxy:
    image: nginx:alpine
    container_name: react-app-proxy
    # ... configuration for react-app.example.local

  # App 2: API on port 3005
  api-proxy:
    image: nginx:alpine
    container_name: api-proxy
    # ... configuration for api.example.local

  # App 3: Admin panel on port 8080
  admin-proxy:
    image: nginx:alpine
    container_name: admin-proxy
    # ... configuration for admin.example.local
```

### Hosts File for Multiple Apps
```
127.0.0.1 react-app.example.local
127.0.0.1 api.example.local
127.0.0.1 admin.example.local
```

## Troubleshooting

### Common Issues

#### 1. Certificate Not Found
```bash
# Check certificate files
ls -la certs/your-app.example.local*

# Regenerate if missing
mkcert your-app.example.local
```

#### 2. Connection Refused
```bash
# Check if your app is running
netstat -tulpn | grep :YOUR_PORT

# Test direct connection
curl http://localhost:YOUR_PORT
```

#### 3. 502 Bad Gateway
```bash
# Check proxy logs
docker logs your-app-proxy

# Check if upstream is reachable
docker exec your-app-proxy wget -qO- http://host.docker.internal:YOUR_PORT
```

#### 4. Domain Not Resolving
```bash
# Check hosts file
cat /etc/hosts | grep your-app.example.local

# Test DNS resolution
nslookup your-app.example.local
```

### Useful Commands

```bash
# Check all running proxies
docker ps | grep proxy

# View proxy logs
docker logs your-app-proxy

# Restart specific proxy
docker-compose restart your-app-proxy

# Test upstream connectivity
docker exec your-app-proxy ping host.docker.internal

# Check Traefik routes
curl http://localhost:8080/api/http/routers
```

## Best Practices

1. **Naming Convention**: Use consistent naming for services, containers, and files
2. **Port Management**: Keep track of which ports are used by which applications
3. **Certificate Management**: Regularly check certificate expiration dates
4. **Logging**: Monitor proxy logs for connection issues
5. **Testing**: Always test the full HTTPS flow after setup

## Quick Reference

| Component | Purpose | Port |
|-----------|---------|------|
| Traefik | HTTPS termination, routing | 80, 443 |
| Nginx Proxy | HTTP proxy to localhost | 80 (internal) |
| Your App | Original application | YOUR_PORT |

### File Structure
```
traefik/
├── docker-compose.yml
├── dynamic/
│   └── tls.yml
├── certs/
│   ├── your-app.example.local.pem
│   └── your-app.example.local-key.pem
└── nginx-proxy-your-app.conf
```

This setup provides a robust, scalable solution for adding HTTPS access to any local application without modifying the original code.
