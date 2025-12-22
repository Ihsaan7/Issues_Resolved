# ------------------------------Troubleshooting: Unexpected Login Redirects---------------------------------------

### Problem

When navigating to a protected route, you get redirected to `/login` immediately with no errors.

### Root Causes & Solutions

#### 1. **Missing Route Definition**

**Symptom:** Click a link/button → instant redirect to login, no network requests

**Cause:** The route path doesn't exist in App.jsx. React Router hits the wildcard `*` route which redirects to `/login`.

**Fix:**

1. Open `Client/src/App.jsx`
2. Check if the route path exists (e.g., `/dashboard/snippets/:id/edit`)
3. Add the missing route inside `<Routes>`:
   ```jsx
   <Route
     path="/your/path"
     element={
       <ProtectedRoute>
         <YourPage />
       </ProtectedRoute>
     }
   />
   ```
4. Import the page component at the top

---

#### 2. **Auth State Lost on Page Refresh**

**Symptom:** Works fine until you refresh the page, then redirects to login

**Cause:** `AuthContext` state (`isVerified`) resets to `false` on refresh because it's stored in memory only.

**Fix:** Persist auth state in localStorage (already implemented):

- Login/register stores user in `localStorage.getItem("sv_user")`
- On app mount, `AuthContext` rehydrates from localStorage
- `isLoading` starts `true` to prevent redirect before hydration completes

**Verify:**

1. Open DevTools → Application → Local Storage
2. Check for `sv_user` key with user data
3. If missing after login, auth persistence isn't working

---

#### 3. **401 Unauthorized from Backend**

**Symptom:** Redirect to login + network shows 401 status

**Cause:** Axios interceptor catches 401 responses and redirects to `/login` automatically.

**Common reasons:**

- Cookies expired or not sent
- Backend route requires auth but token is invalid
- CORS blocking cookies

**Fix:**

1. Open DevTools → Network tab
2. Find the failed request (red, status 401)
3. Check Request Headers → look for cookies
4. If no cookies: Check `withCredentials: true` in axios config
5. If cookies present but 401: Backend token validation failed (re-login)

---

#### 4. **Race Condition: isLoading vs isVerified**

**Symptom:** Brief flash of loading then redirect

**Cause:** `ProtectedRoute` checks `isVerified` before auth state finishes loading.

**Fix:** Start `isLoading` as `true` in `AuthContext`:

```jsx
const [isLoading, setIsLoading] = useState(true);
```

Set to `false` only after hydration:

```jsx
useEffect(() => {
  // ... load from localStorage
  setIsLoading(false);
}, []);
```

---

### Quick Diagnosis Checklist

1. **Check Network Tab**

   - No requests → Missing route (wildcard redirect)
   - 401 status → Auth/cookie issue
   - 404 status → Backend endpoint missing

2. **Check Console**

   - Errors? → Code issue
   - No errors + redirect → Missing route or auth state

3. **Check Application Tab**

   - Cookies → Should see `accessToken` (or similar)
   - Local Storage → Should see `sv_user` if logged in

4. **Check Routes**
   - Does the path exist in `App.jsx`?
   - Is it wrapped in `<ProtectedRoute>`?

---

### Prevention Tips

- Always add routes BEFORE navigating to them
- Test routes individually before wiring navigation
- Keep DevTools Network tab open during testing
- Check auth state persistence after page refresh

---

**Last Updated:** Phase 3 - Snippet CRUD Implementation
