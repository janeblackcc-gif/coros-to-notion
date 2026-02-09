# Property-Based Testing Properties

This document defines testable invariants and falsification strategies for the COROS migration. These properties ensure correctness through automated property-based testing rather than example-based tests.

## API Authentication Properties

### API-Auth-Idempotent-Token
**INVARIANT:** For valid credentials `c`, `login(c)` called twice in the same unexpired session window yields identical non-empty token: `t1 = t2 != ""`.

**FALSIFICATION:** Generate repeated `login(c)` calls with randomized short delays and concurrent double-invocations; assert token equality.

**BOUNDARY CONDITIONS:** Immediate second call; near-expiry second call; concurrent calls sharing one client instance.

---

### API-Auth-Invalid-Creds-Determinism
**INVARIANT:** For any invalid credential pair `c_bad`, `login(c_bad)` always raises `AuthenticationError` and client auth state remains unauthenticated.

**FALSIFICATION:** Fuzz malformed credentials (empty, whitespace, long, unicode, swapped email/password) and assert exception type + no token persisted.

**BOUNDARY CONDITIONS:** Empty email; empty password; both empty; syntactically valid email with wrong password.

---

### API-Auth-Network-Failure-Classification
**INVARIANT:** If `/login` transport fails, outcome is `ConnectionError` (not `AuthenticationError`), and previous token is unchanged.

**FALSIFICATION:** Inject timeout, DNS failure, connection reset, TLS failure during login; verify error taxonomy and state safety.

**BOUNDARY CONDITIONS:** Failure before request send; failure after headers sent; intermittent one-shot failure.

---

### API-Auth-Token-Lifetime-Bound
**INVARIANT:** If login response contains `expiresAt`, then at return time `now < expiresAt`.

**FALSIFICATION:** Generate responses with stale, equal-now, and future `expiresAt`; assert stale/equal values are rejected.

**BOUNDARY CONDITIONS:** `expiresAt` missing; `expiresAt == now`; `expiresAt` timezone-offset timestamps.

---

## API Request Properties

### API-Request-Auth-Header-Injection
**INVARIANT:** After successful auth with token `t`, every protected `_request(...)` includes auth header carrying exactly `t`.

**FALSIFICATION:** Stub request transport and capture outgoing headers for random endpoints/params; fail if token omitted or mutated.

**BOUNDARY CONDITIONS:** Token contains special chars; token with leading/trailing spaces; large query params.

---

### API-Request-PreAuth-Guard
**INVARIANT:** `_request(...)` before successful `login()` always raises `AuthenticationError`.

**FALSIFICATION:** Random state-machine testing over call sequences (`_request`, `login`, `_request`) and assert precondition enforcement.

**BOUNDARY CONDITIONS:** Token set to empty string; token set to `None`; partially initialized client.

---

## Pagination Properties

### API-Pagination-PageSize-Bounds
**INVARIANT:** For each page fetch, returned count satisfies `0 <= n <= min(size, 200)`.

**FALSIFICATION:** Fuzz `size` and `page` over invalid/edge ranges and use mocked API envelopes that over-return; ensure client clamps or rejects.

**BOUNDARY CONDITIONS:** `size=1`; `size=200`; `size=201`; `size=0`; negative size.

---

### API-Pagination-DateRange-Inclusion
**INVARIANT:** For `get_activities(from_date=a, to_date=b)`, every returned activity time `t` satisfies `a <= t <= b` (inclusive).

**FALSIFICATION:** Generate activities around both boundaries (`a-1s`, `a`, `b`, `b+1s`) and assert only inclusive interval survives.

**BOUNDARY CONDITIONS:** `a == b`; empty range; `a > b` (must fail deterministically).

---

### API-Pagination-Monotonic-Order
**INVARIANT:** With newest-first paging, `max(start_time(page N+1)) <= min(start_time(page N))`.

**FALSIFICATION:** Generate synthetic multi-page datasets with shuffled boundaries; assert monotonic order check catches out-of-order pages.

**BOUNDARY CONDITIONS:** Equal timestamps across page boundary; single-item pages; empty final page.

---

## Retry & Error Handling Properties

### API-401-Reauth-Retry-Bound
**INVARIANT:** On first `401`, client performs at most one re-auth and one replay of original request; total auth-retry count per request `<= 1`.

