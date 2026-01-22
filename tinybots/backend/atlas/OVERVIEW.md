# Atlas Overview

## TL;DR

- Atlas is a batch-job service that copies and anonymises robot script data from the typ-e database to the intelligence database.
- Uses Python-based NLP tools (Flair, spaCy, pii-filter) to detect and anonymise personally identifiable information (PII) in Dutch text.
- Runs scheduled batch jobs to migrate scripts, execution data, status checks, and robot references while preserving referential integrity.
- Critical for maintaining GDPR compliance by anonymising sensitive script content and customer conversations.

## Service Purpose & Responsibilities

Atlas bridges two databases by copying robot automation script data from typ-e (the operational database) into intelligence (the analytics/reporting database). Its primary mission is to anonymise PII before persisting data into intelligence, ensuring compliance with data protection regulations while maintaining data utility for analytics.

Key responsibilities:
- **Data Migration**: Incrementally copy new script-related records from typ-e to intelligence in chronological order.
- **PII Anonymisation**: Detect and replace names, email addresses, phone numbers, medicine names, and other sensitive data in script text using multi-layered NLP taggers.
- **UUID Generation**: Create anonymised UUIDs for deleted robots to maintain referential integrity without exposing real robot identifiers.
- **Batch Orchestration**: Execute copy, delete, and prune operations via scheduled jobs or manual invocation.

## Architecture & Data Flow

### Databases

- **typ-e Database**: Source database containing live robot scripts, executions, and operational data.
- **intelligence Database**: Target database for anonymised analytics data consumed by reporting and machine learning systems.

### Data Migration Flow

```
typ-e DB (source)
    |
    v
[Atlas Batch Jobs]
    |
    +---> Generate Anonymised UUIDs (for deleted robots)
    |
    +---> Copy Scripts V1/V2 (copy script_robot, script_reference, script_version, script_step, etc.)
    |        |
    |        +---> Anonymise Text (script names, TTS text, ASR answers)
    |
    +---> Delete Scripts V1/V2 (remove old archived data)
    |
    +---> Prune UUIDs (cleanup unused anonymised UUIDs)
    |
    v
intelligence DB (target)
```

### Anonymisation Pipeline

1. **Capitalizer**: Uses spaCy to normalize capitalization for better NER detection.
2. **PII Filter Tagger**: Rule-based detection of Dutch names, emails, phone numbers, and medicine names.
3. **spaCy Tagger**: Named entity recognition for persons, locations, and organizations using `nl_core_news_lg` model.
4. **Flair Tagger**: Deep learning NER using Flair embeddings for additional entity detection.
5. **Combined Anonymiser**: Merges results from all taggers, prunes overlapping tags, and replaces detected entities with `***`.

## Key Components

### Batch Jobs (`src/batchJobs/`)

- **GenerateUUIDsV1/V2**: Generate anonymised UUIDs for deleted robots to preserve foreign key relationships.
- **CopyScriptsV1**: Copy script_robot-based scripts (older schema version).
- **CopyScriptsV2**: Copy script_reference-based scripts (newer schema version).
- **DeleteScriptsV1/V2**: Remove archived scripts older than retention policy.
- **PruneUUIDs**: Clean up unused anonymised UUIDs from typ-e.

### Repositories

- **CopyTablesRepository**: Copies non-anonymised tables (script_category, script_execution, script_step, status_check_description, report_type).
- **CopyAnonymisedTablesRepository**: Copies tables requiring anonymisation (script_version, script_node_tts, script, script_node_asr).
- **DeleteTablesV1/V2Repository**: Deletes archived data from intelligence based on archive timestamps.
- **HighestIdIntelligenceRepository**: Tracks the highest ID copied for each table to enable incremental batch processing.

### Anonymisers (`src/anonymiser/`)

- **Tagger**: Abstract base class for all PII detection strategies.
- **PIIFilterTagger**: Uses `pii-filter` library for Dutch PII detection (names, emails, phones, medicines).
- **PythonTagger** (Flair/spaCy): Wraps Python NLP models via `python-shell` for advanced NER.
- **CombinedAnonymiser**: Orchestrates multiple taggers and merges their results.

## Dependencies & Integrations

### Internal Services
- **typ-e**: Source database schema for robot operational data.
- **intelligence**: Target database schema for anonymised analytics data (schema not in this monorepo).

### External Libraries
- **Python NLP Stack**: Flair, spaCy (nl_core_news_lg), pii-filter for Dutch language processing.
- **mysql2**: MySQL database connections.
- **python-shell**: Bridge between Node.js and Python scripts for NLP processing.
- **awilix**: Dependency injection container.
- **class-validator**: DTO validation.

## Configuration

