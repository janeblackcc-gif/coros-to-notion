## ADDED Requirements

### Requirement: Personal records data retrieval
The system SHALL retrieve personal records from COROS API or calculate them from activity data if dedicated endpoint does not exist.

#### Scenario: Fetch personal records from API
- **WHEN** COROS API provides personal records endpoint
- **THEN** system fetches records directly from endpoint

#### Scenario: Calculate records from activities
- **WHEN** COROS API lacks personal records endpoint
- **THEN** system calculates records (fastest pace, longest distance, etc.) from activity list

#### Scenario: Handle empty activity history
- **WHEN** user has no COROS activities
- **THEN** system exits without creating personal record entries

### Requirement: Record type mapping
The system SHALL map COROS personal record types to existing Notion record categories.

#### Scenario: Map distance records
- **WHEN** record is for fastest 1K, 5K, 10K, or half marathon
- **THEN** system creates corresponding Notion record with name and time

#### Scenario: Map longest distance records
- **WHEN** record is for longest run or longest ride
- **THEN** system creates Notion record with distance value

#### Scenario: Map power records
- **WHEN** record is for max average power (20 min)
- **THEN** system creates Notion record with power value in watts

#### Scenario: Handle unmapped record types
- **WHEN** COROS provides record type not in mapping
- **THEN** system logs warning and skips record

### Requirement: Record value formatting
The system SHALL format personal record values according to record type (time, distance, power, etc.).

#### Scenario: Format pace records
- **WHEN** record is fastest 1K pace
- **THEN** system formats as "MM:SS /km"

#### Scenario: Format distance records
- **WHEN** record is longest run distance
- **THEN** system formats as "XX.XX km"

#### Scenario: Format power records
- **WHEN** record is max average power
- **THEN** system formats as "XXX W"

### Requirement: Record update detection
The system SHALL detect when personal records are updated and mark old records as non-PR in Notion.

#### Scenario: Update existing record with new date
- **WHEN** new record date is more recent than existing record
- **THEN** system unmarks old record PR checkbox and creates new record marked as PR

#### Scenario: Skip record if unchanged
- **WHEN** record date and value match existing Notion record
- **THEN** system skips update and logs no change

#### Scenario: Create first occurrence of record
- **WHEN** record type does not exist in Notion database
- **THEN** system creates new record marked as PR

### Requirement: Record icon and cover assignment
The system SHALL assign appropriate emoji icons and Unsplash cover images to personal record entries.

#### Scenario: Assign icon for distance records
- **WHEN** record is for running distance (1K, 5K, 10K)
- **THEN** system assigns running shoe emoji (👟)

#### Scenario: Assign icon for cycling records
- **WHEN** record is for longest ride
- **THEN** system assigns bicycle emoji (🚴)

#### Scenario: Assign cover image
- **WHEN** creating any personal record
- **THEN** system assigns relevant Unsplash cover image URL

### Requirement: Record date handling
The system SHALL store the date when the personal record was achieved.

#### Scenario: Record achievement date from API
- **WHEN** COROS API provides record achievement date
- **THEN** system stores date in Notion Date field

#### Scenario: Infer date from activity data
- **WHEN** calculating records from activities
- **THEN** system uses activity date where record was achieved

### Requirement: Multi-record support per category
The system SHALL support multiple records per category (e.g., daily, weekly, monthly step records).

#### Scenario: Store step records variants
- **WHEN** COROS provides most steps (day), (week), and (month)
- **THEN** system creates three separate Notion records

#### Scenario: Format multi-variant record names
- **WHEN** record has variants
- **THEN** system appends variant suffix to record name (e.g., "Most Steps (Day)")
