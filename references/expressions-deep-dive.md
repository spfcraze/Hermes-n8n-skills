# n8n Expressions — Deep Dive Reference

Comprehensive expression syntax guide for n8n workflow automation.

---

## Expression Format Rules

All dynamic content uses **double curly braces**:

```
{{expression}}
```

| Correct | Incorrect |
|---------|-----------|
| `{{$json.email}}` | `$json.email` (no braces) |
| `{{$json['field name']}}` | `{{$json.field name}}` (spaces) |
| `{{$node["HTTP Request"].json.data}}` | `{{$node.HTTP Request.json}}` (no quotes) |
| `{{$json.field}}` | `{{{$json.field}}}` (nested braces) |

---

## Variable Reference

### $json — Current Node Output

```javascript
{{$json.fieldName}}                    // Simple field
{{$json['field with spaces']}}         // Bracket notation
{{$json.nested.property}}              // Nested object
{{$json.items[0].name}}                // Array access
{{$json.items[$json.items.length - 1]}} // Last item
```

### $node — Reference Other Nodes

```javascript
{{$node["Node Name"].json.fieldName}}  // Standard format
{{$node["Webhook"].json.body.email}}   // Webhook node data
{{$node["HTTP Request"].json.data}}    // API response data
```

**Rules**:
- Node names **must** be in quotes
- Node names are **case-sensitive**
- Must match exact node name from workflow

### $now — Current Timestamp (Luxon)

```javascript
{{$now}}                               // ISO datetime
{{$now.toFormat('yyyy-MM-dd')}}        // Date only
{{$now.toFormat('HH:mm:ss')}}          // Time only
{{$now.toFormat('yyyy-MM-dd HH:mm')}}  // DateTime
{{$now.plus({days: 7})}}               // Add time
{{$now.minus({hours: 24})}}            // Subtract time
{{$now.toISO()}}                       // ISO format
{{$now.toLocal()}}                     // Local timezone
```

### $env — Environment Variables

```javascript
{{$env.API_KEY}}
{{$env.DATABASE_URL}}
{{$env.WEBHOOK_SECRET}}
```

**Note**: Environment variables must be set in n8n configuration.

### $input — Raw Input Items

```javascript
{{$input.first().json.id}}             // First item
{{$input.all()[0].json.name}}          // Same as first
{{$input.item.json.field}}             // Current item (Each Item mode)
```

### $workflow — Workflow Metadata

```javascript
{{$workflow.id}}                       // Workflow ID
{{$workflow.name}}                     // Workflow name
```

### $execution — Execution Metadata

```javascript
{{$execution.id}}                      // Execution ID
{{$execution.timestamp}}               // Start time
{{$execution.mode}}                    // execution mode
```

---

## Webhook Data Structure (CRITICAL)

**Most Common Mistake**: Webhook data is nested under `.body`!

```javascript
// Webhook node output:
{
  "headers": {
    "content-type": "application/json",
    "user-agent": "..."
  },
  "params": {},
  "query": {},
  "body": {                    // USER DATA IS HERE
    "name": "John Doe",
    "email": "john@example.com",
    "message": "Hello!"
  }
}

// CORRECT access patterns:
{{$json.body.name}}                    // Current node (if webhook is previous)
{{$node["Webhook"].json.body.email}}   // From specific webhook node
{{$json.body.message}}                 // Payload field

// WRONG access patterns:
{{$json.name}}                         // undefined — not at root
{{$json.email}}                        // undefined
```

**Why**: Webhook node wraps all request data under `body` to preserve headers, params, and query parameters separately.

---

## Advanced Patterns

### Conditional Content

```javascript
// Ternary operator
{{$json.status === 'active' ? 'Active User' : 'Inactive User'}}

// Default/fallback values
{{$json.email || 'no-email@example.com'}}
{{$json.name || 'Anonymous'}}

// Null coalescing style
{{$json.phone ?? 'N/A'}}
```

### Date Manipulation

