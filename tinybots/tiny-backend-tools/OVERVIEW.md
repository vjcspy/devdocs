# tiny-backend-tools — Overview

## TL;DR
- Shared TypeScript toolkit that every Tinybots Node service imports for Express scaffolding, Awilix DI, request context propagation, `/healthcheck` routes, and MySQL lifecycle management (`lib/index.ts`).
- Centralizes Kong authentication, permission enforcement, validation, serialization, GraphQL helpers, and domain models so repos like `megazord-events`, `m-o-triggers`, `sensara-adaptor`, and the Wonkers suite expose consistent HTTP surfaces.
- Ships infrastructure helpers (cron job harnesses, AWS SQS producer/consumer with context-aware logging, async module lifecycle, pools/timers, config loaders) plus security/util services (email, password hashing, TOTP, OAuth API client).
- Mocha + NYC suite exercises controllers, middleware, security services, and Localstack-backed SQS flows; `ci/docker-compose.yml` is what CI runs (Node 22, Yarn 3.8.7, Localstack).

## Table of Contents
- [Repo Purpose & Interactions](#repo-purpose--interactions)
- [Inventory & Layout](#inventory--layout)
- [Controllers / Public Surface](#controllers--public-surface)
- [Key Services / Repository & Logic](#key-services--repository--logic)
- [External Dependencies & Cross-Service Contracts](#external-dependencies--cross-service-contracts)
- [Tests & Tooling](#tests--tooling)
- [Data & Integration Map](#data--integration-map)
- [Gaps & Risks](#gaps--risks)

## Repo Purpose & Interactions

`tiny-backend-tools` is the platform layer for Tinybots’ Node.js services. It provides:

- Base Express application classes (`TinyApp*` and `TinyDatabaseApp*`) that boot servers on port 8080, wire Morgan logging, attach request context IDs, register `/healthcheck`, and expose Awilix containers for controllers/services. Database variants create a MySQL pool via `mysql2`.
- Authentication/authorization utilities that wrap `kong-js`’s `KongAuthenticationProvider`, validate the `x-consumer-*` headers set by Kong, and dial the Dashboard user service to enforce permission constants (see `lib/constants/Permissions.ts`). Repos such as `wonkers-accounts`, `wonkers-api`, and `m-o-triggers` rely on these guards.
- Cross-cutting middleware (validation, serializer, async handler, context logger, Slack-backed error middleware) so every service behaves consistently under Kong, Cron, and queue workloads.
- Infrastructure modules (cron jobs, SQS, pools, timers, OAuth HTTP client) used by automation services (`megazord-events`, `sensara-adaptor`) to connect to AWS, Slack, SMTP, and other HTTP APIs.

Because this package exports everything via `lib/index.ts`, any change propagates into all Tinybots backend repos after a single publish—treat it as a platform component rather than a leaf library.

## Inventory & Layout

```
tiny-backend-tools/
├── lib/                     # TypeScript source exported to consumers
│   ├── controller/          # Base controllers, DbHealthCheck, AuthenticatedController
│   ├── middleware/          # Context/logger, admin validators, validation, error handling
│   ├── validation/          # KongValidationService
│   ├── service/             # EmailService, PasswordService, TotpService
│   ├── providers/           # cron jobs, AWS SQS, permission API, generic pools
│   ├── modules/             # AwilixWrapper + async module lifecycle contracts
│   ├── repository/          # Database + Transaction wrappers
│   ├── model/               # BaseDomain + config/domain DTOs
│   ├── graphql/             # BaseResolver + input validator
│   ├── utils/               # Config loaders, ApplicationError, ContextGroup transformer
│   └── TinyApp*/TinyDatabaseApp*.ts
├── test/                    # Mocha/Chai suites covering every exported surface
├── ci/                      # docker-compose + scripts (Localstack + node runner)
├── dist/, coverage/         # Build outputs (gitignored)
├── package.json             # Yarn 3 workspace metadata, scripts (Node >=20)
└── yarn.lock
```

- `scripts.test` runs `nyc mocha --require ts-node/register` across `test/**/*.ts`, enforces coverage (statements 55%, lines 60%), then prints HTML reports in `coverage/`.
- `ci/docker-compose.yml` spins up Localstack (SQS) and a Node 22 Alpine runner executing `ci/node-verify.sh` (installs deps, runs `yarn test` with `NODE_OPTIONS="--no-experimental-strip-types"`).
- The repo ships compiled JS in `dist/`, but contributors work directly in `lib/` TypeScript.

## Controllers / Public Surface

### Base Express App Classes
- `TinyAppUnauthenticated` & `TinyAppAuthenticated` (`lib/TinyApp*.ts`) build an Express server (morgan logger, body parsers, `/healthcheck`) without a DB. The authenticated variant injects `KongAuthenticationProvider` and `KongValidationService`.
- `TinyDatabaseAppBase` + `TinyDatabaseAppUnauthenticated/Authenticated` bootstrap a MySQL pool via `MySQLConfig`, register a singleton `Database`, expose `app` + `container`, and start on port 8080 while keeping connections alive (keepAliveTimeout 61s). `TinyDatabaseAppAuthenticated` inherits Kong guards.
- `TinyDatabaseAppAuthenticatedPermissions` adds permission enforcement. It registers Awilix entries (`ContainerNames.VALIDATOR_ADMIN`, `ContainerNames.VALIDATOR_PERMISSION`, etc.) and exposes helpers:
  - `useAdminValidatorMiddleware()` – Express `Handler` that authenticates dashboard admins.
  - `usePermissionValidatorMiddleware(permissions)` – ensures the Kong-authenticated admin owns the listed `Permission.*` constants.
  - `usePermissionRoutes()` – pattern matching variant via `matchPermissions`.
- Applications extend one of these classes, register controllers on the Awilix container (`this.container.register('fooController', asClass(FooController))`), and wire routes on `this.app` before calling `app.start()`.

```ts
this.app.route('/internal/v1/foo')
  .get(
    contextMiddleware(randomUUID),
    ValidationMiddleware.headerValidator(KongHeader, false, true, false),
    this.usePermissionValidatorMiddleware([Permission.TAAS_ORDER_READ_ALL]),
    asyncHandler(this.container.resolve<FooController>('fooController').handleFoo)
  )
  .use(errorMiddleware(slackService), serializerMiddleware);
```

### Health Checks
- `HealthCheck` (`lib/controller/HealthCheck.ts`) immediately invokes the provided callback, used by `TinyAppUnauthenticated`.
- `DbHealthCheck` (`lib/controller/DbHealthCheck.ts`) pings the MySQL pool (via the `Database` wrapper) and returns `state: 'no connection with mysql database'` when the pool rejects. Base apps mount it at `GET /healthcheck`.

### Kong Auth & Permission Guards
- `KongValidationService` (`lib/validation/KongValidationService.ts`) wraps `KongAuthenticationProvider` to enforce:
  - `authenticateAdmin` (requires `x-consumer-username: tinybots-dashboard-users` and `role === ADMIN`),
  - `authenticateTessaOwner` (`role === USER`),
  - `authenticateRobot`, `authenticateIntegration`, `authenticateIntegrationSingleOrg`, and `checkUserRobotAccess`.
- `AdminMiddleware` (`lib/middleware/AdminMiddleware.ts`) exposes request handlers:
  - `adminValidator`, `robotValidator`, `tessaOwnerValidator`, `userRobotAccessValidator`.
  - `permissionValidator(permissions)` / `matchPermissions(routes)` integrate with `PermissionAPIProvider` for dashboard permission checks.
  - `getReqUser` returns the cached `DashboardUser|Robot|IntegrationUser` patched onto `req`.
- These middlewares expect `KongValidationService` and optionally an `IPermissionProvider` to be bound in the container (handled automatically by `TinyDatabaseAppAuthenticatedPermissions`).

### Validation & Serialization
- `ValidationMiddleware` (`lib/middleware/ValidationMiddleware.ts`) offers `bodyValidator`, `queryValidator`, `pathValidator`, and `headerValidator` that run `class-transformer` + `class-validator`. When validation fails, it responds with HTTP 400 and aggregated constraint messages.
- `serializerMiddleware` serializes any `BaseDomain` (or arrays) returned by controllers through `BaseDomain.toPlainWithContext`, ensuring `@Expose` + `@ContextGroup` annotations are respected. Use `ContextGroupMiddleware` + `ContextGroup` transformer if you need field-level masking per route grouping.
- `asyncHandler` wraps async Express handlers to funnel errors to `next()`.

### Request Context, Logging & Errors
- `contextMiddleware` attaches an `IRequestContext` to every request containing `callRef` (either from inbound `Call-Ref` header or generated), `serviceRef`, and writes them back to response headers. `contextLoggerMiddleware` then attaches (optionally) a winston child logger to the context so downstream services/loggers can include `callRef`.
- `errorMiddleware` standardizes error responses:
  - Handles `ApplicationError`, `ts-http-errors`’ `ExtendedError`, and arbitrary objects with a `statusCode`.
  - Sends Slack alerts via `tb-ts-slack-notification` whenever a 5xx error occurs, including the `callRef`.
- `serializerMiddleware` and `contextLoggerMiddleware` should be mounted early to ensure `loggerFromCtx` works everywhere.

### GraphQL Helper Surface
- `graphql/BaseResolver` + `InputValidator` enable permission-aware GraphQL resolvers. Call `BaseResolver.SetPermissionApiProvider(provider)` during bootstrap, then wrap resolvers with `BaseResolver.Wrap({ selectUserId, permissions, validateArgDto }, resolver)` to run permission checks and DTO validation before executing user code.

## Key Services / Repository & Logic

### Data Access
- `Database` + `Transaction` (`lib/repository`) wrap `mysql2` pooling with promise-based `query`, `queryOne`, and transaction helpers. `TinyDatabaseAppBase` registers a singleton instance in Awilix.
- `Repository` is a minimal base class that injects the `Database` for domain-specific repositories downstream services define.
- `MySQLConfig` + `SmtpConfig` provide strongly typed config DTOs validated through `class-validator`.

### Domain Models & Serialization
- `BaseDomain` centralizes serialization, validation (`FromPlain`), and error wrapping (throws `ApplicationError` with `ErrorHttpCode` mappings). `Robot`, `SimpleRobot`, `Relation`, `RelationTessaOwner`, `Password`, `LogConfig`, and `PermissionsProviderConfig` describe common Tinybots entities.
- `ContextGroup` transformer lets you annotate sensitive fields to only be serialized when `ContextGroupMiddleware` attaches the matching group to the request context.

### Security & Identity Services
- `PasswordService` enforces password strength via `zxcvbn`, generates salts/hashes with PBKDF2 (SHA-512), and verifies both Node- and Java-generated password strings.
- `TotpService` issues/validates 2FA secrets and tokens using `notp` + Base32 encodings.
- `EmailService` + `EmailSender` render `subject.ejs` + `html.ejs` templates and dispatch via Nodemailer transports created by `createMailTransport`.
- `KongValidationService` (covered above) plus `PermissionAPIProvider` provide HTTP front doors for Kong-authenticated requests and dashboard permission validation.
- `OauthApiClient` is an Axios wrapper that injects Bearer tokens via `ITokenManager`, refreshes on 401s, supports configurable retry counts, and gates concurrent refreshes (waiters) so only one refresh occurs at a time.

### Infrastructure Helpers
- `providers/cron` exposes `ContextCronJob` and `SimpleContextCronJob`, which automatically create request contexts/logger children per execution and catch/log `ApplicationError`s.
- `providers/sqs` implements `ContextSQS`, a combined producer/consumer:
  - `send` serializes payloads, injects the request `callRef` as a message attribute, and retries on missing queues by calling `createQueue`.
  - `poll` returns an async generator of `IContextMessage` objects with `ack()`/`fail()`, and attaches a nested context logger for each message.
  - Implements the `IAsyncModule` contract so it can be registered with `Modules.AwilixWrapper` for lifecycle management.
- `providers/pool` ships `SimpleContextPool` (generic resource pooling with stop logic) and `TimerPool` (blocking scheduler using `setTimeout` resources).
- `modules/AwilixWrapper` tracks every registered async module, calling `init(ctx)` during bootstrap and `stop(ctx)` during shutdown, ensuring Cron/SQS/Pools can clean up gracefully.

### Utilities
- `utils/utils.ts` contains `loadConfigValue` (uses `config` package) and `loadConfigValueV2` (accepts custom providers) to instantiate DTOs or primitives with validation.
- `utils/Errors.ts` defines `ApplicationError` (supports `annotate` for nested errors/plain objects) and `logApplicationError`.
- `constants/ErrorCodes`, `constants/Permissions`, and `errors/Errors.ts` hold HTTP codes and reusable `ExtendedError` subclasses for misuse of Kong guards.
- `logger/ContextLogger` + `logger/Logger` provide typed logger interfaces and default console loggers.

## External Dependencies & Cross-Service Contracts

- **Kong Gateway (`kong-js`)**: `KongAuthenticationProvider` validates `x-consumer-username`, `x-authenticated-userid`, and `x-authenticated-scope` headers. Roles map to `Errors.NotAdminError`, etc. Consumers must forward original Kong headers to downstream services so shared middleware can re-validate.
- **Dashboard User Service**: `PermissionAPIProvider` posts to `/internal/v3/admin/accounts/:userId/validate` on the Wonkers API surface (`kongFig.dashboardService.address`). Failures throw `ForbiddenError` or `InternalServerError`.
- **MySQL**: `TinyDatabaseAppBase` expects a pool config (`host`, `user`, `password`, `database`, `connectionLimit`). Health checks and repositories assume the schema defined in `typ-e` / `wonkers-db`.
- **AWS SQS**: `ContextSQS` uses the v3 AWS SDK; tests rely on Localstack. Message attributes always include `callRef` to propagate tracing info. `ISQSConfig` optionally accepts `endpoint`, `profile`, `maxAttempts`, `maxNumberOfMessages`, `waitTimeSeconds`, etc.
- **Slack**: `errorMiddleware` depends on `tb-ts-slack-notification`’s `SlackService`. Provide an instance that implements `notifyError(message, error)` to route 5xx incidents to the #alerts channel.
- **SMTP + EJS**: Email templates live outside this repo; `EmailService.sendEmail(templatePath, from, to, locals)` expects `html.ejs` and `subject.ejs` to exist beneath `templatePath`.
- **Cron & Timers**: `ContextCronJob` uses the `cron` package for schedules like `'*/5 * * * *'` and automatically builds request contexts/loggers for each run.
- **OAuth / HTTP APIs**: `OauthApiClient` expects an `ITokenManager` implementation (often defined inside consuming repos). It wraps Axios interceptors to refresh tokens.

## Tests & Tooling

- **Application scaffolding tests** (`test/app/*.ts`): `TinyDatabaseAppAuthenticatedPermissionsTest.ts` uses `supertest` + `nock` to prove controllers can be registered, Kong headers validated, permissions enforced, `/healthcheck` wired, and Slack error middleware invoked without a live DB.
- **Middleware coverage** (`test/middleware/*.ts`): Each middleware has unit tests—e.g., `ErrorMiddlewareTest.ts` validates Slack notifications for 5xx errors, `ContextMiddlewareTest.ts` ensures `Call-Ref` propagation, `ContextGroupMiddleware.ts` checks group patching, and `ValidationMiddlewareTest.ts` asserts 400 responses.
- **Security & service tests** (`test/service/*.ts`): `KongValidationServiceTest.ts`, `PasswordServiceTest.ts`, `TotpServiceTest.ts`, and `EmailServiceTest.ts` cover authentication flows, password hashing, TOTP verification, and EJS email rendering with fixtures in `test/fixtures/email/`.
- **Infrastructure tests**:
  - `test/providers/sqs/sqsIT.ts` spins up Localstack via `ci/docker-compose`, validates `createQueue`, `send` retries, streaming `poll`, `ack()/fail()`, and re-delivery semantics.
  - `test/providers/pool/*.ts` assert pooling and timer scheduling semantics.
  - `test/providers/cron/*.ts` check `newCronContext` logger inheritance and `ContextCronJob` scheduling.
- **GraphQL & utils**: `test/modules/AsyncModulesTest.ts`, `test/utils/utils.ts`, `test/utils/MultiIndexedMapTest.ts`, etc., ensure wrappers and helpers behave as expected.
- **Running tests locally**: `yarn test` (Node 20+) runs everything. Integration suites need Localstack on port 4566; use `docker compose -f ci/docker-compose.yml up localstack` or run `ci/local-test.sh`.

## Data & Integration Map

1. **HTTP request flow**: Express routes run through `contextMiddleware` → `contextLoggerMiddleware` → `ValidationMiddleware` → application controllers (`Controller`/`AuthenticatedController`) → domain services/repositories. Responses pass through `serializerMiddleware` and `errorMiddleware`, which log via `loggerFromCtx` and optionally ping Slack.
2. **Authentication & permissions**: Kong attaches user headers. `KongValidationService` authenticates them, caches the user on `req`, then `PermissionAPIProvider` hits the Dashboard service to ensure they own the required `Permission` constants (e.g., `Permission.M_O_TRIGGERS_SETTING_WRITE_ALL`). Robot-facing routes can instead call `robotValidator` or `userRobotAccessValidator`.
3. **Data models**: Downstream repos instantiate `BaseDomain` subclasses (Robot, Relation, Password, LogConfig, SmtpConfig). Serialization respects context groups, ensuring sensitive fields are masked unless a middleware adds the appropriate group symbol.
4. **MySQL interactions**: `Database` handles connection pooling and queries for `tinybots` / `wonkers-db` schemas. `DbHealthCheck` pings the pool used by Cron jobs and controllers.
5. **Async workflows**: Cron jobs (`ContextCronJob`) and SQS consumers/producers (`ContextSQS`) create new request contexts/loggers so queue processing retains the original `callRef`. `TimerPool.enqueue` supports delayed execution without blocking the event loop.
6. **Error propagation**: `ApplicationError` captures nested errors/objects; `errorMiddleware` logs structured JSON, attaches `callRef`/`serviceRef`, and notifies Slack for server errors.
7. **External calls**: `OauthApiClient` wraps axios to add OAuth2 flows for other HTTP APIs. Email/TOTP/password services share credential management across repos so user flows look identical.

## Gaps & Risks

- **Retry accounting in `ContextSQS.retry`**: `retry()` increments the `attempts` counter twice per failure, so `maxAttempts` effectively halves. This might prematurely abort retries under transient AWS errors—verify desired semantics or fix the counter.
- **Password strength enforcement is opt-in**: `PasswordService.createPassword()` does not force a `checkPasswordStrength()` call. Repos must remember to call it or risk allowing weak passwords.
- **Email template path injection**: `EmailService.sendEmail(templatePath, ...)` concatenates `${template}/html.ejs` and `${template}/subject.ejs` without sanitization. Ensure callers only pass trusted template directories to prevent reading arbitrary files.
- **GraphQL permission provider bootstrap**: `BaseResolver` throws if `SetPermissionApiProvider()` was not invoked before calling `Wrap()`, but there is no compile-time enforcement. Services must remember to wire it during startup, otherwise resolvers crash on first request.
- **CI scripts assume AWS ECR credentials**: `ci/test.sh` logs into AWS ECR and Docker Hub before running tests. Local contributors without credentials must use `yarn test` directly or edit the script, otherwise CI helpers fail early.
- **Async module lifecycle discipline**: `ContextSQS`, `ContextCronJob`, and pools rely on consumers to call `init(ctx)` and `stop(ctx)` (often through `Modules.AwilixWrapper`). Forgetting to register them means jobs never start or resources leak on shutdown.

This overview should give you the map needed to extend Tinybots services safely, whether you are adding a new controller, wiring cron jobs, or evolving the shared infrastructure exported by `tiny-backend-tools`.
