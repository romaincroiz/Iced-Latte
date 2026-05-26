# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

**Local infrastructure (required before running the app):**
```bash
docker compose --env-file .env.example up -d postgres redis minio minio-init
```

**Run the application:**
```bash
set -a && source .env.example && set +a && mvn spring-boot:run
```
App: `http://localhost:8083` | Swagger UI: `http://localhost:8083/api/docs/swagger-ui/index.html`

**Build and verify:**
```bash
mvn clean package            # full build + tests (recommended for CI / first run)
mvn compile && mvn test      # compile then test — use this when target/ is absent or stale
mvn test                     # tests only — safe only when compiled classes already exist in target/
```

**Format code before committing:**
```bash
mvn spotless:apply       # apply Palantir Java Format to touched files
```

**Run a single test class:**
```bash
mvn test -Dtest=ProductReviewManagerTest
```

**Run a single test method:**
```bash
mvn test -Dtest=ProductReviewManagerTest#methodName
```

## Code Style

- Formatting is enforced by **Palantir Java Format** via the Spotless Maven plugin. Run `mvn spotless:apply` before finalizing changes; `mvn validate` will fail if formatting is wrong.
- Import order: `java`, `javax`, `jakarta`, `org`, `com`, blank line for static.
- **NullAway** is active via ErrorProne: code in `@NullMarked` packages must annotate nullable returns/params; fields injected by Spring (`@Autowired`, `@Value`) are exempt.
- Do not reformat files you did not touch.

## Architecture

Iced Latte is a **modular monolith** built with Java 25 and Spring Boot 4. All modules are deployed as one backend service but are organized as independent business slices.

**Root package:** `com.zufar.icedlatte`

**Feature modules** (each is a self-contained business slice):

| Package | Owns |
|---|---|
| `security` | registration, login, JWT, sessions, Google OAuth2 |
| `user` | user profiles, addresses, avatars |
| `product` | catalog, product filters, product images |
| `cart` | shopping cart state and operations |
| `order` | orders, order lifecycle, order history |
| `payment` | Stripe checkout, payments, webhooks |
| `review` | product reviews, ratings, AI moderation, Kafka outbox |
| `favorite` | favorites list |
| `email` | email verification and notifications |
| `filestorage` | AWS S3 / MinIO file upload/download |
| `ratelimit` | request rate-limiting infrastructure |
| `astartup` | startup data migration and bootstrap tasks |
| `common` | shared utilities, pagination, correlation, monitoring, HTTP helpers |

**Standard package shape inside each feature:**
```
feature/
├── api/          # public module boundary: interfaces, records, stable DTOs
│   └── dto/      # DTOs in the public contract
├── endpoint/     # REST controllers (implement generated OpenAPI interfaces)
├── service/      # application services
├── entity/       # JPA entities
├── repository/   # Spring Data repositories
├── converter/    # MapStruct DTO/entity mapping
├── validator/    # feature-specific validation
└── exception/    # feature-specific errors and handlers
```

## Feature Packaging Rules

These rules are enforced at test time by **ArchUnit** (`src/test/java/com/zufar/icedlatte/architecture/`):

1. **Keep code in the owning feature.** Do not move feature-specific code to `common` just because it looks reusable.
2. **Cross-feature dependencies must go through `api/`.** Never import another feature's `entity`, `repository`, `service`, or `converter` directly. Depend on `feature.api.*` interfaces only.
3. **`common` must not depend on any feature module.**
4. **REST controllers must not access repositories** — go through services.
5. **`api/` packages must not contain Spring beans** (`@Service`, `@Component`, `@Repository`, `@Controller`, etc.) or MapStruct `@Mapper` classes.
6. **`api/` packages must not depend on generated OpenAPI DTOs** — those are HTTP-edge models that stay in endpoints.
7. **Feature packages must be free of cycles.**

## OpenAPI Code Generation

API contracts are defined in `src/main/resources/api-specs/*.yaml`. The `openapi-generator-maven-plugin` generates:
- API interfaces (implemented by endpoint classes) into `target/generated-sources/openapi/com/zufar/icedlatte/openapi/<feature>/api/`
- DTO classes into `target/generated-sources/openapi/com/zufar/icedlatte/openapi/dto/`

Run `mvn generate-sources` or `mvn compile` to regenerate. Do not edit generated sources directly — edit the YAML specs.

When modifying API contracts, update the corresponding YAML spec file and regenerate.

## Database Migrations

Schema changes use **Liquibase**. Add new migration SQL files under `src/main/resources/db/changelog/` following the existing naming convention (e.g., `DD.MM.YYYY.partN.description.sql`).

The dev profile (`SPRING_PROFILES_ACTIVE=dev`) drops and recreates the schema on every restart with fresh seed data.

## Testing

- Integration tests extend `IntegrationTestBase` (`src/test/java/com/zufar/icedlatte/test/config/`), which starts PostgreSQL and Redis via **Testcontainers**. Docker must be running.
- `JavaMailSender` and `ObjectStorage` are mocked in `IntegrationTestBase` — no real email or S3 access in tests.
- Tests use the `test` Spring profile.
- Test resources (JSON schemas, fixtures) mirror the feature layout under `src/test/resources/<feature>/`.
- Architecture rules are enforced by `ArchitectureRulesTest` and `ModularityTests` — they run as part of `mvn test`.
- **Always run tests after code changes.** Before considering any change done, run `mvn compile && mvn test`. Do not skip this step.
- **`mvn test` requires compiled sources.** On a clean workspace or after `mvn clean`, the Surefire forked process crashes at test *discovery* with `ClassNotFoundException` on OpenAPI-generated types (e.g. `com.zufar.icedlatte.openapi.dto.OrderStatus`). Root cause: `openapi-generator-maven-plugin` writes sources to `target/generated-sources/openapi/` and they must be compiled before the test JVM starts. Always run `mvn compile` (or `mvn clean package`) before `mvn test` when `target/` is missing.

## Key Infrastructure

- **PostgreSQL** — primary data store, managed by Liquibase
- **Redis** — token/session/cache store
- **MinIO** — local S3-compatible object storage (production uses AWS S3 + CloudFront)
- **Stripe** — payment processing (disabled locally unless credentials are provided in `.env.example`)
- **Kafka** — used by the `review` module's outbox pattern for async event publishing
