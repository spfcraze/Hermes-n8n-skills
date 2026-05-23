# Validation & Debugging — Deep Dive Reference

Complete guide to validating n8n workflows and fixing errors.

---

## Validation Philosophy

**Validate early, validate often.**

Validation is iterative:
- Expect 2-3 validate → fix cycles
- This is normal — don't be discouraged
- Each cycle improves configuration quality

---

## Error Severity Levels

### Errors (Must Fix)

**Blocks workflow execution.** Must be resolved before activation.

| Type | Meaning | Example |
|------|---------|---------|
| `missing_required` | Required field not provided | "Channel name is required" |
| `invalid_value` | Value not in allowed options | "Operation must be one of: post, update" |
| `type_mismatch` | Wrong data type | "Expected number, got string" |
| `invalid_reference` | Referenced node doesn't exist | "Node 'HTTP Requets' does not exist" |
| `invalid_expression` | Expression syntax error | "Invalid expression: $json.name" |

### Warnings (Should Fix)

**Doesn't block execution** but may cause issues.

| Type | Meaning | Example |
|------|---------|---------|
| `best_practice` | Recommended improvement | "Add error handling" |
| `deprecated` | Using old API/feature | "This operation is deprecated" |
| `performance` | Potential performance issue | "No LIMIT on query" |

### Suggestions (Optional)

**Nice to have** improvements.

| Type | Meaning | Example |
|------|---------|---------|
| `optimization` | Could be more efficient | "Use batch operations" |
| `alternative` | Better way to achieve result | "Consider using Set node" |

---

## Validation Profiles

### minimal

**Use**: Quick checks during editing.

**Validates**:
- Only required fields
- Basic structure

**Pros**: Fastest, most permissive
**Cons**: May miss issues

### runtime (RECOMMENDED)

**Use**: Pre-deployment validation.

**Validates**:
- Required fields
- Value types
- Allowed values
- Basic dependencies

**Pros**: Balanced, catches real errors
**Cons**: Some edge cases missed

### ai-friendly

**Use**: AI-generated configurations.

**Validates**:
- Same as runtime
- Reduces false positives
- More tolerant of minor issues

**Pros**: Less noisy for AI workflows
**Cons**: May allow questionable configs

### strict

**Use**: Production deployment, critical workflows.

**Validates**:
- Everything
- Best practices
- Performance concerns
- Security issues

**Pros**: Maximum safety
**Cons**: Many warnings, some false positives

---

## The Validation Loop

```
1. Configure node
   ↓
2. Validate (analyze errors)
   ↓
3. Read error messages carefully
   ↓
4. Fix errors
   ↓
5. Validate again
   ↓
6. Repeat until clean (usually 2-3 iterations)
```

**Example**:
```javascript
// Iteration 1
let config = {resource: "channel", operation: "create"};
validate_node({nodeType: "nodes-base.slack", config, profile: "runtime"});
// → Error: Missing "name"

// Iteration 2
config.name = "general";
validate_node({...});
// → Error: Missing "text"

// Iteration 3
config.text = "Hello!";
validate_node({...});
// → Valid! ✅
```

---

## Common Error Types — Detailed

### missing_required

**What**: A required field is not provided.

**Fix**:
1. Use node discovery to see required fields
2. Add the missing field
3. Provide appropriate value

**Example**:
```javascript
// Error
{type: "missing_required", property: "channel", message: "Channel name is required"}

// Fix
config.channel = "#general";
```

### invalid_value

**What**: Value doesn't match allowed options.

**Fix**:
1. Check error for allowed values
2. Update to valid value

**Example**:
```javascript
// Error
{type: "invalid_value", property: "operation", message: "Must be one of: post, update, delete", current: "send"}

// Fix
config.operation = "post";
```

### type_mismatch

**What**: Wrong data type for field.

**Fix**:
1. Check expected type
2. Convert value

**Example**:
```javascript
// Error
{type: "type_mismatch", property: "limit", message: "Expected number, got string", current: "100"}

// Fix
config.limit = 100;  // Number, not string
```

### invalid_expression

**What**: Expression syntax error.

**Fix**:
1. Check for missing `{{}}`
2. Verify node/field references
3. Check for typos

**Example**:
```javascript
// Error
{type: "invalid_expression", property: "text", message: "Invalid expression: $json.name", current: "$json.name"}

// Fix
config.text = "={{$json.name}}";  // Add {{}}
```

### invalid_reference

**What**: Referenced node doesn't exist.

**Fix**:
1. Check node name spelling
2. Verify node exists
3. Update reference

**Example**:
```javascript
// Error
{type: "invalid_reference", property: "expression", message: "Node 'HTTP Requets' does not exist"}

// Fix
config.expression = "={{$node['HTTP Request'].json.data}}";  // Fix typo
```

