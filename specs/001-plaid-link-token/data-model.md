# Data Model: Create Plaid Link Token

This is the logical model for the operation; it does not prescribe database tables. The operation has no persistence layer. Any token-bearing object is short-lived request/response memory only.

## Entities and value objects

### CreateLinkTokenCommand

| Field                      | Type / constraints                                                            | Source and behavior                                                                                                                                 |
| -------------------------- | ----------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| `consumerId`             | Required opaque SPD identifier; non-empty and format-validated                | Caller request. Must be authorized for the trusted principal.                                                                                       |
| `sessionId`              | Required opaque onboarding session identifier; non-empty and format-validated | Caller request. Validate active state, consumer association, and completed phone-verification state before calling Plaid.                           |
| `requestId`              | Required valid request identifier                                             | Caller request; preserve unchanged when valid. Never generate a replacement.                                                                        |
| `correlationId`          | Required valid correlation identifier                                         | Caller request; preserve unchanged when valid. Never generate a replacement.                                                                        |
| `authenticatedPrincipal` | Trusted server-side principal/claims; never deserialize from request body     | Set by SPD authentication middleware/context.                                                                                                       |
| `tenantContext`          | Trusted deployment/routing context                                            | Resolve from authenticated service context, not an untrusted request field. Used to select the approved Plaid environment and Key Vault references. |

### CreateLinkTokenResult

| Field                   | Type / constraints         | Source and behavior                                                                                                       |
| ----------------------- | -------------------------- | ------------------------------------------------------------------------------------------------------------------------- |
| `linkToken`           | Non-empty sensitive string | Plaid response. Keep request-local; exclude from logs, audit, caching, telemetry, exception text, and persistent storage. |
| `expirationTimestamp` | ISO-8601 UTC date-time     | Plaid response `expiration`; validate and return unchanged. A missing or invalid value makes the operation fail.        |
| `requestId`           | String                     | Original validated request value.                                                                                         |
| `correlationId`       | String                     | Original validated request value.                                                                                         |

### PlaidEnvironmentConfiguration

| Field                 | Type / constraints                                                 | Source and behavior                                                                                                                |
| --------------------- | ------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------- |
| `environment`       | `Sandbox`, `Development`, or `Production`                    | Tenant deployment configuration; never caller-selected.                                                                            |
| `baseUrl`           | Allowlisted Plaid host for the selected environment                | Derived from approved environment mapping; reject arbitrary host values to avoid SSRF/configuration drift.                         |
| `keyVaultReference` | Secret references for Plaid client ID/secret                       | Stored in trusted deployment configuration. Secret values retrieved with managed/workload identity; never sent to callers or logs. |
| `clientName`        | Plaid Link display name, valid per Plaid's current API constraints | Non-secret service configuration.                                                                                                  |
| `language`          | Plaid-supported language                                           | Non-secret service configuration.                                                                                                  |
| `plaidClientUserId` | Unique stable non-PII user identifier                              | Derived using the SPD-approved consumer identifier mapping; must not be phone, email, access token, or `sessionId`.              |

### AuditEvent

| Field                            | Type / constraints                                                        | Behavior                                                                                                        |
| -------------------------------- | ------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------- |
| `eventType`                    | Enumerated security/outcome event                                         | At minimum record request decision and final outcome (issued/denied/failed) according to SPD audit conventions. |
| `requestId`, `correlationId` | Valid identifiers when available                                          | Enable trace lookup; preserve unchanged.                                                                        |
| `consumerReference`            | Minimal approved non-sensitive reference, if required by SPD audit schema | Do not include authentication credentials or Link Token.                                                        |
| `outcomeClassification`        | Stable SPD classification                                                 | Do not place raw provider response/message in the caller-visible event.                                         |
| `timestamp`                    | UTC instant                                                               | Audit service or trusted service clock, per SPD conventions.                                                    |

### OperationFailure

| Field                            | Type / constraints                    | Behavior                                                                                      |
| -------------------------------- | ------------------------------------- | --------------------------------------------------------------------------------------------- |
| `code`                         | Stable safe SPD error code            | Distinguishes expected failure class without raw provider detail.                             |
| `message`                      | Generic safe message                  | No credentials, raw provider body, internal hostnames, or sensitive account-existence detail. |
| `retryable`                    | Boolean                               | Based on standardized failure class, not arbitrary provider text.                             |
| `retryAfter`                   | Optional duration                     | Set only when a reliable, safe retry delay is available and SPD API conventions allow it.     |
| `requestId`, `correlationId` | Valid supplied values where available | Preserve for traceability.                                                                    |
| `linkToken`                    | Must not exist in failure result      | Failures never carry partial success data.                                                    |

## Relationships and invariants

- One `CreateLinkTokenCommand` produces either one `CreateLinkTokenResult` or one `OperationFailure`; never both.
- `authenticatedPrincipal` is associated with exactly the consumer authorized by the identity adapter. The client cannot provide or override this principal.
- An active, phone-verified onboarding session must belong to `consumerId`; otherwise no Plaid request is made.
- The Plaid request always uses the selected tenant environment, `products = ["auth"]`, and `country_codes = ["US"]`.
- `sessionId` is an SPD validation value. It is not a Plaid Link Token API field.
- A result is eligible for return only when token and expiration are valid and the required success audit has been recorded.
- Link Token has no persistence/cache lifecycle. It is discarded on any downstream failure, including an audit-write failure.
- Repeated commands are independent attempts; no idempotency or token deduplication is promised.

## State transitions

`Received` → `IdentifiersValidated` → `Authenticated` → `ConsumerAuthorized` → `SessionVerified` → `PlaidRequested` → `PlaidResponseValidated` → `SuccessAudited` → `Returned`.

Any validation/dependency/audit failure transitions to `Failed` with a safe classification and no token in the failure response. No transition to `PlaidRequested` is allowed before IDs, authentication, authorization, and session validation succeed.
