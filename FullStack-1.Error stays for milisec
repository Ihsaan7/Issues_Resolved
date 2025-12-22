# “Error shows for 1 second then disappears” (common full‑stack bug)

#### What you see
- You type wrong credentials (or any request fails).
- The UI shows an error message briefly.
- Then the page “refreshes” / redirects and the error disappears.

#### The real cause (almost always)
A **global auth handler** is doing a **redirect or page reload** when it sees a certain HTTP status (usually **401/403**).

Common places this happens:
- Axios response interceptor (`axios.interceptors.response`)
- Fetch wrapper / API client wrapper
- React Router guard / ProtectedRoute
- Auth context `useEffect` that navigates when user becomes “not logged in”

When that redirect/reload happens:
- Your app reloads or navigates away
- Component state resets (including `error`)
- So the error message vanishes instantly

## The classic mistake
Redirecting on 401 without checking **where** the 401 came from.

Example: login with wrong password returns **401** (normal).
But your interceptor treats *any* 401 as “token expired / user must login” and redirects to `/login`.
If you’re already on `/login`, you can even create a redirect-to-same-page refresh loop.

Also common:
- Comparing the wrong URL values (example: comparing `window.location.href`
  which is a full URL like `http://localhost:5173/login`
  against `"/login"` which will never match).

#### Common fixes (pick 1–2)
##### Fix 1: Don’t redirect for auth endpoints
In your interceptor, skip redirect if the failed request is:
- `/auth/login`
- `/auth/register`
- `/auth/refresh` (depending on your setup)

This prevents “wrong password” from triggering a forced navigation.

### Fix 2: Don’t hard reload the page
Avoid:
- `window.location.href = ...`
- `window.location.reload()`

Prefer:
- React Router navigation (`navigate("/login")`)
- Rendering `<Navigate to="/login" />` in a route guard

Hard reload wipes state instantly, which makes errors “flash”.

##### Fix 3: Only redirect when you’re truly “logged out”
Instead of “if status === 401 => redirect”, use logic like:
- redirect only when access token is missing/invalid AND refresh attempt failed
- redirect only for requests that require auth (protected routes / protected API calls)

##### Fix 4: Make sure you don’t redirect when already on the same page
If you do use window.location:
- check `window.location.pathname === "/login"` (pathname)
- not `window.location.href === "/login"` (href includes domain)

#### Quick mental checklist (next time you see this)
1. Did the request return 401/403?
2. Do I have an interceptor/guard that redirects on 401/403?
3. Is it also running for `/auth/login`?
4. Am I using `window.location.*` (hard reload) anywhere?
5. Is my “already on /login” check using `pathname` (correct) or `href` (often wrong)?

#### One-line takeaway
If errors “flash then disappear”, it’s usually because some global auth logic is
redirecting/reloading on an error response, clearing your UI state.
<img width="1360" height="858" alt="image" src="https://github.com/user-attachments/assets/46a232ea-2c91-4d91-b545-75b987503d2a" />

# ---------------------------------------------------------------------------------------------------------------------------------------------------------


