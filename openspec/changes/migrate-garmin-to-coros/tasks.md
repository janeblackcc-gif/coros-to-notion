## 1. API Exploration (Phase 1 - Must Complete First)

- [ ] 1.1 Create `explore_coros_api.py` script with basic HTTP request setup (base URL: `https://teamcnapi.coros.com`)
- [ ] 1.2 Implement COROS login authentication (POST `/account/login` with MD5-hashed password, accountType=2)
- [ ] 1.3 Extract and store accessToken from `response['data']['accessToken']`, use header key `accesstoken` (lowercase)
- [ ] 1.4 Query `/activity/query` endpoint with params (startDay, endDay, size, pageNumber) to fetch 1-2 sample activities
- [ ] 1.5 Document complete JSON response structure of `data.dataList` items with all fields
- [ ] 1.6 Create field mapping document comparing COROS fields (date, name, sportType, totalTime, distance, calorie, trainingLoad, labelId) vs Garmin fields
- [ ] 1.7 Call `/activity/fit/getImportSportList` to get complete sportType numeric codes mapping
- [ ] 1.8 Test token expiration behavior (send request after delay) and document refresh mechanism
- [ ] 1.9 Explore activity detail endpoint (path TBD: try `/activity/detail` with labelId param) for avgHeartRate, avgPower, maxPower
- [ ] 1.10 Confirm personal records endpoint does NOT exist (verified via research, confirm via API)

## 2. CorosClient Implementation

- [ ] 2.1 Create `coros_client.py` with CorosClient class structure
- [ ] 2.2 Implement `__init__(account, password, base_url="https://teamcnapi.coros.com")` constructor with `requests.Session`
- [ ] 2.3 Implement `login()` method: MD5 hash password, POST `/account/login`, store accessToken
- [ ] 2.4 Implement `_request(method, path, params, json, allow_reauth=True)` helper with auth header injection (`accesstoken` key)
- [ ] 2.5 Add 401 reactive re-authentication (single retry) in `_request()`
- [ ] 2.6 Implement `get_activities(start_day, end_day, size=200, page_number=1)` via GET `/activity/query`
- [ ] 2.7 Implement `get_all_activities(limit=1000)` with pagination loop (pageNumber increment, empty dataList termination)
- [ ] 2.8 Implement error handling for network failures (ConnectionError, Timeout)
- [ ] 2.9 Implement error handling for API errors (non-"0000" result code, malformed response)
- [ ] 2.10 Add retry with exponential backoff for 429/5xx responses
- [ ] 2.11 Implement `get_activity_detail(label_id)` for detailed metrics (path from exploration phase)
- [ ] 2.12 Implement `get_sport_types()` via GET `/activity/fit/getImportSportList`
- [ ] 2.13 Test CorosClient with real API calls and verify all methods work

## 3. Activity Type Mapping

- [ ] 3.1 Create COROS_ACTIVITY_TYPES dictionary with verified codes (100=Running, 101=Treadmill, 103=Road Run, 200=Bike)
- [ ] 3.2 Add runtime fetch from `/activity/fit/getImportSportList` to populate remaining codes
- [ ] 3.3 Create two-level mapping: sportType code → raw name → Notion display name (align with existing ACTIVITY_ICONS keys)
- [ ] 3.4 Add helper function `map_activity_type(sport_code, runtime_map)` returning (activity_type, activity_subtype)
- [ ] 3.5 Add fallback to `"Unknown ({code})"` for unmapped activity types with warning log

## 4. Activity Sync Script (coros-activities.py)

- [ ] 4.1 Create `coros-activities.py` based on `garmin-activities.py` structure
- [ ] 4.2 Replace `from garminconnect import Garmin` with `from coros_client import CorosClient`
- [ ] 4.3 Update environment variable names (GARMIN_EMAIL→COROS_ACCOUNT, GARMIN_PASSWORD→COROS_PASSWORD, GARMIN_ACTIVITIES_FETCH_LIMIT→COROS_ACTIVITIES_FETCH_LIMIT)
- [ ] 4.4 Replace GarminClient initialization with CorosClient (account, password, base_url from env)
- [ ] 4.5 Update `get_all_activities()` to use CorosClient.get_all_activities() with pagination
- [ ] 4.6 Update `format_activity_type()` to use sportType numeric code → two-level mapping
- [ ] 4.7 Implement Fetch-Filter-Detail strategy: list first, filter against Notion, detail only for new/changed
- [ ] 4.8 Update `create_activity()` field mappings: date from `date` field, distance from `distance`(meters), duration from `totalTime`(seconds), calories from `calorie`
- [ ] 4.9 Add default values for missing fields: Training Effect="Unknown", Aerobic=0, Anaerobic=0, PR=False, Fav=False
- [ ] 4.10 Map detail fields when available: avgHeartRate, avgPower, maxPower (null/0 if no detail)
- [ ] 4.11 Update `activity_needs_update()` to compare COROS field structure
- [ ] 4.12 Update `update_activity()` field mappings
- [ ] 4.13 Update pace calculation: compute from distance/totalTime (avgSpeed = distance_m / totalTime_s)
- [ ] 4.14 Preserve ±5min dedup logic and ACTIVITY_ICONS mapping
- [ ] 4.15 Test script locally with Notion database