**FALSIFICATION:** Mock sequence `401 -> 200`, `401 -> 401`, `401 -> login-fail`; assert bounded retries and terminal exception behavior.

**BOUNDARY CONDITIONS:** 401 during pagination mid-run; 401 on first request; 401 after token refresh.

---

### API-429-Backoff-Monotonic-And-Attempt-Bound
**INVARIANT:** For retryable 429 flow with `max_attempts=5`, sleep times are monotone non-decreasing: `sleep(n+1) >= sleep(n)`, and `0 <= attempts <= 5`.

**FALSIFICATION:** Generate 429 streaks with/without `Retry-After`, malformed headers, and random jitter seeds; assert monotonicity + hard cap.

**BOUNDARY CONDITIONS:** `Retry-After=0`; missing `Retry-After`; very large `Retry-After`; success on final attempt.

---

### API-Malformed-Response-FailFast
**INVARIANT:** Non-JSON or schema-incompatible response always raises `APIError`; no partial activity objects are emitted.

**FALSIFICATION:** Fuzz payloads (HTML, truncated JSON, wrong types, missing keys); assert fail-fast and zero leaked records.

**BOUNDARY CONDITIONS:** Valid JSON with wrong shape; empty body; null body.

---

## Synchronization Properties

### SYNC-Retrieval-Limit-Bound
**INVARIANT:** Let `L = env(COROS_ACTIVITIES_FETCH_LIMIT)` else `1000`; fetched activity count `n` satisfies `0 <= n <= L`.

**FALSIFICATION:** Generate source datasets larger than `L`, randomized page sizes, and malformed env values; assert cap behavior.

**BOUNDARY CONDITIONS:** Env missing; `L=500`; `L=1`; `L=0`; non-integer env value.

---

### SYNC-ApiUnavailable-NoSideEffects
**INVARIANT:** If COROS fetch phase is unreachable/fails, Notion state delta is empty: `Δcreate = Δupdate = Δdelete = 0`.

**FALSIFICATION:** Inject failures before first page and mid-fetch; assert no writes were committed.

**BOUNDARY CONDITIONS:** Failure after auth but before first page; failure after some pages retrieved; transient outage recovery.

---

## Activity Type Mapping Properties

### SYNC-TypeMapping-Totality
**INVARIANT:** Mapping obeys required fixed points: `100->Running`, `200->Cycling`, `201->Indoor Cycling`; unknown codes map to `Unknown`.

**FALSIFICATION:** Fuzz `mode_code` over ints/strings/floats/negatives and assert deterministic mapped output set.

**BOUNDARY CONDITIONS:** `mode=None`; `mode=100.0`; very large code; negative code.

---

### SYNC-TypeMapping-RoundTrip
**INVARIANT:** For known codes, `code -> raw_type -> notion_display` preserves canonical class; inverse map returns same equivalence class.

**FALSIFICATION:** Generate known mapping table and random aliases; detect non-bijective collisions that break reversibility.

**BOUNDARY CONDITIONS:** Many-to-one alias (`Road Bike` and `Cycling`); unknown code round-trip.

---

## Field Mapping Properties

### SYNC-FieldRoundTrip-CorePreservation
**INVARIANT:** For core fields `{date, distance, duration, calories, avgPower, maxPower}`, `readback(map(activity)) == normalize(activity)` within numeric precision rules.

**FALSIFICATION:** Property-generate activities with random numeric scales and optional missing fields; compare mapped/read-back normalization.

**BOUNDARY CONDITIONS:** Zero distance; zero duration; high-precision floats; max numeric ranges.

---

### SYNC-DefaultValues-MissingFields
**INVARIANT:** If training-effect fields absent: `Training Effect="Unknown" ∧ Aerobic=0 ∧ Anaerobic=0`; if PR/Fav absent: `PR=False ∧ Fav=False`.

**FALSIFICATION:** Generate payloads with omitted keys, explicit nulls, wrong types (`"NaN"`, `{}`), and assert canonical defaults.

**BOUNDARY CONDITIONS:** Field absent vs null vs empty string; partially present TE fields.

---

### SYNC-Schema-Preservation-19Fields
**INVARIANT:** Output Notion payload preserves required schema cardinality/types (existing 19-field contract) for every synced activity.

