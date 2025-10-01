# Puter on Cloudflare Pages – Development Plan

## Overview
This document outlines the approach to run the Puter GUI as a static site on Cloudflare Pages without a backend, enabling local development login and a visible, interactive desktop. It also captures a roadmap to evolve toward a proper cloud-backed solution.

## Current Architecture (Summary)
- GUI: Webpack-bundled SPA in `src/gui`, builds to `src/gui/dist/`.
- Puter SDK (puter-js): expects a live API for auth, KV, FS, etc.
- Backend (Node/Express): handles login/signup, Turnstile captcha, KV/FS, sessions.
- Cloudflare Pages: static hosting (no server-side processing).

## Static Deployment Challenges
- Auth endpoints (`/login`, `/signup`) and captcha validation require a backend.
- GUI relies on Puter SDK for KV/FS; calls fail without API.
- First-visit/temp-user flow triggers Turnstile and server calls.

## Phase 1 — Local Development Auth (Implemented)
Goals
- Remove captcha blocker.
- Allow local login/signup fully in the browser (no backend).
- Render the Desktop without backend FS calls.

Key Decisions
- “Static mode” flag is passed from the build HTML into the GUI, signaling no backend.
- Login/Signup UI short-circuits to client-only flows when static mode is true.
- Desktop boot avoids network calls and renders a minimal placeholder desktop fsentry.

What Works
- Login with local users, including default `admin` / `password`.
- Local signup: create new users in browser storage.
- Session persistence via `localStorage` (uses existing `update_auth_data`).
- Desktop interface renders with a minimal fsentry in static mode.

What’s Out of Scope (Phase 1)
- Real file system, KV, or sharing features (SDK-backed) — would require backend.
- Email confirmation, 2FA, and recovery code flows.
- Multi-device sync.

## Phase 2 — Cloud Integration (Future)
Priorities
1) Replace local auth with a real backend (options below).
2) Provide SDK endpoints for KV/FS and auth to enable full functionality.

Options
- Lightweight serverless API (Cloudflare Workers/Pages Functions) for:
  - Auth (login/signup JWT issuance)
  - KV via Durable Objects/Workers KV
  - FS abstraction to R2 (read/write/list)
- Supabase or similar backend (Auth + Postgres + Storage) with a thin compatibility layer.

Migration Steps
- Introduce API origin via env (PUTER_API_ORIGIN) and disable static_mode.
- Port static-mode auth flows to call new API.
- Gradually re-enable SDK-backed features (KV, FS, apps).

## Configuration Steps (Current)
- Cloudflare Pages
  - Build command: `npm install --ignore-scripts --workspaces=false && npm run build`
  - Output directory: `src/gui/dist`
  - Env: `PUTER_ENV=prod`; omit PUTER_API_ORIGIN to enable static mode (captcha disabled, local auth).
- Custom domain: use `https://webos.cyopsys.com` for validation.

## Code Changes (Summary)
- Build HTML generator (`src/gui/utils.js`)
  - gui(...) now receives `static_mode: true` when no `PUTER_API_ORIGIN` is set.
- GUI bootstrap (`src/gui/src/index.js`)
  - Reads `options.static_mode`; sets `window.static_mode` and disables Turnstile script in static mode.
- Init flow (`src/gui/src/initgui.js`)
  - If static mode and authed, bypass `/whoami` and use `window.user`.
  - When rendering Desktop: in static mode pass a minimal fsentry; skip FS calls.
  - If unauthenticated: skip `/whoarewe` fetch in static mode and show login directly.
- Login (`src/gui/src/UI/UIWindowLogin.js`)
  - In static mode, authenticate locally from `localStorage` (seed `admin/admin@local` + `password`).
- Signup (`src/gui/src/UI/UIWindowSignup.js`)
  - In static mode, create local users in `localStorage`; bypass Turnstile.

## Data Model (Static Mode)
- Local users array: `localStorage['local_users'] = [{ username, email, password, uuid }]`.
- Session: uses existing `update_auth_data(token, user)` which updates `auth_token`, `user`, `logged_in_users` and persists them.
- Tokens: simple opaque strings prefixed `local-...` (client-only; not cryptographic).

## Security Notes (Static Mode)
- Client-side passwords are stored in cleartext in localStorage for development only. Do NOT use in production.
- Tokens are not secure; they exist only to feed the existing GUI/session plumbing.
- Never enable static mode for production user data; proceed to Phase 2.

## Testing Checklist
Local (before commit)
- Build: `npm run build`
- Serve: `cd src/gui/dist && python3 -m http.server 8000`
- Validate:
  - No captcha popup/error.
  - Login with `admin` / `password` succeeds and reloads.
  - Desktop interface renders (no infinite spinner/blue screen).
  - Refresh persists session; logout/login cycles work (via existing UI where available).
  - Signup flow creates a new local user; immediate login works and desktop renders.

Cloudflare Pages
- Wait for deploy, test `https://webos.cyopsys.com/`.
- Confirm the same validations as local.

## Roadmap
- Phase 1.1: Add a file-backed mock FS (IndexedDB) for demoing file operations locally.
- Phase 2.0: Minimal serverless API for Auth (Turnstile optional), KV, and FS.
- Phase 2.1: Migrate persistent storage to R2/Workers KV and introduce real tokens.
- Phase 2.2: Enable feature parity (apps, sharing, welcome flows, preferences).

## Risks / Limitations
- Many SDK calls will still fail in static mode; code attempts to avoid hard failures for desktop boot.
- Future merges from upstream may touch the same areas; keep patches isolated and behind `window.static_mode` where possible.

## Maintenance
- Document environment choices in `docs/cloudflare-pages.md`.
- Keep `static_mode` code paths small, guarded, and easily removable when backend is introduced.

