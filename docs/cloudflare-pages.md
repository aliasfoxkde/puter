## Puter GUI on Cloudflare Pages (Static PWA)

This guide explains how to deploy the Puter web GUI as a static Progressive Web App (PWA) on Cloudflare Pages. The GUI will talk to a running Puter backend via API. Cloudflare Workers are not required unless you choose to proxy the API.

Important: We fixed the prior login configuration issue by ensuring pub_port is set and CORS is correctly applied. The /login endpoint works when an Origin header is present. Follow this guide to produce a static build that’s compatible with Cloudflare Pages and your API origin.

---

### 1) Prerequisites
- A reachable Puter backend (for example: https://api.your-domain.com or http://puter.localhost:4100 during local testing)
- Node.js 20 on the build environment
- This repository connected to a Cloudflare Pages project

---

### 2) One-time notes about the repo
- Static GUI sources live at: src/gui
- The build produces assets at: src/gui/dist
- We enhanced the build to produce:
  - dist/index.html (bootstraps the GUI)
  - dist/_redirects (enables SPA routing on Cloudflare Pages)
  - PWA assets (manifest.json, icons, favicon) are already included

---

### 3) Configure your API origin (critical)
The static GUI must know where your Puter API lives. Provide this at build time via environment variables:
- PUTER_API_ORIGIN: e.g., https://api.your-domain.com (production) or http://puter.localhost:4100 (local)
- Optional: PUTER_GUI_ORIGIN if you want to override the GUI base origin for metadata

These are embedded into dist/index.html during the build and passed to window.gui({ api_origin, gui_origin }).

Backend-side CORS: The backend already sets Access-Control-Allow-Origin for /login and API traffic. If your API is not served from an api subdomain, enable experimental_no_subdomain in your backend config to allow CORS for all:
- volatile/config/config.json → { "experimental_no_subdomain": true }

---

### 4) Local build (optional sanity check)
From the repo root:

```
PUTER_API_ORIGIN=http://puter.localhost:4100 npm run build
```

This runs node src/gui/build.js and writes static files to src/gui/dist.

You can serve src/gui/dist with any static file server to test (ensure your API is running and CORS allows your test origin). Example: Python http.server (or any equivalent).

---

### 5) Cloudflare Pages settings
Set these in your Cloudflare Pages project:

- Build command:
```
npm install --ignore-scripts --workspaces=false && npm run build
```
Note: The repository does not include a package-lock.json, so `npm ci` would fail. We also disable workspaces during install and use a webpack alias so the GUI build does not rely on workspace linking or run workspace scripts during install.

- Output directory:
```
src/gui/dist
```

- Environment variables:
```
NODE_VERSION=20
PUTER_ENV=prod
PUTER_API_ORIGIN=https://api.your-domain.com
# Optional
PUTER_GUI_ORIGIN=https://your-pages-domain.pages.dev
```

- Framework preset: None (this is a vanilla static site)

The build process will produce:
- index.html that bootstraps the GUI and includes gui.js
- bundle.min.css / bundle.min.js
- manifest.json + icons (installable PWA)
- _redirects to route all paths to index.html (SPA)

---

### 6) PWA notes
- Installability: manifest.json and icons are included; theme-color is set. This is sufficient for “installable” PWA in most browsers.
- Offline: A service worker is not included by default. If you want offline caching, we can add a small service worker in a follow-up change.

---

### 7) Verification checklist
After Pages deploys:
- Visit your Pages URL
- Open DevTools → Application → Manifest to confirm PWA installability
- Attempt login → ensure network calls hit PUTER_API_ORIGIN and succeed (CORS should allow your Pages origin)
- Try core flows: whoami, listing files, opening apps

---

### 8) Troubleshooting
- Login fails with 403: Ensure requests include Origin (browsers do), and that backend CORS allows your Pages origin. Consider setting experimental_no_subdomain=true in backend config if not on api subdomain.
- Mixed content errors: Use HTTPS for both Pages and API in production.
- 404s on deep links: Confirm dist/_redirects exists and Pages is using it.
- Wrong API origin: Rebuild Pages with corrected PUTER_API_ORIGIN.

---

### 9) Summary of changes enabling Cloudflare Pages
- Added generation of dist/index.html and dist/_redirects during GUI build
- Ensured webpack context works when building from repo root
- index.html now calls gui({...}) with api_origin/gui_origin when provided via env

### Troubleshooting: Git Submodule Issues
Cloudflare Pages may attempt to initialize git submodules automatically during the clone phase. If submodules use SSH URLs (git@github.com:...), the clone can fail with an error like:

- "Failed: error occurred while updating repository submodules"

Resolution applied in this repo:
- Switched submodule URLs in .gitmodules to public HTTPS URLs so Cloudflare Pages can fetch them without SSH keys:
  - submodules/v86 → https://github.com/HeyPuter/v86.git
  - submodules/twisp → https://github.com/MercuryWorkshop/twisp.git
  - submodules/epoxy-tls → https://github.com/MercuryWorkshop/epoxy-tls.git
  - submodules/wiki already used HTTPS

Notes:
- The GUI static build in src/gui does not require these submodules, but leaving them fetchable avoids clone failures on Pages.
- If you prefer to avoid submodule fetching entirely, you could remove .gitmodules in a dedicated branch used only by Pages. However, using HTTPS URLs is the simplest, least disruptive fix.

You can now deploy the GUI as a static site on Cloudflare Pages that connects to your Puter backend.

