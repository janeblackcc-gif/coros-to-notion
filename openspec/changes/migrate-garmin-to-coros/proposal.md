## Why

The current project synchronizes fitness data from Garmin Connect (China region) to Notion databases using the `garminconnect` Python library. This change migrates the data source from Garmin to COROS Training Hub to support users who have switched from Garmin devices to COROS watches. The migration is necessary because COROS and Garmin use different APIs and data formats, requiring a complete reimplementation of the data retrieval layer while preserving the existing Notion database structure.

## What Changes

- Replace `garminconnect` Python library with custom COROS Training Hub API client
- Implement new `CorosClient` class for authentication and data retrieval
- Migrate `garmin-activities.py` to `coros-activities.py` with COROS API integration
- Migrate `personal-records.py` to `coros-personal-records.py` with COROS data format
- Update activity type mapping from string keys to numeric codes (100=Run, 200=Cycling, etc.)
- Adapt field mappings to handle missing COROS fields (Training Effect metrics, PR/Favorite flags)
- Update environment variables from `GARMIN_*` to `COROS_*`
- Update GitHub Actions workflow to use new scripts
- Remove `garminconnect` dependency from `requirements.txt`
- Update project documentation (README.md, CLAUDE.md) to reflect COROS integration

**Note**: Daily steps and sleep data synchronization are **removed** from the project (user decision: "移除步数和睡眠同步"). Only Activities and Personal Records will be migrated to COROS.

## Capabilities

### New Capabilities
- `coros-api-client`: HTTP client for COROS Training Hub API with authentication, activity querying, and error handling
- `coros-activity-sync`: Synchronization of COROS activities to Notion with field mapping and type conversion
- `coros-personal-records`: Extraction and synchronization of personal records from COROS data

### Modified Capabilities
<!-- No existing capabilities are being modified at the requirement level. The Notion database structure remains unchanged; only the data source changes. -->

## Impact

**Affected Files**:
- New: `coros_client.py` - Core COROS API client
- New: `coros-activities.py` - Activity synchronization script
- New: `coros-personal-records.py` - Personal records synchronization script
- Modified: `requirements.txt` - Remove garminconnect, ensure requests is present
- Modified: `.github/workflows/sync_garmin_to_notion.yml` - Update environment variables and script calls
- Modified: `README.md` - Update documentation for COROS setup
- Modified: `CLAUDE.md` - Update project description
- Removed: `garmin-activities.py` (replaced by coros-activities.py)
- Removed: `personal-records.py` (replaced by coros-personal-records.py)
- Removed: `daily-steps.py` (user decision: remove steps sync)
- Removed: `sleep-data.py` (user decision: remove sleep sync)

**External Dependencies**:
- COROS Training Hub API (`teamcnapi.coros.com` for China region) - unofficial reverse-engineered API
- Authentication: POST `/account/login` with MD5-hashed password, token via `accesstoken` header
- Activity list: GET `/activity/query` (summary: date, name, sportType, totalTime, distance, calorie, labelId)
- Activity detail: separate endpoint per activity (for avgHeartRate, avgPower, maxPower)
- Sport type mapping: GET `/activity/fit/getImportSportList`
- Personal records: NO API endpoint — must calculate from activity data
- Notion API - no changes required
- Reference implementations: xballoy/coros-api (TypeScript), Dhivakarkd/corus-mcp, jmn8718/coros-connect

**Risks**:
- COROS API is unofficial and undocumented; endpoints verified via community implementations
- Non-official API may change without notice
- Personal Records API endpoint confirmed NOT to exist (must calculate from activities)
- Activity detail requires per-activity API call (N+1 pattern, mitigated by Fetch-Filter-Detail strategy)

**Data Impact**:
- User will clear existing Garmin data from Notion databases before migration
- Notion database schema remains unchanged (COROS-unsupported fields set to default values)
