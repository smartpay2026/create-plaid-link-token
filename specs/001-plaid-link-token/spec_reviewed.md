# Feature Specification: Create Plaid Link Token

**Feature Branch**: Not specified
**Created**: 2026-11-01
**Status**: Draft
**Input**: User description: Create a temporary Plaid Link Token for an authenticated SPD Consumer to securely initiate bank account linking through Plaid Link.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Start secure bank linking (Priority: P1)

An authenticated SPD Consumer requests a Plaid Link Token for their own consumer account so they can begin linking a bank account through Plaid Link.

**Why this priority**: Issuing a valid, short-lived token is the operation's primary user value and enables the bank-linking journey.

**Independent Test**: Submit a request with an authenticated user context associated with a valid Consumer ID and verify that the response contains a Plaid Link Token, its expiration timestamp, and the original Request ID and Correlation ID.

**Acceptance Scenarios**:

1. **Given** a valid authenticated user context associated with the requested Consumer ID, **When** the consumer requests a Link Token, **Then** the operation returns a token and its expiration timestamp along with the same Request ID and Correlation ID.
2. **Given** a valid authenticated user context, **When** token creation succeeds, **Then** the token is usable to initiate the intended Plaid Link flow only until the returned expiration timestamp.
3. **Given** an authenticated user context associated with a different consumer, **When** a token is requested for the mismatched Consumer ID, **Then** the operation rejects the request and does not return a token.

### User Story 2 - Receive a safe, traceable failure (Priority: P2)

A consumer or calling SPD service receives a clear failure outcome when identity validation, token creation, or a required dependency prevents the operation from completing, without exposing credentials or sensitive provider details.

**Why this priority**: Predictable failures protect consumers and allow SPD support and operations teams to investigate incidents without leaking sensitive information.

**Independent Test**: Exercise unauthenticated, invalid-consumer, provider-authentication, provider-unavailable, network, rate-limit, and unexpected-error cases; verify no token is returned and the response retains the request and correlation identifiers when available.

**Acceptance Scenarios**:

1. **Given** no valid authenticated user context, **When** a token is requested, **Then** the operation rejects the request as unauthenticated and does not call Plaid to create a token.
2. **Given** an invalid Consumer ID, **When** a token is requested, **Then** the operation returns an invalid-consumer failure and does not return a token.
3. **Given** Plaid authentication failure, Plaid service unavailability, network failure, or rate limiting, **When** token creation cannot complete, **Then** the operation returns an appropriate safe failure classification, does not return a token, and preserves available request and correlation identifiers.
4. **Given** an unexpected internal failure, **When** the operation cannot complete, **Then** it returns a generic internal-error outcome without exposing secrets or sensitive internal details.

### Edge Cases

- Missing, malformed, or mismatched Request ID or Correlation ID: reject invalid required identifiers where validation is applicable; do not fabricate replacement identifiers, and preserve valid supplied values in the outcome.
- Authenticated context is present but cannot be associated with the requested Consumer ID: reject the request without revealing whether another consumer account exists.
- Plaid returns an unusable token or omits its expiration timestamp: treat token creation as failed and do not return a partial success.
- A dependency times out or returns a rate-limit response: return a safe retryable or throttling failure classification; do not return a partial token response.
- Audit logging is unavailable: do not return a token unless the required audit event has been recorded.
- A token request is repeated: each successful response is an independently issued token; token reuse and deduplication are not guaranteed by this operation.

## Requirements *(mandatory)*

### Functional Requirements

### Inputs and Outputs

**Inputs**:

- Consumer ID
- Correlation ID
- Request ID
- Authenticated User Context

**Successful outputs**:

- Plaid Link Token
- Link Token Expiration Timestamp
- Request ID (unchanged from the request)
- Correlation ID (unchanged from the request)

**Failure output**: A safe, classified failure outcome with no Link Token and with valid supplied Request ID and Correlation ID values preserved where available.

### Functional Requirements

- **FR-001**: The operation MUST accept Consumer ID, Correlation ID, Request ID, and an authenticated user context.
- **FR-002**: The operation MUST validate that the user context is authenticated through the SPD Authentication Service before requesting a Link Token.
- **FR-003**: The operation MUST verify that the authenticated user context is authorized for the supplied Consumer ID. A missing, invalid, or mismatched Consumer ID MUST be rejected without issuing a token.
- **FR-004**: For an authorized request, the operation MUST request a temporary Plaid Link Token from the Plaid Link Token API for the intended bank-linking flow.
- **FR-005**: On success, the operation MUST return the Plaid Link Token, the expiration timestamp supplied for that token, and the original Request ID and Correlation ID.
- **FR-006**: The operation MUST NOT report success or return a token if token creation fails, the token is unusable, or its expiration timestamp is unavailable.
- **FR-007**: The operation MUST classify the specified failures: unauthenticated consumer, invalid Consumer ID, Plaid API authentication failure, Plaid service unavailable, network failure, rate limiting, and unexpected internal server error.
- **FR-008**: Failure responses MUST NOT include a Plaid Link Token, provider credentials, or sensitive internal/provider details. They MUST include the original Request ID and Correlation ID when those values were validly supplied.
- **FR-009**: The operation MUST submit the required security and outcome audit events to the SPD Audit Logging Service. It MUST NOT return a token when a required audit event cannot be recorded.
- **FR-010**: The operation MUST NOT write the Link Token or provider credentials to audit records or routine diagnostic logs.
- **FR-011**: The operation MUST preserve valid Request ID and Correlation ID values across dependency calls and outcomes to support request tracing.
- **FR-011**: The operation MUST preserve valid Request ID and Correlation ID values across dependency calls and outcomes to support request tracing.
- **FR-012**: The operation MUST reject a missing or malformed required Request ID or Correlation ID before requesting a Link Token, and MUST NOT fabricate a replacement identifier.
- { Zubeir }
- **FR-013**: The operation MUST accept and validate an active onboarding `session_id` associated with the consumer's phone verification state.
- **FR-014**: The Link Token configuration MUST explicitly request the `auth` Plaid product (`products: ["auth"]`) to enable downstream routing and account number verification for payment rails.
- **FR-015**: The Link Token request MUST enforce country code constraints restricted to the United States (`country_codes: ["US"]`).
- **FR-016**: The operation MUST handle Plaid runtime environments (Sandbox, Development, Production) dynamically via secure service configuration pulled from Azure Key Vault based on tenant deployment context.
- **FR-017**: The operation MUST classify and propagate explicit Plaid API error codes (e.g., `ITEM_LOGIN_REQUIRED`, `INVALID_API_KEYS`, `RATE_LIMIT_EXCEEDED`) into standardized, safe SPD failure classifications with appropriate retry guidance.
- **FR-018**: The backend service MUST NOT cache or persist the generated temporary Link Token beyond its short-lived TTL expiration window.

