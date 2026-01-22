> **Branch:** develop
> **Last Commit:** (refresh as needed)
> **Last Updated:** Thu Jan 15 2026 +0000

# wadsworth Overview

## TL;DR
- Speech interaction service managing voice commands for TinyBots robots, built on `TinyDatabaseAppAuthenticated` with Awilix wiring.
- Stores robot-specific and default (hardware-type-scoped) speech interactions linking voice commands to script references in MySQL `tinybots` database.
- Exposes REST APIs for robots to retrieve matching commands, users to manage custom interactions, and internal services to bulk-manage interactions by script reference.
- Integrates with micro-manager for script validation and dashboard robot service for hardware type resolution.

## Recent Changes Log
- Initial overview document created to capture existing speech interaction architecture and API surface.

## Repo Purpose & Bounded Context
- Provides the voice command layer for TinyBots robots: users configure "speech interactions" that map spoken phrases (commands) to executable scripts.
- Supports two interaction scopes: **default interactions** (pre-configured per hardware type) and **robot-specific interactions** (user-customized per robot).
- Acts as the bridge between robot voice recognition and script execution by resolving spoken commands to script references consumed by micro-manager.
- Enforces limits on max interactions per robot and max commands per interaction to prevent abuse.

## Project Structure
- `src/server.ts` — entry point; initializes App and starts the server.
- `src/App.ts` — extends `TinyDatabaseAppAuthenticated`; loads configs, registers Awilix container entries, configures middleware, and wires all endpoints.
- `src/controller/*` — HTTP layer:
  - `RobotInteractionController` — robot-facing endpoint for command matching (`/v1/speech-interactions/robot/self`).
  - `UserInteractionController` — user-facing CRUD for robot-specific interactions (`/v1/speech-interactions/:robotId/*`).
  - `DefaultInteractionController` — manage default interaction toggles per robot (`/v1/speech-interactions/:robotId/default/*`).
  - `InternalInteractionController` — internal endpoints for bulk operations and command retrieval (`/internal/v1/speech-interactions/*`).
  - `ErrorController` — centralized error handling.
- `src/service/*` — core business logic:
  - `InteractionService` — CRUD operations for robot-specific interactions with validation (similar command checks, max limits).
  - `DefaultInteractionService` — default interaction management with hardware type filtering and script support validation.
  - `ScriptService` — validates script references via micro-manager integration.
- `src/repository/*` — MySQL query layer:
  - `InteractionRepository` — queries for `script_speech_interaction` and `script_speech_interaction_command` tables.
  - `DefaultInteractionRepository` — queries for default interactions with hardware type and robot override logic.
  - `ScriptRepository` — script reference lookups.
  - `HardwareTypeRepository` — implements `IHardwareTypeProvider` for hardware type caching.
- `src/model/*` — DTOs, config models, and domain objects (`SpeechInteraction`, `RobotAccount`, `ScriptReference`).
- `config/*.json` — configuration defaults (MySQL, Kong, service addresses, limits).
- `docs/` — OpenAPI specs (`wadsworth.yaml` and resource definitions).
- `test/**/*` — integration tests for controllers, repositories, and services.

## Controllers & Public Surface
- `GET /v1/speech-interactions/robot/self?command=<phrase>` (RobotInteractionController) — robot auth required; returns matching enabled interactions (user or default) for the given command phrase.
- `GET /v1/speech-interactions/:robotId` (UserInteractionController) — Kong user auth + robot access validation; lists all interactions for a robot, optionally filtered by `scriptReferenceId`.
- `GET /v1/speech-interactions/:robotId/:speechInteractionId` (UserInteractionController) — retrieves a single interaction.
- `PUT /v1/speech-interactions/:robotId` (UserInteractionController) — creates or updates an interaction; validates `SpeechInteractionDto` body; enforces unique commands and max limits.
- `DELETE /v1/speech-interactions/:robotId/:speechInteractionId` (UserInteractionController) — removes a robot-specific interaction.
- `POST /v1/speech-interactions/:robotId/similar-commands` (UserInteractionController) — finds interactions with similar commands to prevent duplicates.
- `GET /v1/speech-interactions/:robotId/default` (DefaultInteractionController) — lists default interactions for a robot (filtered by hardware type and script support).
- `POST /v1/speech-interactions/:robotId/default/:speechInteractionId/toggle` (DefaultInteractionController) — enables/disables a default interaction for a specific robot.
- `PUT /internal/v1/speech-interactions` (InternalInteractionController) — internal endpoint to create/update interactions without user auth.
- `DELETE /internal/v1/speech-interactions` (InternalInteractionController) — bulk delete interactions by `robotId` and `scriptReferenceId`.
- `GET /internal/v1/speech-interactions/:robotId/commands` (InternalInteractionController) — retrieves all enabled commands for a robot (user + default).

