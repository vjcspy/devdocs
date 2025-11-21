# Node.js Repository Onboarding Generator

## 1. Role & Objective

Act as a **Senior Backend Node.js Engineer and Technical Writer**. Your goal is to reverse-engineer the provided Node.js repository and generate a **"State of the Repo" Onboarding Document**. This document must allow a new AI agent or developer to immediately understand the architecture, data flow, and contribution guidelines.

## 2. Phase 0: Critical Environment Validation (Git Check)

**You must perform this step FIRST. Do not proceed to analysis until this passes.**

1. **Check Branch:** Execute a git command to verify the current branch.
   - *Command logic:* Check if current branch is `develop` (You must check repository branch, not root folder)
2. **STOPPING CONDITION:**
   - **IF** the branch is **NOT** `develop`: **STOP IMMEDIATELY**. Output a warning message: *"⚠️ Aborted: Repository must be on 'develop' branch to generate an official Overview. Current branch is [Branch Name]."*
   - **IF** the branch **IS** `develop`: Proceed to step 3.
3. **Capture Metadata:**
   - Retrieve the last commit hash and date (e.g., `git log -1 --format="%h - %cd"`).
   - **Requirement:** This information must be displayed at the very top of the final output to certify "Freshness".

## 3. Phase 1: Codebase Analysis (Deep Scan)

*Scan only `src`, `test` to extract the following:*

- **Inventory:** Create a structural tree of key files/folders and define their purpose.
- **Controllers & API Surface:**
  - Identify exposed routes, HTTP methods, and schemas.
  - Extract core middleware (Guards, Auth), and key dependencies.
- **Domain Logic (Services/Repos):**
  - Map main functions, inputs/outputs, and side effects (DB, Cache, Queue, HTTP).
  - Identify idempotency logic, retry policies, and error handling patterns.
- **Testing Strategy:**
  - Deep dive into `test/`, prioritizing integration tests (`*IT`).
  - Extract workflows covered, fixtures used, and the definition of "Green" (what constitutes a success).
- **Integration Map:**
  - Infer domain entities and relations.
  - Map **all** external systems (DBs, Brokers, 3rd Party APIs, Internal Services). Note protocols, auth methods, and timeouts.

## 4. Phase 2: Synthesis & Insight

- **Purpose:** Define the repo's bounded context and primary mission.
- **Gap Analysis:** Identify TODOs, missing tests, dead code, potential security risks (secrets), or ambiguous ownership.

## 5. Phase 3: Output Assembly (The Deliverable)

*Generate a single Markdown file using the exact structure below. **Constraint: DO NOT use tables. Use Hierarchical Headings (H2, H3, H4) for all data representation.***

### Structure

1. **Metadata Header:**
   - `> **Branch:** develop`
   - `> **Last Commit:** [Hash] on [Date]`
2. **Title & TL;DR:** Concise summary.
3. **Repo Purpose & Bounded Context:** High-level architectural role.
4. **Project Structure:** The inventory tree.
5. **Controllers & Public Surface:** (Group by functionality).
6. **Core Services & Logic:** (Detail the "How").
7. **External Dependencies & Cross-Service Contracts:**
   - *Crucial:* Explicitly list all external connections found in source code.
8. **Testing & Quality Gates:** Analysis of the `test` folder.
