# Step 2: Deploying the Frontend (React + Vite on Vercel)

This guide walks you through deploying the **TA Coach Frontend** to **Vercel** (100% Free) and connecting it to your deployed Render backend.

---

## Table of Contents
1. [Why Vercel + Rewrites?](#1-why-vercel--rewrites)
2. [Step 2.1: Configure `vercel.json`](#step-21-configure-verceljson)
3. [Step 2.2: Deploy on Vercel](#step-22-deploy-on-vercel)
4. [Step 2.3: Connect Backend & Google OAuth](#step-23-connect-backend--google-oauth)
5. [Verification Checklist](#verification-checklist)

---

## 1. Why Vercel + Rewrites?

The frontend makes requests to relative paths (`/api/v1/...`) and relies on HTTP-only session cookies (`ta_coach_session`).

By using a `vercel.json` **rewrite**:
- The browser visits `https://your-frontend.vercel.app`.
- Any call to `/api/*` is transparently proxied by Vercel's edge network to your Render backend (`https://your-backend.onrender.com/api/*`).
- **No CORS configuration needed**, and browser third-party cookie restrictions are completely avoided because everything stays on the same frontend domain!

---

## 2. Step 2.1: Configure `vercel.json`

In the [`frontend/`](file:///c:/Users/azure/Code/Azure/transaction-analysis/frontend) directory, create a file named `vercel.json` (or copy from [`frontend/vercel.json.example`](file:///c:/Users/azure/Code/Azure/transaction-analysis/frontend/vercel.json.example)):

```json
{
  "rewrites": [
    {
      "source": "/api/:path*",
      "destination": "https://<YOUR-RENDER-BACKEND-URL>/api/:path*"
    },
    {
      "source": "/docs",
      "destination": "https://<YOUR-RENDER-BACKEND-URL>/docs"
    },
    {
      "source": "/openapi.json",
      "destination": "https://<YOUR-RENDER-BACKEND-URL>/openapi.json"
    },
    {
      "source": "/(.*)",
      "destination": "/index.html"
    }
  ]
}
```

> **Replace `<YOUR-RENDER-BACKEND-URL>`** with your actual Render backend domain, e.g.:
> `https://ta-coach-backend.onrender.com`

---

## 3. Step 2.2: Deploy on Vercel

1. Log in to [Vercel.com](https://vercel.com) (free account with GitHub or Google).
2. Click **Add New...** > **Project**.
3. Import your GitHub repository.
4. Configure the project settings:
   - **Framework Preset**: `Vite`
   - **Root Directory**: Click *Edit* and select **`frontend`**
   - **Build Command**: `npm run build` (default)
   - **Output Directory**: `dist` (default)
   - **Install Command**: `npm install` (default)
5. Click **Deploy**.

Vercel will build your React application in ~30 seconds and assign you a live production URL (e.g. `https://ta-coach-frontend.vercel.app`).

---

## 4. Step 2.3: Connect Backend & Google OAuth

Now that your frontend has a live production URL, update two final settings:

### 1. Update Backend `FRONTEND_URL` on Render
1. Open your [Render Dashboard](https://dashboard.render.com).
2. Go to your backend service > **Environment**.
3. Update the `FRONTEND_URL` variable to your new Vercel domain:
   ```text
   FRONTEND_URL=https://<your-vercel-domain>.vercel.app
   ```
4. Save changes. Render will automatically redeploy the backend with the new setting.

### 2. Update Google Cloud Console Credentials
1. Go to [Google Cloud Console Credentials](https://console.cloud.google.com/apis/credentials).
2. Edit your OAuth 2.0 Web Client.
3. Under **Authorized redirect URIs**, add:
   ```text
   https://<your-vercel-domain>.vercel.app/api/v1/auth/google/callback
   ```
4. Under **Authorized JavaScript origins**, add:
   ```text
   https://<your-vercel-domain>.vercel.app
   ```
5. Click **Save**.

---

## Verification Checklist

1. Open `https://<your-vercel-domain>.vercel.app` in your browser.
2. Click **Sign in with Google**.
3. Authenticate with your Google account.
4. Verify you are redirected back into the app dashboard.
5. Click on any scenario (e.g. *Delegating without micromanaging*), send a message, and ensure AI responses stream back smoothly!
