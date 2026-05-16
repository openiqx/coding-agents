# Backend Engineering Standards

Every change you make must follow these rules without exception.

---

## Principle 1 — Layers own exactly one concern

Every feature is split into exactly four layers. Nothing more, nothing less.

```
Routes / Handlers   →   parse request, call service, write response
Service             →   all business logic
Repository          →   all data access, nothing else
Validation          →   all five validation types + input transformation
```

**The hard rules:**

A function that formats an HTTP response never touches the database.
A function that queries the database never knows what a 200 status code is.
A service never imports a route or handler.
A repository never imports a service.
A handler never imports a repository directly — always through the service.
Business logic never lives in a route, handler, or repository.
Database calls never live in a service.

**What each layer is allowed to do:**

| Layer | Can do | Cannot do |
|---|---|---|
| Route / Handler | Parse request, validate input, call service, write response | Business logic, DB calls |
| Service | Business rules, call repository, cache, call other services | Handle HTTP, write responses, direct DB calls |
| Repository | Query and mutate the database, map DB errors to typed errors | Business logic, cache, HTTP concerns |
| Validation | Define input shape, run all five checks, transform input | DB calls, business logic, side effects |

---

## Principle 2 — Errors propagate up typed, never sideways

**One global handler catches everything.** Nothing else catches except to rethrow.

Every layer throws or returns a typed error. Callers never inspect return values
to detect failure. Failures are errors — not nulls, not booleans, not empty arrays.

### Typed error hierarchy

Define these error types in every project. Map each to an HTTP status code.

```
ValidationError      400   input failed any validation check
AuthenticationError  401   no session, expired session, bad credentials
AuthorizationError   403   wrong role, insufficient permissions
NotFoundError        404   resource does not exist
ConflictError        409   duplicate, unique constraint violation
UnprocessableError   422   FK violation, business rule violation, impossible state
InternalError        500   unexpected — programmer error, not user error
```

### DB error mapping

The repository never returns raw database errors.
Map database-specific error codes to typed errors before returning:

```
unique constraint violation  →  ConflictError 409
foreign key violation        →  UnprocessableError 422
not null violation           →  ValidationError 400
no rows / not found          →  NotFoundError 404
everything else              →  InternalError 500
```

Name constraints descriptively: `table_column_unique`, `table_column_fk_othertable`.
Extract the field name from the constraint name to include in the error.

### Error response envelope

Every error response — no matter where it originates — produces this exact shape:

```json
{
  "success": false,
  "error": {
    "code": "DUPLICATE_USERNAME",
    "message": "This username is already taken.",
    "fields": {
      "username": "This username is already taken."
    }
  }
}
```

`fields` is only present on `ValidationError`. It maps field names to messages.
`code` is always a SCREAMING_SNAKE_CASE string. Define all codes in one file.
The global error handler is the only place that writes error responses.
In development it includes the stack trace. In production it never leaks internals.

---

## Principle 3 — External input is never trusted

Every boundary where data enters the system is validated before it touches
the service. HTTP bodies, query params, path params, environment variables,
webhook payloads — all of it, every time.

### The five validation types — applied in order

**1. Syntactic** — is the structure correct? Are required fields present?
```
username is present
body is valid JSON / XML
id param exists in the path
```

**2. Type** — is the value the correct type after parsing?
```
page is a number (coerce from string if it comes from a query param)
ids is an array
isActive is a boolean
```

**3. Semantic** — does the value make sense in isolation?
```
username: 3–32 chars, lowercase alphanumeric + underscores only
password: 8–128 chars, at least one letter and one number
email: valid format if provided
```

**4. Complex** — do the values make sense together? Cross-field rules.
```
startDate must be before endDate
if paymentType == card: cardNumber is required
if role == admin: stateId is required
```

**5. Transform** — normalise before the service ever sees the data.
```
username → trim whitespace, convert to lowercase
email → trim whitespace, convert to lowercase
unknown / undeclared fields → strip entirely
```

### Rules

Validation runs at the boundary — in middleware or the handler — before the
service is called. The service receives clean, typed, already-transformed data.
It never re-validates.

