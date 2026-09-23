# 🚀 Deployment Guide - Specialist App

## Architecture Overview

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│                 │     │                 │     │                 │
│   React/Next.js │────▶│   NestJS API    │────▶│   PostgreSQL    │
│    (Vercel)     │     │    (Fly.io)     │     │   (Supabase)    │
│                 │     │                 │     │                 │
└─────────────────┘     └─────────────────┘     └─────────────────┘
```

---

## 📋 Prerequisites

1. GitHub account (for CI/CD)
2. [Fly.io](https://fly.io) account (free tier)
3. [Supabase](https://supabase.com) account (free tier)
4. [Vercel](https://vercel.com) account (free tier)

---

## 🗄️ Step 1: Database Setup (Supabase)

### 1.1 Create Project
1. Go to [supabase.com](https://supabase.com)
2. Create new project
3. Choose region closest to your users (São Paulo for Argentina)
4. Save the password!

### 1.2 Get Connection String
1. Go to **Settings > Database**
2. Copy the **Connection string (URI)**
3. It looks like: `postgresql://postgres.[ref]:[password]@aws-0-sa-east-1.pooler.supabase.com:6543/postgres`

### 1.3 Run Migrations
```bash
cd specialist-be
DATABASE_URL="your-supabase-url" npx prisma migrate deploy
```

---

## 🔧 Step 2: Backend Deployment (Fly.io)

### 2.1 Install Fly CLI
```bash
# macOS
brew install flyctl

# Linux
curl -L https://fly.io/install.sh | sh

# Windows
powershell -Command "iwr https://fly.io/install.ps1 -useb | iex"
```

### 2.2 Login
```bash
fly auth login
```

### 2.3 Launch App (first time only)
```bash
cd specialist-be
fly launch --no-deploy
```

When prompted:
- App name: `specialist-api` (or choose your own)
- Region: `gru` (São Paulo) for Argentina
- PostgreSQL: **No** (we're using Supabase)

### 2.4 Set Secrets
```bash
# Required secrets
fly secrets set DATABASE_URL="postgresql://postgres.[ref]:[password]@aws-0-sa-east-1.pooler.supabase.com:6543/postgres"
fly secrets set JWT_SECRET="your-super-secret-jwt-key-min-32-chars"
fly secrets set CORS_ORIGINS="https://your-app.vercel.app"

# Optional (for social login)
fly secrets set GOOGLE_CLIENT_ID="..."
fly secrets set GOOGLE_CLIENT_SECRET="..."
fly secrets set FACEBOOK_APP_ID="..."
fly secrets set FACEBOOK_APP_SECRET="..."
```

### 2.5 Deploy
```bash
fly deploy --dockerfile Dockerfile.prod
```

### 2.6 Verify
```bash
# Check status
fly status

# Check logs
fly logs

# Test health endpoint
curl https://specialist-api.fly.dev/api/health
```

---

## 🌐 Step 3: Frontend Deployment (Vercel)

### 3.1 Push to GitHub
```bash
git add .
git commit -m "Prepare for deployment"
git push origin main
```

### 3.2 Import to Vercel
1. Go to [vercel.com](https://vercel.com)
2. Click **New Project**
3. Import your GitHub repository
4. Set root directory to `specialist-fe`

### 3.3 Configure Build
- **Framework Preset**: Next.js
- **Root Directory**: `specialist-fe`
- **Build Command**: `npm run build`
- **Output Directory**: `.next`

### 3.4 Set Environment Variables
In Vercel dashboard, add:

| Name | Value |
|------|-------|
| `NEXT_PUBLIC_API_URL` | `https://specialist-api.fly.dev` |

### 3.5 Deploy
Click **Deploy** and wait for completion.

---

## 🔄 Continuous Deployment

### GitHub Actions (Optional but Recommended)

Create `.github/workflows/deploy.yml`:

```yaml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy-backend:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: superfly/flyctl-actions/setup-flyctl@master
      - run: flyctl deploy --remote-only --dockerfile Dockerfile.prod
        working-directory: specialist-be
        env:
          FLY_API_TOKEN: ${{ secrets.FLY_API_TOKEN }}

  # Vercel deploys automatically when connected to GitHub
```

Get your Fly.io token:
```bash
fly tokens create deploy
```

Add it as `FLY_API_TOKEN` secret in GitHub repository settings.

---

## 🔐 Environment Variables Reference

### Backend (Fly.io Secrets)

| Variable | Required | Description |
|----------|----------|-------------|
| `DATABASE_URL` | ✅ | Supabase PostgreSQL connection string |
| `JWT_SECRET` | ✅ | Secret key for JWT tokens (min 32 chars) |
| `CORS_ORIGINS` | ✅ | Comma-separated frontend URLs |
| `NODE_ENV` | Auto | Set to `production` by fly.toml |
| `PORT` | Auto | Set to `8080` by fly.toml |
| `GOOGLE_CLIENT_ID` | ❌ | For Google OAuth |
| `GOOGLE_CLIENT_SECRET` | ❌ | For Google OAuth |
| `FACEBOOK_APP_ID` | ❌ | For Facebook OAuth |
| `FACEBOOK_APP_SECRET` | ❌ | For Facebook OAuth |

### Frontend (Vercel Environment Variables)

| Variable | Required | Description |
|----------|----------|-------------|
| `NEXT_PUBLIC_API_URL` | ✅ | Backend API URL (e.g., `https://specialist-api.fly.dev`) |

---

## 📊 Monitoring

### Health Endpoints
- **Health Check**: `GET /api/health`
- **Readiness**: `GET /api/health/ready`
- **Liveness**: `GET /api/health/live`

### Fly.io Dashboard
- CPU/Memory usage
- Request metrics
- Logs

### Vercel Dashboard
- Build logs
- Analytics
- Edge network stats

---

## 🆘 Troubleshooting

### Backend won't start
```bash
# Check logs
fly logs -a specialist-api

# SSH into container
fly ssh console -a specialist-api
```

### Database connection issues
1. Verify `DATABASE_URL` is correct
2. Check if Supabase project is active
3. Ensure IP is not blocked in Supabase

### CORS errors
1. Verify `CORS_ORIGINS` includes your frontend URL
2. Ensure it's the full URL with `https://`
3. No trailing slash!

### Frontend API calls fail
1. Check browser Network tab
2. Verify `NEXT_PUBLIC_API_URL` is set correctly
3. Redeploy after changing env vars

---

## 💰 Cost Summary (Free Tier)

| Service | Free Tier |
|---------|-----------|
| **Fly.io** | 3 shared-cpu VMs, 256MB each |
| **Supabase** | 500MB database, 1GB file storage |
| **Vercel** | Unlimited bandwidth, 100GB |

**Total cost: $0/month** (for reasonable usage)

---

## 🔮 Future Migration to AWS

When ready for AWS, the architecture translates to:

| Current | AWS |
|---------|-----|
| Fly.io | ECS Fargate |
| Supabase | RDS PostgreSQL |
| Vercel | S3 + CloudFront |
| fly.toml | Terraform |

No code changes needed!

