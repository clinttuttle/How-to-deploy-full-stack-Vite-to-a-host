<u>**KEY things to get right**</u>

**Root Directory** is the big one — setting it to `backend` or `frontend` respectively tells Render to scope all commands and file watching to that subfolder, which is essential for a monorepo setup.

**The env variable decision rule** is simple: if it's a secret, a credential, or something that changes between local and production — it's an env var. If it's just a value hardcoded in your source, leave it in the code.

**The Vite `VITE_` prefix trap** catches a lot of people — any env var you want accessible in your React app *must* start with `VITE_`, and after adding or changing it you *must* trigger a manual redeploy since Vite bakes them in at build time, not runtime.

**CORS** is the other common failure point — make sure your backend's `ALLOWED_ORIGIN` is set to the exact frontend Render URL (no trailing slash), and that you're reading it from `process.env` rather than hardcoding it.
