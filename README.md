# Claim Settlement Decision API

A small ASP.NET Core Web API vertical slice targeting .NET 10. It evaluates low-value home insurance claims against policy data retrieved from a configured Policy Admin HTTP endpoint and records decisions in memory.

## Architecture

- **Domain**: request/response/policy models and deterministic settlement rules.
- **Application**: use-case orchestration and interfaces for policy retrieval and decision recording.
- **Infrastructure**: typed `HttpClient` adapter and thread-safe in-memory decision repository.
- **API**: HTTP endpoint, request validation, error mapping, configuration and dependency injection.
- **Tests**: deterministic unit tests using fakes; no live Policy Admin system is required to run tests.

Dependency direction is toward the Domain/Application abstractions. The core rule engine has no HTTP, ASP.NET Core, or storage dependency.

## Requirements

- .NET 10 SDK
- A reachable Policy Admin API for manual end-to-end execution (not needed for tests)

## Configure and run

Set `PolicyAdmin:BaseAddress` in `src/ClaimSettlement.Api/appsettings.json` or override it with an environment variable. For example:

```bash
# Linux/macOS
export PolicyAdmin__BaseAddress="http://localhost:5099/"

# Windows PowerShell
$env:PolicyAdmin__BaseAddress = "http://localhost:5099/"
```

The remote service must expose `GET /policies/{policyNumber}` and return JSON such as `{"coverageLimit":10000,"isActive":true}`. The configured base address must end with `/` so the relative `policies/...` path resolves as intended.

From the solution directory:

```bash
dotnet restore
dotnet build --no-restore
dotnet test --no-build
 dotnet run --project src/ClaimSettlement.Api
```

The API will use the URL shown in the startup output. Example request:

```bash
curl -X POST http://localhost:5000/claim-settlement \
  -H "Content-Type: application/json" \
  -d '{"policyNumber":"POL-123","claimAmount":2500,"propertyAgeYears":10}'
```

Use the actual local URL printed by `dotnet run` if it differs from port 5000.

## HTTP contract

`POST /claim-settlement`

Request:

```json
{"policyNumber":"POL-123","claimAmount":2500,"propertyAgeYears":10}
```

Successful decision response:

```json
{"policyNumber":"POL-123","isApproved":true,"statusReason":"Auto-Settled"}
```

- `400 Bad Request`: blank policy number, non-positive claim amount, negative property age, or malformed JSON.
- `404 Not Found`: Policy Admin returns 404 / no policy exists. No decision is recorded.
- `502 Bad Gateway`: Policy Admin is unavailable, returns another unsuccessful status, or returns invalid JSON/data. Internal exception details are not returned.
- `200 OK`: decision was made and recorded, whether approved or referred for review.

## Rules and precedence

1. Inactive policy -> `Policy Inactive`.
2. Claim amount greater than coverage -> `Coverage Limit Exceeded`.
3. Claim amount greater than the automatic-settlement threshold -> `Adjuster Review Required`.
4. Otherwise -> `Auto-Settled`.

The threshold is 3,000 for property age <= 30 and 1,000 for property age > 30. Threshold and coverage boundaries are inclusive. Review is not a rejection of the underlying insurance claim.

## Assumptions and trade-offs

- Policy data is retrieved synchronously for each claim; no caching is implemented.
- No additional settlement business rules are applied beyond the conditions and precedence in the brief. Invalid JSON or an empty policy response is treated as a dependency failure.
- Decisions are recorded in a thread-safe in-memory collection and are lost when the process restarts. A production system would use durable storage and define retention/audit requirements.
- A 5-second `HttpClient` timeout is configured. No retry/circuit breaker is included to keep the exercise focused.
- Request cancellation is passed to the HTTP call and repository, and checked before evaluation/recording. Cancellation is not converted into a settlement result.
- Authentication, authorization, durable persistence, distributed tracing, metrics, and a separate Policy Admin stub are outside this exercise.

## Production evolution discussion

Add structured logs and correlation/trace IDs without logging secrets or unnecessary personal data; instrument outbound HTTP latency, status and failure rates; use bounded retries only for appropriate transient failures and safe/idempotent requests; add circuit breaking and timeouts with a documented budget; persist decisions transactionally with an audit trail; and add contract tests against the Policy Admin schema.