## Core Services & Logic
- **InteractionService**: Manages robot-specific interactions; `createInteraction` validates no similar commands exist and enforces `maxInteractions` limit; `updateInteraction` handles command diff (add/remove) and enforces `maxCommands`; `getSimilarCommands` performs fuzzy matching by removing punctuation and whitespace.
- **DefaultInteractionService**: Manages default interactions scoped to hardware type; `getDefaultInteractions` filters by robot's hardware type and validates script support via micro-manager; `toggleDefaultInteraction` persists robot-specific enable/disable state; `getDefaultInteractionsByCommand` matches spoken phrases against default commands.
- **ScriptService**: Validates script references by calling micro-manager's `/internal/v3/scripts/robots/:robotId/scripts/default` to confirm scripts are supported for the given robot.

## Database Schema
Tables in `tinybots` database:
- `script_speech_interaction` — stores interaction metadata (id, title, script_reference_id, robot_id NULL for defaults).
- `script_speech_interaction_command` — stores voice commands linked to interactions (speech_interaction_id, command).
- `disabled_script_speech_interaction` — tracks robot-specific disables of default interactions (robot_id, script_speech_interaction_id).

Key constraints:
- `idx_robot_title` unique index ensures no duplicate titles per robot.
- Default interactions have `robot_id = NULL` in `script_speech_interaction`.

## External Dependencies & Cross-Service Contracts
- **MySQL (tinybots)**: Primary store; connection via `tiny-backend-tools` Database wrapper; transaction support for atomic command updates.
- **micro-manager**: Script validation calls to `/internal/v3/scripts/robots/:robotId/scripts/default` to filter supported scripts per robot.
- **Dashboard Robot Service**: `DashboardRobotService.getRobotsV5ByRobotAccountId` resolves robot hardware type for default interaction filtering.
- **Kong / Identity**: Robot and user auth via `KongHeader` validation; `robotValidator` for robot endpoints; `userRobotAccessValidator` for user endpoints.
- **tiny-backend-tools**: Provides `TinyDatabaseAppAuthenticated`, middleware (context, logging, serialization, error handling), validation, and Repository base class.
- **tiny-internal-services**: `DashboardRobotService` client and `HardwareTypeLoader` for hardware type caching.
- **tiny-types**: Shared DTOs (`SpeechInteractionDto`, `SpeechInteraction`, `ScriptVersion`).

## Configuration
From `config/default.json`:
- `maxInteractions: 50` — max custom interactions per robot.
- `maxCommands: 5` — max commands per interaction.
- `scriptsServiceAddress` — micro-manager URL for script validation.
- `dashboardRobotServiceAddress` — dashboard robot service URL.
- `mysql` — database connection (host, port, user `wadsworth-rw`, database `tinybots`).
- `kong` — Kong admin/API/service addresses.
- `appConfig.port: 8080` — HTTP server port.

## Testing & Quality Gates
- Test runner: `yarn test` uses mocha + ts-node with nyc coverage.
- Coverage thresholds: 85% statements, 90% functions, 65% branches, 85% lines (unit tests have stricter 90%+ gates).
- Integration tests in `test/controller/*IT.ts` exercise full request flows with DB fixtures.
- Repository and service tests validate query correctness and business logic.
- Uses `tiny-backend-testing-tools` for MySQL fixtures and test helpers.
