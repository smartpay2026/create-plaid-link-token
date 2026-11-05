# Quickstart: Validate Create Plaid Link Token

This guide describes the end-to-end checks for the planned implementation. The current repository contains no application project, so these commands become runnable after the service and test projects are added and the proposed paths are aligned with SPD conventions.

## Prerequisites

- .NET 10 SDK at the latest supported servicing patch.
- Access to the existing SPD host conventions and the actual identity, onboarding-session, and audit contracts.
- A Plaid Sandbox tenant configuration and Key Vault references. Use developer identity locally and managed/workload identity in deployed environments; do not put Plaid credentials in source files or committed settings.
- Deterministic fake implementations for SPD identity/session/audit adapters and a fake Plaid client for ordinary automated tests.

## Automated validation

From the repository root, after the projects are created:

1. Restore and build the solution using the repository's solution file and standard CI commands.
2. Run all unit and contract tests with `dotnet test`.
3. Run integration tests in `tests/CreatePlaidLinkToken.IntegrationTests/` with fake SPD adapters and a fake Plaid response.
4. Run the Plaid contract test and verify the outbound request has exactly the required `products: ["auth"]` and `country_codes: ["US"]`, uses the selected allowlisted environment, and contains only a stable non-PII `client_user_id`.
5. Run the HTTP-boundary contract tests against the host's actual authentication/error conventions after those are bound.

### Required scenarios and expected outcomes

- Valid authenticated principal + authorized Consumer ID + active phone-verified session + valid Plaid response + successful audit → non-empty token, original IDs, exact provider expiration.
- Missing/malformed Request ID or Correlation ID → validation failure before Plaid; no fabricated ID.
- Missing/invalid authentication, consumer mismatch, missing/inactive/mismatched session, or incomplete phone verification → safe failure and zero Plaid calls.
- Plaid `INVALID_API_KEYS`, `RATE_LIMIT_EXCEEDED`, `ITEM_LOGIN_REQUIRED`, timeout, 5xx, malformed response, empty token, and missing/invalid expiration → expected safe classification, correct retry guidance, no token in failure.
- Required audit failure after Plaid success → fail closed; token is discarded and not returned or logged.
- Repeated requests → independently issued responses; no idempotency or token cache assumptions.
- Log/audit redaction test → Link Token, Plaid secret, and client ID are absent from captured records and exception messages.

## Sandbox smoke test

After adapter contracts and deployment configuration are bound:

1. Configure an isolated Sandbox tenant using Key Vault and the supported developer/workload identity flow.
2. Authenticate as a test SPD consumer and use a session known to be active, associated with that consumer, and phone-verified.
3. Invoke the create-token operation from the approved SPD API client. Do not copy tokens into tickets, logs, screenshots, or source files.
4. Verify the response contains the token and Plaid-provided expiration, and that Request ID/Correlation ID are unchanged.
5. Verify the audit service records the result without the Link Token or credentials.
6. Launch Plaid Link only through the approved client flow and confirm the token works for the configured Auth/US use case. Do not proceed to public-token exchange or downstream account processing; those are outside this feature.

## Performance and release gate

Run a representative non-production load test with dependency latency measured and included. Confirm at least 95% of successful requests complete within 3 seconds. Record the workload/concurrency, environment, dependency latency, and test window with the result. Do not hide slow dependency calls by excluding them from measurement, and do not add create-token retries to improve the metric.

## Readiness gates

Before implementation is considered ready to start:

- Bind the four logical adapters in `contracts/create-link-token.md` to the actual SPD contracts.
- Confirm route, HTTP statuses, identifier formats, tenant-to-environment mapping, Key Vault secret names, Plaid `client_name`/language, and non-PII `client_user_id` mapping with owners.
- Add FR-013–FR-018 to the approved requirements traceability table.
- Replace the placeholder project constitution if the repository is expected to enforce project-wide principles.
