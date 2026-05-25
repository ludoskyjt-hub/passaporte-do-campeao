# Deployment Guide — Vercel (Frontend) + Railway (Backend)

This monorepo is configured for an independent deployment split:

- **Frontend** (`artifacts/passaporte`) → Vercel (static SPA build)
- **Backend** (`artifacts/api-server`) → Railway (Node.js + PostgreSQL)
- **Database** → Railway-managed PostgreSQL (or any other managed Postgres)
- **Stripe** → your own Stripe account
- **SendGrid** → your own SendGrid account

No Replit connectors, no platform lock-in.

---

## 1. Prerequisites

- A GitHub repository with this project pushed
- Accounts on: Vercel, Railway, Stripe, SendGrid
- A verified sender on SendGrid (e.g. `info@passaportedocampeao.com.br`)

---

## 2. Backend — Railway

### 2.1 Create the services

1. New Project → Deploy from GitHub repo
2. Add a **PostgreSQL** plugin to the project — Railway gives you a `DATABASE_URL`
3. **Service settings → leave Root Directory as the repo root (`/`)** — do NOT set it to `artifacts/api-server`.
   The build needs the workspace root to install all packages, then targets the api-server via pnpm filter.
4. Railway will detect `nixpacks.toml` at the repo root and use it to install + build + start the API.
   The `start` command is `node --enable-source-maps artifacts/api-server/dist/index.mjs` (from repo root).

### 2.2 Required environment variables (Railway → Variables)

| Variable | Description | Example |
|---|---|---|
| `DATABASE_URL` | Postgres connection string | injected by the Postgres plugin |
| `SESSION_SECRET` | JWT signing secret (≥ 32 random chars) | `openssl rand -hex 32` |
| `STRIPE_SECRET_KEY` | Stripe secret key | `sk_live_...` |
| `STRIPE_PUBLISHABLE_KEY` | Stripe publishable key | `pk_live_...` |
| `STRIPE_WEBHOOK_SECRET` | Stripe webhook signing secret | `whsec_...` |
| `SENDGRID_API_KEY` | SendGrid API key | `SG.xxxx` |
| `SENDGRID_FROM_EMAIL` | Verified sender email | `info@passaportedocampeao.com.br` |
| `APP_BASE_URL` | Public URL of the **frontend** (used in reset emails) | `https://passaportedocampeao.com.br` |
| `CORS_ORIGINS` | Comma-separated allowed frontend origins | `https://passaportedocampeao.com.br,https://www.passaportedocampeao.com.br` |
| `PORT` | Port the app listens on | Railway sets this automatically |
| `NODE_ENV` | `production` | set by `nixpacks.toml` |

### 2.3 Optional environment variables

| Variable | Default |
|---|---|
| `PRIZE_POOL_CAP_BRL` | `5000` |
| `PARIS_TRIP_SUBSTITUTION_BRL` | `35000` |
| `ADMIN_BOOTSTRAP_KEY` | unset (only needed by the `promote-admin` script) |

### 2.4 Run database migrations

After first deploy, open the Railway shell (or use Railway CLI) and run:

```bash
pnpm --filter @workspace/db run push
```

(Or pre-build a one-off task that runs this on deploy.)

### 2.5 Stripe webhook

In your Stripe Dashboard → Developers → Webhooks → add endpoint:

```
https://<your-railway-domain>/api/stripe/webhook
```

Subscribe to: `payment_intent.succeeded`, `payment_intent.payment_failed`.
Copy the signing secret into `STRIPE_WEBHOOK_SECRET`.

---

## 3. Frontend — Vercel

### 3.1 Import the repo

1. Vercel → New Project → Import from GitHub
2. **Root Directory**: `artifacts/passaporte`
3. **Framework Preset**: Other (or Vite)
4. Vercel reads `artifacts/passaporte/vercel.json` for the build/output config

### 3.2 Required environment variables (Vercel → Settings → Environment Variables)

| Variable | Description | Example |
|---|---|---|
| `VITE_API_URL` | Public URL of the Railway backend | `https://api.passaportedocampeao.com.br` |

That's it for the frontend — the bundle is fully static.

### 3.3 Custom domain

In Vercel → Domains → add `passaportedocampeao.com.br`. Update DNS as instructed.

Once the domain is live, add it to the backend's `CORS_ORIGINS` and `APP_BASE_URL`.

---

## 4. Local development without Replit

```bash
# Install deps
pnpm install

# Create .env files (see .env.example)
cp .env.example .env

# Push the DB schema
pnpm --filter @workspace/db run push

# Run the API (port 8080 by default)
PORT=8080 pnpm --filter @workspace/api-server run dev

# In another terminal, run the frontend (port 5173 by default)
VITE_API_URL=http://localhost:8080 pnpm --filter @workspace/passaporte run dev
```

---

## 5. Going to production checklist

- [ ] DNS records pointing to Vercel + Railway
- [ ] All env vars set on both platforms (see tables above)
- [ ] Stripe in **live mode** — webhook endpoint added & secret rotated
- [ ] SendGrid sender verified
- [ ] First admin promoted: `pnpm --filter @workspace/scripts run promote-admin <email>`
- [ ] DB seeded with 106 matches: `pnpm --filter @workspace/scripts run seed-matches`
- [ ] HTTPS enforced on both Vercel and Railway (default)
- [ ] `CORS_ORIGINS` set to **exact** frontend domain(s) only
- [ ] `APP_BASE_URL` matches the public frontend URL
