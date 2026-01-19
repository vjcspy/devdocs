# Create Overview

## Role & Objective

Act as a **Senior Polyglot Software Architect** and **Technical Writer**.
Your goal is to generate or update the "State of the Repo" Onboarding Document (Context Overview).
This document serves as the primary context source for other AI Agents or Developers working on this repository.

**Input Variables:**

- `TARGET_PATH`: [User Defined Path, e.g., devdocs/services/payment/OVERVIEW.md]

---

## Phase 0: Environment & Safety Validation (CRITICAL)

**Action:** Verify execution environment constraints before scanning code.

1. **Get Current Context:**
    - Current Branch: Execute `git branch --show-current` -> Store as `{CURRENT_BRANCH}`.
    - Current Hash: Execute `git log -1 --format="%h"` -> Store as `{CURRENT_HASH}`.
    - Current Date: Execute `git log -1 --format="%cd"` -> Store as `{CURRENT_DATE}`.

2. **Branch Allow-list Check:**
    - IF `{CURRENT_BRANCH}` is NOT `develop` OR `master`:
      - **STOP IMMEDIATELY.**
      - Output: "⚠️ **Aborted:** Documentation can only be generated from 'develop' or 'master'. Current branch is '{CURRENT_BRANCH}'."

3. **Existing File Consistency Check:**
    - Check if `{TARGET_PATH}` exists.
    - **Scenario A (File Exists):**
      - Read the file content. Look for the metadata line: `> **Branch:** [Old_Branch_Name]`.
      - **Consistency Rule:** IF `[Old_Branch_Name]` != `{CURRENT_BRANCH}`:
        - **STOP IMMEDIATELY.**
        - Output: "⚠️ **Aborted:** Branch Mismatch. The existing documentation belongs to branch '{Old_Branch_Name}', but you are currently on '{CURRENT_BRANCH}'. Please handle the merge or delete the old file manually."
    - **Scenario B (File Missing):**
      - Proceed to Phase 1 (Status: NEW GENERATION).

---

## Phase 1: Lifecycle & Freshness Check

**Action:** Determine if an update is strictly necessary based on Git history.

1. **Parse Existing Metadata (If file exists):**
    - Extract `> **Last Commit:** [Old_Hash]` from `{TARGET_PATH}`.

2. **Compare Hashes:**
    - IF `[Old_Hash]` == `{CURRENT_HASH}`:
      - **STOP.** Output: "✅ Documentation is up to date. No changes detected."
    - IF `[Old_Hash]` != `{CURRENT_HASH}`:
      - Set Status: **[UPDATE REQUIRED]**
      - **Delta Analysis:** Execute `git diff --name-only [Old_Hash] HEAD`.
      - **Instruction:** Keep this file list. You must analyze these specific files to write the "Recent Changes Log". Focus on *structural* changes (new modules, renamed services, changed signatures), not just trivial code edits.

---

## Phase 2: Architectural Deep Scan (Language-Agnostic)

**Action:** Scan the codebase to identify architectural patterns. Auto-detect the language (Java/Node/Python/etc.) to use correct terminology, but map them to the following universal concepts:

1. **Project Structure:**
    - Map the directory tree. Identify where logic, configs, and tests live.

2. **Public Surface (Entry Points):**
    - *For API Repos:* Identify Controllers, Route Handlers, gRPC definitions, or GraphQL Resolvers.
    - *For CLI/Scripts:* Identify main execution entry points and arguments.
    - *For Workers:* Identify Event Listeners (Kafka/RabbitMQ consumers) or Cron jobs.

3. **Core Services & Domain Logic:**
    - Identify the "How". Where is the business logic isolated? (e.g., Service classes in Java/NestJS, Use Cases in Clean Arch, utils/logic folders in Python).
    - Map data flow: Entry Point -> Service -> Data Access.

4. **External Dependencies (Crucial):**
    - **Scan imports and config files (package.json, pom.xml, requirements.txt, .env.example)**.
    - List ALL external systems the code communicates with:
      - Databases (Postgres, Mongo, Redis).
      - Message Brokers (SQS, Kafka, RabbitMQ).
      - Downsteam APIs(External APIs like: 3rd party vendors, other internal microservices).

---

## Phase 3: Synthesis & Output Generation

**Action:** Overwrite `{TARGET_PATH}` with the content below.
**Constraint:** DO NOT use tables. Use Hierarchical Headings (H2, H3, H4) and Bullet points.

### Output Template

```markdown
## Metadata Header
> **Branch:** {CURRENT_BRANCH}
> **Last Commit:** {CURRENT_HASH} (Updated from [Old_Hash] if applicable)
> **Last Updated:** {CURRENT_DATE}

## Title & TL;DR
[A concise summary of what this repository does in 1-2 sentences]

## Recent Changes Log (Only if Updating)
[Based on the git diff analysis, briefly explain STRUCTURAL changes. e.g., "Added PaymentService", "Modified User Schema", "Refactored Auth Controller". If New Generation, leave blank or state "Initial Documentation".]

## Repo Purpose & Bounded Context
- **Role:** [High-level architectural role, e.g., "Manages Order Lifecycle"]
- **Domain:** [The specific business domain it belongs to]

## Project Structure
[Tree view or bullet list of KEY top-level directories and their purposes]

## Controllers & Public Surface (Inbound)
[Group by functionality. List endpoints/commands exposed to the world]
- **[Group Name]:**
  - [Description of key endpoints/commands]

## Core Services & Logic (Internal)
[Detail the "How". List key services/modules and their responsibilities]
- **[Service Name]:** [What logic does it handle?]

## External Dependencies & Cross-Service Contracts (Outbound)
**Crucial:** Explicitly list all external connections found in source code.
- **Databases:** [List DBs]
- **Message Queues:** [List Queues/Topics produced or consumed]
- **External APIs:** [List downstream services called]
```
