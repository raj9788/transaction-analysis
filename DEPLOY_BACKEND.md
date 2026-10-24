# Step 1: Deploying the Backend (FastAPI + Database)

This guide provides end-to-end instructions for deploying the **TA Coach Backend** to production on **Render** (100% Free) or **Railway**.

---

## Table of Contents
1. [Important: Render Free Tier & Database Options](#1-important-render-free-tier--database-options)
2. [Google Cloud Console OAuth Setup](#2-google-cloud-console-oauth-setup)
3. [Option A: Deploy on Render with Neon Free Postgres (Recommended for 100% Free)](#3-option-a-deploy-on-render-with-neon-free-postgres-recommended)
4. [Option B: Deploy on Render with Ephemeral SQLite (Zero Config / Demo Only)](#4-option-b-deploy-on-render-with-ephemeral-sqlite-zero-config--demo-only)
5. [Option C: Deploy on Railway (Free Trial with Disk Volume)](#5-option-c-deploy-on-railway-free-trial-with-disk-volume)
6. [Verifying Backend Health & Database](#6-verifying-backend-health--database)
7. [Next Steps: Connecting with Frontend (Step 2)](#7-next-steps-connecting-with-frontend-step-2)

---

## 1. Important: Render Free Tier & Database Options

Render's **Free Web Service** tier **does not support persistent disks** (disks require upgrading to paid plans). 

Because containers on Render Free sleep after 15 minutes of inactivity and restart on new deploys:
- **If you use local SQLite on Render Free**, files are stored in ephemeral storage. When the server sleeps or restarts, SQLite resets back to initial state (seeded default scenarios). This is fine for testing or demoing, but not for permanent use.
- **To get 100% free, permanent data storage**, connect the backend to a **free cloud PostgreSQL database** like **[Neon.tech](https://neon.tech)** (free forever, 0.5 GB, serverless, no credit card required).

The backend already includes `psycopg[binary]`, `psycopg2-binary`, and SQLAlchemy with built-in connection pool pre-pinging (`pool_pre_ping=True`), so Neon serverless sleep cycles and PgBouncer connection drops are handled seamlessly without errors. Switching from SQLite to PostgreSQL only requires changing the `DATABASE_URL` environment variable.

---

## 2. Google Cloud Console OAuth Setup

1. Open the [Google Cloud Console Credentials Page](https://console.cloud.google.com/apis/credentials).
2. Select your project (or create a new one).
3. If not already configured, set up your **OAuth consent screen** (User Type: *External*, add app name and contact email).
4. Go to **Credentials** > click **Create Credentials** > select **OAuth client ID**.
5. Select **Web application** for Application Type.
6. Under **Authorized redirect URIs**, add:
   - `https://<your-backend-app-name>.onrender.com/api/v1/auth/google/callback`
   *(Once your frontend is deployed in Step 2, you will also add `https://<your-frontend-domain>/api/v1/auth/google/callback` here).*
7. Click **Create** and copy:
   - **Client ID**
   - **Client Secret**

---

## 3. Option A: Deploy on Render with Neon Free Postgres (Recommended)

### Step 3.1: Get a Free PostgreSQL Database on Neon (2 minutes)
1. Go to [neon.tech](https://neon.tech) and sign up (free with GitHub or Google).
2. Create a new project (e.g. `ta-coach-db`).
3. On the dashboard, copy the connection string under **Connection Details**:
   ```
   postgresql://user:password@ep-xyz.us-east-2.aws.neon.tech/neondb?sslmode=require
   ```

### Step 3.2: Create the Web Service on Render
1. Go to [Render.com](https://render.com) and log in.
2. Click **New +** > select **Web Service**.
3. Connect your GitHub repository.
4. Set the basic configuration:
   - **Name**: `ta-coach-backend` (or a name of your choice)
   - **Region**: Same or nearest region to your Neon database
   - **Branch**: `main`
   - **Root Directory**: `backend`
   - **Runtime**: `Python 3`
   - **Instance Type**: `Free`
   - **Build Command**:
     ```bash
     pip install uv && uv sync --frozen --no-dev
     ```
     *(Or with standard pip: `pip install -e .`)*
   - **Start Command**:
     ```bash
     uv run uvicorn main:app --app-dir src --host 0.0.0.0 --port $PORT --proxy-headers --forwarded-allow-ips='*'
     ```

### Step 3.3: Set Environment Variables on Render
Go to the **Environment Variables** section on Render and add:

| Key | Value | Description |
| :--- | :--- | :--- |
| `DATABASE_URL` | *(Paste your Neon connection string)* | Connects to Neon free Postgres |
| `SESSION_SECRET` | *(Generate a random 32+ character string)* | Encrypts session cookies |
| `GROQ_API_KEY` | `gsk_...` | Groq API key |
| `GOOGLE_CLIENT_ID` | `...apps.googleusercontent.com` | Google OAuth Client ID |
| `GOOGLE_CLIENT_SECRET` | `GOCSPX-...` | Google OAuth Client Secret |
| `COOKIE_SECURE` | `true` | Enforces HTTPS-only cookies |
| `FRONTEND_URL` | `/docs` *(change to frontend URL in Step 2)* | Redirect target after Google sign-in |
| `PYTHON_VERSION` | `3.11.11` | Ensures compatible Python runtime |

*(Optional variables: `LLM_MODEL=openai/gpt-oss-120b`, `MAX_TURNS=10`, `SUPERADMIN_EMAILS=your-email@gmail.com`)*

### Step 3.4: Deploy
Click **Create Web Service**. 
Render will build the backend, connect to Neon, run `init_db()` and `seed_scenarios()` automatically on startup.

---

## 4. Option B: Deploy on Render with Ephemeral SQLite (Zero Config / Demo Only)

If you only want to quickly test or demo the app without creating a database:

1. Follow **Step 3.2** above on Render.
2. Set the environment variables, but for `DATABASE_URL` use:
   ```text
   DATABASE_URL=sqlite:///./conversation.db
   ```
3. **Do not attach any disk**.
4. Deploy the service.

> **Note:** Every time the free Render instance goes to sleep (after 15 minutes of idle time) or restarts, data will reset to the 3 default public scenarios.

---

## 5. Option C: Deploy on Railway (Free Trial with Disk Volume)

Railway provides free trial credits and supports mounting persistent disk volumes directly to services:

1. Log in to [Railway.app](https://railway.app).
2. Click **New Project** > **Deploy from GitHub repo**.
3. In service **Settings**:
   - **Root Directory**: `backend`
   - **Build Command**: `pip install uv && uv sync --frozen --no-dev`
   - **Start Command**: `uv run uvicorn main:app --app-dir src --host 0.0.0.0 --port $PORT --proxy-headers --forwarded-allow-ips='*'`
4. Click **+ New** > **Volume** and mount it to `/data`.
5. Set `DATABASE_URL=sqlite:////data/conversation.db`.
6. Add the remaining environment variables (`SESSION_SECRET`, `GROQ_API_KEY`, `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, `COOKIE_SECURE=true`).
7. In **Settings** > **Networking**, click **Generate Domain**.

---

## 6. Verifying Backend Health & Database

Once your backend deployment status is **Live**:

1. **Test Health Endpoint**:
   Open in your browser:
   ```
   https://<your-backend-url>/health
   ```
   Expected response:
   ```json
   {"status": "ok"}
   ```

2. **Test Interactive Swagger Docs**:
   Open:
   ```
   https://<your-backend-url>/docs
   ```
   All endpoints (`/api/v1/auth/...`, `/api/v1/scenarios/...`, etc.) should appear.

3. **Check Logs**:
   Look at Render / Railway deployment logs to confirm that:
   - `init_db()` ran successfully.
   - `seed_scenarios()` seeded the default scenarios into your database.

4. **Test Google Sign-In**:
   Navigate to:
   ```
   https://<your-backend-url>/api/v1/auth/google/login
   ```
   Sign in with Google. It should redirect you to `/docs` with a valid session cookie.

---

## 7. Next Steps: Connecting with Frontend (Step 2)

Once your backend is live:
1. Save your backend URL (e.g. `https://ta-coach-backend.onrender.com`).
2. Proceed to **Step 2 (Vercel Frontend)**, where you will set up a `vercel.json` rewrites file in `frontend/` so `/api/:path*` proxies to this backend URL.
3. Once Vercel assigns your frontend URL (e.g. `https://ta-coach.vercel.app`):
   - Update `FRONTEND_URL` in your backend environment variables.
   - Add `https://ta-coach.vercel.app/api/v1/auth/google/callback` to Google Cloud Console.
