# MerchantPay — Backend (Spring Boot)

Payment Gateway & Merchant Acquiring Platform — REST API backend for acquiring banks,
PSPs, and fintechs. Covers merchant onboarding/KYB, terminal provisioning, transaction
authorisation, settlement & funding, chargeback/dispute handling, fraud monitoring, and
in-app notifications, with JWT-based RBAC and a full audit trail.

## Tech Stack

| Concern        | Choice                                             |
|----------------|----------------------------------------------------|
| Language       | Java 17 (compiles/runs on 17–21)                   |
| Framework      | Spring Boot 3.3.x (Web, Data JPA, Security, Actuator) |
| Security       | Stateless JWT (HS256) + method-level RBAC          |
| Persistence    | Spring Data JPA / Hibernate                        |
| Database       | H2 (local) · MySQL / PostgreSQL (prod)             |
| API Docs       | springdoc-openapi (Swagger UI)                     |
| Build          | Maven                                              |

## Architecture

Layered, modular monolith. Each business capability is a self-contained vertical slice
under `com.merchantpay.<module>` (`domain` → `repository` → `service` → `controller` + `dto`).
Modules are decoupled — cross-module references use plain ID fields, not JPA relationships —
so the codebase can later be split into microservices with minimal churn.

```
com.merchantpay
├── common         # BaseEntity, enums, exceptions, error handling, PageResponse
├── config         # OpenAPI config, local data seeder
├── security       # JWT service/filter, principal, SecurityConfig (RBAC)
├── iam            # Users, authentication, audit log              (Module 4.1)
├── onboarding     # Merchant applications, KYB, merchant accounts  (Module 4.2)
├── terminal       # POS/terminal provisioning & lifecycle          (Module 4.3)
├── transaction    # Payment transactions & batches                 (Module 4.4)
├── settlement     # Settlement runs & merchant funding             (Module 4.5)
├── chargeback     # Chargeback cases & dispute evidence            (Module 4.6)
├── fraud          # Fraud flags & merchant risk scoring            (Module 4.7)
└── notification   # In-app notifications & alerts                  (Module 4.8)
```

### Cross-cutting industrial-standards features
- **RBAC** — 6 roles (`MERCHANT`, `ONBOARDING_OFFICER`, `ACQ_OPS`, `DISPUTE_ANALYST`,
  `FRAUD_ANALYST`, `ADMIN`) enforced via `@PreAuthorize`. Merchant users are scoped to
  their own `merchantId`.
- **Audit trail** — every sensitive mutation (overrides, chargeback decisions, fee/risk
  changes, settlement funding) is written to `audit_log` in its own transaction.
- **Validation** — Jakarta Bean Validation on all request DTOs.
- **Consistent errors** — `GlobalExceptionHandler` returns a uniform `ApiError` body.
- **Optimistic locking & auditing timestamps** — via `BaseEntity` (`@Version`, created/updated).
- **Card-data security** — only the card BIN (≤6 digits) is ever accepted/stored; no PAN.
- **Observability** — Spring Actuator (`/actuator/health`, `/metrics`, `/prometheus`).
- **Stateless** — no server session; horizontally scalable behind a load balancer.

## Running locally

Prerequisites: JDK 17+ and Maven (or use the bundled commands below).

```bash
# from the project root
mvn spring-boot:run            # starts on http://localhost:8080/api  (profile: local, H2 in-memory)
# or
mvn clean package
java -jar target/merchantpay-backend-1.0.0.jar
```

On startup (local profile) a default admin is seeded:

```

```

### Key URLs (note the /api context path)
- Swagger UI:      http://localhost:8080/api/swagger-ui.html
- OpenAPI JSON:    http://localhost:8080/api/v3/api-docs
- H2 console:      http://localhost:8080/api/h2-console  (JDBC `jdbc:h2:mem:merchantpay`, user `sa`)
- Health:          http://localhost:8080/api/actuator/health

## Authentication flow

1. `POST /api/auth/login` with `{ "email", "password" }` → returns `accessToken` + `refreshToken`.
2. Send `Authorization: Bearer <accessToken>` on every subsequent request.
3. When the access token expires, `POST /api/auth/refresh` with the refresh token.

In Swagger UI, click **Authorize** and paste the access token.

## Production profile

Run with `-Dspring.profiles.active=prod` and supply env vars:
`DB_URL`, `DB_USERNAME`, `DB_PASSWORD`, `JWT_SECRET` (Base64 256-bit), `CORS_ORIGINS`.
`ddl-auto` is `validate` in prod — manage schema via migrations (Flyway/Liquibase recommended).

## API reference

See [API_DOCUMENTATION.md](API_DOCUMENTATION.md) for the full endpoint catalogue, or the
live Swagger UI which is generated from the code.
