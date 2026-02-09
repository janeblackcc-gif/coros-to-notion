## Context

The current project uses the `garminconnect` Python library to fetch activity and personal records data from Garmin Connect (China region) and sync it to Notion databases via the `notion-client` library. The system runs as a scheduled GitHub Actions workflow that executes four independent Python scripts. This design document addresses migrating the data source from Garmin to COROS while maintaining the existing Notion database schema and workflow architecture.

**Current Architecture:**
```
Garmin Connect API → garminconnect library → Python Scripts → notion-client → Notion Databases
                                                ↑
                                        GitHub Actions (daily cron)
```

**Target Architecture:**
```
COROS Training Hub API (teamcnapi.coros.com) → CorosClient (custom) → Python Scripts → notion-client → Notion Databases
                                                      ↑
                                              GitHub Actions (daily cron)
```

**Key Constraints:**
- COROS API is unofficial and undocumented (reverse-engineered from xballoy/coros-api TypeScript implementation)
- Must preserve existing Notion database structure (19 fields including Training Effect metrics)
- User has COROS Training Hub account and can test immediately
- Windows environment with Git Bash shell
- Python 3.11+

## Goals / Non-Goals

**Goals:**
- Create a stable CorosClient that handles authentication and activity retrieval
- Migrate activities and personal records synchronization to COROS data source
- Maintain backward compatibility with existing Notion database schema
- Preserve GitHub Actions automation with minimal configuration changes
- Handle missing COROS fields gracefully (default values)

**Non-Goals:**
- Migrating daily steps synchronization (excluded per user request)
- Migrating sleep data synchronization (excluded per user request)
- Supporting dual Garmin/COROS data sources simultaneously
- Applying for official COROS API access (using unofficial API)
- Preserving existing Garmin historical data (user will clear databases)

## Decisions

### Decision 1: Custom HTTP Client vs. Third-Party Library
**Choice:** Implement custom `CorosClient` using `requests` library
**Rationale:**
- No official Python SDK exists for COROS API
- Existing TypeScript implementation (xballoy/coros-api) is small enough to port
- Direct control over authentication flow and error handling
- Minimal dependencies (only `requests`, which is ubiquitous)

**Alternatives Considered:**
- Port xballoy/coros-api to Python → Rejected: TypeScript-specific tooling overhead
- Use generic HTTP library wrapper → Rejected: No value over direct `requests` usage

### Decision 2: API Exploration Phase First
**Choice:** Implement exploration script before full client
**Rationale:**
- COROS API response structure is undocumented
- Field names, units, and data types are unknown
- Must verify API behavior before committing to field mappings
- Reduces rework if API differs from TypeScript implementation

**Implementation:**
- Phase 1: Create minimal client with login + single activity query
- Document actual JSON response
- Map fields to Garmin equivalents
- Phase 2: Build full CorosClient based on findings

### Decision 3: Activity Type Mapping Strategy
**Choice:** Maintain separate mapping dictionaries (COROS codes → display names → icons)
**Rationale:**
- COROS uses numeric codes (100, 200, 300...) vs. Garmin string keys
- Existing `ACTIVITY_ICONS` dict maps display names to icon URLs
- Two-step translation: numeric code → display name → icon URL

**Implementation:**
```python
COROS_ACTIVITY_TYPES = {
    100: "Running",
    200: "Road Bike",
    300: "Pool Swim",
    # ... etc
}
activity_type = COROS_ACTIVITY_TYPES.get(mode, "Unknown")
icon_url = ACTIVITY_ICONS.get(activity_type)
```

### Decision 4: Handling Missing COROS Fields
**Choice:** Use default values, not null/None
**Rationale:**
- Notion requires valid values for typed fields (number, select, checkbox)
- Consistent with existing code patterns
- Enables queries/filters in Notion without null checks

**Default Values:**
- Training Effect → "Unknown" (select field)
- Aerobic/Anaerobic numbers → 0
- PR/Favorite checkboxes → False

### Decision 5: Script File Replacement vs. In-Place Modification
**Choice:** Create new files (`coros-activities.py`), keep originals temporarily
**Rationale:**
- Allows side-by-side comparison during testing
- Easy rollback if issues arise
- Clear git history (add new files, delete old files in separate commits)

