---
name: bitbucket
description: Provides Bitbucket API integration for interacting with repositories, pull requests, comments, and tasks. Use when working with Bitbucket PRs, reviewing code, or automating PR workflows.
user-invocable: false
disable-model-invocation: false
---

# Bitbucket API Integration

This skill provides context and examples for interacting with Bitbucket Cloud API v2.0.

## Authentication

Use **Basic Authentication** with App Password via environment variables:

```bash
--user "${BITBUCKET_USER}:${BITBUCKET_APP_PASSWORD}"
```

**Environment Variables (Required):**

| Variable | Description | Example |
|----------|-------------|---------|
| `BITBUCKET_USER` | Bitbucket username/email | `user@example.com` |
| `BITBUCKET_APP_PASSWORD` | Bitbucket App Password | `ATATTxxxxx...` |

> **Setup:** Create an App Password at: https://bitbucket.org/account/settings/app-passwords/

**Required Scopes:** `read:repository`, `read:pullrequest`

## Base URL

```
https://api.bitbucket.org/2.0
```

## API Endpoints

### 1. Get Pull Request Info

```bash
curl --request GET \
  --url 'https://api.bitbucket.org/2.0/repositories/{workspace}/{repo_slug}/pullrequests/{pr_id}' \
  --user "${BITBUCKET_USER}:${BITBUCKET_APP_PASSWORD}" \
  --header 'Accept: application/json'
```

**Response fields:**
- `id`: PR number
- `title`: PR title
- `state`: `OPEN`, `MERGED`, `DECLINED`, `SUPERSEDED`
- `author.display_name`: Author name
- `source.branch.name`: Source branch
- `destination.branch.name`: Target branch
- `description`: PR description

### 2. List All Pull Requests

```bash
curl --request GET \
  --url 'https://api.bitbucket.org/2.0/repositories/{workspace}/{repo_slug}/pullrequests' \
  --user "${BITBUCKET_USER}:${BITBUCKET_APP_PASSWORD}" \
  --header 'Accept: application/json'
```

**Query parameters:**
- `state`: Filter by state (`OPEN`, `MERGED`, `DECLINED`, `SUPERSEDED`)
- `pagelen`: Results per page (default: 10, max: 50)
- `page`: Page number

### 3. Get PR Comments

```bash
curl --request GET \
  --url 'https://api.bitbucket.org/2.0/repositories/{workspace}/{repo_slug}/pullrequests/{pr_id}/comments' \
  --user "${BITBUCKET_USER}:${BITBUCKET_APP_PASSWORD}" \
  --header 'Accept: application/json'
```

**Response structure:**
```json
{
  "values": [
    {
      "id": 740667474,
      "content": {
        "raw": "Comment text here"
      },
      "user": {
        "display_name": "Reviewer Name"
      },
      "inline": {
        "path": "src/file.ts",
        "to": 29
      },
      "deleted": false
    }
  ],
  "size": 7,
  "page": 1
}
```

**Key fields:**
- `id`: Comment ID (used to link with tasks)
- `content.raw`: Comment text
- `user.display_name`: Commenter name
- `inline.path`: File path (for inline comments)
- `inline.to`: Line number
- `deleted`: Whether comment was deleted

### 4. Get Single Comment

```bash
curl --request GET \
  --url 'https://api.bitbucket.org/2.0/repositories/{workspace}/{repo_slug}/pullrequests/{pr_id}/comments/{comment_id}' \
  --user "${BITBUCKET_USER}:${BITBUCKET_APP_PASSWORD}" \
  --header 'Accept: application/json'
```

### 5. Get PR Tasks

```bash
curl --request GET \
  --url 'https://api.bitbucket.org/2.0/repositories/{workspace}/{repo_slug}/pullrequests/{pr_id}/tasks' \
  --user "${BITBUCKET_USER}:${BITBUCKET_APP_PASSWORD}" \
  --header 'Accept: application/json'
```

**Response structure:**
```json
{
  "values": [
    {
      "id": 58006601,
      "state": "UNRESOLVED",
      "content": {
        "raw": "Task description"
      },
      "creator": {
        "display_name": "Reviewer Name"
      },
      "comment": {
        "id": 740667474
      }
    }
  ]
}
```

