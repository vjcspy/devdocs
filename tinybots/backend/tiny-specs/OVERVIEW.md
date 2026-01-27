# tiny-specs

## Purpose

**tiny-specs** is the centralized OpenAPI specification repository for TinyBots backend services. It generates TypeScript types and runtime validators from OpenAPI schemas, ensuring type consistency between backend services and frontend applications.

## Key Capabilities

| Capability | Description |
|------------|-------------|
| **OpenAPI Schemas** | Centralized API contract definitions for all TinyBots services |
| **TypeScript Types** | Auto-generated type definitions from OpenAPI specs |
| **Runtime Validators** | JSON schema validators for request/response validation |
| **Frontend Integration** | Shared types consumed by webapp and dashboard frontends |

## Architecture

### Build Targets

tiny-specs generates two separate bundles:

| Target | Purpose | Services Included |
|--------|---------|-------------------|
| **webapp** | Types for webapp frontend | checkpoint, megazord-events, micro-manager, prowl-user-account |
| **dashboard** | Types for dashboard frontend | wonkers-buy-subscriptions, wonkers-relations, wonkers-robots, wonkers-taas, wonkers-overview |

### Directory Structure

```
tiny-specs/
├── specs/local/                      # OpenAPI source files
│   ├── components/                   # Schema definitions
│   │   ├── common/                   # Shared schemas (errors, security)
│   │   └── <service>/v<version>/     # Service-specific schemas
│   │       └── schemas.yaml
│   ├── paths/                        # API path definitions
│   │   └── <service>/v<version>/
│   │       └── paths.yaml
│   └── <service>-main.yaml           # Service entry point
├── src/                              # TypeScript source (parser, generator)
│   └── main.ts                       # Build orchestration
├── dist/                             # Generated output
│   ├── webapp/
│   │   ├── typings/                  # TypeScript type definitions
│   │   └── validators/               # Runtime validators
│   └── dashboard/
│       ├── typings/
│       └── validators/
└── generated/                        # Intermediate validator output
```

### Adding New Schemas

To add schemas for a service (e.g., `micro-manager` v6):

1. **Create/update schemas:** `specs/local/components/micro-manager/v6/schemas.yaml`
2. **Create/update paths:** `specs/local/paths/micro-manager/v6/paths.yaml`
3. **Update main file:** `specs/local/micro-manager-main.yaml` (reference new paths)
4. **Build:** `yarn run all`

## Usage in Dependent Repositories

### Installing tiny-specs

```json
// package.json
{
  "dependencies": {
    "tiny-specs": "git+ssh://git@bitbucket.org/tinybots/tiny-specs.git#master"
  }
}
```

### Importing Types

```typescript
// Import from webapp bundle
import type { 
  ExecutionDetailResponse,
  TriggeredExecutionSummary 
} from 'tiny-specs/dist/webapp/typings'

// Import from dashboard bundle
import type { 
  AdminOverview,
  TessaOwnerRobotV4 
} from 'tiny-specs/dist/dashboard/typings'
```

### Using Validators

```typescript
import { validateExecutionDetailResponse } from 'tiny-specs/dist/webapp/validators'

const isValid = validateExecutionDetailResponse(data)
```

## Development Workflow

### Local Development

```bash
cd tinybots/backend/tiny-specs

# Install dependencies
yarn install

# Build everything (typings + validators)
yarn run all

# Build only typings
yarn run build

# Validate OpenAPI specs
yarn run validate
```

### Testing Changes in Dependent Repo

When developing new schemas, use a feature branch:

```bash
# In tiny-specs
git checkout -b feature/micro-manager-new-api
# ... make changes ...
yarn run all
git commit && git push -u origin feature/micro-manager-new-api

# In dependent repo (e.g., micro-manager)
# Update package.json to reference the branch:
"tiny-specs": "git+ssh://git@bitbucket.org/tinybots/tiny-specs.git#feature/micro-manager-new-api"
yarn install
```

## AI Agent Command

**IMPORTANT:** When updating tiny-specs, use the dedicated command:

```
devdocs/agent/commands/tinybots/update-tiny-specs.md
```

This command provides:
- Pre-flight checks (clean working directory, correct branch)
- Step-by-step workflow for schema changes
- Integration steps for dependent repositories
- Error handling guidance

### When to Use the Command

| Scenario | Action |
|----------|--------|
| Adding new API schemas | Use `update-tiny-specs` command |
| Adding response types for frontend | Use `update-tiny-specs` command |
| Fixing schema validation issues | Use `update-tiny-specs` command |
| Just reading/understanding specs | Read files directly |

## Common Patterns

### Date/Time Fields

All datetime fields should be ISO8601 strings:

```yaml
executedAt:
  type: string
  format: date-time
  description: ISO8601 datetime
  example: "2026-01-06T09:15:00.000Z"
```

### Nested Response Objects

Use `$ref` for reusable components:

```yaml
TriggeredExecutionSummary:
  type: object
  properties:
    script:
      $ref: '#/components/schemas/ScriptInfo'
    trigger:
      $ref: '#/components/schemas/TriggerInfo'
```

### Error Responses

Reference common error schemas:

```yaml
responses:
  '404':
    description: Not found
    content:
      application/json:
        schema:
          $ref: '../../../components/common/schemas.yaml#/components/schemas/NotFoundError'
```

## Dependencies

| Service | Uses tiny-specs For |
|---------|---------------------|
| micro-manager | Script execution DTOs |
| checkpoint | Robot pairing DTOs |
| prowl | User account DTOs |
| megazord-events | Event subscription DTOs |
| wonkers-* | Dashboard DTOs |

## Related Resources

- **Source Code:** `tinybots/backend/tiny-specs/`
- **AI Command:** `devdocs/agent/commands/tinybots/update-tiny-specs.md`
- **Build Output:** `tiny-specs/dist/`