**Migration Path:**
1. Create new `coros-*.py` scripts
2. Test thoroughly
3. Update GitHub Actions to use new scripts
4. Delete old `garmin-*.py` files after validation

### Decision 6: Personal Records Strategy
**Choice:** Wait for API exploration; prepare fallback calculation method
**Rationale:**
- Unknown if COROS API provides personal records endpoint
- Fallback: calculate records from activity list (max distance, fastest pace, etc.)
- Defer decision until Phase 1 exploration completes

## Risks / Trade-offs

### Risk 1: COROS API Structure Differs from TypeScript Implementation
**Mitigation:**
- Phase 1 exploration before full implementation
- Document actual responses with JSON examples
- Maintain detailed error logging

### Risk 2: Unofficial API May Break Without Notice
**Mitigation:**
- Add comprehensive error handling with clear messages
- Log full request/response details for debugging
- Consider adding API health check before each sync run

**Trade-off:** Long-term maintenance burden vs. applying for official API access (requires approval)

### Risk 3: Personal Records Endpoint May Not Exist
**Mitigation:**
- Implement fallback: calculate records from activity list
- Compare against xballoy/coros-api source to find alternate endpoints

**Trade-off:** Calculation-based records may miss server-side calculations

### Risk 4: Token Expiration Handling
**Mitigation:**
- Implement token refresh logic if API provides refresh mechanism
- Re-authenticate automatically on 401 responses
- Cache token with expiry timestamp

**Trade-off:** Extra API calls vs. risk of mid-sync authentication failures

### Risk 5: Data Migration Timing
**Impact:** User must manually clear Notion databases before first sync
**Mitigation:**
- Provide clear backup instructions in documentation
- Suggest exporting to CSV before migration
- Note: No automated data migration tool provided

**Trade-off:** Manual step required vs. complexity of automated migration script

## Migration Plan

### Phase 1: API Exploration (Must Complete First)
1. Create `explore_coros_api.py` script
2. Implement basic authentication
3. Query 1-2 activities
4. Document JSON response structure
5. Create field mapping document
6. Verify activity type codes

**Validation:** Successfully authenticate and retrieve activity JSON

### Phase 2: CorosClient Implementation
1. Create `coros_client.py` with `CorosClient` class
2. Implement methods: `login()`, `get_activities()`, `_request()`
3. Add error handling (token expiry, network errors, rate limiting)
4. Add logging (request/response details)
5. Unit test with real API calls

**Validation:** Client can authenticate and fetch activities consistently

### Phase 3: Script Adaptation
1. Create `coros-activities.py` based on `garmin-activities.py`
2. Replace GarminClient with CorosClient
3. Update activity type mapping
4. Handle missing fields with defaults
5. Test locally with Notion sandbox database

**Validation:** Activities sync to Notion with correct field mappings

### Phase 4: Personal Records Migration
1. Explore COROS records API (if exists)
2. Create `coros-personal-records.py`
3. Implement fallback calculation if needed
4. Test with Notion PR database

**Validation:** Records update correctly in Notion

### Phase 5: Configuration & Deployment
1. Update `requirements.txt`
2. Update GitHub Actions workflow
3. Update environment variables (GitHub Secrets)
4. Update documentation
5. Test workflow in GitHub Actions

**Validation:** Automated sync runs successfully

### Rollback Strategy
If issues arise post-deployment:
1. Revert GitHub Actions workflow to use `garmin-*.py` scripts
2. Restore Garmin environment variables
3. Re-add `garminconnect` to requirements.txt
4. User can re-import backup data to Notion

## Explicit Constraints (Multi-Model Analysis Results)

### Authentication & Token Management
**Constraint:** Token lifecycle and retry behavior MUST be explicitly defined to prevent infinite loops and thundering herd issues.

**Parameters:**
- `max_auth_retries_per_request = 1` (single retry on 401, then hard fail)
- Token preemptive refresh: `if expiresAt exists and now >= expiresAt - 60s`
- Reactive refresh on 401: retry once, then raise `AuthenticationError`
- Never retry on 401 from `login()` itself (fail fast)

