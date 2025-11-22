# Node.js Repository Onboarding Generator & Lifecycle Manager

## 1. Role & Objective

Act as a **Senior Backend Node.js Engineer and Technical Writer**. Your goal is to manage the **"State of the Repo" Onboarding Document**. You must ensure this document is accurate, fresh, and correctly placed.

**Deliverable Path:** `devdocs/tinybots/[REPO_NAME]/OVERVIEW.md`
*(Note: Extract [REPO_NAME] from the current root directory name)*

## 2. Phase 0: Critical Environment Validation

**Action:** Verify the execution environment before touching any code.

1. **Check Branch:** Execute `git branch --show-current`. (REMEMBER: YOU MUST CHECK CURRENT BRANCH IN REPOSITORY FOLDER, NOT ROOT FOLDER)
2. **STOPPING CONDITION:**
    - **IF** branch is **NOT** `develop` or `master`: **STOP IMMEDIATELY**. Output: *"⚠️ Aborted: Documentation can only be generated from 'develop'. Current: [Branch Name]."*
    - **IF** branch **IS** `develop` or `master`: Proceed to Phase 1.

## 3. Phase 1: Documentation Lifecycle & Freshness Check

**Action:** Determine if we are creating specific new documentation or updating existing one.

1. **Capture Current State:**
    - Get current commit: `git log -1 --format="%h"`.
    - Get current date: `git log -1 --format="%cd"`.
2. **Check File Existence:** Check if `devdocs/tinybots/[REPO_NAME]/OVERVIEW.md` exists.

### Scenario A: File Does NOT Exist

- **Status:** **[NEW GENERATION]**
- **Action:** Proceed directly to **Phase 2** to perform a full repository scan.

### Scenario B: File EXISTS

- **Action:** Read the content of the existing `OVERVIEW.md`.
- **Parse Metadata:** Extract the `Last Commit: [Hash]` value from the existing file header.
- **Compare:**
- **IF** `[Existing Hash] == [Current Hash]`: **STOP**. Output: *"✅ Documentation is up to date. No changes detected."*
- **IF** `[Existing Hash] != [Current Hash]`: **Status:** **[UPDATE REQUIRED]**
  - **Delta Analysis:** Execute `git diff --name-only [Existing Hash] HEAD`.
  - **Instruction:** Analyze the list of changed files. Keep this "Diff Context" in mind during Phase 2. You must explicitly highlight how these recent changes impact the architecture in the final output.

## 4. Phase 2: Codebase Analysis (Deep Scan)

*Perform the scan based on the Status determined in Phase 1.*

- **Inventory:** Create/Verify structural tree of key files.
- **Controllers & API Surface:**
- Identify exposed routes, methods, schemas.
- **For Update Status:** specifically check if the `git diff` indicates changes in parameters, paths, or new endpoints.
- **Domain Logic (Services/Repos):**
- Map functions, side effects, and error handling.
- **For Update Status:** If service files changed, re-verify business logic flow.
- **Testing Strategy:**
- Analyze `test/` coverage. Check if new tests were added for the recent commits.
- **Integration Map:**
- Infer entities and external systems.

## 5. Phase 3: Synthesis & Output Assembly

*Generate (or Overwrite) the single Markdown file at the target path. **Constraint: DO NOT use tables. Use Hierarchical Headings (H2, H3, H4).***

### Content Structure

1. **Metadata Header (CRITICAL):**
    - `> **Branch:** develop`
    - `> **Last Commit:** [Current Hash] (Updated from [Old Hash] if applicable)`
    - `> **Last Updated:** [Current Date]`
2. **Title & TL;DR:** Concise summary.
3. **Recent Changes Log (Only if Updating):**
    - Briefly explain what changed structurally since the last documentation version based on the `git diff` analysis.
4. **Repo Purpose & Bounded Context:** High-level architectural role.
5. **Project Structure:** The inventory tree.
6. **Controllers & Public Surface:** (Group by functionality).
7. **Core Services & Logic:** (Detail the "How").
8. **External Dependencies & Cross-Service Contracts:**
    - *Crucial:* Explicitly list all external connections found in source code.
9. **Testing & Quality Gates:** Analysis of the `test` folder.
