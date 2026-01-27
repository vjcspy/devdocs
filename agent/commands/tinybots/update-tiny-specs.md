# Update Tiny-Specs

## Role & Objective

Act as a **Senior Backend Developer** managing shared OpenAPI specifications.

Your goal is to safely update the `tiny-specs` repository and integrate changes into dependent repositories following the established workflow.

## Input Variables

- `SOURCE_REPO`: The repository that needs tiny-specs updates (e.g., `micro-manager`, `wonkers-graphql`)
- `BRANCH_NAME`: Branch name for tiny-specs changes (convention: `feature/<source-repo>-<short-description>`)

---

## Phase 1: Pre-flight Checks

**Action:** Verify tiny-specs repository state before making changes.

### Step 1.1: Navigate to tiny-specs

```bash
cd /path/to/tinybots/backend/tiny-specs
```

### Step 1.2: Check Current Branch

```bash
git branch --show-current
```

**Expected:** `master`

**If NOT on master:** STOP and inform user:
> "tiny-specs is currently on branch `{branch_name}`. Please switch to master or confirm you want to continue on this branch."

### Step 1.3: Check for Uncommitted Changes

```bash
git status --porcelain
```

**Expected:** Empty output (no changes)

**If changes exist:** STOP and inform user:
> "tiny-specs has uncommitted changes. Please commit or stash them before proceeding."

### Step 1.4: Pull Latest Changes

```bash
git pull origin master
```

---

## Phase 2: Create Feature Branch

**Action:** Create a new branch for the changes.

### Step 2.1: Create and Checkout Branch

```bash
git checkout -b {BRANCH_NAME}
```

**Branch naming convention:**
- `feature/<source-repo>-<description>` for new features
- `fix/<source-repo>-<description>` for fixes

**Examples:**
- `feature/micro-manager-triggered-execution-apis`
- `fix/wonkers-graphql-user-schema`

---

## Phase 3: Implement Changes

**Action:** Make the required changes to tiny-specs.

### Common Change Locations

| Change Type | File Path |
|-------------|-----------|
| Schemas | `specs/local/components/<service>/v<version>/schemas.yaml` |
| Paths | `specs/local/paths/<service>/v<version>/paths.yaml` |
| Common schemas | `specs/local/components/common/schemas.yaml` |

### Guidelines

1. Follow existing OpenAPI 3.0 conventions in the repo
2. Use `$ref` for reusable components
3. Include proper descriptions and examples
4. Match response schemas to actual API responses

---

## Phase 4: Build and Validate

**Action:** Ensure changes are valid and build successfully.

### Step 4.1: Install Dependencies (if needed)

```bash
yarn install
```

### Step 4.2: Build

```bash
yarn run build
```

**Expected:** No errors

### Step 4.3: Run All Checks

```bash
yarn run all
```

**Expected:** All checks pass

**If errors occur:** Fix them before proceeding.

---

## Phase 5: Decision Point

**Action:** Ask user about next steps.

### If NO source repo needs the changes:
> "tiny-specs changes are complete. Would you like me to commit and push?"

### If source repo needs the changes (like micro-manager):
> "tiny-specs changes are ready. To use these in `{SOURCE_REPO}`:
> 1. I'll commit and push the tiny-specs branch
> 2. Update `{SOURCE_REPO}/package.json` to reference this branch
> 3. Run `yarn install` to fetch the new specs
> 4. Update code to import from tiny-specs
>
> Proceed? (yes/no)"

---

## Phase 6: Commit and Push tiny-specs

**Action:** Commit changes and push to remote.

### Step 6.1: Stage Changes

```bash
git add .
```

### Step 6.2: Commit

```bash
git commit -m "feat(<service>): <description>

- Added schemas for <feature>
- Added paths for <endpoints>"
```

### Step 6.3: Push

```bash
git push -u origin {BRANCH_NAME}
```

---

## Phase 7: Integrate into Source Repository

**Action:** Update source repo to use new tiny-specs branch.

### Step 7.1: Navigate to Source Repo

```bash
cd /path/to/tinybots/backend/{SOURCE_REPO}
```

### Step 7.2: Update package.json

Find the `tiny-specs` dependency and update:

```json
{
  "dependencies": {
    "tiny-specs": "git+ssh://git@bitbucket.org/tinybots/tiny-specs.git#{BRANCH_NAME}"
  }
}
```

### Step 7.3: Install Updated Dependency

```bash
yarn install
```

### Step 7.4: Update Imports

Replace local DTOs with imports from tiny-specs:

```typescript
// Before (local DTO)
import { MyResponse } from '../schemas/body/MyResponse'

// After (from tiny-specs)
import { MyResponse } from 'tiny-specs/dist/local/<service>/v<version>/schemas'
```

### Step 7.5: Verify Build

```bash
yarn build
# or
yarn tsc --noEmit
```

### Step 7.6: Run Tests

```bash
yarn test
# or use just command
just test-<source-repo>
```

---

## Phase 8: Cleanup (After PR Merged)

**Action:** After tiny-specs PR is merged to master, update source repo.

### Step 8.1: Update package.json to master

```json
{
  "dependencies": {
    "tiny-specs": "git+ssh://git@bitbucket.org/tinybots/tiny-specs.git#master"
  }
}
```

### Step 8.2: Reinstall

```bash
yarn install
```

---

## Error Handling

### Build Errors in tiny-specs

```bash
# Check TypeScript errors
yarn tsc --noEmit

# Check OpenAPI validation
yarn run validate
```

### Import Errors in Source Repo

1. Verify the export path exists in tiny-specs
2. Check `tiny-specs/dist/` structure after build
3. Ensure correct version number in import path

### yarn install Fails

```bash
# Clear yarn cache
yarn cache clean

# Remove node_modules and reinstall
rm -rf node_modules yarn.lock
yarn install
```

---

## Example Usage

```
User: Add response schemas for micro-manager triggered execution APIs

Agent:
1. Checks tiny-specs is on master with no changes
2. Creates branch: feat/micro-manager-triggered-execution-apis
3. Adds schemas to specs/local/components/micro-manager/v6/schemas.yaml
4. Adds paths to specs/local/paths/micro-manager/v6/paths.yaml
5. Runs yarn build && yarn all
6. Asks user to confirm push
7. Updates micro-manager/package.json
8. Updates imports in micro-manager
9. Runs tests
```

---

## Related Files

- tiny-specs repo: `tinybots/backend/tiny-specs`
- Build output: `tiny-specs/dist/`
- Import pattern: `tiny-specs/dist/local/<service>/v<version>/schemas`
