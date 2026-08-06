# Paging Process — Sequence Diagram

```mermaid
sequenceDiagram
    autonumber
    actor Frontend as Frontend
    participant Backend as Backend
    participant PG as PostgreSQL

    Note over Frontend, PG: Initial request (first load)
    Frontend->>Backend: 1. Paging request: params(1) { page, size, ...filters }
    Backend->>PG: 2a. get ids -> size × 10 rows (id batch / window) (2)
    PG-->>Backend: ids (2) [size × 10]
    Backend->>PG: 2b. get data of ids -> first `size` rows
    PG-->>Backend: data [size]
    Backend-->>Frontend: 3. response { data [size], ids (2) [size × 10], total items }

    Note over Frontend, Backend: Next page with the SAME params(1)
    alt params(1) unchanged
        Frontend->>Backend: 4. Paging request: params(1) + ids (2) + total items
        alt ids from frontend in range (requested page within cached window)
            Backend->>PG: 5a. get data of ids [size]  (no `get ids` call)
            PG-->>Backend: data [size]
            Backend-->>Frontend: response { data [size], ids (2), total items }
        else ids out of range -> recall to get next ids
            Backend->>PG: 5b. get new ids [size × 10]
            PG-->>Backend: new ids (2)
            Backend->>PG: get data of ids [size]
            PG-->>Backend: data [size]
            Backend-->>Frontend: response { data [size], new ids (2), total items }
        end
    end
```

## Flow Summary

1. **Frontend → Backend**: sends paging params `(1)` — `page`, `size`, plus any filters.
2. **Backend processing**:
   - `get ids` returns a batch of `size × 10` IDs (the reusable paging window) `(2)`.
   - `get data of ids` returns data for only `size` IDs (the current page).
3. **Backend → Frontend**: returns the page `data`, the full `ids (2)` batch, and `total items`.
4. **Same params** → Frontend sends back `params (1)` together with the cached `ids (2)` and `total items`.
5. **Backend decides**:
   - If the requested IDs are **in range** of the cached window → fetch `get data of ids (size)` straight away, skipping a redundant ID query.
   - If the IDs are **out of range** → re-fetch the next ID batch (`recall get ids`) before loading the page data.
