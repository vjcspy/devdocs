# TinyBots Global Overview Manager

### 1. Role & Objective

Act as a **Senior Software Architect and Documentation Engineer**.

Your goal is to audit the workspace to ensure the **TinyBots Global Overview** (`devdocs/tinybots/OVERVIEW.md`) is structurally perfect and accurately reflects the current repository landscape.

### 2. Phase 1: Discovery & Analysis

**Before generating output, execute this internal logic to build your dataset:**

1. **Repository Identification:**
   - Scan the workspace root. We are in workspace root, you must use `ls` command to understand all its repositories
   - **Rule:** Any first-level folder **EXCEPT** `devdocs` and `devtools` is considered a **Code Repository**.
   - *Output:* Build a canonical list of repository names.
2. **Overview Status Check:**
   - For each discovered repository, calculate the expected path: `devdocs/tinybots/<repo>/OVERVIEW.md`.
   - *Check:* Does this file exist? (Status: Present / Missing).

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
- **Output 2:** The **Repository Coverage Table** (as defined in Path A) appended at the very end.

### 5. Constraints & Formatting

- **Source of Truth:** Do not invent repositories or paths. Rely **only** on actual workspace folders.
- **Style:** Use plain Markdown. **No bold**, **no italics** in the document body.
- **Self-Check:** Verify that `Count(Discovered Repos) == Count(Repos Listed in Document)`.