{ /

Zubeir }

- A token request is repeated: the operation does not guarantee idempotency or deduplication; multiple requests may result in multiple tokens.

### Non-Functional Requirements

- **NFR-001**: For successful requests under normal dependency conditions, at least 95% of responses MUST be returned within 3 seconds. Time spent waiting for external dependencies is included.
- **NFR-002**: The operation MUST transmit and handle the Link Token only in a manner that prevents access by unauthenticated users and consumers other than the authorized consumer.
- **NFR-003**: The token MUST be treated as temporary and sensitive; it MUST NOT be retained longer than required to complete the Link initiation flow.
- **NFR-004**: Failure outcomes MUST be distinguishable by the calling SPD service for the specified error conditions while remaining safe to expose to the caller.
- **NFR-005**: Audit and trace records MUST support investigation by Request ID and Correlation ID without storing the Link Token or provider credentials.

### External Dependencies

- Plaid Link Token API
- SPD Authentication Service
- SPD Audit Logging Service
- { Zubeir }
- Azure Key Vault (Credentials management)  { /Zubeir }

### Acceptance Criteria

- **AC-001**: An authenticated consumer authorized for the supplied Consumer ID receives a non-empty Link Token, its corresponding expiration timestamp, and unchanged Request ID and Correlation ID.
- **AC-002**: No token is issued for an unauthenticated request or an invalid or unauthorized Consumer ID.
- **AC-003**: Each listed dependency and service failure produces a safe, correctly classified failure response with no token.
- **AC-004**: No success response is returned without a usable token, its expiration timestamp, and successful recording of required audit events.
- **AC-005**: Neither audit records nor routine diagnostics contain the Link Token or provider credentials.
- **AC-006**: Under normal dependency conditions, at least 95% of successful responses complete within 3 seconds.
- {ZUbeir}
- 
- **AC-007**: A request containing a valid `session_id` and correct payload configuration successfully receives a Plaid Link Token configured with `products: ["auth"]` and `country_codes: ["US"]`.
- **AC-008**: Requests with missing, malformed, or inactive `session_id` values are rejected prior to calling the Plaid API.
- **AC-009**: Provider-specific errors returned by Plaid are intercepted, translated into safe client-facing error codes, and logged with full request tracing identifiers without exposing provider secrets.
- </Zubeir}

### Key Entities

- **Consumer**: The SPD customer whose bank-linking journey is being initiated; identified by Consumer ID and bound to the authenticated user context.
- **Authenticated User Context**: Trusted identity and authorization information used to establish the caller's authentication state and access to the requested Consumer ID.
- **Link Token**: A temporary, sensitive credential returned to an authorized caller to start the Plaid Link flow.
- **Request ID**: Identifier for the individual operation request, returned unchanged for traceability.
- **Correlation ID**: Identifier used to correlate this operation with related SPD activity, returned unchanged for traceability.
- **Expiration Timestamp**: The point in time after which the returned Link Token is no longer valid.

## Assumptions

- SPD Authentication Service provides a trusted authenticated user context that can be checked against the requested Consumer ID.
- The Plaid Link Token API determines the token's validity period and supplies the expiration timestamp; the operation returns that timestamp without extending token validity.
- Request ID and Correlation ID are supplied by the caller and are expected to be valid before the operation starts.
- Required audit events are a security control; if the SPD Audit Logging Service cannot record them, the operation fails without returning a token.
- Plaid and SPD dependencies are configured and managed outside this operation.
- A provider or network failure is reported safely; automatic retries are not promised by this operation.

## Out-of-Scope Items

- Completing Plaid Link, selecting a financial institution, or collecting bank credentials.
- Exchanging a public token for an access token, retrieving account details, or managing linked accounts.
- Creating, refreshing, revoking, or deduplicating Link Tokens outside the single request described here.
- Defining the Plaid Link user interface or client-side integration.
- Defining SPD user enrollment, authentication policy, or Consumer ID lifecycle.
- Setting provider-specific retry schedules, rate-limit quotas, or operational service-level objectives beyond the response-time criterion above.
