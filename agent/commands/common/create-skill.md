# Create Skill

## Role & Objective

Act as a **Senior AI Agent Developer** and **Prompt Engineer**.
Your goal is to create a Claude Code skill (SKILL.md and supporting files) following the [Agent Skills open standard](https://code.claude.com/docs/en/skills).

A skill extends Claude's capabilities - it's a set of instructions Claude uses when relevant, or that users can invoke directly via `/skill-name`.

**Input Variables:**

- `SKILL_NAME`: Name of the skill (lowercase, hyphens allowed, max 64 chars)
- `SKILL_DESCRIPTION`: What the skill does and when to use it
- `SKILL_SCOPE`: `personal` | `project` | `plugin`
- `INVOCATION_MODE`: `both` | `user-only` | `model-only`
- `EXECUTION_CONTEXT`: `inline` | `fork` (default: `inline`)

---

## Phase 0: Requirements Gathering (CRITICAL)

**Action:** Clarify the skill's purpose and behavior before implementation.

1. **Define the Skill Intent:**
   - What task or knowledge does this skill provide?
   - Is it **reference content** (conventions, patterns, domain knowledge) or **task content** (step-by-step actions)?

2. **Determine Invocation Mode:**
   - `both` (default): User can invoke via `/skill-name`, Claude can invoke automatically when relevant
   - `user-only`: Only user can invoke (for workflows with side effects like deploy, commit)
   - `model-only`: Only Claude can invoke (background knowledge users shouldn't invoke directly)

3. **Determine Execution Context:**
   - `inline` (default): Runs in the current conversation context
   - `fork`: Runs in an isolated subagent context (use for research, exploration, or tasks that need isolation)

4. **Identify Supporting Files:**
   - Does the skill need templates, examples, scripts, or reference docs?
   - Plan the directory structure if supporting files are needed

---

## Phase 1: Directory Structure Setup

**Action:** Create the skill directory based on scope.

### Skill Location by Scope

| Scope | Path |
|-------|------|
| Personal | `~/.claude/skills/{SKILL_NAME}/SKILL.md` |
| Project | `.claude/skills/{SKILL_NAME}/SKILL.md` |
| Plugin | `{plugin-path}/skills/{SKILL_NAME}/SKILL.md` |

### Basic Structure

```
{SKILL_NAME}/
├── SKILL.md           # Main instructions (required)
└── ... (optional supporting files)
```

### Extended Structure (with supporting files)

```
{SKILL_NAME}/
├── SKILL.md           # Main instructions (required)
├── templates/         # Templates for Claude to fill in
│   └── template.md
├── examples/          # Example outputs showing expected format
│   └── sample.md
├── scripts/           # Scripts Claude can execute
│   └── helper.py
└── reference/         # Detailed reference documentation
    └── api-docs.md
```

---

## Phase 2: Frontmatter Configuration

**Action:** Configure the YAML frontmatter based on requirements.

### Frontmatter Reference

```yaml
---
# Required/Recommended
name: {SKILL_NAME}                    # Display name (defaults to directory name)
description: {SKILL_DESCRIPTION}      # RECOMMENDED: When to use this skill

# Invocation Control
disable-model-invocation: false       # true = only user can invoke
user-invocable: true                  # false = only Claude can invoke

# Arguments
argument-hint: "[arg1] [arg2]"        # Hint shown during autocomplete

# Tool Restrictions
allowed-tools: Read, Grep, Glob       # Limit available tools

# Execution
model: claude-sonnet-4-20250514             # Override model
context: fork                         # Run in isolated subagent
agent: Explore                        # Subagent type (if context: fork)

# Hooks (optional)
hooks:                                # Lifecycle hooks
  pre-invoke: "./scripts/setup.sh"
---
```

### Invocation Mode Mapping

| INVOCATION_MODE | Frontmatter |
|-----------------|-------------|
| `both` | (defaults, no extra fields) |
| `user-only` | `disable-model-invocation: true` |
| `model-only` | `user-invocable: false` |

### Execution Context Mapping

| EXECUTION_CONTEXT | Frontmatter |
|-------------------|-------------|
| `inline` | (default, no extra fields) |
| `fork` | `context: fork` and optionally `agent: Explore|Plan|general-purpose` |

---

## Phase 3: Skill Content Writing

**Action:** Write the markdown content following these guidelines.

### Content Structure Guidelines

1. **Start with Context:**
   - Brief explanation of what the skill does
   - When it should be used

2. **Provide Clear Instructions:**
   - Step-by-step actions for task skills
   - Guidelines and patterns for reference skills

3. **Use String Substitutions:**
   - `$ARGUMENTS` - All arguments passed when invoking
   - `${CLAUDE_SESSION_ID}` - Current session ID

4. **Dynamic Context Injection (Optional):**
   - Use `!`command`` syntax to inject shell command output
   - Commands run before skill content is sent to Claude

### Reference Content Template

```markdown
---
name: {SKILL_NAME}
description: {SKILL_DESCRIPTION}
---

When [doing X], follow these guidelines:

## [Category 1]

- [Guideline 1]
- [Guideline 2]

## [Category 2]

- [Guideline 3]
- [Guideline 4]
```

### Task Content Template

```markdown
---
name: {SKILL_NAME}
description: {SKILL_DESCRIPTION}
disable-model-invocation: true
---

# {Task Name}

Execute the following steps for $ARGUMENTS:

## Step 1: [Action]

[Detailed instructions]

## Step 2: [Action]

[Detailed instructions]

## Output

[Expected deliverable]
```

### Subagent Task Template

```markdown
---
name: {SKILL_NAME}
description: {SKILL_DESCRIPTION}
context: fork
agent: Explore
allowed-tools: Read, Grep, Glob
---

# Research Task

Investigate $ARGUMENTS thoroughly:

1. [Step 1]
2. [Step 2]
3. [Step 3]

## Output Format

Summarize findings with specific file references.
```

---

## Phase 4: Supporting Files (Optional)

**Action:** Create supporting files if needed.

### When to Use Supporting Files

- **Templates:** Structured outputs Claude should fill in
- **Examples:** Sample outputs showing expected format
- **Scripts:** Automation scripts Claude can execute
- **Reference:** Detailed docs too large for SKILL.md

### Referencing Supporting Files

In SKILL.md, reference files so Claude knows when to load them:

```markdown
## Additional Resources

- For complete API details, see [reference.md](reference/api-docs.md)
- For usage examples, see [examples.md](examples/sample.md)
- Use the template at [template.md](templates/template.md) for output
```

### Size Guidelines

- Keep `SKILL.md` under 500 lines
- Move detailed reference material to separate files
- Large reference docs load on-demand, not every invocation

---

## Phase 5: Validation & Testing

**Action:** Verify the skill works correctly.

### Validation Checklist

1. **File Structure:**
   - [ ] `SKILL.md` exists in correct location
   - [ ] Frontmatter is valid YAML between `---` markers
   - [ ] Supporting files are properly referenced

2. **Naming:**
   - [ ] Skill name uses lowercase letters, numbers, and hyphens only
   - [ ] Skill name is max 64 characters

3. **Description:**
   - [ ] Description explains what the skill does
   - [ ] Description includes keywords users would naturally say

4. **Content:**
   - [ ] Instructions are clear and actionable
   - [ ] String substitutions are correct (`$ARGUMENTS`, not `{ARGUMENTS}`)
   - [ ] Dynamic context uses correct syntax (`!`command``, not `$(command)`)

### Testing Methods

1. **Direct Invocation:**
   ```
   /{SKILL_NAME} [arguments]
   ```

2. **Automatic Invocation (if enabled):**
   - Ask Claude something that matches the skill description
   - Verify Claude loads and uses the skill

3. **List Available Skills:**
   ```
   What skills are available?
   ```

---

## Phase 6: Output & Delivery

**Action:** Create the skill files and confirm creation.

### Output Checklist

- [ ] Created `SKILL.md` with valid frontmatter and content
- [ ] Created supporting files (if applicable)
- [ ] Verified skill appears in available skills
- [ ] Tested invocation works as expected

### Delivery Format

Report to user:
- Skill location path
- How to invoke: `/{SKILL_NAME}` or automatic
- Summary of what the skill does
- Any supporting files created

---

## Examples

### Example 1: Code Review Skill (User-Only Task)

```markdown
---
name: review-pr
description: Review pull request changes for code quality
disable-model-invocation: true
argument-hint: "[pr-number]"
allowed-tools: Bash(gh:*)
---

# PR Review

Review PR #$ARGUMENTS:

## Current PR Context
- PR diff: !`gh pr diff $ARGUMENTS`
- PR description: !`gh pr view $ARGUMENTS`

## Review Checklist

1. **Code Quality:** Check for SOLID violations, code smells
2. **Security:** Look for injection risks, exposed secrets
3. **Performance:** Identify N+1 queries, expensive operations
4. **Tests:** Verify adequate test coverage

## Output

Provide structured feedback with specific line references.
```

### Example 2: API Conventions Skill (Reference Content)

```markdown
---
name: api-conventions
description: REST API design patterns and conventions for this codebase
---

When writing API endpoints, follow these conventions:

## Naming

- Use plural nouns for resources: `/users`, `/orders`
- Use kebab-case for multi-word resources: `/order-items`
- Use path parameters for resource IDs: `/users/{id}`

## Response Format

- Always return JSON
- Use consistent error format: `{ "error": { "code": "...", "message": "..." } }`
- Include pagination metadata for list endpoints

## Status Codes

- 200: Success with body
- 201: Created
- 204: Success without body
- 400: Bad request (validation)
- 401: Unauthorized
- 404: Not found
- 500: Server error
```

### Example 3: Research Skill (Subagent Fork)

```markdown
---
name: deep-research
description: Research a topic thoroughly in the codebase
context: fork
agent: Explore
allowed-tools: Read, Grep, Glob, SemanticSearch
---

# Deep Research

Research "$ARGUMENTS" thoroughly:

1. **Discovery:** Use Glob and Grep to find relevant files
2. **Analysis:** Read and understand the code structure
3. **Connections:** Identify dependencies and relationships
4. **Patterns:** Note design patterns and conventions used

## Output Format

Summarize findings with:
- List of relevant files with descriptions
- Key functions/classes and their purposes
- Data flow diagrams (ASCII if helpful)
- Recommendations or concerns
```
