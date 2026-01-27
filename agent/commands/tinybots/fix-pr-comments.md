# Fix PR Comments

## Role & Objective

Act as a **Senior Code Reviewer** and **Developer**.

Your goal is to analyze Bitbucket PR comments and tasks from reviewers, evaluate their validity, provide expert assessment, and fix approved items only after user confirmation.

## Input Variables

- `WORKSPACE`: Bitbucket workspace (default: `tinybots`)
- `REPO_SLUG`: Repository name (e.g., `micro-manager`)
- `PR_ID`: Pull request number

## Environment Variables (Required)

| Variable | Description | Example |
|----------|-------------|---------|
| `BITBUCKET_USER` | Bitbucket username/email | `user@example.com` |
| `BITBUCKET_APP_PASSWORD` | Bitbucket App Password | `ATATTxxxxx...` |

> **Note:** Create an App Password at: https://bitbucket.org/account/settings/app-passwords/
> Required permissions: `Repositories: Read`, `Pull requests: Read`

---

## Phase 1: Fetch PR Data

**Action:** Use the `bitbucket` skill to fetch all necessary data.

### Step 1.1: Get PR Info

```bash
curl --request GET \
  --url 'https://api.bitbucket.org/2.0/repositories/{WORKSPACE}/{REPO_SLUG}/pullrequests/{PR_ID}' \
  --user "${BITBUCKET_USER}:${BITBUCKET_APP_PASSWORD}" \
  --header 'Accept: application/json'
```

### Step 1.2: Get All Comments

```bash
curl --request GET \
  --url 'https://api.bitbucket.org/2.0/repositories/{WORKSPACE}/{REPO_SLUG}/pullrequests/{PR_ID}/comments' \
  --user "${BITBUCKET_USER}:${BITBUCKET_APP_PASSWORD}" \
  --header 'Accept: application/json'
```

### Step 1.3: Get All Tasks

```bash
curl --request GET \
  --url 'https://api.bitbucket.org/2.0/repositories/{WORKSPACE}/{REPO_SLUG}/pullrequests/{PR_ID}/tasks' \
  --user "${BITBUCKET_USER}:${BITBUCKET_APP_PASSWORD}" \
  --header 'Accept: application/json'
```

---

## Phase 2: Analyze & Map Relationships

**Action:** Build a comprehensive view of comments and tasks.

### Step 2.1: Create Comment-Task Mapping

Build a mapping structure:

```
┌──────────────────────────────────────────────────────────────────┐
│ TASKS (Required Work)                                            │
├──────────────────────────────────────────────────────────────────┤
│ [TASK-1] "Task description"                                      │
│   └── Linked Comment: #comment_id                                │
│       └── File: path/to/file.ts (Line X)                         │
│       └── Content: "Comment text..."                             │
│   └── Status: UNRESOLVED / RESOLVED                              │
├──────────────────────────────────────────────────────────────────┤
│ [TASK-2] "Standalone task" (no linked comment)                   │
│   └── Status: UNRESOLVED                                         │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│ ORPHAN COMMENTS (Nice-to-have, no task created)                  │
├──────────────────────────────────────────────────────────────────┤
│ [COMMENT-X] File: path/to/file.ts (Line Y)                       │
│   └── Content: "Suggestion text..."                              │
│   └── Author: Reviewer Name                                      │
└──────────────────────────────────────────────────────────────────┘
```

### Step 2.2: Categorize Items

Group by priority:

1. **MUST FIX**: Tasks with `state: UNRESOLVED`
2. **SHOULD CONSIDER**: Orphan comments (no linked task)
3. **ALREADY DONE**: Tasks with `state: RESOLVED`

---

## Phase 3: Expert Assessment

**Action:** Provide professional evaluation of each item.

For EACH task/comment, analyze:

### Assessment Template

