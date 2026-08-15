# Deploying AEGIS AI

AEGIS is split into two deployable units:

- **Frontend** — Next.js 16 app (root) → deployed to **Vercel**
- **Backend** — Express + Socket.IO + MongoDB worker (`server/`) → deployed to **Render**

The frontend talks to the backend over HTTP + WebSockets using the
`NEXT_PUBLIC_BACKEND_URL` environment variable (falls back to `localhost:3001`
in dev). The backend is told which frontend to allow via `FRONTEND_URL`.

---

## 1. Prerequisites

- Node.js 20 (`.nvmrc` is included)
- An AWS-free [MongoDB Atlas](https://www.mongodb.com/cloud/atlas) cluster
- [Clerk](https://clerk.com) account (auth) — publishable + secret key
- [Cloudinary](https://cloudinary.com) account (image upload)
- [Groq](https://groq.com) API key (LLM verification pipeline)
- [OpenAI](https://platform.openai.com) API key
- [Copernicus Data Space Ecosystem](https://dataspace.copernicus.eu/) OAuth
  client (for Sentinel Hub satellite imagery)

---

## 2. Deploy the Backend to Render

`render.yaml` defines the service. You can deploy with one click or manually.

### Option A — "Deploy with Render" from the repo (recommended)

1. Push this repo to GitHub.
2. Go to **Render → New + → Web Service** and connect the GitHub repo.
3. Render reads `render.yaml` automatically:
   - Build command: `npm install`
   - Start command: `npm run start:backend`
   - Health check: `/api/health`
4. **Set the required environment variables** in the Render Dashboard
   (`aegis-backend → Environment → Environment Variables`):

   | Variable                | Source                                   |
   | ----------------------- | ---------------------------------------- |
   | `MONGODB_URI`           | MongoDB Atlas connection string          |
   | `CLOUDINARY_CLOUD_NAME` | Cloudinary cloud name                    |
   | `CLOUDINARY_API_KEY`    | Cloudinary API key                       |
   | `CLOUDINARY_API_SECRET` | Cloudinary API secret                    |
   | `GROQ_API_KEY`          | Groq API key                             |
   | `OPENAI_API_KEY`        | OpenAI API key                           |
   | `SENTINEL_CLIENT_ID`    | Copernicus client id                     |
   | `SENTINEL_CLIENT_SECRET`| Copernicus client secret                 |
   | `FRONTEND_URL`          | Your Vercel frontend URL (set in step 3) |

5. Click **Create Web Service**. Render builds and starts the backend.
   Confirm `https://<your-service>.onrender.com/api/health` returns
   `{"status":"ok",...}`.

### Option B — Manual (no render.yaml)

1. Create a **Web Service** on Render, connect the repo.
2. Build command: `npm install`
3. Start command: `npm run start:backend`
4. Leave root directory as `/` and set the env vars above.
5. Add a health check on `/api/health`.

> The free plan is fine for a hackathon; it sleeps after ~15 min of inactivity
> (cold start). Upgrade to a paid plan for always-on realtime.

---

## 3. Deploy the Frontend to Vercel

1. Push this repo to GitHub (or link Vercel directly).
2. In Vercel → **New Project** → import the repo.
3. Framework preset: **Next.js** (auto-detected). Build command: `next build`.
   Output directory: `.next`.
4. **Set the required environment variables** in Vercel
   (`Settings → Environment Variables`). Each var should be available in
   **Production** (and optionally Preview/Development).

   #### Client-exposed (prefix `NEXT_PUBLIC_`):

   | Variable                          | Value                                          |
   | --------------------------------- | --------------------------------------------- |
   | `NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY` | Clerk publishable key                       |
   | `NEXT_PUBLIC_CLOUDINARY_CLOUD_NAME` | Cloudinary cloud name                       |
   | `NEXT_PUBLIC_BACKEND_URL`         | `https://<your-render-backend>.onrender.com` |
   | `NEXT_PUBLIC_SOCKET_URL`          | `https://<your-render-backend>.onrender.com` |

   #### Server-side only (no prefix):

   | Variable                  | Value                            |
   | ------------------------- | --------------------------------- |
   | `CLERK_SECRET_KEY`        | Clerk secret key                  |
   | `SENTINEL_CLIENT_ID`      | Copernicus client id (used by the Next.js `/api/sentinel/image` route) |
   | `SENTINEL_CLIENT_SECRET`  | Copernicus client secret          |

5. Click **Deploy**. After build, verify the live URL works and auth/sign-up
   flow completes.

---

## 4. Environment Variable Quick Reference

```
# Backend (Render)                           Frontend (Vercel)
MONGODB_URI                                  NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY
CLOUDINARY_CLOUD_NAME                        CLERK_SECRET_KEY
CLOUDINARY_API_KEY                           NEXT_PUBLIC_CLOUDINARY_CLOUD_NAME
CLOUDINARY_API_SECRET                        NEXT_PUBLIC_BACKEND_URL
GROQ_API_KEY                                 NEXT_PUBLIC_SOCKET_URL
OPENAI_API_KEY                               SENTINEL_CLIENT_ID   (server-side only)
SENTINEL_CLIENT_ID                           SENTINEL_CLIENT_SECRET (server-side only)
SENTINEL_CLIENT_SECRET
FRONTEND_URL  <-- Vercel URL
```

---

## 5. Local Development

```bash
npm install
npm run dev   # runs frontend (3000) + backend (3001) concurrently
```

Both read from `.env.local` (already present and git-ignored). The frontend
proxies to `http://localhost:3001` automatically via the env fallback.

---

## 6. Smoke Test After Deploy

1. Visit your Vercel frontend URL.
2. Sign up / sign in via Clerk.
3. Open the dashboard — incidents & alerts should load (via the Render backend).
4. Open browser devtools → Networking → confirm WebSocket connects to the
   Render backend (`wss://<your-backend>.onrender.com`) and you receive
   `dashboard:sync` events.
5. Submit a report from `/report` and watch the AI verification queue update
   in real time.
