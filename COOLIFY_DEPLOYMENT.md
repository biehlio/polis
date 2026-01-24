# Polis Deployment on Coolify - Minimal Setup Guide

This guide walks you through deploying Polis on Coolify with the absolute minimum required configuration. No development services, no emulators, no optional components—just the core system.

## What's Included

The minimal Polis stack consists of **5 essential services**:

| Service | Purpose | Technology |
|---------|---------|-----------|
| **PostgreSQL** | Primary relational database | PostgreSQL |
| **Server** | API backend and main application logic | Node.js/Express |
| **Math** | Sentiment analysis and math processing engine | Clojure/JVM |
| **File Server** | Serves compiled frontend applications | Node.js |
| **Nginx Proxy** | Reverse proxy and request router | Nginx |

## Pre-Deployment Checklist

Before deploying on Coolify, you need:

### 1. External Services (Not Provided)

- [ ] **OIDC Provider** (Auth0, Okta, Azure AD, etc.)
  - You'll need the issuer URL, client ID, audience, and JWKS URI
  - Set `DEV_MODE=false` when using external OIDC

- [ ] **Email Service** (AWS SES, Mailgun, SendGrid, etc.)
  - Not included in this minimal setup
  - Configure through the server environment variables

- [ ] **AI Model APIs** (Optional)
  - OpenAI (for GPT models)
  - Anthropic (for Claude)
  - Google Gemini
  - Only required if you want AI-powered features

### 2. Domain/SSL Setup

