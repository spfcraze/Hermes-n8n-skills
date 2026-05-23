---
name: n8n-master
description: >-
  Comprehensive n8n workflow automation expert. Covers expression syntax,
  JavaScript/Python Code nodes, node configuration, validation, workflow
  patterns, MCP tools, error handling, and best practices. Use for ANY n8n
  task — building workflows, writing expressions, configuring nodes,
  debugging errors, or designing automation architecture.
category: automation
version: 2.0.0
n8n_version: '>=1.0.0, tested through 2.22.x'
---

# n8n Master Skill

Complete guide to n8n workflow automation — from expressions to production deployments.

---

## Table of Contents

1. [Quick Start](#quick-start)
2. [Expression Syntax](#expression-syntax)
3. [Code Nodes](#code-nodes)
4. [Node Configuration](#node-configuration)
5. [Workflow Patterns](#workflow-patterns)
6. [AI & LangChain](#ai--langchain)
7. [Trigger Data Structures](#trigger-data-structures)
8. [Testing & Debugging](#testing--debugging)
9. [Validation](#validation)
10. [MCP Tools (if available)](#mcp-tools)
11. [Error Catalog](#error-catalog)
12. [Best Practices](#best-practices)

---

## Quick Start

### The 5 Golden Rules

1. **Expressions use `{{ }}`** — Always wrap dynamic content in double curly braces
2. **Webhook data is under `.body`** — `$json.body.field`, not `$json.field`
3. **Code nodes return `[{json: {...}}]`** — Array of objects with `json` property
4. **No `{{ }}` inside Code nodes** — Use JavaScript/Python directly
5. **Validate early, validate often** — Run validation after every configuration change

### Expression vs Code Node Decision Tree

```
Need dynamic value in a field? → EXPRESSION ({{$json.field}})
Need complex logic / transform? → CODE NODE (JavaScript/Python)
Simple field mapping?           → SET NODE (no code needed)
```

---

## Expression Syntax

### Basic Format

All dynamic content uses **double curly braces**:

```
{{expression}}
```

**Examples**:
```javascript
{{$json.email}}                    // Current node output
{{$json.body.name}}                // Webhook payload data
{{$node["HTTP Request"].json.data}} // Data from another node
{{$now.toFormat('yyyy-MM-dd')}}    // Current timestamp
{{$env.API_KEY}}                   // Environment variable
```

### Core Variables Reference

| Variable | Purpose | Example |
|----------|---------|---------|
| `$json` | Current node output | `{{$json.field}}` |
| `$node["Name"]` | Reference other nodes | `{{$node["Webhook"].json.body.email}}` |
| `$now` | Current datetime (Luxon) | `{{$now.toFormat('HH:mm')}}` |
| `$env.VAR` | Environment variables | `{{$env.DATABASE_URL}}` |
| `$input` | Raw input items | `{{$input.first().json.id}}` |
| `$workflow` | Workflow metadata | `{{$workflow.id}}` |
| `$execution` | Execution metadata | `{{$execution.id}}` |

### Webhook Data Structure (CRITICAL)

**Most Common Mistake**: Webhook data is NOT at the root!

```javascript
// Webhook node output structure:
{
  "headers": {...},
  "params": {...},
  "query": {...},
  "body": {           // USER DATA IS HERE
    "name": "John",
    "email": "john@example.com"
  }
}

// CORRECT access:
{{$json.body.name}}     // From current node
{{$node["Webhook"].json.body.email}}  // From Webhook node

// WRONG access:
{{$json.name}}          // undefined — data is under .body
```

### Advanced Expressions

```javascript
// Ternary / conditional
{{$json.status === 'active' ? 'Active' : 'Inactive'}}

// Default values
{{$json.email || 'no-email@example.com'}}

// Date arithmetic
{{$now.plus({days: 7}).toFormat('yyyy-MM-dd')}}
{{$now.minus({hours: 24}).toISO()}}

// String methods
{{$json.email.toLowerCase()}}
{{$json.name.trim().toUpperCase()}}

// Array operations
{{$json.items[0].name}}
{{$json.items.length}}
{{$json.tags.split(',').join(' | ')}}

// Math
{{$json.price * 1.1}}
{{$json.quantity + 5}}
```

### Expression Validation Rules

| Rule | Correct | Incorrect |
|------|---------|-----------|
| Wrap in `{{ }}` | `{{$json.field}}` | `$json.field` |
| Quotes for spaces | `{{$json['field name']}}` | `{{$json.field name}}` |
| Quotes for node names | `{{$node["HTTP Request"]}}` | `{{$node.HTTP Request}}` |
| Case-sensitive names | `{{$node["HTTP Request"]}}` | `{{$node["http request"]}}` |
| No nested braces | `{{$json.field}}` | `{{{$json.field}}}` |

---

## Code Nodes

### Mode Selection

| Mode | When to Use | Data Access | Performance |
|------|-------------|-------------|-------------|
| **Run Once for All Items** (default) | 95% of cases — batch processing, aggregation, filtering | `$input.all()`, `$input.first()` | Faster for multiple items |
| **Run Once for Each Item** | Per-item independent operations | `$input.item` | Slower for large datasets |

**Decision shortcut**: Need to look at multiple items? → All Items. Each item completely independent? → Each Item.

### JavaScript Code Node

```javascript
// Template: Process all items
const items = $input.all();

const processed = items.map(item => ({
  json: {
    ...item.json,
    processed: true,
    timestamp: new Date().toISOString()
  }
}));

return processed;
```

```javascript
// Template: Single item + HTTP request
const response = await $helpers.httpRequest({
  method: 'POST',
  url: 'https://api.example.com/webhook',
  headers: {
    'Authorization': 'Bearer ' + $env.API_TOKEN,
    'Content-Type': 'application/json'
  },
  body: {
    name: $json.body.name,
    email: $json.body.email
  }
});

return [{json: {success: true, id: response.id}}];
```

**Built-in JavaScript helpers**:
- `$helpers.httpRequest()` — HTTP requests from code
- `DateTime` (Luxon) — Advanced date/time operations
- `$jmespath()` — JSON querying
- `$getWorkflowStaticData()` — Persistent storage across executions
- Node.js modules: `crypto`, `Buffer`, `URL`

### Python Code Node (Beta)

```python
# Template: Process all items
items = _input.all()

processed = []
for item in items:
    processed.append({
        "json": {
            **item["json"],
            "processed": True,
            "timestamp": datetime.now().isoformat()
        }
    })

return processed
```

**Critical Python limitations**:
- **NO external libraries** — No requests, pandas, numpy, beautifulsoup
- **Standard library only**: json, datetime, re, base64, hashlib, urllib.parse, math, random, statistics
- **Webhook data under** `_json["body"]`
- **Return format**: `[{"json": {...}}]`
- **Use JavaScript for 95% of cases** — Python only when you specifically need Python stdlib

### Code Node Return Format (CRITICAL)

```javascript
// CORRECT — Array of objects with json property
return [{json: {result: 'success'}}];
return [
  {json: {id: 1, name: 'Alice'}},
  {json: {id: 2, name: 'Bob'}}
];

// INCORRECT — Common mistakes
return {json: {result: 'success'}};     // Missing array wrapper
return [{result: 'success'}];            // Missing json property
return "success";                         // Plain string
return $input.all();                      // Missing .map()
```

### Data Access Patterns

| Pattern | JavaScript | Python | Use Case |
|---------|-----------|--------|----------|
| All items | `$input.all()` | `_input.all()` | Batch processing |
| First item | `$input.first()` | `_input.first()` | Single object/API response |
| Current item | `$input.item` | `_input.item` | Each Item mode |
| Other node | `$node["Name"].json` | `_node["Name"]["json"]` | Cross-node references |
| Webhook body | `$json.body.field` | `_json["body"]["field"]` | Webhook payload |

---

## Node Configuration

### Progressive Discovery Approach

1. **Start with essentials** — Get required fields and common options
2. **Add fields iteratively** — Configure, validate, repeat
3. **Use dependencies when stuck** — Understand what fields unlock others
4. **Full schema as last resort** — Complete documentation for edge cases

### Operation-Aware Configuration

**Not all fields are always required** — it depends on the operation!

```javascript
// Slack: post message
{
  "resource": "message",
  "operation": "post",
  "channel": "#general",   // Required for post
  "text": "Hello!"         // Required for post
}

// Slack: update message (DIFFERENT required fields!)
{
  "resource": "message",
  "operation": "update",
  "messageId": "123",      // Required for update
  "text": "Updated!"       // Required for update
  // channel NOT required
}
```

### Property Dependencies

Fields appear/disappear based on other field values via `displayOptions`:

```javascript
// HTTP Request: POST shows body field
{
  "method": "POST",
  "url": "https://api.example.com",
  "sendBody": true,        // Enables body field
  "body": {                // Now visible and required
    "contentType": "json",
    "content": {...}
  }
}
```

### Common Node Type Prefixes

| Context | Prefix | Example |
|---------|--------|---------|
| Search/Validate/Get | `nodes-base.*` | `nodes-base.slack` |
| Workflow JSON | `n8n-nodes-base.*` | `n8n-nodes-base.slack` |
| LangChain nodes | `@n8n/n8n-nodes-langchain.*` | `@n8n/n8n-nodes-langchain.agent` |

---

## Workflow Patterns

### The 5 Core Patterns

| Pattern | Trigger | Flow | Example |
|---------|---------|------|---------|
| **Webhook Processing** | HTTP request | Webhook → Validate → Transform → Respond | Form submission → Slack notification |
| **HTTP API Integration** | Schedule/Manual | Trigger → HTTP Request → Transform → Action | GitHub issues → Jira tickets |
| **Database Operations** | Schedule | Schedule → Query → Transform → Write | Postgres → MySQL sync |
| **AI Agent Workflow** | Webhook/Chat | Trigger → AI Agent (Model + Tools + Memory) → Output | Conversational AI assistant |
| **Scheduled Tasks** | Cron | Schedule → Fetch → Process → Deliver | Daily analytics report |

### Workflow Data Flow Patterns

```
Linear:     Trigger → Transform → Action → End
Branching:  Trigger → IF → [True] → Action A
                         └→ [False] → Action B
Parallel:   Trigger → [Branch 1] → Merge
                   └→ [Branch 2] ↗
Loop:       Trigger → Split Batches → Process → Loop (until done)
Error:      Main Flow → [Success]
                   └→ [Error Trigger → Handler]
```

### Workflow Creation Checklist

**Planning**:
- [ ] Identify pattern (webhook/API/database/AI/scheduled)
- [ ] List required nodes
- [ ] Understand data flow (input → transform → output)
- [ ] Plan error handling strategy

**Implementation**:
- [ ] Create workflow with trigger
- [ ] Add and configure nodes
- [ ] Set up credentials (never hardcode!)
- [ ] Add transformations
- [ ] Configure error handling

**Validation**:
- [ ] Validate each node
- [ ] Validate complete workflow
- [ ] Test with sample data
- [ ] Handle edge cases (empty data, errors)

**Deployment**:
- [ ] Review settings (execution order, timeout)
- [ ] Activate workflow (manual step in n8n UI)
- [ ] Monitor executions
- [ ] Document purpose and data flow

---

## AI & LangChain

n8n includes **50+ AI nodes** powered by LangChain for building intelligent workflows.

### AI Node Types

| Node Type | Purpose | Example |
|-----------|---------|---------|
| **AI Agent** | Conversational AI with tool access | Customer support bot |
| **Language Model** | LLM integration (OpenAI, Anthropic, etc.) | GPT-4, Claude |
| **AI Tool** | Custom tools the AI can invoke | Search database, send email |
| **AI Memory** | Conversation context persistence | Window buffer, vector store |
| **AI Chain** | LangChain chains for structured tasks | QA, summarization |
| **Output Parser** | Structured output from LLMs | JSON, regex parsing |

### AI Agent Workflow Pattern

```
Trigger → AI Agent
   ├─ Language Model (brain)
   ├─ Memory (context)
   ├─ Tools (capabilities)
   └─ Output Parser (structured response)
→ Action / Response
```

**Example: Customer Support AI**
```
1. Webhook (receive chat message)
2. AI Agent
   ├─ OpenAI Chat Model (ai_languageModel)
   │   System prompt: "You are a helpful support agent"
   ├─ HTTP Request Tool (ai_tool)
   │   Search knowledge base API
   ├─ Database Tool (ai_tool)
   │   Query user order history
   └─ Window Buffer Memory (ai_memory)
       Keep last 10 messages for context
3. Respond to Webhook (send AI reply)
```

### Memory Options

| Memory Type | How It Works | Best For |
|-------------|--------------|----------|
| **Window Buffer** | Keeps last N messages | Simple chatbots, short conversations |
| **Vector Store** | Semantic search over history | Long conversations, knowledge retrieval |
| **No Memory** | Stateless (each call independent) | One-shot tasks, cost-sensitive |

### Tool Configuration

Tools are regular n8n nodes wrapped for AI use:

```javascript
// AI Tool node configuration
{
  "name": "SearchDocs",
  "description": "Search documentation for answers",
  "workflow": "sub-workflow-id"  // Or inline node config
}
```

**Best practices for AI tools**:
- Give clear, descriptive names
- Write detailed descriptions (AI uses these to choose tools)
- Return structured data when possible
- Handle errors gracefully (AI will retry or ask user)

### Expression Variables for AI Nodes

```javascript
{{$json.message}}              // User input to AI
{{$json.response}}             // AI generated response
{{$json.tool_calls}}           // Tools the AI decided to use
{{$json.metadata.model}}       // Which model was used
{{$json.metadata.tokens}}      // Token usage
```

**See**: `references/ai-langchain-deep-dive.md` for comprehensive AI workflow guide.

---

## Trigger Data Structures

**CRITICAL**: Different trigger nodes have DIFFERENT data structures. Using the wrong path causes `undefined` errors.

### Webhook Trigger

```javascript
// Data is under .body
{
  "headers": {...},
  "params": {...},
  "query": {...},
  "body": {                    // USER DATA HERE
    "name": "John",
    "email": "john@example.com"
  }
}

// Access: {{$json.body.name}} or {{$node["Webhook"].json.body.email}}
```

### Schedule Trigger

```javascript
// Data is at ROOT (no .body wrapper)
{
  "timestamp": "2025-01-15T09:00:00.000Z",
  "readableDate": "Wed, 15 Jan 2025 09:00:00 GMT"
}

// Access: {{$json.timestamp}} or {{$json.readableDate}}
// WRONG: {{$json.body.timestamp}} — undefined!
```

### Manual Trigger

```javascript
// Data is at ROOT
{
  "json": {
    "myField": "value"         // Only if you configured input fields
  }
}

// Access: {{$json.myField}}
```

### Error Trigger

```javascript
// Error object structure
{
  "error": {
    "message": "Node execution failed",
    "description": "HTTP request returned 404",
    "node": {
      "name": "HTTP Request",
      "type": "n8n-nodes-base.httpRequest"
    }
  }
}

// Access: {{$json.error.message}} or {{$json.error.node.name}}
```

### Trigger Comparison Table

| Trigger | Data Location | Example Access | Common Mistake |
|---------|--------------|----------------|----------------|
| Webhook | `$json.body` | `{{$json.body.email}}` | Using `$json.email` |
| Schedule | `$json` (root) | `{{$json.timestamp}}` | Using `$json.body.timestamp` |
| Manual | `$json` (root) | `{{$json.myField}}` | Using `$json.body.myField` |
| Error | `$json.error` | `{{$json.error.message}}` | Using `$json.message` |

---

## Testing & Debugging

### Testing Workflows

#### Test Data Injection

Use "Execute Workflow" with test data:
```
1. Open workflow
2. Click "Test Workflow" (play button)
3. For webhook workflows: provide test JSON payload
4. For scheduled workflows: runs immediately with current time
```

#### Pinning Data for Testing

Pin output from any node to reuse in testing:
```
1. Run workflow once (or execute single node)
2. Click node → View output
3. Click "Pin" icon on output data
4. Now you can edit downstream nodes without re-running upstream
```

#### Mock Executions

For nodes that call external APIs:
```javascript
// In Code node, create mock data for testing
const mockData = {
  id: "123",
  status: "active",
  email: "test@example.com"
};

// Use mock instead of real API call during development
// return [{json: mockData}];
```

### Debugging Techniques

#### Execution Log Analysis

```
1. Open Executions tab (left sidebar)
2. Find your workflow execution
3. Click to expand each node
4. Green = success, Red = error, Yellow = warning
5. Click any node to see input/output data
```

#### Step-by-Step Execution

```
1. Enable "Pause after each node" in workflow settings
2. Run workflow
3. Inspect data at each step
4. Continue or stop as needed
```

#### Expression Editor Debug

```
1. Click any field with expression
2. Click "fx" icon
3. Live preview shows result
4. Red highlighting = syntax error
5. Use sample data dropdown to test different inputs
```

#### Code Node Debugging

```javascript
// Add console.log for visibility
console.log("Input data:", $input.all());
console.log("First item:", JSON.stringify($input.first(), null, 2));

// Check data structure
const data = $input.first().json;
console.log("Has body?", data.hasOwnProperty("body"));
console.log("Keys:", Object.keys(data));

return [{json: {debugged: true}}];
```

### Testing Checklist

- [ ] Test with empty input data
- [ ] Test with maximum expected data volume
- [ ] Test error scenarios (bad API response, missing fields)
- [ ] Test webhook with malformed payload
- [ ] Verify all expressions resolve correctly
- [ ] Check Code node return format
- [ ] Validate workflow before activation
- [ ] Monitor first 5 production executions

---

### Validation Profiles

| Profile | Use When | Strictness |
|---------|----------|------------|
| `minimal` | Quick checks during editing | Fast, permissive |
| `runtime` (recommended) | Pre-deployment validation | Balanced |
| `ai-friendly` | AI-generated configurations | Reduces false positives |
| `strict` | Production, critical workflows | Maximum safety |

### The Validation Loop

```
1. Configure node
   ↓
2. Validate (think about errors)
   ↓
3. Fix errors
   ↓
4. Validate again
   ↓
5. Repeat until clean (usually 2-3 iterations)
```

### Common Error Types

| Error | Meaning | Fix |
|-------|---------|-----|
| `missing_required` | Required field not provided | Add the missing field |
| `invalid_value` | Value not in allowed options | Check and use valid value |
| `type_mismatch` | Wrong data type | Convert to correct type |
| `invalid_expression` | Expression syntax error | Check `{{}}`, node names |
| `invalid_reference` | Referenced node doesn't exist | Fix node name spelling |

### Auto-Sanitization

On any workflow update, n8n automatically fixes:
- **Binary operators** (equals, contains) → removes `singleValue`
- **Unary operators** (isEmpty, isNotEmpty) → adds `singleValue: true`
- **IF/Switch nodes** → adds missing metadata

Cannot fix: broken connections, branch count mismatches, corrupt states.

---

## MCP Tools

If you have n8n-mcp tools available, use this workflow:

### Tool Selection Guide

| Task | Tool | Key Parameters |
|------|------|----------------|
| Find nodes | `search_nodes` | `query`, `mode`, `limit` |
| Get node info | `get_node` | `nodeType`, `detail` |
| Validate config | `validate_node` | `nodeType`, `config`, `profile` |
| Create workflow | `n8n_create_workflow` | `name`, `nodes`, `connections` |
| Edit workflow | `n8n_update_partial_workflow` | `id`, `operations` |
| Auto-fix | `n8n_autofix_workflow` | `id`, `applyFixes` |
| Test workflow | `n8n_test_workflow` | `workflowId`, `data` |

### Node Discovery Workflow

```javascript
// Step 1: Search
search_nodes({query: "slack", limit: 20})
// → Returns: nodes-base.slack

// Step 2: Get details
get_node({
  nodeType: "nodes-base.slack",
  detail: "standard",        // "minimal", "standard", "full"
  includeExamples: true
})
// → Returns: operations, properties, examples

// Step 3: Validate
validate_node({
  nodeType: "nodes-base.slack",
  config: {resource: "message", operation: "post", channel: "#general", text: "Hello"},
  mode: "full",
  profile: "runtime"
})
```

### Workflow Editing Pattern

```javascript
// Iterative building — NOT one-shot!
await n8n_update_partial_workflow({
  id: "workflow-id",
  operations: [{type: "addNode", node: {...}}]
});

await n8n_update_partial_workflow({
  id: "workflow-id",
  operations: [{type: "addConnection", source: "A", target: "B"}]
});

await n8n_validate_workflow({id: "workflow-id"});
await n8n_autofix_workflow({id: "workflow-id", applyFixes: true});
```

### Smart Parameters for Multi-Output Nodes

```javascript
// IF node — use semantic branch names
{type: "addConnection", source: "IF", target: "True Handler", branch: "true"}
{type: "addConnection", source: "IF", target: "False Handler", branch: "false"}

// Switch node — use case numbers
{type: "addConnection", source: "Switch", target: "Handler A", case: 0}
```

---

## Error Catalog

### Top 10 n8n Errors and Fixes

| # | Error | Cause | Fix |
|---|-------|-------|-----|
| 1 | `Cannot read property 'X' of undefined` | Missing parent object | Check data path, use optional chaining |
| 2 | Expression shows as literal text | Missing `{{ }}` | Add double curly braces |
| 3 | `X is not a function` | Wrong variable type | Check variable type before calling methods |
| 4 | Empty code / missing return | Forgot return statement | Always return `[{json: {...}}]` |
| 5 | `{{ }}` in Code node | Expression syntax confusion | Use direct JS/Python access |
| 6 | Webhook data undefined | Accessing `$json.field` instead of `$json.body.field` | Add `.body` to path |
| 7 | Return format error | Missing array wrapper or `json` property | Wrap in `[{json: ...}]` |
| 8 | Node not found | Wrong nodeType prefix or typo | Use correct prefix (`nodes-base.*` vs `n8n-nodes-base.*`) |
| 9 | Missing required field | Operation needs different fields | Check operation-specific requirements |
| 10 | `ModuleNotFoundError` (Python) | Importing external library | Use standard library only, or switch to JS |

### Debugging Checklist

- [ ] Check expression editor (fx icon) for live preview
- [ ] Verify webhook data is under `.body`
- [ ] Confirm node names are exact and case-sensitive
- [ ] Test with sample data before activation
- [ ] Check execution log for detailed error traces
- [ ] Use `console.log()` / `print()` in Code nodes for debugging
- [ ] Validate after every configuration change
- [ ] Review workflow settings → Execution Order (v1 recommended)

---

## Best Practices

### Do

- Start with the simplest pattern that solves the problem
- Use descriptive node names
- Add error handling to all workflows
- Test with sample data before activation
- Use credentials system — never hardcode secrets
- Document complex workflows (notes field)
- Monitor executions after deployment
- Use "Run Once for All Items" mode by default
- Access webhook data via `.body`
- Use `.get()` (Python) or optional chaining (JS) for safe access

### Don't

- Build workflows in one shot (iterate!)
- Skip validation before activation
- Use expressions in Code nodes
- Forget to handle empty data cases
- Hardcode API keys or passwords
- Mix multiple patterns without clear boundaries
- Deploy without testing
- Use `detail: "full"` when `standard` suffices
- Ignore error scenarios

### Security Checklist

- [ ] Credentials stored in n8n credential system
- [ ] No secrets in workflow JSON or expressions
- [ ] Webhook paths are non-guessable (for sensitive data)
- [ ] Error handling doesn't leak sensitive data
- [ ] API calls use HTTPS
- [ ] Rate limiting considered for high-volume workflows

---

## Credentials & Deployment

### Credential Configuration

n8n supports **399+ credential types** for secure API authentication.

**Creating Credentials**:
```
1. Left sidebar → Credentials → "Add Credential"
2. Select service (e.g., "Slack API", "OpenAI")
3. Fill in required fields (API key, token, etc.)
4. Save with descriptive name
```

**Using Credentials in Nodes**:
```javascript
// In node configuration, select credential from dropdown
// NOT: hardcoding in expressions or code

// WRONG
apiKey: "xoxb-1234567890"

// CORRECT
// Select "Slack API" credential from node settings panel
```

**Credential Types**:
| Type | Use Case | Example |
|------|----------|---------|
| API Key | Simple key-based auth | OpenAI, SendGrid |
| OAuth2 | User-authorized access | Google, GitHub, Slack |
| Basic Auth | Username/password | Internal APIs |
| Bearer Token | Token-based auth | Custom APIs |
| SSH | Secure shell access | SFTP, Git |

**Environment Variable Credentials**:
```javascript
// For Docker/self-hosted deployments
// Set in .env file or container environment:
N8N_CREDENTIALS_DEFAULT_NAME="Production"

// Reference in credential:
{{$env.MY_API_KEY}}
```

### Self-Hosting Basics

**Docker Deployment**:
```yaml
# docker-compose.yml
version: '3'
services:
  n8n:
    image: n8nio/n8n:latest
    ports:
      - "5678:5678"
    environment:
      - N8N_BASIC_AUTH_ACTIVE=true
      - N8N_BASIC_AUTH_USER=admin
      - N8N_BASIC_AUTH_PASSWORD=secure_password
      - WEBHOOK_URL=https://your-domain.com/
    volumes:
      - ~/.n8n:/home/node/.n8n
```

**Key Environment Variables**:
| Variable | Purpose | Example |
|----------|---------|---------|
| `N8N_BASIC_AUTH_ACTIVE` | Enable basic auth | `true` |
| `N8N_BASIC_AUTH_USER` | Admin username | `admin` |
| `N8N_BASIC_AUTH_PASSWORD` | Admin password | `secure_pass` |
| `WEBHOOK_URL` | Public webhook URL | `https://n8n.example.com/` |
| `N8N_PORT` | HTTP port | `5678` |
| `N8N_PROTOCOL` | HTTP or HTTPS | `https` |
| `DB_TYPE` | Database type | `postgresdb` |
| `DB_POSTGRESDB_HOST` | Postgres host | `postgres` |

**Backup/Restore**:
```bash
# Backup workflows (export all)
n8n export:workflow --all --output=backup/

# Backup credentials (encrypted)
n8n export:credentials --all --output=backup/

# Restore
n8n import:workflow --input=backup/workflows/
n8n import:credentials --input=backup/credentials/
```

---

## Performance & Scaling

### Batch Processing

Process large datasets efficiently:
```
1. Database/HTTP node → returns 10,000 records
2. Split In Batches (batch size: 100)
3. Process each batch
4. Loop back until all batches complete
```

**Recommended batch sizes**:
| Operation | Batch Size | Why |
|-----------|-----------|-----|
| Database inserts | 100-500 | Balance speed vs memory |
| API calls | 10-50 | Respect rate limits |
| Email sends | 50-100 | ESP limits |
| File operations | 20-50 | I/O throughput |

### Rate Limiting

Handle API rate limits gracefully:
```javascript
// In HTTP Request node settings:
// Retry on fail: Yes
// Wait between retries: 1000ms
// Max retries: 3

// Or in Code node with custom backoff:
const delay = (ms) => new Promise(resolve => setTimeout(resolve, ms));
for (const item of items) {
  await makeApiCall(item);
  await delay(100); // 100ms between calls
}
```

### Memory Management

Prevent out-of-memory errors:
- Use "Split In Batches" for large datasets
- Don't load entire datasets into Code node memory
- Use `$input.first()` when only first item needed
- Clear pinned data after testing

### Execution Timeouts

Set appropriate timeouts:
```
Workflow Settings → Execution Timeout
- Default: No timeout
- Recommended: 5 minutes for most workflows
- Long-running: 30 minutes for ETL jobs
```

### Scaling Considerations

| Scale | Recommendation |
|-------|---------------|
| < 100 executions/day | Single instance, SQLite |
| 100-1,000/day | Single instance, Postgres |
| 1,000-10,000/day | Multiple workers, Postgres, Redis |
| > 10,000/day | Queue mode, multiple instances, monitoring |

---

## Related Resources

- **n8n Documentation**: https://docs.n8n.io/
- **Expression Reference**: https://docs.n8n.io/code-examples/expressions/
- **Code Node Guide**: https://docs.n8n.io/code/code-node/
- **Built-in Methods**: https://docs.n8n.io/code-examples/methods-variables-reference/
- **Luxon DateTime**: https://moment.github.io/luxon/
- **n8n Community**: https://community.n8n.io/

---

## Skill Version

- **Version**: 2.0.0
- **n8n Compatibility**: >= 1.0.0, tested through 2.22.x
- **Consolidated from**: n8n-expression-syntax, n8n-code-javascript, n8n-code-python, n8n-node-configuration, n8n-validation-expert, n8n-workflow-patterns, n8n-mcp-tools-expert
- **Last Updated**: 2026-05-23