Configuration files in `config/`:
- `default.json`: Default database connection settings for typ-e and intelligence.
- `custom-environment-variables.json`: Environment variable mappings for deployment.

Key environment variables:
- `MYSQL_HOST_TYPE`, `MYSQL_PORT_TYPE`, `MYSQL_USERNAME_TYPE`, `MYSQL_PASSWORD_TYPE`, `MYSQL_DATABASE_TYPE`: typ-e database credentials.
- `MYSQL_HOST_INTELLIGENCE`, `MYSQL_PORT_INTELLIGENCE`, etc.: intelligence database credentials.

## Operational Notes

### Running Batch Jobs

Batch jobs are invoked via npm scripts:
```bash
yarn generate-uuid-deleted-v1    # Generate UUIDs for deleted robots (V1)
yarn generate-uuid-deleted-v2    # Generate UUIDs for deleted robots (V2)
yarn copy-scripts-v1              # Copy scripts from typ-e to intelligence (V1)
yarn copy-scripts-v2              # Copy scripts from typ-e to intelligence (V2)
yarn delete-scripts-v1            # Delete archived scripts (V1)
yarn delete-scripts-v2            # Delete archived scripts (V2)
yarn prune-uuids                  # Cleanup unused anonymised UUIDs
```

### Deployment

- **Dockerfile**: Uses `nikolaik/python-nodejs:python3.8-nodejs20-slim` base image.
- **Python Setup**: Installs dependencies from `requirements.txt`, downloads spaCy `nl_core_news_lg` model, and loads Flair embeddings.
- **Port**: Exposes port 3000 (though primarily a batch job runner, not an HTTP service).

### Testing

- **Unit Tests**: `yarn unit-test` (Mocha + nyc for coverage, 90% threshold for statements/functions/lines).
- **Performance Tests**: `yarn performance-test` (timeout 80s).
- Test structure mirrors `src/` in `test/` directory.

### Data Retention & Compliance

- Scripts are archived in typ-e and deleted from intelligence after retention period.
- Anonymisation ensures GDPR compliance by removing PII from customer conversations and robot scripts.
- Anonymised UUIDs maintain referential integrity without exposing real robot identifiers.

## Key Data Models

### Tables Copied (typ-e → intelligence)

**Non-Anonymised:**
- `script_robot`, `script_reference`: Robot identifiers (anonymised UUID only).
- `script_category`: Script categorization.
- `script_execution`, `script_step`, `script_next_goto`, `script_next_then`: Execution flow metadata.
- `script_node_goto`, `script_node_report`, `script_node_status_check`: Node-level metadata.
- `status_check_description`, `report_type`: Lookup tables.

**Anonymised:**
- `script_version`: Script name anonymised.
- `script`: Script name and command anonymised.
- `script_node_tts`: Text-to-speech content anonymised.
- `script_node_asr`: Automatic speech recognition answers anonymised (yes/no/other/silence).

### Tag Model (`src/model/Tag.ts`)

Represents a detected PII entity:
- `type`: Entity type (e.g., `first_name`, `email_address`, `PER`, `LOC`).
- `start`, `end`: Character position in text.

## Known Limitations & Future Considerations

- **Dutch Language Only**: NLP models and pii-filter are configured for Dutch (`nl_core_news_lg`, Dutch language model).
- **Manual Job Invocation**: Batch jobs are not scheduled within Atlas; must be triggered externally (cron, orchestrator).
- **No HTTP API**: Atlas is purely a batch job runner, not a REST service despite exposing port 3000.
- **Python Dependency**: Requires Python runtime and large NLP models (~500MB+ for spaCy + Flair), increasing Docker image size.
- **Schema Version Dual Support**: Maintains V1 (script_robot) and V2 (script_reference) migration paths for legacy and new schemas.

## Troubleshooting

### Common Issues

1. **Python Script Hangs**: Check `python-shell` process lifecycle; ensure spaCy/Flair models are loaded correctly.
2. **Missing Anonymised UUIDs**: Run `generate-uuid-deleted-v1/v2` before copy jobs.
3. **Incremental Copy Failures**: Verify `HighestIdIntelligenceRepository` state; highest IDs may be out of sync.
4. **Coverage Failures**: Unit test coverage thresholds are high (90%); ensure all repository methods are tested.

### Logs

- Console logging via `console.info` and `console.error`.
- No structured logging framework; consider adding `winston` or `pino` for production observability.

## Related Documentation

- Global TinyBots overview: `devdocs/tinybots/OVERVIEW.md`
- typ-e database schema: `devdocs/tinybots/typ-e/OVERVIEW.md`
- Intelligence database: (external schema, not documented in this monorepo)
