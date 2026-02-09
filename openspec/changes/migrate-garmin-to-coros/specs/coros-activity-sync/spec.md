## ADDED Requirements

### Requirement: COROS activity data retrieval
The system SHALL fetch activity data from COROS Training Hub using CorosClient and retrieve a configurable number of recent activities.

#### Scenario: Fetch recent activities with default limit
- **WHEN** script runs without COROS_ACTIVITIES_FETCH_LIMIT environment variable
- **THEN** system fetches up to 1000 most recent activities

#### Scenario: Fetch activities with custom limit
- **WHEN** COROS_ACTIVITIES_FETCH_LIMIT=500 is set
- **THEN** system fetches up to 500 most recent activities

#### Scenario: Handle COROS API unavailable
- **WHEN** COROS API is unreachable
- **THEN** system logs error and exits without modifying Notion database

### Requirement: Activity type mapping from numeric codes
The system SHALL convert COROS numeric activity type codes to human-readable display names matching existing Notion select options.

#### Scenario: Map standard activity types
- **WHEN** activity has mode=100 (Running)
- **THEN** system maps to "Running" display name

#### Scenario: Map cycling variants
- **WHEN** activity has mode=200 (Road Bike) or mode=201 (Indoor Cycling)
- **THEN** system maps to "Cycling" and "Indoor Cycling" respectively

#### Scenario: Handle unknown activity type
- **WHEN** activity has unmapped numeric code
- **THEN** system uses "Unknown" as display name and logs warning

### Requirement: Field mapping with default values
The system SHALL map COROS activity fields to Notion properties, using default values for fields not provided by COROS API.

#### Scenario: Map core activity fields
- **WHEN** COROS activity contains date, distance, duration, calories
- **THEN** system maps to Notion Date, Distance (km), Duration (min), Calories fields

#### Scenario: Map power metrics
- **WHEN** COROS activity contains avgPower and maxPower
- **THEN** system maps to Notion Avg Power and Max Power fields

#### Scenario: Handle missing Training Effect fields
- **WHEN** COROS activity lacks aerobic/anaerobic training effect data
- **THEN** system sets Training Effect="Unknown", Aerobic=0, Anaerobic=0

#### Scenario: Handle missing PR and Favorite flags
- **WHEN** COROS activity lacks PR or favorite indicators
- **THEN** system sets PR=False and Fav=False

### Requirement: Duplicate activity detection
The system SHALL detect duplicate activities in Notion database using date, activity type, and activity name as composite key.

#### Scenario: Detect exact duplicate
- **WHEN** activity with same date (±5 minutes), type, and name exists in Notion
- **THEN** system skips creating duplicate and logs skip message

#### Scenario: Update changed activity
- **WHEN** activity exists but distance, duration, or other metrics changed
- **THEN** system updates existing Notion page with new values

#### Scenario: Create new activity
- **WHEN** no matching activity exists in Notion
- **THEN** system creates new Notion page with activity data

### Requirement: Activity icon assignment
The system SHALL assign activity type icons to Notion pages using existing ACTIVITY_ICONS mapping.

#### Scenario: Assign icon for known activity type
- **WHEN** activity type is "Running"
- **THEN** system sets Notion page icon to Running icon URL from ACTIVITY_ICONS

#### Scenario: Skip icon for unknown activity type
- **WHEN** activity type is not in ACTIVITY_ICONS
- **THEN** system creates page without custom icon

### Requirement: Pace calculation
The system SHALL calculate average pace (min/km) from activity speed or distance/duration.

#### Scenario: Calculate pace from average speed
- **WHEN** COROS provides averageSpeed in m/s or km/h
- **THEN** system converts to pace format "MM:SS /km"

#### Scenario: Handle zero speed activities
- **WHEN** activity has zero or missing speed (e.g., strength training)
- **THEN** system sets Avg Pace to empty string or "N/A"

### Requirement: Unit conversion
The system SHALL convert COROS distance and duration units to match Notion field expectations.

#### Scenario: Convert distance to kilometers
- **WHEN** COROS provides distance in meters
- **THEN** system converts to kilometers with 2 decimal places

#### Scenario: Convert duration to minutes
- **WHEN** COROS provides duration in seconds
- **THEN** system converts to minutes with 2 decimal places

### Requirement: Entertainment activity name formatting
The system SHALL preserve existing entertainment activity name formatting (e.g., "ENTERTAINMENT" → "Netflix").

#### Scenario: Format entertainment activity
- **WHEN** activity name is "ENTERTAINMENT"
- **THEN** system replaces with "Netflix" in Notion

#### Scenario: Preserve other activity names
- **WHEN** activity name is not "ENTERTAINMENT"
- **THEN** system uses original activity name
