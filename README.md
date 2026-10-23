# Inspect Buddy — AWS Lightsail + Docker + GitHub CI/CD Deployment Guide

A complete, production-ready, step-by-step runbook for packaging **Inspect Buddy Chat** into a Docker container and deploying it on an **AWS Lightsail instance in Mumbai (ap-south-1)** with automated CI/CD via GitHub Actions and free automatic SSL via Caddy.

---

## Architecture Overview

- **Source Code**: GitHub (`vedainspections/inspect-buddy-chat`)
- **Container Registry**: GitHub Container Registry (`ghcr.io`) — Free for public repos / generous free tier for private
- **Compute Server**: AWS Lightsail Ubuntu 24.04 LTS (Mumbai `ap-south-1`) — ~$3.50 to $5.00/month
- **Reverse Proxy & SSL**: Caddy Server (automatic HTTPS certificate issuance and renewal via Let's Encrypt)
- **Database & Storage**: Supabase Mumbai (`ap-south-1`) — Free tier
- **CI/CD Pipeline**: GitHub Actions (auto-build on push to `main`, SSH deploy to Lightsail)

---

## Phase 1: Prepare Codebase for Docker

In your local repository or GitHub project, add these four configuration files.

### 1.1 Create `Dockerfile` in the root folder

```dockerfile
# Stage 1: Build the Nitro/Vite web application
FROM node:20-alpine AS builder

WORKDIR /app

# Enable corepack for pnpm/bun/npm consistency
RUN corepack enable

# Copy dependency manifests
COPY package.json package-lock.json* pnpm-lock.yaml* bun.lockb* ./

# Install dependencies
RUN npm ci || npm install

# Copy application source
COPY . .

# Build environment variables (passed during docker build)
ARG VITE_SUPABASE_URL
ARG VITE_SUPABASE_PUBLISHABLE_KEY

ENV VITE_SUPABASE_URL=$VITE_SUPABASE_URL
ENV VITE_SUPABASE_PUBLISHABLE_KEY=$VITE_SUPABASE_PUBLISHABLE_KEY
ENV NODE_ENV=production

# Build the Nitro server output
RUN npm run build

# Stage 2: Minimal Production Runtime
FROM node:20-alpine AS runner

WORKDIR /app

ENV NODE_ENV=production
ENV PORT=3000
ENV HOST=0.0.0.0

# Copy built server from Nitro output (.output directory)
COPY --from=builder /app/.output ./.output

EXPOSE 3000

# Nitro node server entrypoint
CMD ["node", ".output/server/index.mjs"]
```

### 1.2 Create `.dockerignore`

```text
node_modules
.git
.github
.cache
dist
.output
.env
.env.local
npm-debug.log*
```

### 1.3 Create `docker-compose.yml`

This file runs both your web app and Caddy reverse proxy together.

```yaml
version: "3.8"

services:
  app:
    image: ghcr.io/vedainspections/inspect-buddy-chat:latest
    restart: always
    environment:
      - NODE_ENV=production
      - PORT=3000
      - HOST=0.0.0.0
      - VITE_SUPABASE_URL=${VITE_SUPABASE_URL}
      - VITE_SUPABASE_PUBLISHABLE_KEY=${VITE_SUPABASE_PUBLISHABLE_KEY}
    networks:
      - webnet

  caddy:
    image: caddy:2-alpine
    restart: always
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
      - caddy_data:/data
      - caddy_config:/config
    networks:
      - webnet
    depends_on:
      - app

networks:
  webnet:
    driver: bridge

volumes:
  caddy_data:
  caddy_config:
```

### 1.4 Create `Caddyfile`

Replace `your-domain.com` with your actual domain or subdomain (e.g. `app.vedainspections.com`):

```caddyfile
your-domain.com {
    reverse_proxy app:3000
    encode zstd gzip

    header {
        Strict-Transport-Security "max-age=31536000; includeSubDomains; preload"
        X-Content-Type-Options "nosniff"
        X-Frame-Options "DENY"
        Referrer-Policy "strict-origin-when-cross-origin"
    }
}
```

*(Note: If you do not have a custom domain yet and want to test with the Lightsail IP address directly over plain HTTP, replace `your-domain.com` with `:80`)*

---

## Phase 2: Launch AWS Lightsail Instance (Mumbai)

1. Log in to the [AWS Management Console](https://console.aws.amazon.com/).
2. Search for **Lightsail** in the top search bar and open it.
3. In the top-right corner of Lightsail, verify your region is set to **Mumbai (ap-south-1)**.
4. Click **Create instance**.
5. Configure your instance:
   - **Platform**: Linux/Unix
   - **Blueprint**: OS Only -> **Ubuntu 24.04 LTS**
   - **Instance plan**: Choose the **$3.50/month** plan (512 MB RAM, 1 vCPU, 20 GB SSD, 1 TB transfer) or the **$5.00/month** plan (1 GB RAM, 1 vCPU, 40 GB SSD, 2 TB transfer - recommended for building/running Node).
   - **Instance name**: `inspect-buddy-prod`
6. Click **Create instance**.
7. Wait 1–2 minutes until the status shows **Running**.

### 2.1 Attach a Static IP (Crucial)

By default, AWS dynamic IPs change if the instance reboots. You must attach a static IP (free while attached to an instance):
1. On the Lightsail dashboard, click the **Networking** tab.
2. Click **Create static IP**.
3. Choose **Mumbai**, attach it to `inspect-buddy-prod`, and name it `inspect-buddy-ip`.
4. Click **Create**. Note down this Static IP address (e.g., `13.235.xx.xx`).

### 2.2 Configure Firewall Ports

1. In Lightsail, click on your `inspect-buddy-prod` instance.
2. Go to the **Networking** tab.
3. Under **IPv4 Firewall**, ensure the following rules exist:
   - **SSH**: Port 22 (TCP)
   - **HTTP**: Port 80 (TCP)
   - **HTTPS**: Port 443 (TCP)
4. If HTTP or HTTPS are missing, click **Add rule**, select HTTP (port 80) and HTTPS (port 443), and save.

---

## Phase 3: Configure the Server & Install Docker

1. In Lightsail, click the orange terminal icon `>_` next to your instance to open the browser-based SSH terminal.
2. Update the system packages and install Docker:

```bash
sudo apt-get update && sudo apt-get upgrade -y
sudo apt-get install -y ca-certificates curl gnupg lsb-release git

# Add Docker official GPG key
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add Docker repository
echo   "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu   $(. /etc/os-release && echo "$VERSION_CODENAME") stable" |   sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker & Docker Compose Plugin
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Allow ubuntu user to run Docker without sudo
sudo usermod -aG docker ubuntu
```

3. Enable Swap Space (Crucial for 512MB/1GB instances):

```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo "/swapfile none swap sw 0 0" | sudo tee -a /etc/fstab
```

4. Create the application directory on the server:

```bash
mkdir -p /home/ubuntu/inspect-buddy
cd /home/ubuntu/inspect-buddy
```

---

## Phase 4: Configure GitHub Secrets & CI/CD Pipeline

To enable automated deployments whenever you push to GitHub, set up a GitHub Actions workflow.

### 4.1 Generate an SSH Key Pair for Deployments

In your Lightsail browser terminal, generate a deployment key:

```bash
ssh-keygen -t ed25519 -C "github-actions-deploy" -f ~/.ssh/github_deploy -N ""
cat ~/.ssh/github_deploy.pub >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys

# Display the private key (copy everything including BEGIN and END lines):
cat ~/.ssh/github_deploy
```

### 4.2 Add Secrets to your GitHub Repository

1. Open your repository on GitHub: `https://github.com/vedainspections/inspect-buddy-chat`.
2. Go to **Settings** -> **Secrets and variables** -> **Actions**.
3. Click **New repository secret** and add:

| Secret Name | Example / Suggested Value | Description |
|---|---|---|
| `LIGHTSAIL_HOST` | `13.235.xx.xx` | Your Lightsail Static IP |
| `LIGHTSAIL_USER` | `ubuntu` | Server username |
| `LIGHTSAIL_SSH_KEY` | *(Content of `~/.ssh/github_deploy`)* | Ed25519 Private Key |
| `VITE_SUPABASE_URL` | `https://xdpadqxhycctuzskopim.supabase.co` | Your Mumbai Supabase URL |
| `VITE_SUPABASE_PUBLISHABLE_KEY` | `sb_publishable_...` | Your Supabase Publishable Key |

### 4.3 Create GitHub Actions Workflow File

Create `.github/workflows/deploy.yml` in your GitHub repository:

```yaml
name: Build & Deploy to AWS Lightsail

on:
  push:
    branches:
      - main

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Log in to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Extract metadata (tags, labels) for Docker
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=raw,value=latest
            type=sha

      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          build-args: |
            VITE_SUPABASE_URL=${{ secrets.VITE_SUPABASE_URL }}
            VITE_SUPABASE_PUBLISHABLE_KEY=${{ secrets.VITE_SUPABASE_PUBLISHABLE_KEY }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy:
    needs: build-and-push
    runs-on: ubuntu-latest
    steps:
      - name: Execute SSH commands to update server
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.LIGHTSAIL_HOST }}
          username: ${{ secrets.LIGHTSAIL_USER }}
          key: ${{ secrets.LIGHTSAIL_SSH_KEY }}
          script: |
            cd /home/ubuntu/inspect-buddy
            
            # Authenticate with GHCR to pull package
            echo "${{ secrets.GITHUB_TOKEN }}" | docker login ghcr.io -u ${{ github.actor }} --password-stdin
            
            # Pull latest image
            docker compose pull app
            
            # Recreate containers without downtime
            docker compose up -d --remove-orphans
            
            # Prune old images to save disk space
            docker image prune -af --filter "until=24h"
```

---

## Phase 5: Initial Server Setup & First Launch

On the Lightsail server terminal (`/home/ubuntu/inspect-buddy`):

1. Copy your `docker-compose.yml` and `Caddyfile` into `/home/ubuntu/inspect-buddy`.
2. Create an `.env` file for runtime variables:

```bash
nano /home/ubuntu/inspect-buddy/.env
```

Paste your Supabase environment values:
```text
VITE_SUPABASE_URL=https://xdpadqxhycctuzskopim.supabase.co
VITE_SUPABASE_PUBLISHABLE_KEY=sb_publishable_d6f17BTE8eBX9zXiU3AVuw_Tvev1Ue_
```

3. Configure your domain DNS:
   - In your domain DNS manager (e.g. Cloudflare, GoDaddy, Route 53), add an **A record**:
     - **Name**: `@` or `app`
     - **Type**: `A`
     - **Value**: Your Lightsail Static IP (`13.235.xx.xx`)
     - **TTL**: Auto or 300 seconds

4. Launch the application with Docker Compose:

```bash
cd /home/ubuntu/inspect-buddy
docker compose up -d
```

5. Check logs to verify both containers are running healthy:

```bash
docker compose logs -f
```

---

## Phase 6: Operational Verification Checklist

- [ ] Static IP responds to HTTP/HTTPS ping
- [ ] Domain resolves to your Lightsail Static IP (`dig your-domain.com +short`)
- [ ] Caddy issues automatic Let's Encrypt certificate (padlock visible in browser)
- [ ] Web app loads successfully
- [ ] Login / Sign up works against Mumbai Supabase
- [ ] Booking creation and inspection checklist updates save properly
- [ ] Evidence photo upload saves into `inspection-evidence` Supabase bucket
- [ ] Pushing a commit to GitHub `main` triggers GitHub Actions and updates the server automatically

---

## Cost Analysis & Comparison

| Component | AWS Lightsail + Docker | Cloudflare + Supabase (Current) |
|---|---|---|
| **Server / Compute** | $3.50 – $5.00 / month | $0.00 (Cloudflare free tier) |
| **Static IP** | $0.00 (Free with instance) | $0.00 |
| **SSL Certificate** | $0.00 (Automated Caddy) | $0.00 (Cloudflare universal) |
| **Container Registry** | $0.00 (GHCR free tier) | N/A |
| **Database & Auth** | $0.00 (Supabase free tier) | $0.00 (Supabase free tier) |
| **File Storage** | $0.00 (Supabase 1 GB free) | $0.00 (Supabase 1 GB free) |
| **Total Monthly Cost** | **~$3.50 – $5.00 / mo** | **$0.00 / mo** |

---

## Maintenance & Monitoring Commands

- **Check container status**: `docker compose ps`
- **View live application logs**: `docker compose logs -f app`
- **View SSL / web server logs**: `docker compose logs -f caddy`
- **Restart application**: `docker compose restart app`
- **Check server disk space**: `df -h`
- **Check server memory & swap**: `free -m`