**Key fields:**
- `id`: Task ID
- `state`: `UNRESOLVED` or `RESOLVED`
- `content.raw`: Task description
- `comment.id`: Linked comment ID (optional - some tasks are standalone)
- `resolved_on`: Timestamp when resolved
- `resolved_by`: Who resolved it

### 6. Get PR Diff

```bash
curl --request GET \
  --url 'https://api.bitbucket.org/2.0/repositories/{workspace}/{repo_slug}/pullrequests/{pr_id}/diff' \
  --user "${BITBUCKET_USER}:${BITBUCKET_APP_PASSWORD}" \
  --header 'Accept: text/plain'
```

### 7. Get Commits in PR

```bash
curl --request GET \
  --url 'https://api.bitbucket.org/2.0/repositories/{workspace}/{repo_slug}/pullrequests/{pr_id}/commits' \
  --user "${BITBUCKET_USER}:${BITBUCKET_APP_PASSWORD}" \
  --header 'Accept: application/json'
```

## Understanding Comments vs Tasks Relationship

```
┌─────────────────────────────────────────────────────────────┐
│                        PR Review                            │
├─────────────────────────────────────────────────────────────┤
│  Comments (inline feedback on code)                         │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ Comment #1: "Why lowercase?" (file.ts:29)           │   │
│  │ Comment #2: "Add to tiny-specs" (file.ts:8)         │   │
│  │ Comment #3: "Fix typo" (test.ts:15) [no task]       │   │
│  └─────────────────────────────────────────────────────┘   │
│                           │                                 │
│                           ▼                                 │
│  Tasks (actionable items to track)                          │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ Task #1: "Separate model" → linked to Comment #1    │   │
│  │ Task #2: "Move dto's to tiny-specs" → Comment #2    │   │
│  │ Task #3: "Fix formatting" → standalone (no comment) │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

**Relationships:**
1. **Task with comment**: Task has `comment.id` field linking to specific comment
2. **Standalone task**: Task without `comment` field (general action item)
3. **Comment without task**: Nice-to-have suggestions, not tracked as required work

## Common Workflows

### Fetch All PR Review Data

```bash
# 1. Get PR info
PR_INFO=$(curl -s --user "${BITBUCKET_USER}:${BITBUCKET_APP_PASSWORD}" \
  'https://api.bitbucket.org/2.0/repositories/{workspace}/{repo}/pullrequests/{pr_id}')

# 2. Get comments
COMMENTS=$(curl -s --user "${BITBUCKET_USER}:${BITBUCKET_APP_PASSWORD}" \
  'https://api.bitbucket.org/2.0/repositories/{workspace}/{repo}/pullrequests/{pr_id}/comments')

# 3. Get tasks
TASKS=$(curl -s --user "${BITBUCKET_USER}:${BITBUCKET_APP_PASSWORD}" \
  'https://api.bitbucket.org/2.0/repositories/{workspace}/{repo}/pullrequests/{pr_id}/tasks')
```

### Parse with jq

```bash
# Extract comment summaries
echo "$COMMENTS" | jq '.values[] | {id, text: .content.raw, file: .inline.path, line: .inline.to, author: .user.display_name}'

# Extract unresolved tasks with linked comments
echo "$TASKS" | jq '.values[] | select(.state == "UNRESOLVED") | {id, task: .content.raw, comment_id: .comment.id}'

# Count unresolved tasks
echo "$TASKS" | jq '[.values[] | select(.state == "UNRESOLVED")] | length'
```

## Error Handling

Common errors:
- `401`: Invalid credentials
- `403`: Missing required scopes
- `404`: Repository/PR not found

Check permissions:
```json
{
  "type": "error",
  "error": {
    "message": "Your credentials lack one or more required privilege scopes.",
    "detail": {
      "required": ["read:pullrequest:bitbucket"],
      "granted": ["admin:repository:bitbucket"]
    }
  }
}
```

## Workspace Reference

| Project | Workspace | Common Repos |
|---------|-----------|--------------|
| Tinybots | `tinybots` | `micro-manager`, `sensara-adaptor`, `tiny-specs` |
