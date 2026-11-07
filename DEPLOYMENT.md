# Blox Fruit Services — Complete Vercel Deployment Guide

This guide provides end-to-end instructions for deploying the **Blox Fruit Services** full-stack application to **Vercel**.

---

## 1. Architecture Overview

- **Frontend**: High-performance React 19 + Vite 8 SPA styled with Tailwind CSS v4, built to static HTML/JS/CSS in the `dist/` directory.
- **Backend API**: Express serverless handler located at `api/index.ts` (proxied from `server.ts`) and routed through `vercel.json`.
- **Database**:
  - **Neon PostgreSQL**: Serverless database queried via `@neondatabase/serverless` (HTTP driver with zero connection pool limits).
  - **Firebase Firestore**: Production persistence and Firebase Authentication (Google Sign-In with server-controlled Custom Claims).
  - **Local In-Memory / File Fallback**: Automatic seamless fallback ensuring the application boots and serves orders even before databases are connected.
- **Security**:
  - Staff sessions use `HttpOnly; Secure; SameSite=Lax` cookies.
  - Google Sign-In validates Firebase ID tokens server-side and checks for `{ admin: true }` Custom Claims.
  - Automated research cron endpoints are secured with `CRON_SECRET`.

---

## 2. Vercel Free Tier & Commercial Eligibility

> [!NOTE]
> ### Roblox Virtual Currency & Vercel Hobby Tier
> - **In-Game Roblox Currency Only**: Blox Fruit Services deals strictly with **in-game virtual fruits and gamepasses (M units)**. There is zero fiat processing, credit card checkout, or real-money payment processing.
> - **Eligibility**: As an in-game gaming community co-op service, running this on the **Vercel Hobby Tier ($0/month)** is fully compliant. If you ever introduce real-world fiat monetization (e.g. Stripe, PayPal, paid subscriptions), Vercel's Terms of Service require upgrading to Vercel Pro.

---

## 3. Deployment Method A: Deploy via GitHub (Recommended)

This is the easiest and most maintainable method. Every `git push` to your repository will automatically trigger an atomic preview or production build.

### Step 1: Push Code to GitHub
1. Initialize or push your project repository to GitHub:
   ```bash
   git add .
   git commit -m "Configure app for Vercel deployment"
   git push origin main
   ```

