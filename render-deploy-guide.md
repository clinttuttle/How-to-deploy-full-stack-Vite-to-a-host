# Render Deployment Guide

## Vite React Frontend + Node Backend (Monorepo)

> **Repo structure assumed:**
> 
> ```
> /
> ├── frontend/   ← Vite React app
> └── backend/    ← Node.js API server
> ```

---

## Part 1 — Deploy the Backend (Web Service)

### Step 1: Create a New Web Service

1. Log in to [render.com](https://render.com) and click **New → Web Service**
2. Connect your GitHub account if you haven't already
3. Select your repository

### Step 2: Configure the Service Settings

| Field              | Value                                                                              |
| ------------------ | ---------------------------------------------------------------------------------- |
| **Name**           | `my-app-backend` (or whatever you like)                                            |
| **Region**         | Choose closest to your users                                                       |
| **Branch**         | `main` (or your prod branch)                                                       |
| **Root Directory** | `backend`                                                                          |
| **Runtime**        | `Node`                                                                             |
| **Build Command**  | `npm install`                                                                      |
| **Start Command**  | `node index.js` *(or `node server.js`, `npm start`, etc. — match your entry file)* |
| **Instance Type**  | Free (to start)                                                                    |

> **Root Directory is critical.** Setting it to `backend` tells Render to treat that folder as the project root — all commands run from inside it, and it watches only that folder for deploys.

> **Build Command:** If you have a build step (e.g. TypeScript compilation), use `npm install && npm run build`. Otherwise `npm install` alone is fine for plain Node.

---

## Part 2 — Backend Environment Variables

### Step 3: Add Environment Variables

In the service dashboard go to **Environment → Add Environment Variable**.

#### ✅ What TO include (secrets & config that differ by environment):

| Variable         | Example Value                 | Why                                                                                                                         |
| ---------------- | ----------------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| `NODE_ENV`       | `production`                  | Enables production optimizations                                                                                            |
| `PORT`           | `5001`                        | NOTE: Render assigns this automatically so if you're using  `process.env.PORT` you can omit PORT from environment variables |
| `DATABASE_URL`   | `postgresql://...`            | Any DB connection string                                                                                                    |
| `JWT_SECRET`     | `some-long-random-string`     | Auth secrets if you're using JWT                                                                                            |
| `API_KEY`        | `sk-...`                      | Third-party API keys (OpenAI, Stripe, etc.)                                                                                 |
| `SESSION_SECRET` | `...`                         | Express session secret                                                                                                      |
| `ALLOWED_ORIGIN` | `https://my-app.onrender.com` | Your frontend URL for CORS                                                                                                  |

#### ❌ What NOT to include:

- Anything already in your code (base URLs, route strings, app name)
- `NODE_MODULES` paths or system-level variables — Render handles those
- Local-only variables like `VITE_*` — those belong in the frontend
- Localhost URLs — your backend is no longer running locally
- Database credentials that are already embedded in `DATABASE_URL`

#### 💡 The Rule of Thumb:

> If it's a **secret**, **changes between environments** (local vs. prod), or is a **credential** — it's an env variable. If it's just a constant in your code, it doesn't belong here.

### Step 4: Make Sure Your Server Uses `process.env.PORT`

Render dynamically assigns a port. Your server **must** listen on it:

```js
// backend/index.js
const PORT = process.env.PORT || 3000;
app.listen(PORT, () => console.log(`Server running on port ${PORT}`));
```

---

## Part 3 — Deploy the Frontend (Static Site)

### Step 5: Create a New Static Site

1. Click **New → Static Site**
2. Select the same repository

### Step 6: Configure the Static Site Settings

| Field                 | Value                          |
| --------------------- | ------------------------------ |
| **Name**              | `my-app-frontend`              |
| **Branch**            | `main`                         |
| **Root Directory**    | `frontend`                     |
| **Build Command**     | `npm install && npm run build` |
| **Publish Directory** | `dist`                         |

> **Publish Directory** is relative to the Root Directory. Vite always outputs to `dist` by default, so this is almost always `dist` unless you've customized `vite.config.js`.

---

## Part 4 — Connect Frontend to Backend API URL

### Step 7: Get Your Backend URL

Once the backend deploys successfully, Render gives it a URL like:

```
https://my-app-backend.onrender.com
```

Copy this from the backend service's dashboard at the top of the page.

### Step 8: Add the API URL to the Frontend Environment Variables

In your **frontend** Static Site dashboard → **Environment → Add Environment Variable**:

| Variable       | Value                                 |
| -------------- | ------------------------------------- |
| `VITE_API_URL` | `https://my-app-backend.onrender.com` |

> **Vite requires the `VITE_` prefix** for any env variable to be exposed to the browser. Variables without it are ignored by Vite at build time.

### Step 9: Use It in Your Frontend Code

```js
// frontend/src/api.js (or wherever you make requests)
const BASE_URL = import.meta.env.VITE_API_URL;

export const fetchData = async () => {
  const res = await fetch(`${BASE_URL}/api/your-endpoint`);
  return res.json();
};
```

### Step 10: Trigger a Frontend Redeploy

Environment variables in Vite are **baked in at build time**, not runtime. After adding `VITE_API_URL` you must redeploy the frontend:

1. Go to your Static Site dashboard
2. Click **Manual Deploy → Deploy latest commit**

> The new build will pick up `VITE_API_URL` and embed it into the compiled JS bundle.

---

## Part 5 — Backend CORS Configuration

Your backend needs to explicitly allow requests from your frontend's Render URL.

```js
// backend/index.js
import cors from 'cors';

app.use(cors({
  origin: process.env.ALLOWED_ORIGIN, // set this in Render env vars
  credentials: true,
}));
```

Set `ALLOWED_ORIGIN` in your backend's environment variables:

```
ALLOWED_ORIGIN = https://my-app-frontend.onrender.com
```

---

## Quick Reference Cheatsheet

```
BACKEND (Web Service)
├── Root Directory:   backend
├── Build Command:    npm install
├── Start Command:    node index.js
└── Key Env Vars:     NODE_ENV, DATABASE_URL, JWT_SECRET, ALLOWED_ORIGIN

FRONTEND (Static Site)
├── Root Directory:   frontend
├── Build Command:    npm install && npm run build
├── Publish Dir:      dist
└── Key Env Vars:     VITE_API_URL
```

---

## Common Gotchas

- **Free tier spins down after 15 min of inactivity** — the first request after idle will be slow (~30s). Upgrade to a paid tier or use a cron job to ping it.
- **Forgetting to redeploy the frontend** after adding `VITE_API_URL` — Vite bakes env vars at build time, so a redeploy is always required after changing them.
- **Using `VITE_` prefix on backend vars** — those are frontend-only. Backend reads vars with `process.env` directly, no prefix needed.
- **Hardcoding `localhost` in frontend API calls** — always use `import.meta.env.VITE_API_URL` so it works in both environments.
