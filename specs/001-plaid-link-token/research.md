# Research: Create Plaid Link Token

**Feature**: `001-plaid-link-token`  
**Date**: 2026-11-04  
**Requirements source**: `spec_reviewed.md`; approved review: `checklists/requirements_reviewed.md`

## Findings and decisions

### 1. Runtime and service framework

- **Decision**: Target C# on .NET 10 LTS and expose the operation through ASP.NET Core on the existing SPD host.
- **Rationale**: This is the user's selected stack. Microsoft's support policy lists .NET 10 as LTS with support through November 14, 2028 (the latest servicing patch is required). The repository has no app code, project file, target framework, or hosting manifest, so the exact host conventions must be adopted when the service is integrated.
- **Alternatives considered**: Use .NET 8 to match an unknown existing service, or remain stack-neutral. .NET 8 support ends November 10, 2026 and no existing codebase was found to inherit; .NET 10 LTS is the better new-service target.
- **Source**: [Microsoft .NET support policy](https://dotnet.microsoft.com/en-us/platform/support/policy/dotnet-core).

### 2. Plaid API client

- **Decision**: Isolate Plaid behind a typed `HttpClient` adapter calling Plaid's REST API. Use `IHttpClientFactory` for client lifetime/configuration and a bounded request timeout; do not retry `/link/token/create` automatically.
- **Rationale**: Plaid documents `/link/token/create`, the required `client_name`, `language`, `country_codes`, and product fields, plus the returned `link_token` and ISO-8601 `expiration`. The current Plaid API reference provides SDK samples for several languages but the reviewed page does not establish a supported .NET SDK for this project. A narrow REST adapter avoids choosing an unverified SDK and keeps provider serialization/error mapping testable. ASP.NET Core documents typed clients as a DI-supported pattern for encapsulating a backend endpoint.
- **Alternatives considered**: Use a third-party/generated Plaid SDK client; defer until the SPD package policy confirms a maintained .NET SDK and version. Retry transient errors; rejected for token creation because the requirements explicitly make repeated calls independently token-producing and do not promise idempotency.
- **Provider configuration**: `products` is fixed to `["auth"]`; `country_codes` to `["US"]`. Supply the configured `client_name` and `language`. Supply a stable, unique, non-PII `client_user_id` (normally a suitably opaque SPD consumer key, subject to SPD privacy policy). Never send the SPD `session_id` to Plaid unless an approved provider field is later specified; use it for SPD validation only.
- **Error handling**: Prefer Plaid's `error_type` and `error_code` over HTTP status alone. Translate them to SPD classifications; do not surface provider `display_message`, raw body, or credentials. Suggested explicit mappings are `INVALID_API_KEYS` → non-retryable `PLAID_AUTHENTICATION_FAILURE`; `RATE_LIMIT_EXCEEDED` → retryable `RATE_LIMITED`; `ITEM_LOGIN_REQUIRED` → non-retryable `LINK_REAUTH_REQUIRED` (consumer must complete the appropriate Link recovery/update flow). Validate these mappings against the current Plaid error catalog when implementing.
- **Source**: [Plaid Link API: `/link/token/create`](https://plaid.com/docs/api/link/#linktokencreate); [ASP.NET Core `IHttpClientFactory`](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/http-requests?view=aspnetcore-10.0).

### 3. Secret and environment management

- **Decision**: Resolve the tenant's Plaid environment and Key Vault secret references from trusted deployment configuration. Retrieve secrets using Azure Identity with workload/managed identity in deployed environments; use developer identity for local development. Do not select the environment from request data or store secrets in source control/configuration files.
- **Rationale**: The reviewed spec requires Sandbox, Development, and Production behavior to be selected by tenant deployment context and credentials managed in Azure Key Vault. Microsoft's .NET Key Vault quickstart recommends `DefaultAzureCredential` for passwordless local/deployed identity flows.
- **Alternatives considered**: Pass Plaid credentials in requests, commit them in settings, or hardcode a global Plaid environment; all conflict with tenant isolation and secret-handling requirements.
- **Source**: [Azure Key Vault secrets client library for .NET](https://learn.microsoft.com/en-us/azure/key-vault/secrets/quick-create-net?tabs=windows).

### 4. SPD identity, onboarding-session, and audit integrations

- **Decision**: Define narrow ports for authentication/consumer authorization, active onboarding-session validation, and audit recording. Bind the adapters to the existing SPD contracts during implementation; fail closed for token issuance if required authentication/session/audit checks cannot be completed.
- **Rationale**: User confirmed existing SPD contracts, but this documentation-only repository contains no contract files, endpoints, schemas, or integration packages. The design must not invent their wire protocols. The `session_id` check must establish active state, consumer association, and completed phone-verification state before any Plaid call.
- **Alternatives considered**: Trust caller-provided identity/session flags or build new SPD services. Neither is supported by repository evidence or the reviewed scope.
- **Implementation binding**: Record actual SPD service names, endpoints, auth mechanism, request/response schemas, timeouts, and audit event names in the contract tests when those existing definitions are supplied.

### 5. Token lifetime, audit order, and storage

- **Decision**: Keep Link Token only in request-local memory until response emission; do not persist, cache, or log it. Validate the token/expiration, record the required successful issuance audit event without the token or credentials, then return the response. If the required audit write fails after Plaid issued a token, discard the token and return a safe failure.
- **Rationale**: FR-006/009/010/018 and NFR-003 require complete success, fail-closed auditing, and no token retention. Plaid supplies the expiration; the service must return it unchanged and must not extend or locally infer it.
- **Alternatives considered**: Persist tokens for retry/reuse or return the token before audit completion; both conflict with the approved security and audit requirements.

### 6. Contract and testing strategy

- **Decision**: Specify a logical JSON operation contract and proposed HTTP mapping in `contracts/create-link-token.md`; align the route and status-code conventions with the existing SPD host before implementation. Use xUnit unit tests, fake-adapter integration tests, and contract tests around the Plaid payload and SPD adapter boundaries.
- **Rationale**: The workspace has no API conventions or test project to reuse. The contract can be specified without claiming an existing route or SPD wire format.
- **Alternatives considered**: Add a new standalone service deployment or database; unnecessary for a single stateless operation and unsupported by project evidence.

## Requirements review and artifact caveats

1. `spec_reviewed.md` includes FR-013–FR-018; the user confirmed these are in scope. `requirements_reviewed.md` only traces FR-001–FR-012. Add traceability entries for the later requirements before implementation so test coverage and review sign-off remain consistent.
2. The repository's `.specify/memory/constitution.md` is an unfilled template. No formal constitution rules could be inferred; the plan treats the reviewed feature's security, audit, traceability, and performance requirements as gates.
3. The Speckit setup script identifies `spec.md` as its default input, but this plan deliberately uses the explicitly approved `spec_reviewed.md`. No source code or SPD API contracts are present in the workspace.

## References

- [Plaid Link API](https://plaid.com/docs/api/link/#linktokencreate)
- [Microsoft .NET support policy](https://dotnet.microsoft.com/en-us/platform/support/policy/dotnet-core)
- [ASP.NET Core `IHttpClientFactory`](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/http-requests?view=aspnetcore-10.0)
- [Azure Key Vault secrets client library for .NET](https://learn.microsoft.com/en-us/azure/key-vault/secrets/quick-create-net?tabs=windows)