### Step 2: Import into Vercel
1. Log in to [vercel.com](https://vercel.com) (sign up with GitHub if you haven't already).
2. On your Vercel Dashboard, click **Add New…** → **Project**.
3. Locate your repository from the list and click **Import**.

### Step 3: Configure Build & Output Settings
Vercel will detect Vite automatically. Verify the following project settings:
- **Framework Preset**: `Vite`
- **Root Directory**: `./` (leave default)
- **Build Command**: `npm run build`
- **Output Directory**: `dist`
- **Install Command**: `npm install`

### Step 4: Add Environment Variables
Before clicking Deploy, expand the **Environment Variables** section in the Vercel setup screen and add the required variables (see [Environment Variables Reference](#5-environment-variables-reference) below):

| Name | Recommended Value | Purpose |
| :--- | :--- | :--- |
| `NODE_ENV` | `production` | Enables production optimizations |
| `ADMIN_PASSWORD` | *(Your secure password)* | Master password for fallback staff login |
| `SESSION_SECRET` | *(64-character random string)* | Signs HMAC-SHA256 staff session cookies |
| `VITE_DISCORD_INVITE_URL` | `https://discord.gg/JeZSJZb2aA` | Official Discord community & proof server |
| `CRON_SECRET` | *(Strong random string)* | Protects automated research updates |
| `DATABASE_URL` | *(Optional Neon connection string)* | Enables Neon PostgreSQL storage |
| `GEMINI_API_KEY` | *(Optional Google AI API key)* | Enables live Google Search Grounding for fruit values |

### Step 5: Click Deploy
Click **Deploy**. Vercel will:
1. Install dependencies.
2. Build the Vite frontend into `dist/`.
3. Package `api/index.ts` into a lightweight serverless function.
4. Provide you with a production URL (e.g. `https://blox-fruit-services.vercel.app`).

---

## 4. Deployment Method B: Deploy via Vercel CLI

If you prefer deploying directly from your terminal:

1. Install the Vercel CLI globally:
   ```bash
   npm install -g vercel
   ```
2. Log in to your Vercel account:
   ```bash
   vercel login
   ```
3. Deploy to production from the project root:
   ```bash
   vercel --prod
   ```
4. Follow the interactive prompts:
   - *Set up and deploy?* **Y**
   - *Which scope?* Select your account/team
   - *Link to existing project?* **N**
   - *Project name?* `blox-fruit-services`
   - *Directory?* `./`
   - *Want to modify settings?* **N** (Vercel reads `vercel.json` automatically)
5. Add your environment variables either in the Vercel web dashboard under **Project Settings > Environment Variables**, or via CLI:
   ```bash
   vercel env add ADMIN_PASSWORD
   vercel env add SESSION_SECRET
   vercel env add VITE_DISCORD_INVITE_URL
   ```

---

## 5. Environment Variables Reference

| Variable | Required? | Description |
| :--- | :---: | :--- |
| `NODE_ENV` | Yes | Set to `production`. |
| `ADMIN_PASSWORD` | Yes | Strong passphrase used to log in to `/admin` staff portal. Can also be a bcrypt hash (starting with `$2b$`). |
| `SESSION_SECRET` | Yes | 64-character random string for signing staff cookies. |
| `VITE_DISCORD_INVITE_URL` | Yes | Official Discord invite link: `https://discord.gg/JeZSJZb2aA`. |
| `CRON_SECRET` | Recommended | Token required to call `POST /api/cron/research-update`. |
| `DATABASE_URL` | Optional | Neon PostgreSQL connection URI (`postgresql://user:pass@ep-xyz.neon.tech/neondb?sslmode=require`). |
| `GEMINI_API_KEY` | Optional | Used for live Google Search Grounding when researching Blox Fruits trade tier lists. |
| `FIREBASE_PROJECT_ID` | Optional | Custom Firebase Project ID if overriding the built-in config. |
| `FIREBASE_DATABASE_ID` | Optional | Custom Firestore Database ID if using a named database. |
| `FIREBASE_SERVICE_ACCOUNT` | Optional | Single-line JSON string of your Firebase Service Account key for server-side Admin SDK operations. |

---

## 6. Configuring Firebase Authentication for Vercel

If you use Google Sign-In for Admin Login on your deployed Vercel site:

1. Open the [Firebase Console](https://console.firebase.google.com/).
2. Select your project (`gen-lang-client-0400929459` or your linked project).
3. In the left navigation, go to **Build** → **Authentication** → **Settings** tab.
4. Scroll to **Authorized domains**.
5. Click **Add domain** and enter your Vercel domain (e.g. `your-project-name.vercel.app` or your custom domain).
6. Click **Save**.

> This ensures Google Sign-In popup will not fail with `auth/unauthorized-domain` when accessed on your Vercel URL.

---

## 7. Automated Scheduling (Vercel Cron Jobs)

To automate market research and catalogue value refreshes without manual intervention:

You can configure Vercel Cron Jobs by adding a `crons` array to `vercel.json`:

```json
{
  "version": 2,
  "rewrites": [
    {
      "source": "/api/(.*)",
      "destination": "/api"
    },
    {
      "source": "/(.*)",
      "destination": "/index.html"
    }
  ],
  "crons": [
    {
      "path": "/api/cron/research-update",
      "schedule": "0 4 * * *"
    }
  ]
}
```

*Note: Vercel Cron Jobs invoke the endpoint with header `x-vercel-cron: 1`. You can also trigger it from an external scheduler or GitHub Actions workflow by passing the `x-cron-secret: <YOUR_CRON_SECRET>` header.*

---

## 8. Post-Deployment Verification Checklist

Once deployed, run through this quick checklist to ensure everything operates smoothly:

- [ ] **Home & Catalog (`/services`)**: Carries load with their verified in-game trade values (M units).
- [ ] **Booking Engine (`/order`)**: Submit a test booking; verify that an order code (`BFS-XXXX`) and private tracking token are generated.
- [ ] **50/50 Escrow Calculator (`/track?token=...`)**: Verify the initial 50% deposit and remaining 50% on completion calculate accurately.
- [ ] **Discord Community Links**: Verify clicking the Discord buttons in the Header, Footer, and Support page opens `https://discord.gg/JeZSJZb2aA` in a new tab.
- [ ] **Staff Admin Portal (`/admin`)**:
  - Test Admin Login with `ADMIN_PASSWORD` or Google Sign-In.
  - Verify orders can be viewed, updated, or accepted.
- [ ] **Proofs Wall (`/proofs`) & Ongoing Carries (`/ongoing`)**: Confirm anonymous public privacy masks work correctly.
