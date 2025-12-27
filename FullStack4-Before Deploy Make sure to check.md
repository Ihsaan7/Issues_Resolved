# SnippetVault — Deployment Issues & Fixes (Vercel)
This file is a short “read before deploy” checklist of issues that happened during deployment and how we fixed them.

## Deployment style used
We deploy as two separate Vercel projects:
- Frontend: `Client/`
- Backend: `Server/`

## Frontend: React Router refresh = Page Not Found
### Symptom
Refreshing routes like `/dashboard/snippets` or `/public/snippets/:id` shows 404.

### Fix
Add SPA rewrite:
- `Client/vercel.json`

## Backend: Serverless function crash (uploads directory)
### Symptom
Backend crashed with:
- `ENOENT: no such file or directory, mkdir '/var/task/uploads'`

### Cause
Vercel serverless filesystem is read-only except `/tmp`.

### Fix
Use `/tmp/uploads` on Vercel:
- `Server/src/middlewares/multer.mware.js`

## Backend: Function crash / unstable DB connection in serverless
### Symptom
Serverless function crashed or kept reconnecting.

### Fix
- Cache Mongo connection/promise (serverless-friendly):
  - `Server/src/DB/db.js`
- Health route should not hard-fail if DB/env is misconfigured:
  - `Server/api/index.js`

## Backend: Express on Vercel (serverless entry)
### Fix
We added a Vercel entrypoint and routing rules:
- `Server/api/index.js`
- `Server/vercel.json`

Routes are forwarded to the handler:
- `/api/v1/*`
- `/health`

## Backend: CORS blocked after deploy
### Symptom
Frontend requests fail due to CORS.

### Fix
- Allow `FRONTEND_URL`
- Also allow Vercel preview/prod domains (`*.vercel.app`) so previews don’t break
File:
- `Server/src/app.js`

## Auth broke on deploy (cookies cross-domain)
### Symptom
Protected endpoints returned “Unauthorized” on Vercel even after login.

### Cause
Cross-site cookies may not be set/sent reliably between `*.vercel.app` domains.

### Fix (deploy-safe)
Use Authorization header tokens:
- Backend now returns tokens in response body for login/register
- Frontend stores token in localStorage and injects it on every request

Files:
- `Server/src/controllers/auth.controller.js` (returns `accessToken` + `refreshToken`)
- `Server/src/middlewares/auth.mware.js` (prefers `Authorization: Bearer ...`)
- `Client/src/context/AuthContext.jsx` (stores `sv_access_token`)
- `Client/src/utils/axios.js` (adds Authorization header)

## Backend: Misleading 404 for unauthorized
### Symptom
Browser showed 404 for protected endpoints.

### Cause
Auth middleware used status code 404 for missing token.

### Fix
Return 401 for unauthorized:
- `Server/src/middlewares/auth.mware.js`

## Backend: Register response shape mismatch
### Symptom
Frontend didn’t set user state properly after register.

### Fix
Return consistent shape (same as login):
- `Server/src/controllers/auth.controller.js`

## Backend: `next is not a function` (Mongoose middleware)
### Symptom
Register caused:
- `TypeError: next is not a function` from `User.model.js`

### Cause
Async Mongoose middleware should not use `next` callback.

### Fix
Use promise-based pre-save hook:
- `Server/src/models/User.model.js`

## Frontend: Hidden required avatar input caused browser warning
### Symptom
Submitting register without image:
- `An invalid form control ... is not focusable`

### Fix
- Remove `required` from hidden file input
- Do manual validation + show UI error
File:
- `Client/src/components/auth/RegisterationForm.jsx`

## Required Vercel Environment Variables
### Backend (Server project)
- `MONGODB_URI`
- `ACCESS_TOKEN`
- `ACCESS_TOKEN_EXPIRY`
- `REFRESH_TOKEN`
- `REFRESH_TOKEN_EXPIRY`
- `FRONTEND_URL`
- `CLOUDI_NAME`
- `CLOUDI_API_KEY`
- `CLOUDI_API_SECRET`

### Frontend (Client project)
- `VITE_BACKEND_URL` = `https://<backend>.vercel.app/api/v1`

## Quick post-deploy smoke tests
1) Backend:
- `/health` returns JSON
- `/api/v1/snippets/public` returns JSON

2) Frontend:
- Refresh on `/public/snippets` works
- Login works and `sv_access_token` is saved in LocalStorage
- Create snippet works (no Unauthorized)
