# Supabase wit ECS & S3 Compatible Storage

# Setting up the Supabase Server

## Initial Setup

Create a new Server Instance - do not reuse your S3 server for that (if you have that. If you don't have an S3 Server: you're all set with 1 server instance).

First steps:
1. Log in to that server and run:
```bash
git clone --depth 1 https://github.com/supabase/supabase .
cd supabase/docker
cp docker-compose.yml docker-compose.yml.bkp  # Create a backup file
cp .env.example .env
```

## Security Configuration

### Secure the Kong API Gateway
Search for the `kong:` section in `docker-compose.yml` and remove the `ports:` section.

### Configure Supabase Studio (Dashboard)
Go to `docker/volumes/api/kong.yml` and remove the entire "Protected Dashboard" section that contains `- dashboard`.

### Add nginx
Add these services to your `docker-compose.yml` under the `services:` section:

```yaml
nginx:
  image: 'jc21/nginx-proxy-manager:latest'
  restart: unless-stopped
  ports:
    # These ports are in format <host-port>:<container-port>
    - '80:80' # Public HTTP Port
    - '443:443' # Public HTTPS Port
    - '81:81' # Admin Web Port
  volumes:
    - ./nginx-data:/data
    - ./nginx-letsencrypt:/etc/letsencrypt
    - ./nginx-snippets:/snippets:ro
  environment:
    TZ: 'Europe/Berlin'
```

## S3 Configuration

### If Using Custom S3
In `docker-compose.yml`, find the `storage` section and configure its environment:

```yaml
environment:
    STORAGE_BACKEND: s3
    GLOBAL_S3_BUCKET: supabase
    GLOBAL_S3_ENDPOINT: https://storage.yourdomain.com
    GLOBAL_S3_PROTOCOL: https 
    REGION: eu-south #whatever your region is
    AWS_DEFAULT_REGION: eu-south
    GLOBAL_S3_FORCE_PATH_STYLE: true
    AWS_ACCESS_KEY_ID: your_s3_access_key
    AWS_SECRET_ACCESS_KEY: your_secret_s3_access_key
```

### If Not Using S3
You may want to modify the storage path in the `volumes:` section of `docker-compose.yml` under `storage.volumes`.

## Credential Configuration

### Configure Environment Variables
In your `.env` file:

1. Set database password:
```bash
POSTGRES_PASSWORD=some-very-complicated-database-password
```

2. Generate and set JWT secret:
```bash
node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"
```

3. Generate and set API keys using the [self-hosting guide](https://supabase.com/docs/guides/self-hosting/docker#generate-api-keys):
```bash
ANON_KEY=your_generated_key
SERVICE_ROLE_KEY=your_generated_key
```

4. Set logger keys:
```bash
LOGFLARE_LOGGER_BACKEND_API_KEY=your_key
LOGFLARE_API_KEY=your_key
```

### Configure Studio and API URLs
In `.env`:
```bash
API_EXTERNAL_URL=your-api.yourdomain.com
SUPABASE_PUBLIC_URL=your-api.yourdomain.com
```

### Configure Application URLs
```bash
SITE_URL=https://yourapp.com
ADDITIONAL_REDIRECT_URLS=http://localhost:3000,http://localhost:6000
```

## Database Access

To enable external database access, remove `127.0.0.1` from the `db.ports:` section in `docker-compose.yml`.

## Starting the Server

1. Pull and start containers:
```bash
docker compose pull
docker compose up -d
```

2. Verify status:
```bash
docker ps
```

## Nginx Configuration

1. Access your protected proxy (e.g., supabase-proxy.yourdomain.com)

2. Configure API Proxy:
   - Add new proxy with `your-api.yourdomain.com`
   - Set scheme=http, hostname=kong, port=8000
   - Enable Websocket Support
   - Request new SSL certificate

3. Configure Dashboard Proxy:
   - Set up with studio.yourdomain.com
   - Use hostname=studio, port=3000
   - Add Advanced configuration:

```nginx
location / {
   proxy_set_header  Authorization $http_authorization;
   proxy_pass_header Authorization;
   proxy_pass $forward_scheme://studio:3000;
}

location /storage {
   proxy_set_header  Authorization $http_authorization;
   proxy_pass_header Authorization;
   proxy_pass $forward_scheme://kong:8000;
}

location /auth {
   proxy_set_header  Authorization $http_authorization;
   proxy_pass_header Authorization;
   proxy_pass $forward_scheme://kong:8000;
}
```
