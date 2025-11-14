Act like a senior backend Node.js engineer and technical writer.

Goal: Scan a Node.js backend repository and generate a crisp, on-boarding-friendly Markdown brief that lets a new AI agent/developer understand and contribute quickly.

Inputs you will receive: Current repository folder. Assume key folders exist: src (source code), test (unit/integration tests). Integration tests in controllers often use the “*IT” suffix.

Task: Analyze the codebase and produce a single Markdown file ready to commit into the repo.

Steps:
1) Inventory: Build a structure file tree (just define important files, folders, and its purpose)
2) Controllers: List all exposed controllers/routes. For each: HTTP method(s), path(s), request/response schema, core middleware/guards, key dependencies (services, repositories), and cross-cutting concerns (auth, validation, tracing).
3) Services and Repositories Layer: Identify important services. For each: purpose, main functions, inputs/outputs, side effects (DB, cache, queue, HTTP calls), idempotency/retries, and error taxonomy.
4) Tests: Read test/ thoroughly. Prioritize controller *IT tests. Extract workflows covered, fixtures, mocks, happy-path vs edge cases, and what “green” implies functionally.
5) Data & Integration Map: Infer domain entities, schemas, and relations. Map external systems (databases, message brokers, third-party APIs, internal repos). For each integration: protocol, endpoints/topics, auth, timeouts/retries, and where called from.
6) Purpose Synthesis: State the repo’s primary mission, bounded context, and how it interacts with other repos/services (directionality and contracts).
7) Gaps & Risks: List TODOs, missing tests, dead code, secrets risk, unclear ownership.
8) Output Assembly:
   - Title, TL;DR, Table of Contents
   - “Repo Purpose & Interactions” (narrative + integration table)
   - “Controllers / Public Surface”
   - “Key Services/Repository & Logic”
   - Runtime / Request Flow
   - External Dependencies & Cross-Service Contracts: Analyze the source code and identify any dependencies or connections to external repositories
   - Notes (Important things)

Deliverable: Update docs to OVERVIEW.md. Take a deep breath and work on this problem step-by-step. You can always define new terms and concepts if needed.