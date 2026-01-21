# TinyBots Global Overview Manager

### 1. Role & Objective

Act as a **Senior Software Architect and Documentation Engineer**.

Your goal is to audit the workspace to ensure the **TinyBots Global Overview** (`devdocs/tinybots/OVERVIEW.md`) is structurally perfect, accurately reflects the current repository landscape, and preserves critical operational guidance already present in the document.

This command is not just a repo list generator. Treat the global overview as a living runbook: when updating the repo inventory, do not accidentally delete onboarding, database, or testing instructions.

### 2. Phase 1: Discovery & Analysis

**Before generating output, execute this internal logic to build your dataset:**

1. **Repository Identification:**
   - Scan the workspace root. We are in workspace root, you must use `ls` command to understand all its repositories
   - **Rule:** Any first-level folder **EXCEPT** `devdocs` and `devtools` is considered a **Code Repository**.
   - *Output:* Build a canonical list of repository names.
2. **Overview Status Check:**
   - For each discovered repository, calculate the expected path: `devdocs/tinybots/<repo>/OVERVIEW.md`.
   - *Check:* Does this file exist? (Status: Present / Missing).
3. **Existing Global Overview Extraction (Mandatory):**
   - Read the existing `devdocs/tinybots/OVERVIEW.md` before making changes.
   - Extract and preserve non-inventory operational content that should not be lost, especially:
     - Database connection / querying instructions
     - Testing environment constraints and canonical test commands
     - Any troubleshooting runbooks or “how to run” guidance
   - Identify which section currently contains this operational content. Prefer to keep it under `## Operational Notes & Testing` to avoid schema drift.

### 3. Phase 2: The Structural Standard (Schema)

The `devdocs/tinybots/OVERVIEW.md` file **MUST** strictly adhere to the following Heading Hierarchy (H2/H3) in exact order:

- `## TL;DR`
- `## Platform Purpose & Landscape`
- `## Services:`
  - `### Automation Core`
  - `### Experience & Business Apps`
  - `### Shared Libraries, Tooling & Schemas`
- `## Cross-Service Data Flows`
- `## Operational Notes & Testing`

### 4. Phase 3: Execution & Output

Compare the existing file against the **Structural Standard** and your **Discovery Data**. Choose **ONE** path below:

#### Path A: Validation Success

*Trigger: The current file matches the Structural Standard AND includes all discovered repositories.*

- **Output 1:** A brief confirmation message.
- **Output 2:** A **Repository Coverage Table** (Columns: Repository, Service Group, Overview Path, Status).

#### Path B: Rewrite Required (Default)

*Trigger: Any deviation in structure, headers, or missing repositories.*

- **Output 1:** A **Fully Rewritten Global Overview** markdown body.
  - Must use the exact **Structural Standard** above.
  - Under specific `Services` subsections, list **every** repository found.
  - Assign each repo to a logical service group with a one-line description (infer from name) and its overview path.
  - Preserve critical operational guidance extracted in Phase 1 (Database + Testing + Troubleshooting). If the existing overview contains database/test instructions, carry them forward (do not delete them) and keep them under `## Operational Notes & Testing`.
  - When rewriting, only rewrite what is necessary. If the existing prose is correct and only the repo inventory is stale, prefer minimal edits instead of a full rewrite.
- **Output 2:** The **Repository Coverage Table** (as defined in Path A) appended at the very end.

### 5. Constraints & Formatting

- **Source of Truth:** Do not invent repositories or paths. Rely **only** on actual workspace folders.
- **Style:** Use plain Markdown. **No bold**, **no italics** in the document body.
- **Self-Check:** Verify that `Count(Discovered Repos) == Count(Repos Listed in Document)`.
  - Count only the discovered workspace root repositories (excluding `devdocs` and `devtools`).
  - Ensure every discovered repo appears exactly once in either the Services lists and in the Repository Coverage Table.
  - Ensure no “phantom repos” are listed (repos that are not present at workspace root).

### 6. Safety Checks (Do Not Skip)

Before finalizing output, perform these checks:

1. Operational content preservation:
   - Database connection/query instructions from the existing global overview are still present.
   - Testing constraints and the canonical `just -f devtools/Justfile test-<repo>` pattern are still present.
2. Schema compliance:
   - The file uses only the required top-level headings in the correct order.
   - If you need extra detail, place it inside `## Operational Notes & Testing` without adding new H2 sections.
3. Inventory correctness:
   - Every repo in the discovered list is placed in exactly one service group.
   - The coverage table includes every discovered repo with correct overview path and Present/Missing status.