```markdown
### [TASK/COMMENT-ID] "{Title/Summary}"

**Location:** `path/to/file.ts` (Line X)
**Reviewer:** {Reviewer Name}
**Type:** Task / Comment-only

**Reviewer's Point:**
> "{Original comment text}"

**Expert Assessment:**
- **Validity:** ✅ Valid / ⚠️ Partially Valid / ❌ Not Recommended
- **Reasoning:** [Explain why this feedback is valid/invalid from an expert perspective]
- **Impact:** [What happens if we fix vs. don't fix]
- **Effort:** Low / Medium / High

**Recommendation:**
- [ ] FIX - {Brief description of what to do}
- [ ] SKIP - {Reason to skip}
- [ ] DISCUSS - {Need more context}
```

### Assessment Criteria

Evaluate based on:

1. **Code Quality**: Does it improve maintainability, readability?
2. **Consistency**: Does it align with project conventions?
3. **Performance**: Does it have performance implications?
4. **Architecture**: Is it architecturally sound?
5. **Practicality**: Is the effort justified by the benefit?

---

## Phase 4: Present Report to User

**Action:** Generate comprehensive report and ASK for approval.

### Report Format

```markdown
# PR #{PR_ID} Review Analysis

## Summary
- **PR Title:** {title}
- **Author:** {author}
- **Branch:** {source} → {destination}
- **Total Tasks:** X (Y unresolved)
- **Total Comments:** Z (W without tasks)

---

## 🔴 UNRESOLVED TASKS (Must Address)

### [T1] {Task Title}
{Assessment from Phase 3}

### [T2] {Task Title}
{Assessment from Phase 3}

---

## 🟡 ORPHAN COMMENTS (Nice-to-have)

### [C1] {Comment Summary}
{Assessment from Phase 3}

---

## 📋 ACTION PLAN

Based on my analysis, here's my recommended action plan:

| ID | Item | Recommendation | Effort |
|----|------|----------------|--------|
| T1 | {Task} | ✅ FIX | Low |
| T2 | {Task} | ⚠️ DISCUSS | Medium |
| C1 | {Comment} | ❌ SKIP | - |

---

## ⏳ AWAITING YOUR DECISION

Please review and tell me which items to fix:
- "Fix all" - I'll fix all recommended items
- "Fix T1, T2" - I'll fix specific items
- "Skip C1" - I'll skip specific items
- Ask questions about any item
```

**CRITICAL:** DO NOT proceed to fix anything until user explicitly approves.

---

## Phase 5: Execute Fixes (After Approval Only)

**Action:** Fix only the approved items.

### Step 5.1: Read Target Files

Before making changes, read the relevant source files to understand context.

### Step 5.2: Make Changes

For each approved item:

1. Navigate to the file/line mentioned
2. Understand the surrounding code context
3. Apply the fix following project conventions
4. Verify the fix doesn't break existing functionality

### Step 5.3: Report Changes

After fixing, report:

```markdown
## ✅ Completed Fixes

### [T1] {Task Title}
- **File:** `path/to/file.ts`
- **Change:** {Description of what was changed}
- **Lines affected:** X-Y

{Show code diff or summary}

### [T2] {Task Title}
...
```

---

## Phase 6: Post-Fix Verification

**Action:** Verify changes and suggest next steps.

### Checklist

- [ ] All approved items have been addressed
- [ ] No linter errors introduced
- [ ] Code follows project conventions
- [ ] Related tests still pass (if applicable)

### Suggest Next Steps

```markdown
## 🚀 Next Steps

1. Review the changes I made
2. Run tests locally: `npm test` / `just test`
3. Commit changes with message: `fix: address PR review comments`
4. Push and request re-review from {Reviewer Name}
```

---

## Error Handling

- **API Error 401/403:** Check credentials and permissions
- **PR Not Found:** Verify workspace, repo slug, and PR ID
- **No Comments/Tasks:** Report "No review feedback found"
- **Ambiguous Feedback:** Ask user for clarification before proceeding

---

## Example Usage

```
User: Check PR 126 on micro-manager and help me fix the comments

Agent:
1. Fetches PR #126 data from tinybots/micro-manager
2. Analyzes 7 comments and 3 tasks
3. Maps relationships between comments and tasks
4. Provides expert assessment for each item
5. Presents report and waits for user approval
6. After approval, fixes only the approved items
7. Reports changes and suggests next steps
```
