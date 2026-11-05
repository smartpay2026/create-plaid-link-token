# Create Plaid Link Token: Logical Contract

**Status**: Proposed logical contract for SPD owner review. The workspace contains no existing SPD API conventions or service schemas. Adopt the actual SPD route, authentication middleware, error envelope, and identifier formats when binding this contract to the existing host.

## SPD operation

- **Method**: `POST`
- **Proposed route**: `/consumers/{consumerId}/plaid/link-token` (confirm route/versioning with the existing SPD API owner)
- **Authentication**: Existing SPD authentication middleware. The authenticated principal is trusted server context, not a JSON body field.
- **Content type**: `application/json`

### Request

The route's `consumerId` is the same logical value as the required Consumer ID input. If existing SPD conventions place Consumer ID in the body instead, keep one canonical representation and reject conflicting values.

| Field             | Required | Meaning                                                                                   |
| ----------------- | -------: | ----------------------------------------------------------------------------------------- |
| `sessionId`     |      Yes | Active onboarding session associated with this consumer and completed phone verification. |
| `requestId`     |      Yes | Valid caller-supplied request identifier.                                                 |
| `correlationId` |      Yes | Valid caller-supplied correlation identifier.                                             |

The caller must not provide a trusted user identity, Plaid environment, Plaid credentials, products, or country codes. The service derives identity from authenticated context and fixes Plaid product/country configuration server-side.

### Success response

Return only after Plaid returns a usable token/expiration and the required SPD audit outcome has been durably recorded.

| Field                   | Meaning                                                                                            |
| ----------------------- | -------------------------------------------------------------------------------------------------- |
| `linkToken`           | Plaid Link Token; sensitive and short-lived.                                                       |
| `expirationTimestamp` | Plaid `expiration`, parsed as an ISO-8601 date-time and serialized without changing its instant. |
| `requestId`           | Original validated request ID.                                                                     |
| `correlationId`       | Original validated correlation ID.                                                                 |

Do not include the Plaid provider request ID unless the SPD response standard explicitly requires it; retain it only in approved operational metadata if needed. Never log the token-bearing response body.

### Failure response

Return a stable SPD error envelope, preserving valid supplied `requestId` and `correlationId` values. No failure response contains `linkToken`, credentials, raw Plaid bodies/messages, or sensitive internal details.

| SPD code (proposed)              | Condition                                                                                 | Retry guidance                                                                                                           |
| -------------------------------- | ----------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| `INVALID_REQUEST`              | Missing/malformed required identifiers or request fields                                  | No; correct request.                                                                                                     |
| `UNAUTHENTICATED`              | No valid SPD principal                                                                    | No; authenticate.                                                                                                        |
| `INVALID_CONSUMER`             | Consumer ID invalid or principal not authorized for it                                    | No; use a generic response that does not reveal whether another consumer exists.                                         |
| `INVALID_SESSION`              | Session missing, inactive, not associated with consumer, or phone verification incomplete | No; complete/restart onboarding.                                                                                         |
| `PLAID_AUTHENTICATION_FAILURE` | Plaid reports invalid API credentials (for example `INVALID_API_KEYS`)                  | No; operator action; alert through approved internal monitoring.                                                         |
| `PLAID_UNAVAILABLE`            | Plaid unavailable or transient upstream service failure                                   | Retry according to safe bounded caller guidance; this operation itself does not retry token creation.                    |
| `NETWORK_FAILURE`              | Plaid network timeout/connectivity failure                                                | Retry guidance may be provided to the caller; no automatic create retry.                                                 |
| `RATE_LIMITED`                 | Caller or Plaid rate limit (including `RATE_LIMIT_EXCEEDED`)                            | Retry only after a safe delay; expose `Retry-After` only if the SPD contract permits.                                  |
| `LINK_REAUTH_REQUIRED`         | `ITEM_LOGIN_REQUIRED` or equivalent requires the consumer to remediate/re-authenticate  | No blind retry; restart/update the appropriate Link flow. Validate this mapping against the current Plaid error catalog. |
| `AUDIT_UNAVAILABLE`            | Required audit recording failed                                                           | No token returned; safe retry may produce a separate token, so callers must not assume idempotency.                      |
| `INTERNAL_ERROR`               | Unexpected failure                                                                        | Generic response; no provider/internal details.                                                                          |

Exact HTTP status codes must follow SPD conventions. Suggested mapping is 400 for malformed input, 401 for unauthenticated, 404 (or SPD's equivalent generic authorization response) for invalid/unauthorized consumer or session, 429 for rate limits, 502/503 for upstream/configuration availability failures, and 500 for unexpected internal failures. Keep response codes stable even if Plaid changes human-readable messages.

## Required validation and call order

1. Validate `requestId`, `correlationId`, Consumer ID, and `sessionId`; do not manufacture identifiers.
2. Authenticate the request through the SPD authentication boundary.
3. Authorize the principal for Consumer ID without disclosing other consumers' existence.
4. Confirm the session is active, belongs to that consumer, and satisfies phone-verification state.
5. Resolve tenant configuration and secret references from trusted context.
6. Call Plaid only after all prior checks succeed.
7. Validate the token and expiration; record the required outcome audit event; then return the success response.

## Plaid provider contract

- **Endpoint**: `/link/token/create` at the allowlisted base URL for the configured tenant environment.
- **Transport**: HTTPS POST with JSON. Put `client_id` and `secret` in the provider headers where supported by the selected API version; never place them in logs or caller-visible diagnostics.
- **Required configured request values**: `client_name`, `language`, `products: ["auth"]`, `country_codes: ["US"]`, and `user.client_user_id` (stable unique non-PII SPD identifier).
- **SPD-only input**: `sessionId` is validated against SPD onboarding state and is not forwarded as a Plaid field.
- **Response fields used**: `link_token` (non-empty) and `expiration` (valid ISO-8601 date-time). Any missing/invalid field is a provider failure; no partial success.
- **Provider error parsing**: Use stable `error_type`/`error_code` values for internal classification. Do not rely on provider display messages or raw HTTP status alone. Never pass raw provider details to the client.
- **Retries**: No automatic retry of token creation. Preserve trace IDs on the outbound request using SPD's approved header/context contract; do not assume arbitrary header propagation is safe.

## SPD adapter boundaries (logical, wire format TBD)

| Adapter                               | Required behavior                                                                                                                                                                              |
| ------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `IConsumerAccessValidator`          | Validate the trusted principal and authorize it for the requested Consumer ID. Return a non-enumerating denial result.                                                                         |
| `IOnboardingSessionValidator`       | Verify session exists, is active, belongs to Consumer ID, and phone verification is complete.                                                                                                  |
| `IAuditRecorder`                    | Record required decision/outcome events with trace identifiers; never accept Link Token or provider credentials in its event model. Required success-audit failure prevents returning a token. |
| `ITenantPlaidConfigurationResolver` | Resolve allowlisted Plaid environment, Key Vault references, client name, language, and approved non-PII Plaid user ID mapping using trusted tenant context.                                   |

Bind these logical boundaries to the existing SPD contracts before implementation; this document does not invent service URLs, claims, HTTP payloads, or audit event schemas.

- Security and privacy requirements
- Keep Link Token in memory only for the request/response; never persist, cache, audit, or write it to routine diagnostics.
- Redact request/response bodies and secret-bearing headers in HTTP client logs and exception telemetry.
- Do not include PII in Plaid `client_user_id`; confirm consumer-key treatment with SPD privacy policy.
- Never trust environment, credentials, product selection, authorization state, or tenant routing from the request body.
- Preserve valid request/correlation IDs end to end; do not substitute generated identifiers.
