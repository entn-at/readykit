# Railway Deployment

Automated CI/CD deployment to Railway. Push to your deploy branch and your app deploys automatically.

## One-Time Setup

### 1. Create Railway Account

Sign up at https://railway.app (sign in with GitHub recommended).

### 2. Create Empty Project

1. Click "New Project"
2. Select "Empty Project"
3. Name it (e.g., `readykit`)

### 3. Add PostgreSQL

1. In your project, click "Add Service"
2. Select "Database" → "PostgreSQL"
3. Railway auto-provisions and sets `DATABASE_URL`

### 4. Add Redis

1. Click "Add Service"
2. Select "Database" → "Redis"
3. Railway auto-provisions and sets `REDIS_URL`

### 5. Create API Token

1. Go to Account Settings → Tokens
2. Click "Create Token"
3. Name it `github-deploy`
4. Copy the token

### 6. Add GitHub Secret

1. Go to your repo → Settings → Secrets and variables → Actions
2. New repository secret: `RAILWAY_TOKEN` = (paste token)

### 7. Set Environment Variables

In Railway dashboard, click on your app service and add these variables:

```bash
# Required
SECRET_KEY=your_64_char_hex_key
SECURITY_PASSWORD_SALT=your_secure_salt
SECURITY_TOTP_SECRETS=your_totp_secret

# Database (Railway sets DATABASE_URL, but Flask needs this format)
SQLALCHEMY_DATABASE_URI=${{DATABASE_URL}}

# Redis (reference Railway's Redis service)
REDIS_SESSION=${{REDIS_URL}}
CELERY_BROKER_URL=${{REDIS_URL}}
CELERY_RESULT_BACKEND=${{REDIS_URL}}

# App
FLASK_APP=run.py
FLASK_DEBUG=0
```

Generate secure keys:
```bash
openssl rand -hex 32  # Run 3 times for each secret
```

### 8. Deploy

Push to your deploy branch or trigger manually:
- Go to Actions tab → "Deploy to Railway" → "Run workflow"

Your app will be live at the URL shown in Railway dashboard.

## Post-Deployment

### Create Admin User

1. In Railway dashboard, click on your app service
2. Go to "Settings" → "Railway Shell" (or use Railway CLI)
3. Run:

```bash
flask create-db
flask install
```

Or via Railway CLI:
```bash
npm install -g @railway/cli
railway login
railway link  # Select your project
railway run flask create-db
railway run flask install
```

## Connecting Services

Railway uses service references. In your app's variables:

| Variable | Value |
|----------|-------|
| `SQLALCHEMY_DATABASE_URI` | `${{Postgres.DATABASE_URL}}` |
| `REDIS_SESSION` | `${{Redis.REDIS_URL}}` |
| `CELERY_BROKER_URL` | `${{Redis.REDIS_URL}}` |

The `${{ServiceName.VAR}}` syntax auto-links to other services in your project.

## Troubleshooting

### View Logs

In Railway dashboard → Your service → "Deployments" → Click on deployment → "View Logs"

Or via CLI:
```bash
railway logs
```

### Redeploy Manually

```bash
railway up
```

### Check Service Status

Railway dashboard shows real-time status, metrics, and logs for each service.

### Database Connection Issues

1. Verify PostgreSQL service is running
2. Check `SQLALCHEMY_DATABASE_URI` references the correct service
3. Railway uses `postgres://` - SQLAlchemy accepts both `postgres://` and `postgresql://`

## Cost Estimate (Hobby Plan - $5/month)

- **App**: ~$2-3/month (shared CPU, 512MB RAM)
- **PostgreSQL**: ~$1-2/month (minimal usage)
- **Redis**: ~$0.50/month (minimal usage)

Total: Usually under $5/month for a small SaaS.

## Differences from Fly.io

| Feature | Railway | Fly.io |
|---------|---------|--------|
| Database | Add as service | Separate `flyctl postgres create` |
| Secrets | Dashboard or CLI | `flyctl secrets set` |
| Logs | Dashboard | `flyctl logs` |
| Shell | Dashboard + CLI | `flyctl ssh console` |
| Pricing | Usage-based | Fixed + usage |