Validation failures return a structured field errors map.
Every field that failed gets its own entry. The client always knows exactly
which field failed and why.

Validation schemas are defined in the validation layer — never inline in a
handler or service.

### Environment variables

All environment variables are loaded into a validated config struct at boot.
The application crashes immediately with a clear message if any required variable
is missing. No silent defaults for secrets.

Never read environment variables directly outside the config module.

```
Required for every project:
  APP_ENV          development | production | test
  PORT             HTTP port
  DATABASE_URL     connection string
  SECRET_KEY       min 32 chars — generate with: openssl rand -hex 32
```

---

## Principle 4 — Structured logging

Plain text logs are useless in production. You cannot filter them by request ID,
you cannot query them across multiple servers, you cannot build dashboards from them.

**Every log entry is structured JSON in production.**
In development, a human-readable format is acceptable.

### Log levels

```
debug   development only — request details, query params, cache hits/misses
info    significant events — server start, user login, resource created
warn    recoverable problems — validation rejected, rate limit hit, cache miss
error   failures — unhandled errors, DB connection lost, 5xx responses
```

### Every log entry includes

```json
{
  "level": "info",
  "timestamp": "2025-05-16T10:30:00.000Z",
  "message": "User created",
  "service": "my-app",
  "environment": "production",
  "traceId": "abc123",
  "module": "users.service"
}
```

`traceId` — a unique ID attached to every request. Every log line in the same
request shares the same traceId. This is how you reconstruct what happened
across multiple log entries. Attach it via middleware on request start.

`module` — which layer/file emitted this log. Makes filtering trivial.

### Rules

Never use `console.log`, `print`, `fmt.Println`, or equivalent in production code.
Always use the structured logger.

Never log passwords, tokens, secret keys, or PII.
Redact sensitive fields before logging.

The request logger logs every request on completion:
```json
{
  "method": "POST",
  "path": "/v1/users",
  "statusCode": 201,
  "durationMs": 43,
  "traceId": "abc123"
}
```

---

## Principle 5 — Tests define the contract

Writing tests alongside the code forces you to think about failure modes
before you write the happy path. That changes how you design the function.

### Two test types — both required

**Unit tests** — test a single function or method in isolation.
Mock all I/O: database, cache, external APIs, logger.
Fast. No network calls. No file system. No real dependencies.

Cover:
- The happy path
- Every failure branch
- Every validation rule
- Edge cases (empty input, boundary values, null/undefined)

**Integration tests** — test the full request lifecycle against real infrastructure.
Use a dedicated test database — never the development environment.
Truncate all data before each test. Every test starts with a clean state.

Cover:
- Happy path end to end (request in → middleware → handler → service → DB → response out)
- `401` unauthenticated
- `403` wrong role / insufficient permissions
- `400` each category of validation failure
- `404` resource not found
- All domain-specific error cases (`409` duplicate, `422` constraint violation)
- Sensitive fields absent from responses (passwords, secrets, hashes)

### Test naming

Test names describe behaviour, not implementation:
```
✓ creates a user and returns 201 with the user object
✓ returns 409 when username is already taken
✓ returns 400 when password has no numbers
✗ test_create_user          ← too vague
✗ calls repository.create   ← tests implementation, not behaviour
```

### Seed helpers

Every integration test suite has seed helpers — `seedUser`, `seedAdmin`,
`createAuthenticatedClient` or equivalent. Tests never manually insert raw data.
Helpers live in a shared `tests/helpers` or `testutils` file.

---

## Response format

Every response uses the same envelope — no exceptions.

```json
// Success
{ "success": true, "data": { } }

// Created
{ "success": true, "data": { } }

// Paginated list
{
  "success": true,
  "data": [],
  "meta": {
    "page": 1,
    "limit": 20,
    "total": 143,
    "totalPages": 8
  }
}

// Error
{
  "success": false,
  "error": {
    "code": "ERROR_CODE",
    "message": "Human readable message.",
    "fields": { }
  }
}
```