- [ ] **Domain name** pointed to your Coolify instance
- [ ] **SSL certificates** (Coolify will auto-provision via Let's Encrypt)
- [ ] **DNS records** configured correctly

### 3. Environment Configuration

- [ ] Generate JWT keys (see below)
- [ ] Prepare `.env` file with all required values

## Setup Instructions

### Step 1: Clone the Repository

```bash
git clone https://github.com/biehlio/polis.git
cd polis
```

### Step 2: Generate JWT Keys

Polis uses RSA keys to sign participant authentication tokens:

```bash
# Generate private key
openssl genpkey -algorithm RSA -out jwt-private.pem -pkeyopt rsa_keygen_bits:4096

# Generate public key from private key
openssl rsa -in jwt-private.pem -pubout -out jwt-public.pem
```

Then base64-encode both keys and set them in your `.env`:

```bash
# On macOS/Linux
JWT_PRIVATE_KEY=$(cat jwt-private.pem | base64)
JWT_PUBLIC_KEY=$(cat jwt-public.pem | base64)

echo "JWT_PRIVATE_KEY=$JWT_PRIVATE_KEY"
echo "JWT_PUBLIC_KEY=$JWT_PUBLIC_KEY"
```

Copy the output values into your `.env` file.

### Step 3: Configure Environment Variables

1. Copy the example configuration:
```bash
cp coolify.env .env
```

2. Edit `.env` and fill in all `[REQUIRED]` fields:

```bash
nano .env  # or your preferred editor
```

**Critical Variables**:

```env
# Database - use a strong password!
POSTGRES_PASSWORD=your_strong_database_password

# Domain - your Coolify instance domain
API_PROD_HOSTNAME=polis.yourdomain.com
DOMAIN_OVERRIDE=polis.yourdomain.com

# OIDC Authentication
AUTH_ISSUER=https://your-auth-provider.com/
AUTH_CLIENT_ID=your_client_id_here
AUTH_AUDIENCE=your_audience_here
JWKS_URI=https://your-auth-provider.com/.well-known/jwks.json

# JWT Keys (base64-encoded, from Step 2)
JWT_PRIVATE_KEY=base64_encoded_private_key_here
JWT_PUBLIC_KEY=base64_encoded_public_key_here

# Email
POLIS_FROM_ADDRESS="Your Service <noreply@yourdomain.com>"
```

### Step 4: Deploy on Coolify

#### Option A: Using Coolify UI

1. **Create New Project** in Coolify dashboard
2. **Add Application** → Docker Compose
3. **Configure:**
   - **Repository**: `https://github.com/biehlio/polis`
   - **Branch**: `edge`
   - **Docker Compose File Path**: `docker-compose.coolify.yml`
   - **Build Command**: Leave blank (uses Dockerfile)

4. **Environment Variables**:
   - Copy entire contents of your `.env` file into Coolify's environment editor
   - Click **Deploy**

#### Option B: Manual Docker Compose (CLI)

```bash
# Test the configuration locally first
docker-compose -f docker-compose.coolify.yml config

# Build and start
docker-compose -f docker-compose.coolify.yml up -d --build

# View logs
docker-compose -f docker-compose.coolify.yml logs -f server

# Stop
docker-compose -f docker-compose.coolify.yml down
```

### Step 5: Verify Deployment

Once deployed, verify all services are running:

```bash
docker-compose -f docker-compose.coolify.yml ps
```

Expected output:
```
NAME               COMMAND                SERVICE    STATUS
polis-nginx-proxy  "docker-entrypoint.s…"  nginx-proxy   Up
polis-server       "docker-entrypoint.s…"  server       Up
polis-math         "lein run"             math         Up
polis-file-server  "npm start"            file-server  Up
polis-postgres     "postgres"             postgres     Up (healthy)
```

Then visit your domain in a browser:

```
https://polis.yourdomain.com
```

You should see the Polis home page.

## Minimal Configuration Explained

### What's NOT Included

- **Development services** (no oidc-simulator, no hot-reload)
- **Local cloud emulators** (MinIO, DynamoDB, LocalStack, SES)
- **Ollama** (local LLM server)
- **Delphi** (AI narrative generation - optional advanced feature)

All these can be added later if needed. For now, we're keeping it minimal.

### Database Initialization

When the PostgreSQL container starts for the first time, it automatically:
1. Creates the database (`POSTGRES_DB`)
2. Runs all migrations from `/server/postgres/migrations/*.sql`
3. Initializes the schema

No manual database setup required.

### Port Mapping

- **Port 80/443** → Nginx proxy (external users)
- **Port 5000** → Server API (internal to Docker network)
- **Port 8080** → File server (internal to Docker network)

Only Nginx ports are exposed. All other services communicate via internal Docker network.

## Scaling & Advanced Configuration

### To Scale the Server

Edit `docker-compose.coolify.yml` and modify the `server` service's `scale` property:

```yaml
server:
  deploy:
    replicas: 3  # Run 3 instances behind Nginx
```

Then redeploy.

### To Add Email Service

Configure in `.env`:

**AWS SES:**
```env
AWS_REGION=us-east-1
AWS_ACCESS_KEY_ID=your_key
AWS_SECRET_ACCESS_KEY=your_secret
```

**Mailgun:**
```env
MAILGUN_API_KEY=your_api_key
MAILGUN_DOMAIN=your_domain
```

See the main `docs/configuration.md` for all email options.

### To Add AI Features

Set any of these in `.env`:

```env
OPENAI_API_KEY=sk-...
ANTHROPIC_API_KEY=sk-ant-...
GEMINI_API_KEY=your_key
```

Then optionally add the Delphi service later for advanced narrative generation.

## Troubleshooting

### Services Won't Start

**Check logs:**
```bash
docker-compose -f docker-compose.coolify.yml logs server
docker-compose -f docker-compose.coolify.yml logs postgres
docker-compose -f docker-compose.coolify.yml logs math
```

**Common issues:**
- `DATABASE_URL` format incorrect
- Missing required environment variables
- Port conflicts (80/443 already in use)

### Database Connection Errors

Verify the database is healthy:

```bash
docker-compose -f docker-compose.coolify.yml exec postgres pg_isready
```

If unhealthy, check if data volume exists:

```bash
docker volume ls | grep postgres
```

### Nginx Returns 502 Bad Gateway

Server might be starting slowly. Check:

```bash
docker-compose -f docker-compose.coolify.yml logs nginx-proxy
docker-compose -f docker-compose.coolify.yml logs server
```

Wait 30-60 seconds for server to fully initialize.

## Backup & Recovery

### Backup Database

```bash
docker-compose -f docker-compose.coolify.yml exec postgres pg_dump -U postgres polis > backup.sql
```

### Restore Database

```bash
docker-compose -f docker-compose.coolify.yml exec -T postgres psql -U postgres polis < backup.sql
```

### Backup All Volumes

```bash
docker run --rm -v polis-prod_postgres_data:/data -v $(pwd):/backup \
  busybox tar czf /backup/postgres-backup.tar.gz -C /data .
```

## Security Notes

- **Never commit `.env`** to version control (add to `.gitignore`)
- **Use strong database passwords** (minimum 16 characters, mixed case, numbers, symbols)
- **Enable database SSL** in production (`DATABASE_SSL=true`)
- **Rotate JWT keys** periodically
- **Keep Docker images updated** (`docker pull` regularly)
- **Monitor logs** for suspicious activity
- **Set appropriate admin UIDs** in `ADMIN_UIDS` environment variable

## Getting Help

- **Polis Documentation**: https://github.com/compdemocracy/polis/tree/develop/docs
- **Configuration Reference**: See `docs/configuration.md` in the repository
- **Issues & Discussion**: https://github.com/compdemocracy/polis/discussions

## Next Steps

Once running, you can:

1. **Create an admin account** and log in
2. **Create your first conversation**
3. **Configure advanced features** (see `docs/configuration.md` for all options)
4. **Set up monitoring & backups** appropriate for your deployment
5. **Add AI features** if desired (see "Scaling & Advanced Configuration" above)

Happy polling! 🎈
