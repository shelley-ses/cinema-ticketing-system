---
name: deployment
description: Governed deployment procedures, pre-flight checks, and production release guidelines for the BJRS Cinema Ticketing System (Laravel 11 backend, React Vite frontend, Supabase database, and background queue workers).
---

# Production Deployment & Release Guidelines

Standardized procedures for building, staging, and releasing the **BJRS Cinema Ticketing System** to production hosting environments (e.g., Vercel / Netlify / Cloudflare for Frontend, Railway / Render / VPS / Forge for Laravel Backend, and Supabase for Managed Database).

---

## 🛡️ Pre-Flight Verification Checklist

Before triggering a production deployment, verify all checks pass:

1. **Clean Git State:**
   - Confirm you are on the release branch (`main` or release tag).
   - Ensure all working trees are clean (`git status`).
2. **Automated Test Validation:**
   ```bash
   cd backend && php artisan test
   cd ../frontend && npm run lint && npm run build
   ```
3. **Environment Security Check:**
   - Confirm production `.env` files contain no default or debug values (`APP_DEBUG=false`, `APP_ENV=production`).
   - Ensure `APP_KEY` is generated and configured.
   - Verify CORS allowed origins match the production frontend domain (`FRONTEND_URL`).

---

## 🚀 Backend Deployment (Laravel 11 API)

### 1. Dependency & Cache Optimization
Run during the build/release hook on the server:

```bash
# Navigate to backend root
cd backend

# Install production dependencies without dev packages
composer install --no-dev --prefer-dist --optimize-autoloader

# Run database migrations safely in production
php artisan migrate --force

# Optimize configuration and route caching
php artisan config:cache
php artisan route:cache
php artisan view:cache
php artisan event:cache

# Link storage directory
php artisan storage:link
```

### 2. Required Production Environment Variables (`backend/.env`)
```ini
APP_NAME="BJRS Cinema"
APP_ENV=production
APP_DEBUG=false
APP_URL=https://api.yourcinemadomain.com
FRONTEND_URL=https://yourcinemadomain.com

# Supabase PostgreSQL Connection
DB_CONNECTION=pgsql
DB_HOST=aws-0-us-east-1.pooler.supabase.com
DB_PORT=5432
DB_DATABASE=postgres
DB_USERNAME=postgres.your-project-id
DB_PASSWORD=your-production-db-password
DB_SSLMODE=require

# Session & Sanctum Stateful Domains
SANCTUM_STATEFUL_DOMAINS=yourcinemadomain.com
SESSION_DRIVER=database
QUEUE_CONNECTION=database

# Production Mail Delivery (SMTP / Resend / Mailgun)
MAIL_MAILER=smtp
MAIL_HOST=smtp.resend.com
MAIL_PORT=587
MAIL_USERNAME=resend
MAIL_PASSWORD=your-production-mail-key
MAIL_ENCRYPTION=tls
MAIL_FROM_ADDRESS="no-reply@yourcinemadomain.com"
```

### 3. Background Queue Worker (Supervisor / Daemon)
Ensure the Laravel queue daemon is active to process manager credentials and customer QR e-tickets:

```bash
php artisan queue:work --tries=3 --timeout=90 --sleep=3 --max-jobs=1000
```

---

## ⚡ Frontend Deployment (React 18 + Vite)

### 1. Build Production Bundle
```bash
cd frontend

# Install exact dependencies
npm ci

# Build optimized static assets (outputs to frontend/dist)
npm run build
```

### 2. Frontend Production Environment Variables (`frontend/.env.production`)
```ini
VITE_API_BASE_URL=https://api.yourcinemadomain.com/api/v1
VITE_APP_NAME="BJRS Cinema"
```

### 3. Static Hosting SPA Rewrite Rule
For single-page routing (React Router) on Nginx, Apache, or static hosts (Vercel/Netlify):
- **Vercel (`vercel.json`):**
  ```json
  {
    "rewrites": [{ "source": "/(.*)", "destination": "/index.html" }]
  }
  ```
- **Nginx:**
  ```nginx
  location / {
    try_files $uri $uri/ /index.html;
  }
  ```

---

## 🔄 Post-Deployment Verification & Health Checks

1. **API Status Check:**
   - Execute `GET https://api.yourcinemadomain.com/api/v1/movies` and verify `200 OK` JSON response.
2. **Database Connectivity:**
   - Verify Supabase connection pool responds within $< 100\text{ ms}$.
3. **Queue Health:**
   - Trigger a test dispatch or view failed jobs table:
     ```bash
     php artisan queue:failed
     ```
4. **CORS & Auth Verification:**
   - Log into the React frontend and verify Sanctum authentication token is received without cross-origin policy errors.