**Authentication Contract (Verified via API Research):**
- Base URLs: `https://teamcnapi.coros.com` (China), `https://teamapi.coros.com` (America), `https://teameuapi.coros.com` (Europe)
- Endpoint: `POST {base_url}/account/login`
- Request body: `{"account": "<email>", "accountType": 2, "pwd": "<MD5_HASHED_PASSWORD>"}`
- Password MUST be MD5 hashed before sending (`hashlib.md5(password.encode()).hexdigest()`)
- Required headers: `Content-Type: application/json`
- Response structure: `{"result": "0000", "data": {"accessToken": "..."}}`
- Auth header for subsequent requests: `accesstoken: <token>` (lowercase key, NOT `Authorization: Bearer`)
- Fallback: if `result != "0000"` or `accessToken` not present, raise `AuthenticationError` immediately

### Retry & Backoff Strategy
**Constraint:** All network operations MUST use exponential backoff with jitter to handle transient failures and rate limiting.

**Parameters:**
- Timeout: `connect=5s, read=20s`
- Retriable failures: network timeouts, connection reset, HTTP 429, HTTP 5xx
- Non-retriable: HTTP 400/401/403/404 (client errors)
- Backoff: `base=0.5s, factor=2, max_sleep=30s, max_attempts=5`
- For HTTP 429: use `Retry-After` header if present, else follow backoff schedule

### Pagination Strategy
**Constraint:** Pagination MUST have explicit termination conditions to prevent infinite loops.

**Parameters (Verified):**
- Fixed page size: `size = 200` (per xballoy/coros-api)
- Pagination params: `pageNumber` (1-based), `size`, `startDay` (YYYYMMDD), `endDay` (YYYYMMDD)
- Endpoint: `GET {base_url}/activity/query`
- Response: `data.dataList` (array of activity summary objects)
- Termination conditions (any of):
  - Empty page returned
  - Page length < size
  - Collected count >= limit
  - Repeated page signature detected (deduplicate by activity ID)
- Safety guard: `max_pages = ceil(limit/size) + 2`

### Activity Type Mapping
**Constraint:** Two-level mapping to preserve COROS raw type names for debugging while maintaining Notion display consistency.

**Strategy (Verified sport type codes):**
```python
# Level 1: COROS sportType code -> raw type name (verified + dynamic)
# Verified codes: 100=Running, 101=Treadmill, 103=Road Run, 200=Bike
# Full list available via GET /activity/fit/getImportSportList
COROS_MODE_TO_RAW_TYPE = {
    100: "Running",
    101: "Treadmill Running",
    103: "Running",
    200: "Cycling",
    # Dynamic: fetch remaining from /activity/fit/getImportSportList at runtime
}

# Level 2: Raw type -> Notion display type
RAW_TYPE_TO_NOTION_DISPLAY = {
    "Road Bike": "Cycling",
    "Mountain Bike": "Cycling",
    "Running": "Running",
    # ... etc
}

# Unknown mode handling
if mode not in COROS_MODE_TO_RAW_TYPE:
    raw_type = f"Unknown ({mode})"  # Log for user reporting
    notion_display = "Unknown"
```

### Default Values for Missing Fields
**Constraint:** Default values MUST distinguish between "no data" and "zero value" to enable accurate Notion filtering.

**Parameters:**
- Training Effect (select): `"Unknown"`
- Aerobic/Anaerobic (number): `0.0` (represents "no data")
- PR/Favorite (checkbox): `False`
- Power metrics (number): `null` (empty in Notion) - distinguishes "no power meter" from "power = 0W"
- Distance/Duration: required fields, no defaults (fail if missing)

### Duplicate Detection
**Constraint:** Primary key MUST be stable activity ID to prevent false duplicates from fuzzy matching.

**Strategy:**
- Primary key: `coros_activity_id` stored in Notion custom field
- Fallback key (only if ID absent): `(start_time ±2min, activity_type, normalized_name, distance ±0.02km)`
- Timezone normalization: store UTC timestamps for matching
- Comparison: use timezone-aware datetime objects

### Unit Conversion Validation
**Constraint:** Unit conversions MUST be validated during exploration phase before implementation.

**Validation Checklist:**
- Distance: confirm unit is meters before `/1000` conversion
- Duration: confirm unit is seconds before `/60` conversion
- Speed: confirm unit (m/s vs km/h) before pace calculation
- Power: confirm unit is watts
- Store unit metadata in `mapping_schema.json` artifact from exploration phase

