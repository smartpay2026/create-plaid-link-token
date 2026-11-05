# Implementation Plan: Create Plaid Link Token

**Branch**: `001-plaid-link-token` | **Date**: 2026-11-04 | **Spec**: [spec_reviewed.md](./spec_reviewed.md)

**Approved requirements review**: [requirements_reviewed.md](./checklists/requirements_reviewed.md)

## Summary

Implement a stateless SPD backend operation that issues a Plaid Link Token only after validating the authenticated principal, consumer ownership, required active onboarding session, and request identifiers. Use an ASP.NET Core service on the existing SPD host, narrow adapters for SPD identity/session/audit services, a typed Plaid REST client, and tenant-aware environment/credential resolution backed by Azure Key Vault. Return the token only after validating Plaid's token and expiration and recording the required audit outcome. Never persist or log the token or provider credentials.

The design includes FR-001–FR-018 and NFR-001–NFR-005 in `spec_reviewed.md`, as explicitly requested. `requirements_reviewed.md` marks requirements FR-001–FR-012 in its traceability table; the FR-013–FR-018 additions therefore need their traceability rows added as a review follow-up before implementation. The plan does not silently remove these reviewed-spec requirements.

## Technical Context

**Language/Version**: C# / .NET 10 LTS (latest servicing patch)

**Primary Dependencies**: ASP.NET Core; `IHttpClientFactory` typed client for Plaid REST; `Azure.Identity` and `Azure.Security.KeyVault.Secrets`; xUnit and ASP.NET Core integration-test host for tests

**Storage**: None for this operation. No Link Token cache, database, or durable token store.

**Testing**: xUnit unit tests, adapter/contract tests with deterministic fakes, ASP.NET Core integration tests, and a performance test for the 95th-percentile target

**Target Platform**: Existing SPD hosting platform, with Azure Key Vault and tenant deployment context. The workspace contains no deployment definition; use the existing host's managed identity/workload identity and deployment conventions.

**Project Type**: Backend HTTP API operation

**Performance Goals**: At least 95% of successful requests complete within 3 seconds, including dependency latency (NFR-001 / AC-006).

**Constraints**: Authentication, consumer authorization, active session validation, and required audit recording precede successful response; products fixed to `auth`; country codes fixed to `US`; provider environment and credentials are server-side tenant configuration; no automatic create-token retry or token persistence; errors must be safe and traceable.

**Scale/Scope**: One operation and its external-service adapters; request volume and host sizing follow the existing SPD service's capacity policy. No new database or background worker.

## Constitution Check

**Pre-design gate**: The repository's `.specify/memory/constitution.md` contains only template placeholders and defines no ratified principles or project-specific gates. No constitution-derived violation can be assessed. Proceed using the approved security, audit, traceability, and performance requirements as the design gates; filling the constitution is a repository governance follow-up, not a feature dependency.

**Requirements gates**:

- **PASS** — no Plaid call before request-ID/correlation-ID validation, authentication, consumer authorization, and active-session validation.
- **PASS** — Link Token is transient response data only; exclude it and all provider credentials from logs, audit records, caches, and storage.
- **PASS** — return success only after Plaid response validation and successful required audit recording; fail closed if audit recording fails.
- **PASS** — preserve valid tracing identifiers through adapters and all outcomes.
- **PASS** — use bounded dependency calls and measure the 95th-percentile response target; do not retry token creation automatically because duplicate issuance is not guaranteed to be idempotent.
- **FOLLOW-UP** — update requirements traceability for FR-013–FR-018 before implementation; this discrepancy is recorded, not resolved by changing the approved artifacts.

**Post-design gate**: PASS against the requirements above. The only follow-ups are the specification traceability update and binding the abstract SPD adapters to the existing contracts during implementation setup.

## Design Overview

1. The HTTP boundary obtains identity only from the SPD-authenticated request context; it never accepts caller-supplied identity claims as trusted data.
2. Validate required IDs and request shape, authenticate/authorize the principal for the requested Consumer ID, and validate that `session_id` is active, belongs to that consumer, and satisfies the required phone-verification state.
3. Resolve the tenant's Plaid environment and credentials through server-side configuration backed by Azure Key Vault. Do not accept an environment selector or secret from the caller.
4. Call Plaid `/link/token/create` with `products: ["auth"]` and `country_codes: ["US"]`, plus configured `client_name`, `language`, and a stable non-PII Plaid user identifier. `session_id` is an SPD validation input, not a Plaid field.
5. Validate that the response has a non-empty Link Token and a parseable expiration timestamp. Map provider errors to stable SPD error classifications and retry guidance; never return raw provider messages/codes to the consumer.
6. Record the required success/failure audit event without token or credentials. Return the token only after required success audit has completed; discard it and fail closed otherwise.

Detailed logical entities are in [data-model.md](./data-model.md); API and adapter boundaries are in [contracts/create-link-token.md](./contracts/create-link-token.md); validation scenarios are in [quickstart.md](./quickstart.md).

## Project Structure

### Documentation (this feature)

```text
specs/001-plaid-link-token/
├── plan.md
├── research.md
├── data-model.md
├── quickstart.md
└── contracts/
    └── create-link-token.md
```

### Proposed Source Code (repository currently contains no application source)

```text
src/CreatePlaidLinkToken.Api/
├── Api/                         # Request/response mapping and endpoint
├── Application/                 # Orchestration and validation
├── Domain/                      # Result, classifications, and configuration models
└── Infrastructure/
    ├── Plaid/                   # Typed HTTP client and provider error translation
    ├── Spd/                     # Identity, onboarding-session, and audit adapters
    └── Configuration/           # Tenant/environment and Key Vault resolution

tests/CreatePlaidLinkToken.UnitTests/
tests/CreatePlaidLinkToken.IntegrationTests/
tests/CreatePlaidLinkToken.ContractTests/
```

**Structure Decision**: Start with one ASP.NET Core API project and focused test projects. Keep external systems behind narrow adapters, but do not add a generic repository layer or persistence project because the operation stores no feature data. Adapt these paths to the existing SPD solution conventions when the service is added; no application directories currently exist in this repository.

## Complexity Tracking

No constitution violations are identified. The adapter boundaries are justified by the four external integration concerns (identity/authorization, onboarding session, audit, Plaid/Key Vault) and enable deterministic failure-path tests; no additional service or data store is proposed.
