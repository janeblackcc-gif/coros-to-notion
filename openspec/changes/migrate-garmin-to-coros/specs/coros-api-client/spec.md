## ADDED Requirements

### Requirement: Authentication with COROS Training Hub
The CorosClient SHALL authenticate with COROS Training Hub using email and password credentials, obtaining an access token for subsequent API requests.

#### Scenario: Successful login
- **WHEN** user provides valid COROS email and password
- **THEN** client successfully authenticates and stores accessToken for subsequent requests

#### Scenario: Invalid credentials
- **WHEN** user provides invalid email or password
- **THEN** client raises AuthenticationError with clear error message

#### Scenario: Network failure during login
- **WHEN** network request to /login endpoint fails
- **THEN** client raises ConnectionError with retry suggestion

### Requirement: Activity retrieval with pagination
The CorosClient SHALL retrieve activity data from COROS Training Hub API with support for pagination and date filtering.

#### Scenario: Fetch activities with default parameters
- **WHEN** get_activities() is called without parameters
- **THEN** client returns first page (up to 200 activities) from most recent activities

#### Scenario: Fetch activities with date range
- **WHEN** get_activities(from_date="20240101", to_date="20240131") is called
- **THEN** client returns activities within the specified date range (inclusive)

#### Scenario: Fetch activities with pagination
- **WHEN** get_activities(page=2, size=100) is called
- **THEN** client returns second page with up to 100 activities

#### Scenario: Handle empty activity list
- **WHEN** no activities exist in the specified date range
- **THEN** client returns empty list without errors

### Requirement: Token expiration handling
The CorosClient SHALL detect expired access tokens and automatically re-authenticate.

#### Scenario: API call with expired token
- **WHEN** API request returns 401 Unauthorized
- **THEN** client automatically calls login() to obtain new token and retries the original request

#### Scenario: Re-authentication failure
- **WHEN** automatic re-authentication fails
- **THEN** client raises AuthenticationError without infinite retry loop

### Requirement: Error handling and logging
The CorosClient SHALL provide comprehensive error handling with detailed logging for debugging.

#### Scenario: API rate limiting
- **WHEN** API returns 429 Too Many Requests
- **THEN** client logs rate limit details and raises RateLimitError with retry-after information

#### Scenario: Malformed API response
- **WHEN** API returns non-JSON response or unexpected structure
- **THEN** client logs raw response and raises APIError with descriptive message

#### Scenario: Request logging
- **WHEN** any API request is made
- **THEN** client logs request method, endpoint, parameters, and response status

### Requirement: HTTP request abstraction
The CorosClient SHALL provide internal _request() method that handles common HTTP operations including authentication headers and error translation.

#### Scenario: Request with authentication
- **WHEN** _request(endpoint, params) is called after successful login
- **THEN** request includes accessToken in headers

#### Scenario: Request before authentication
- **WHEN** _request() is called before login() completes
- **THEN** client raises AuthenticationError indicating login is required