### Dependency Management
**Decision:** Remove `garminconnect` dependency and delete `daily-steps.py` and `sleep-data.py` scripts.

**Rationale:**
- User confirmed: "移除步数和睡眠同步"
- Simplifies migration scope
- Eliminates dual-dependency maintenance burden

**Impact:**
- Remove `garminconnect` from `requirements.txt`
- Delete `daily-steps.py` and `sleep-data.py` files
- Update GitHub Actions workflow to remove steps/sleep sync jobs
- Remove `GARMIN_EMAIL`, `GARMIN_PASSWORD`, `NOTION_STEPS_DB_ID`, `NOTION_SLEEP_DB_ID` secrets

## Open Questions (Resolved via Multi-Model Analysis + API Research)

1. **COROS Activity Response Format**
   - What are the exact field names in the activity JSON?
   - What units are used for distance/duration (meters/km, seconds/minutes)?
   - Does the API include Training Effect equivalents?

   **Resolution (Verified):** Activity list (`/activity/query`) returns `data.dataList` with fields: `date` (YYYYMMDD string), `name`, `sportType` (int), `totalTime` (seconds), `distance` (meters), `calorie` (int), `trainingLoad` (int), `labelId` (activity ID). Detailed metrics (avgHeartRate, avgPower, maxPower) require separate detail endpoint call per activity. COROS does NOT provide Training Effect equivalents — use default "Unknown".

2. **Personal Records Endpoint**
   - Does `/personal-records` or similar endpoint exist?
   - If not, what data is needed to calculate records from activities?

   **Resolution (Verified):** No personal records endpoint exists in the COROS API. Must implement fallback: calculate records from activity list using rule-based reducer (fastest time per distance bucket, longest distance, max power). Distance buckets with tolerance (e.g., 4.9-5.2km → 5K candidate).

3. **Token Lifetime**
   - How long does accessToken remain valid?
   - Is there a refresh token mechanism?

   **Resolution:** No `expiresAt` or `refreshToken` confirmed in login response. Implement reactive 401 re-authentication (single retry). Monitor during API exploration phase.

4. **Rate Limiting**
   - Are there API rate limits?
   - What headers indicate limit status?

   **Resolution:** No documented rate limits. Implement defensive retry/backoff strategy as defined above.

5. **Activity Pagination**
   - Maximum page size (200 per xballoy/coros-api)?
   - How to handle users with >200 activities?

   **Resolution:** Pagination uses `pageNumber` (1-based) + `size` + `startDay`/`endDay` params. Loop until empty `dataList` returned.

### Activity List vs Detail Endpoint (New Finding)

**Critical Architecture Decision:** The activity list endpoint only returns summary data. Detailed metrics require per-activity API calls.

**List endpoint fields** (`GET /activity/query`):
- `date`, `name`, `sportType`, `totalTime`, `distance`, `calorie`, `trainingLoad`, `labelId`

**Detail endpoint fields** (path TBD, requires exploration):
- `avgHeartRate`, `maxHeartRate`, `avgPower`, `maxPower`, lap data, HR/power streams

**Strategy (Fetch-Filter-Detail):**
- Fetch activity list first
- Filter against Notion to identify new/changed activities only
- Call detail endpoint ONLY for those activities (avoids N+1 for unchanged data)
- Daily sync typically processes 0-1 new activities → minimal detail calls

### Personal Records Calculation Strategy (New)

**Approach:** Rule-based reducer over all historical activities.

**Rules:**
- 1K: Running activities, min(totalTime) where distance ≈ 1000m
- 5K: Running activities, min(totalTime) where 4900m ≤ distance ≤ 5200m
- 10K: Running activities, min(totalTime) where 9800m ≤ distance ≤ 10400m
- Longest Run: Running activities, max(distance)
- Longest Ride: Cycling activities, max(distance)
- Max Avg Power (20 min): Cycling/Running, max(avgPower) — requires detail endpoint

**Limitations:**
- Cannot calculate "best split" PRs (e.g., fastest 5K within a 10K) without raw track data
- Step-based records (day/week/month) not available from COROS activity data