JSON keys are always `camelCase`.
Use `PATCH` for partial updates — never `PUT`.
Never expose internal fields in responses: password hashes, tokens, raw DB error
messages, or stack traces in production.

### Pagination

Every list endpoint supports and validates:
- `page` — integer, min 1, default 1
- `limit` — integer, min 1, max 100, default 20
- `sortBy` — enum of allowed column names for this resource
- `sortOrder` — `asc` or `desc`, default `desc`

---

## Authentication

Two valid approaches. Choose one per project and stay consistent.

### Option A — Server-side sessions (Redis)
Generate a cryptographically random token (minimum 32 bytes) on login.
Store session data in Redis with a TTL. Send the token as an `HttpOnly` cookie.
Best for: apps that need immediate session revocation (deactivate user = instant lockout).

### Option B — JWT
Sign a JWT on login containing the user ID and role. Send it as an `HttpOnly` cookie.
**Never store JWTs in localStorage or sessionStorage** — they are readable by
JavaScript and vulnerable to XSS. The `HttpOnly` cookie is the only secure
storage for a JWT in a browser context.
Set a short expiry (15–60 minutes) and use a refresh token flow for long sessions.
Best for: stateless APIs, microservices, third-party client integrations.

### Password hashing

Use `argon2id` where available — it is the current best practice.
`bcrypt` with a cost factor of 12 or higher is also acceptable.
Never use MD5, SHA-1, SHA-256, or any non-adaptive hashing algorithm for passwords.
Never store plaintext passwords. Never log plaintext passwords.

### Regardless of approach

- Rate limit all auth endpoints
- Lock out or slow down after repeated failed attempts
- Return the same error for wrong username and wrong password — never reveal which one failed
- Invalidate all sessions / revoke all tokens on password change

---

## Security baseline

- All secrets in environment variables — never in code or version control
- `HttpOnly; Secure; SameSite=Strict` on all auth cookies in production
- Rate limiting on all auth endpoints
- HTTPS only in production
- Security headers on every response (HSTS, CSP, X-Frame-Options, X-Content-Type-Options)
- Input length limits on all string fields to prevent DoS
- Never log or return raw database error messages to clients
- Never expose internal error details in production responses

---

## Health checks

Three endpoints — always present:

```
GET /health         liveness — no dependency checks, always 200 if process is alive
GET /health/ready   readiness — checks DB and cache, 503 if any dependency is down
GET /health/details full diagnostics with latencies — auth-gated, for ops use only
```

---

## Database conventions

Every table has:
- `id` — UUID, primary key, generated by the database
- `created_at` — timestamp with timezone, set on insert
- `updated_at` — timestamp with timezone, updated on every change

Soft-deletable resources have `deleted_at` (nullable timestamp).
Always filter `WHERE deleted_at IS NULL` in queries on soft-deletable tables.

Index every column used in WHERE, ORDER BY, or JOIN.
Migrations are append-only. Never edit a migration after it has been applied.

---

## API documentation

Every endpoint is documented with an OpenAPI 3.0 annotation co-located with
the route or handler it describes. Never maintain a separate YAML file — it drifts.

All shared schemas are defined once and referenced via `$ref`.
Every endpoint documents: summary, description, request body, all response codes.

In production: docs are auth-gated.
In development: docs are open at `/v1/docs`.

---

## What never to do

- Never read environment variables outside the config module
- Never use print/console.log/fmt.Println — always use the structured logger
- Never log passwords, tokens, or PII
- Never return raw database errors — always map them to typed domain errors first
- Never swallow errors silently — log and propagate
- Never put business logic in a handler or repository
- Never put database calls in a service
- Never use PUT — PATCH only for partial updates
- Never expose password hashes, tokens, or secrets in API responses
- Never store JWTs in localStorage or sessionStorage — HttpOnly cookie only
- Never skip validation — every external input goes through all five types
- Never skip the typed error hierarchy — every error must be catchable by type
- Never edit applied migrations
- Never ship console.log / debug statements in production code
- Never hardcode secrets, connection strings, or environment-specific values
