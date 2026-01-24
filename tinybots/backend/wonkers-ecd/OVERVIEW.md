> **Branch:** develop
> **Last Commit:** d1fe232
> **Last Updated:** Fri Nov 14 16:05:45 2025 +0800

# wonkers-ecd Overview

## TL;DR

- Exposes two authenticated webhook endpoints that translate Ecare Puur and ZSP notifications into TinyBots order lifecycle actions.
- Maps and validates incoming form payloads, enriches with Ecare user and client data, and calls internal Wonkers TaaS Orders plus Concept TaaS Orders services.
- Persists ECD client–order relations and OAuth client credentials in MySQL; emits Slack and email notifications via shared tooling.

## Repo Purpose & Bounded Context

wonkers-ecd bridges external care platforms (Ecare Puur and ZSP) to TinyBots order flows. It authenticates integration users via Kong headers, validates notification payloads, enriches data from Ecare APIs, and creates/returns Tessa orders in downstream TaaS services (production and concept/experimental flows). The service owns minimal persistence (ECD order linkage and OAuth secrets) and relies on other domains for order storage and fulfillment.

## Project Structure

- `src/server.ts` – bootstraps the app.
- `src/App.ts` – DI container, configuration loading, endpoint wiring, email/slack setup.
- `src/controller/` – `EcarePuurNotificationController`, `ZspNotificationController`.
- `src/service/`  
  - `ecare/` – auth, API, mapping, team resolution, error messages.  
  - `zsp/` – mapping and orchestration for ZSP flows.  
  - `WonkersTaasOrderService`, `OrderStatusService`, `PhoneNumberService`.
- `src/repository/` – `ClientIdRepository` (ecd_order table), `EcdOAuthRepository` (ecd_oauth table).
- `src/model/` – DTOs for notifications, config, email, address/client representations.
- `config/` – default and env-mapped settings for DB, APIs, Kong, SMTP, Slack, email.
- `test/` – mocha/nyc tests covering controllers, services, repositories, and mapping utils plus fixtures.

## Controllers & Public Surface

- `POST /ext/v1/ecd/puur/notify`  
  - Middlewares: Kong header auth + body validation (`PuurNotificationDto`).  
  - Handler: `EcarePuurNotificationController.notify` -> delegates to EcarePuurService. Returns `200` with `orderId` or propagates mapped errors; 500s log UUID and notify Slack.
- `POST /ext/v1/ecd/zsp/notify`  
  - Middlewares: Kong header auth + body validation (`ZSPPuurNotificationDto`).  
  - Handler: `ZspNotificationController.notify` -> delegates to ZspService. Returns `204` on success; mapped errors or generic 500.

## Core Services & Logic

- **EcarePuurService**  
  - Switches on `Type` (`Aanmeldbericht` subscribe vs `Afmeldbericht` unsubscribe).  
  - Subscribe: maps form to `SubscribeFields`, fetches requester/contact from Ecare Employee API when missing, fetches client and care teams, builds `OrderV2Dto`, places order via WonkersTaasOrderService, stores client-order link in `ecd_order`.  
  - Unsubscribe: maps `UnsubscribeFields`, fetches returner contact if absent, resolves order IDs from `ecd_order`, validates status via OrderStatusService, deletes cancellable orders or issues return via WonkersTaasOrderService.
- **ZspService**  
  - Subscribe: maps payload, derives requester from embedded employee, extracts client/address/careteam from embedded patient data, creates concept order + form via `ConceptService.createConceptOrder`.  
  - Unsubscribe: maps return data, optionally enriches returner from employee, builds concept return form, calls `ConceptService.createConceptReturn`.
- **Mapping & Validation**  
  - `EcarePuurMappingService` and `ZspMappingService` convert AdditionalFields to typed DTOs, enforce phone/address formats, enforce optional/required logic, truncate long notes, select client identifier strategy based on relation lists.  
  - `PhoneNumberService` normalizes/validates numbers; shared error messages in `EcarePuurFieldErrors`.
- **Order & Status Helpers**  
  - `WonkersTaasOrderService` wraps HTTP calls for place, fetch, return, and delete order operations (internal endpoints).  
  - `OrderStatusService` guards against duplicate/invalid returns; classifies deletable vs returnable orders.
- **Repositories & Auth**  
  - `EcdOAuthRepository` reads OAuth client credentials per integration for Ecare token exchange.  
  - `ClientIdRepository` persists client/order mappings for follow-up lookups.

## External Dependencies & Cross-Service Contracts

- **Wonkers TaaS Orders (internal)** – `wonkersTaasOrderAddress` for POST `/internal/v4/taas-orders`, GET `/internal/v1/taas-orders`, DELETE returns. Expects TinyBots order DTOs.  
- **Ecare Puur** – Token endpoint `/token` using client credentials from DB; Patient API `/fhir/r3/patient/{id}` and careteam; Employee API `/employees/{id}/teams` and `/HealthProfessionals/{id}` with `Version` header.  
- **Concept TaaS Orders** – `tb-concept-taas-orders` client for concept order/return creation and email/slack helpers.  
- **Kong Gateway** – Header validation (`KongHeader`) and integration user resolution via `tiny-backend-tools`.  
- **MySQL** – `ecd_order` and `ecd_oauth` tables via `tiny-backend-tools` `Database`. Connection from `mysql` config (host/user/db/env overrides).  
- **SMTP** – `SmtpConfig` for mailer; selects email config by `environment` (`dev`, `academy`, `production`).  
- **Slack** – Error notifications via `tb-ts-slack-notification`; hooks provided in config (`slack`, `conceptSlackConfig`).  
- **FHIR parser** – `@smile-cdr/fhirts` used to read patient data for addresses and teams.  
- **Other libs** – `awilix` DI, `axios` HTTP, `class-validator/transformer` for DTO validation, `uuid` for error correlation.