---

## Auto-Sanitization

**Automatically fixes common issues on ANY workflow update.**

### What It Fixes

#### Binary Operators

Operators: equals, notEquals, contains, notContains, greaterThan, lessThan, startsWith, endsWith

**Fix**: Removes `singleValue` property.

```javascript
// Before
{type: "boolean", operation: "equals", singleValue: true}  // Wrong!

// After (automatic)
{type: "boolean", operation: "equals"}  // singleValue removed
```

#### Unary Operators

Operators: isEmpty, isNotEmpty, true, false

**Fix**: Adds `singleValue: true`.

```javascript
// Before
{type: "boolean", operation: "isEmpty"}  // Missing singleValue

// After (automatic)
{type: "boolean", operation: "isEmpty", singleValue: true}
```

#### IF/Switch Metadata

**Fix**: Adds complete `conditions.options` metadata for IF v2.2+ and Switch v3.2+.

### What It CANNOT Fix

| Issue | Solution |
|-------|----------|
| Broken connections | Use `cleanStaleConnections` operation |
| Branch count mismatches | Add missing connections or remove rules |
| Paradoxical corrupt states | May require manual intervention |

---

## False Positives

### What Are They?

Validation warnings that are technically "wrong" but acceptable.

### Common False Positives

#### "Missing error handling"

**Acceptable when**:
- Simple workflows
- Testing/development
- Non-critical notifications

**Fix when**: Production workflows with important data.

#### "No retry logic"

**Acceptable when**:
- APIs with built-in retry
- Idempotent operations
- Manual trigger workflows

**Fix when**: Flaky external services, production automation.

#### "Missing rate limiting"

**Acceptable when**:
- Internal APIs
- Low-volume workflows
- APIs with server-side limits

**Fix when**: Public APIs, high-volume workflows.

#### "Unbounded query"

**Acceptable when**:
- Small known datasets
- Aggregation queries
- Development/testing

**Fix when**: Production queries on large tables.

### Reducing False Positives

Use `ai-friendly` profile:
```javascript
validate_node({nodeType: "nodes-base.slack", config: {...}, profile: "ai-friendly"})
```

---

## Validation Result Structure

```javascript
{
  valid: false,
  errors: [
    {
      type: "missing_required",
      property: "channel",
      message: "Channel name is required",
      fix: "Provide a channel name"
    }
  ],
  warnings: [
    {
      type: "best_practice",
      property: "errorHandling",
      message: "Slack API can have rate limits",
      suggestion: "Add onError: 'continueRegularOutput'"
    }
  ],
  suggestions: [
    {
      type: "optimization",
      message: "Consider using batch operations"
    }
  ],
  summary: {
    hasErrors: true,
    errorCount: 1,
    warningCount: 1,
    suggestionCount: 1
  }
}
```

### How to Read Results

1. **Check `valid` field**
   ```javascript
   if (result.valid) { /* Configuration is valid */ }
   ```

2. **Fix errors first**
   ```javascript
   result.errors.forEach(e => console.log(`${e.property}: ${e.message}`));
   ```

3. **Review warnings**
   ```javascript
   result.warnings.forEach(w => console.log(`Warning: ${w.message}`));
   ```

4. **Consider suggestions**
   ```javascript
   result.suggestions.forEach(s => console.log(`Suggestion: ${s.message}`));
   ```

---

## Debugging Checklist

- [ ] Check expression editor (fx icon) for live preview
- [ ] Verify webhook data is under `.body`
- [ ] Confirm node names are exact and case-sensitive
- [ ] Test with sample data before activation
- [ ] Check execution log for detailed traces
- [ ] Use console.log() / print() in Code nodes
- [ ] Validate after every configuration change
- [ ] Review workflow settings → Execution Order
- [ ] Check credentials are configured (not hardcoded)
- [ ] Handle empty data cases
- [ ] Test error scenarios
- [ ] Monitor first executions after deployment

---

## Quick Fixes Reference

| Error Message | Quick Fix |
|---------------|-----------|
| "Cannot read property 'X' of undefined" | Check parent object exists; use optional chaining |
| "X is not a function" | Check variable type |
| "Unexpected token" | Check braces, quotes, syntax |
| Expression shows literal text | Add `{{ }}` around expression |
| "Node not found" | Check nodeType prefix and spelling |
| "Missing required field" | Check operation-specific requirements |
| "ModuleNotFoundError" (Python) | Use standard library only |
| "Invalid expression" | Verify `{{}}`, node names, field paths |
| "Type mismatch" | Convert to correct data type |
| "Invalid value" | Use allowed value from error message |