## 5. Personal Records Script (coros-personal-records.py)

- [ ] 5.1 Create `coros-personal-records.py` based on `personal-records.py` structure
- [ ] 5.2 Replace GarminClient with CorosClient
- [ ] 5.3 Update environment variable names (GARMIN_*→COROS_*)
- [ ] 5.4 Implement rule-based PR calculation from activity list (no API endpoint exists)
- [ ] 5.5 Define distance bucket rules: 1K(±100m), 1mi(±100m), 5K(±200m), 10K(±400m)
- [ ] 5.6 Implement Longest Run (max distance, running), Longest Ride (max distance, cycling)
- [ ] 5.7 Implement Max Avg Power (20 min) from detail endpoint (lazy fetch for cycling/running candidates)
- [ ] 5.8 Rename `format_garmin_value()` to `format_coros_value()` with same formatting logic
- [ ] 5.9 Preserve icon/cover mapping, update/create/archive Notion logic
- [ ] 5.10 Remove step-based records (Most Steps Day/Week/Month) — not available from COROS activities
- [ ] 5.11 Test script locally with Notion PR database

## 6. Dependency Management

- [ ] 6.1 Update `requirements.txt`: remove `garminconnect`, add `requests`
- [ ] 6.2 Verify `requests` is in requirements.txt
- [ ] 6.3 Verify all other dependencies remain (notion-client, pytz, python-dotenv)
- [ ] 6.4 Remove unused dependencies: `withings-sync`, `lxml`, `datetime` (evaluate necessity)
- [ ] 6.5 Test local install with `pip install -r requirements.txt`

## 7. Configuration Updates

- [ ] 7.1 Rename workflow: "Sync Garmin to Notion" → "Sync COROS to Notion"
- [ ] 7.2 Update environment variable references: GARMIN_EMAIL→COROS_ACCOUNT, GARMIN_PASSWORD→COROS_PASSWORD
- [ ] 7.3 Update script execution: `python coros-activities.py`
- [ ] 7.4 Update script execution: `python coros-personal-records.py`
- [ ] 7.5 Remove `python daily-steps.py` and `python sleep-data.py` from workflow
- [ ] 7.6 Remove NOTION_STEPS_DB_ID and NOTION_SLEEP_DB_ID from env section
- [ ] 7.7 Add COROS_ACTIVITIES_FETCH_LIMIT variable reference

## 8. Data Migration Preparation

- [ ] 8.1 Document backup instructions for users in README
- [ ] 8.2 Test Notion database export to CSV (manual step for user)
- [ ] 8.3 Verify Notion database structure has all required fields
- [ ] 8.4 User action: Clear Activities database in Notion
- [ ] 8.5 User action: Clear Personal Records database in Notion

## 9. Testing & Validation

- [ ] 9.1 Run coros-activities.py locally and verify activities appear in Notion
- [ ] 9.2 Verify activity field values are correct (dates, distances, durations)
- [ ] 9.3 Verify activity type icons are displayed correctly
- [ ] 9.4 Verify pace calculations are accurate
- [ ] 9.5 Run coros-personal-records.py locally and verify records appear
- [ ] 9.6 Verify record formatting matches expected format
- [ ] 9.7 Test duplicate detection by running scripts twice
- [ ] 9.8 Test update detection by modifying activity in COROS and re-syncing
- [ ] 9.9 Trigger GitHub Actions workflow manually and verify success
- [ ] 9.10 Verify automated cron job runs successfully (wait for scheduled run)
- [ ] 9.11 Check workflow logs for any errors or warnings

## 10. Documentation Updates

- [ ] 10.1 Update README.md title from "Garmin to Notion" to "COROS to Notion"
- [ ] 10.2 Update README.md feature descriptions to reference COROS
- [ ] 10.3 Update README.md prerequisites section (COROS Training Hub account)
- [ ] 10.4 Update README.md environment variables section (COROS_EMAIL, COROS_PASSWORD)
- [ ] 10.5 Add note about unofficial COROS API limitations
- [ ] 10.6 Update script execution examples in README
- [ ] 10.7 Update CLAUDE.md project description to reference COROS
- [ ] 10.8 Add migration notes for users switching from Garmin version

## 11. Cleanup (Optional)

- [ ] 11.1 Delete `garmin-activities.py` after verifying new script works
- [ ] 11.2 Delete `personal-records.py` after verifying new script works
- [ ] 11.3 Remove Garmin-related comments from codebase
- [ ] 11.4 Archive field mapping documentation for reference
