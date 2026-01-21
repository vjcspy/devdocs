---
name: testing-guidelines
description: TinyBots testing best practices and assertion patterns for writing integration tests
---

# TinyBots Testing Guidelines

When writing tests for TinyBots repositories, follow these guidelines for consistent, maintainable test code.

## Assertion Best Practices

### Use `deep.include` for Object Assertions

Prefer `expect(obj).to.deep.include({...})` over checking individual fields one by one. This provides a balance between conciseness and flexibility.

```javascript
// ❌ Avoid: Verbose individual field checks
expect(res.body.id).to.equal(scheduledExecutionId)
expect(res.body.executionType).to.equal('scheduled')
expect(res.body.executedAt).to.equal(plannedIso)
expect(res.body.schedule.scheduleId).to.equal(880011)

// ✅ Prefer: deep.include for partial matching
expect(res.body).to.deep.include({
  id: scheduledExecutionId,
  executionType: 'scheduled',
  executedAt: plannedIso,
  schedule: { scheduleId: 880011 }
})
```

### Why `deep.include` over `deep.equal`

| Benefit | Description |
|---------|-------------|
| **Ignores dynamic fields** | Doesn't fail on `createdAt`, `updatedAt`, etc. |
| **Less brittle** | Won't break if new fields are added to the response |
| **Validates structure** | Still validates nested object structure correctly |

### When to Use Individual Checks

Use separate assertions when you need:

- **Array length checks**: Verify collection sizes
- **Type checks**: Confirm data types explicitly
- **Specific error messages**: Need custom failure messages for specific fields

```javascript
// Combine deep.include with specific checks
expect(res.body).to.deep.include({ id: expectedId, status: 'active' })
expect(res.body.items).to.be.an('array').with.lengthOf(3)
```

## Test Structure

### Arrange-Act-Assert Pattern

```javascript
it('should return execution details', async () => {
  // Arrange - setup test data
  const executionId = await createTestExecution()
  
  // Act - perform the action
  const res = await request(app)
    .get(`/v6/scripts/user/robots/1/executions/${executionId}`)
    .set('x-authenticated-userid', '1')
  
  // Assert - verify results
  expect(res.status).to.equal(200)
  expect(res.body).to.deep.include({
    id: executionId,
    executionType: 'triggered'
  })
})
```

### Descriptive Test Names

- Use `should` prefix for clarity
- Describe the expected behavior, not the implementation
- Include relevant context (e.g., "when user is not authenticated")

```javascript
// ✅ Good
it('should return 404 when execution does not exist')
it('should return paginated results when limit is specified')

// ❌ Avoid
it('test execution endpoint')
it('works correctly')
```

## Running Tests

> **IMPORTANT:** All TinyBots tests must run inside Docker containers via `just` commands.
> See `devdocs/agent/rules/tinybots/run-tests.md` for execution instructions.

```bash
# Run tests for a specific repository
just -f devtools/tinybots/local/Justfile test-<repo>
```