```javascript
// Add time
{{$now.plus({days: 7}).toFormat('yyyy-MM-dd')}}
{{$now.plus({months: 1, days: 5})}}

// Subtract time
{{$now.minus({hours: 24}).toISO()}}
{{$now.minus({weeks: 2})}}

// Specific date
{{DateTime.fromISO('2025-12-25').toFormat('MMMM dd, yyyy')}}

// Date diff
{{$now.diff(DateTime.fromISO($json.created_at), 'days').days}}
```

### String Manipulation

```javascript
// Case
{{$json.email.toLowerCase()}}
{{$json.name.toUpperCase()}}

// Substring
{{$json.email.substring(0, 5)}}

// Replace
{{$json.message.replace('old', 'new')}}

// Split and join
{{$json.tags.split(',').join(' | ')}}

// Trim
{{$json.name.trim()}}

// Includes
{{$json.email.includes('@company.com') ? 'Internal' : 'External'}}
```

### Array Operations

```javascript
// First item
{{$json.users[0].email}}

// Last item
{{$json.users[$json.users.length - 1].name}}

// Length
{{$json.users.length}}

// Map (limited)
{{$json.items.map(i => i.name).join(', ')}}

// Filter (limited)
{{$json.items.filter(i => i.active).length}}
```

### Number Operations

```javascript
// Math
{{$json.price * 1.1}}                  // Add 10%
{{$json.quantity + 5}}
{{$json.total / $json.count}}
{{$json.value % 2}}

// Formatting
{{$json.price.toFixed(2)}}
{{$json.amount.toString()}}
```

---

## When NOT to Use Expressions

### Code Nodes

Code nodes use **direct language access**, NOT expressions:

```javascript
// WRONG in Code node
const email = '={{$json.email}}';
const name = '{{$json.body.name}}';

// CORRECT in Code node (JavaScript)
const email = $json.email;
const name = $json.body.name;
const allItems = $input.all();
```

```python
# WRONG in Python Code node
email = "{{$json.email}}"

# CORRECT in Python Code node
email = _json["email"]
body = _json["body"]
```

### Webhook Paths

```javascript
// WRONG — dynamic paths not supported
path: "{{$json.user_id}}/webhook"

// CORRECT — static paths only
path: "user-webhook"
path: "form-submit"
```

### Credential Fields

```javascript
// WRONG — never use expressions for secrets
apiKey: "={{$env.API_KEY}}"

// CORRECT — use n8n credential system
// Configure credentials in node settings panel
```

---

## Expression Helpers

### Available Methods

**String**:
- `.toLowerCase()`, `.toUpperCase()`
- `.trim()`, `.replace()`, `.substring()`
- `.split()`, `.includes()`, `.startsWith()`, `.endsWith()`

**Array**:
- `.length`, `.map()`, `.filter()`
- `.find()`, `.join()`, `.slice()`, `.includes()`

**DateTime (Luxon)**:
- `.toFormat()`, `.toISO()`, `.toLocal()`
- `.plus()`, `.minus()`, `.set()`, `.diff()`

**Number**:
- `.toFixed()`, `.toString()`, `.toPrecision()`
- Math operations: `+`, `-`, `*`, `/`, `%`, `**`

**Object**:
- `Object.keys()`, `Object.values()`, `Object.entries()`
- `JSON.stringify()`, `JSON.parse()`

---

## Debugging Expressions

### Expression Editor

1. Click field with expression
2. Open expression editor (click "fx" icon)
3. See live preview of result
4. Check for errors highlighted in red

### Common Error Messages

| Error | Cause | Solution |
|-------|-------|----------|
| "Cannot read property 'X' of undefined" | Parent object doesn't exist | Check data path, verify structure |
| "X is not a function" | Calling method on wrong type | Check variable type |
| "Unexpected token" | Syntax error in expression | Check braces, quotes, operators |
| Shows as literal text | Missing `{{ }}` | Add double curly braces |
| `[Object object]` | Trying to display object | Access specific property |

---

## Quick Reference Card

```
ESSENTIAL RULES:
1. Wrap in {{ }}
2. Webhook data is under .body
3. No {{ }} in Code nodes
4. Use quotes for node names with spaces
5. Use bracket notation for fields with spaces
6. Node names are case-sensitive
7. Test in expression editor
```
