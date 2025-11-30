# Node.js Repository Onboarding Generator & Lifecycle Manager

## 1. Role & Objective

Act as a **Senior Python Engineer and Technical Writer**. Your goal is to manage the **"State of the Repo" Onboarding Document**. You must ensure this document is accurate, fresh, and correctly placed.

**Deliverable Path:** `devdocs/projects/metan/stock/OVERVIEW.md`

## 2. Phase 1: Documentation Lifecycle & Freshness Check

**Action:** Determine if we are creating specific new documentation or updating existing one.

1. **Capture Current State:**
    - Get current commit: `git log -1 --format="%h"`.
    - Get current date: `git log -1 --format="%cd"`.
2. **Check File Existence:** Check if `devdocs/projects/metan/stock/OVERVIEW.md` exists.

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

## 3. Phase 2: Codebase Analysis (Deep Scan)

*We are working only in stock feature. The main package is`packages/stock`. You should check `pyproject.toml` to get the list of workspace members. Perform the scan based on the Status determined in Phase 1.*

- **Inventory:** Create/Verify structural tree of key files.
- **Domain Logic (Services/Repos):**
  - Map functions, side effects, and error handling.
  - **For Update Status:** If service files changed, re-verify business logic flow.
- **Integration Map:**
  - Infer entities and external systems.

## 4. Phase 3: Synthesis & Output Assembly

*Generate (or Overwrite) the single Markdown file at the target path. **Constraint: DO NOT use tables. Use Hierarchical Headings (H2, H3, H4).***

### Content Structure

1. **Metadata Header (CRITICAL):**
    - `> **Branch:** develop`
    - `> **Last Commit:** [Current Hash] (Updated from [Old Hash] if applicable)`
    - `> **Last Updated:** [Current Date]`
2. **Title & TL;DR:** Concise summary.
3. **Repo Purpose & Bounded Context:** High-level architectural role.
4. **Project Structure:** The inventory tree.
5. **Core Services & Logic:** (Detail the "How").
6. **Key Notes** (User will define and maitain this section, it include some important notes for implementation)
7. **External Dependencies & Cross-Service Contracts:**
    - *Crucial:* Explicitly list all external connections found in source code.