**FALSIFICATION:** Generate varied activity shapes and validate produced Notion property keys/types against schema oracle.

**BOUNDARY CONDITIONS:** Optional icon missing; unknown activity type; sparse activity payload.

---

## Duplicate Detection Properties

### SYNC-Dedup-Idempotency
**INVARIANT:** Running sync twice on same input set does not increase Notion cardinality under composite key `(date±5m, type, name)`.

**FALSIFICATION:** Re-run identical generated batches; assert `count_after_second_run == count_after_first_run`.

**BOUNDARY CONDITIONS:** Timestamp exactly `±5m`; case-different names; timezone-normalized timestamps.

---

### SYNC-UpdateVsCreate-Exclusivity
**INVARIANT:** If key matches existing row and any mapped metric differs, exactly one update occurs and zero creates for that key.

**FALSIFICATION:** Generate one-field-at-a-time mutations and assert operation counts (`updates=1`, `creates=0`).

**BOUNDARY CONDITIONS:** Changes below rounding threshold; only icon change; only PR/Fav change.

---

### SYNC-Count-Preservation-InDateRange
**INVARIANT:** After successful sync in range `R`, unique Notion activity count equals unique COROS activity count in `R`.

**FALSIFICATION:** Generate datasets with duplicates, unknown modes, and missing optional fields; assert count equality over normalized keys.

**BOUNDARY CONDITIONS:** Empty range; all duplicates already exist; all-new dataset.

---

## Commutativity Properties

### SYNC-Commutativity-BatchOrder
**INVARIANT:** For activity multiset `A`, final Notion state is permutation-invariant: `sync(A) == sync(permutation(A))`.

**FALSIFICATION:** Run sync over multiple random permutations of same batch and compare canonicalized Notion state snapshots.

**BOUNDARY CONDITIONS:** Duplicate keys with conflicting metrics; equal timestamps; mixed known/unknown activity types.

---

## Icon & Formatting Properties

### SYNC-Icon-Assignment-Determinism
**INVARIANT:** If mapped activity type is in `ACTIVITY_ICONS`, icon URL equals mapped URL; otherwise icon is absent.

**FALSIFICATION:** Fuzz activity types across known/unknown sets and assert exact icon behavior.

**BOUNDARY CONDITIONS:** Empty icon map; malformed URL in map; unknown type fallback.

---

### SYNC-Pace-Formula-Correctness
**INVARIANT:** If speed `v>0`, pace satisfies `pace_sec_per_km = 1000 / v_mps`; formatted as `MM:SS /km`; if `v<=0` or missing then pace is empty/`N/A`.

**FALSIFICATION:** Generate speeds in both `m/s` and `km/h` with unit tags and verify computed pace against oracle conversion.

**BOUNDARY CONDITIONS:** `v=0`; very small positive `v`; very high `v`; missing unit metadata.

---

## Unit Conversion Properties

### SYNC-UnitConversion-Bounds
**INVARIANT:** `distance_km = round(distance_m / 1000, 2)`, `duration_min = round(duration_s / 60, 2)`, and both are non-negative finite numbers.

**FALSIFICATION:** Fuzz meters/seconds across ints/floats/extremes/NaN/negative values and assert finite bounded outputs or deterministic rejection.

**BOUNDARY CONDITIONS:** `0`, `1`, `59`, `60`, `61`, `999`, `1000`, `1001`; negative and NaN inputs.

---

### SYNC-Entertainment-Formatting-Idempotency
**INVARIANT:** Formatter is deterministic and idempotent: `f("ENTERTAINMENT")="Netflix"` and `f(f(name))=f(name)` for all names.

**FALSIFICATION:** Generate random names including exact token, substrings, case variants, unicode; assert deterministic/idempotent output.

**BOUNDARY CONDITIONS:** `"ENTERTAINMENT"` exact; `" entertainment "`; `"ENTERTAINMENT_SHOW"`; lowercase `"entertainment"`.

---

## Implementation Notes

These properties should be implemented using a property-based testing framework such as:
- **Python**: `hypothesis` library
- **Test Organization**: Group by module (API client tests, sync logic tests, mapping tests)
- **Coverage Target**: All 26 properties must pass with 1000+ generated test cases each
- **CI Integration**: Run PBT suite on every commit; fail build on property violations
