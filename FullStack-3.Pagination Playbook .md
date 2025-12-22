# Pagination Playbook (Full Stack)

Use these steps to add or debug pagination in any project. Swap in your own route names and fields; the flow stays the same.

## Backend Essentials

1. **Accept paging params**: Parse `page` (default 1) and `limit` (default 10, clamp to a sane max). Coerce to integers and guard against negatives.
2. **Build the query**: Add filters (search, tags, owner). For search, prefer a safe regex or text index. Example shape:
   - `query = { owner: userId, ...filters }`
   - `sort = { createdAt: -1 }`
3. **Apply skip/limit**: `skip = (page - 1) * limit`, then `find(query).sort(sort).skip(skip).limit(limit)`.
4. **Count total**: `total = countDocuments(query)`. Compute `totalPages = Math.max(1, Math.ceil(total / limit))`.
5. **Respond with both data and meta**:
   ```json
   {
     "items": [...],
     "pagination": { "page": 1, "limit": 10, "total": 42, "totalPages": 5 }
   }
   ```
6. **Validate inputs**: Reject non-numeric `page/limit`; cap `limit` (e.g., 100) to avoid heavy queries.

## Frontend Essentials

1. **State**: Keep `page`, `limit`, `search/filter`, `items`, and `pagination` metadata.
2. **Fetch**: Call the list endpoint with `?page=&limit=&search=...` whenever `page` or filters change. Debounce search inputs.
3. **Render list**: Show items for the current page; handle loading/error/empty states.
4. **Controls**: Prev/Next buttons disabled at edges; show "Page X of Y". Optionally allow page size or direct page jump.
5. **Reset on filter**: When search/filter changes, reset `page` to 1 before fetching.

## Quick Wiring Checklist

- Backend route accepts `page` and `limit` and returns `{ items, pagination }`.
- `skip` and `limit` use parsed integers; no string math.
- `totalPages` computed from the same `query` used for data.
- Frontend passes `page/limit/search` as query params.
- Buttons disable when `page <= 1` or `page >= totalPages`.
- Search/filter change resets `page` to 1.

## Common Pitfalls

- **String math**: Forgetting `parseInt` causes skip/limit to be wrong.
- **Unbounded limit**: Large `limit` can slow the DB; always cap.
- **Mismatched counts**: Counting with a different query than the fetch makes `totalPages` wrong.
- **Off-by-one**: Allowing page 0 or negative numbers breaks skip math.
- **Missing meta**: Frontend cannot render controls if backend omits `total/totalPages`.

## Minimal Example (Pseudo)

- **Backend**: `GET /items?page=2&limit=10&search=foo`
  - Parse ints, build `query`, run `find().skip().limit()`, `countDocuments(query)`, return `{ items, pagination }`.
- **Frontend**: state `{ page, limit, search, items, pagination }`; `useEffect` fetches on change; controls call `setPage(page +/- 1)`; reset page to 1 on search.

## Copy/Paste Templates

### Backend (Express + Mongo)

```js
// GET /items?page=1&limit=10&search=foo
app.get("/items", async (req, res) => {
  const { page = 1, limit = 10, search = "" } = req.query;
  const pageNumber = Math.max(1, parseInt(page, 10) || 1);
  const limitNumber = Math.max(1, Math.min(100, parseInt(limit, 10) || 10));

  const query = {};
  const trimmed = search.trim();
  if (trimmed) {
    const escaped = trimmed.replace(/[.*+?^${}()|[\]\\]/g, "\\$&");
    const regex = new RegExp(escaped, "i");
    query.$or = [
      { title: regex },
      { description: regex },
      { tags: { $elemMatch: { $regex: regex } } },
    ];
  }

  const items = await Item.find(query)
    .sort({ createdAt: -1 })
    .skip((pageNumber - 1) * limitNumber)
    .limit(limitNumber);

  const total = await Item.countDocuments(query);
  const totalPages = Math.max(1, Math.ceil(total / limitNumber));

  res.json({
    items,
    pagination: { page: pageNumber, limit: limitNumber, total, totalPages },
  });
});
```

### Frontend (React)

```jsx
function ItemsList() {
  const [items, setItems] = useState([]);
  const [page, setPage] = useState(1);
  const [limit, setLimit] = useState(10);
  const [search, setSearch] = useState("");
  const [pagination, setPagination] = useState({ page: 1, totalPages: 1 });
  const [isLoading, setIsLoading] = useState(false);

  useEffect(() => {
    const timer = setTimeout(() => {
      setIsLoading(true);
      fetch(
        `/items?page=${page}&limit=${limit}&search=${encodeURIComponent(
          search
        )}`
      )
        .then((r) => r.json())
        .then((data) => {
          setItems(data.items);
          setPagination(data.pagination);
        })
        .finally(() => setIsLoading(false));
    }, 300);
    return () => clearTimeout(timer);
  }, [page, limit, search]);

  const disablePrev = page <= 1 || isLoading;
  const disableNext =
    isLoading || (pagination.totalPages && page >= pagination.totalPages);

  return (
    <div>
      <input
        value={search}
        onChange={(e) => {
          setSearch(e.target.value);
          setPage(1);
        }}
      />
      {isLoading ? (
        <p>Loading...</p>
      ) : (
        items.map((item) => <div key={item._id}>{item.title}</div>)
      )}
      <button
        onClick={() => setPage((p) => Math.max(1, p - 1))}
        disabled={disablePrev}
      >
        Prev
      </button>
      <span>
        Page {page} of {pagination.totalPages || 1}
      </span>
      <button
        onClick={() =>
          setPage((p) =>
            pagination.totalPages
              ? Math.min(pagination.totalPages, p + 1)
              : p + 1
          )
        }
        disabled={disableNext}
      >
        Next
      </button>
      <select
        value={limit}
        onChange={(e) => {
          setLimit(Number(e.target.value));
          setPage(1);
        }}
      >
        {[5, 10, 20].map((n) => (
          <option key={n} value={n}>
            {n} / page
          </option>
        ))}
      </select>
    </div>
  );
}
```

## Testing Steps

1. Seed 15–20 records.
2. Load page 1 (limit 5) → should show 5 items, `totalPages = 4`.
3. Click Next until the end; verify last page has the remainder and Next is disabled.
4. Change limit (e.g., 5 → 10); `totalPages` should adjust.
5. Add search/filter; results should reset to page 1 and counts reflect filtered results.
6. Hard refresh; pagination state should refetch correctly.

**Keep this handy**: the shapes change per project, but the pattern stays the same—parse params, query with skip/limit, count, return meta, and wire the UI to those numbers.
