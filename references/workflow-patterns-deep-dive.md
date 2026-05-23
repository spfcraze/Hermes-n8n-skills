# Workflow Patterns — Deep Dive Reference

Architectural patterns for building production n8n workflows.

---

## Pattern 1: Webhook Processing

**Use when**: Receiving data from external systems, building integrations, instant response to events.

**Architecture**:
```
Webhook → Validate → Transform → Action → Response
```

**Example: Form Submission to Slack**
```
1. Webhook (path: "form-submit", POST)
   - Receives: {body: {name, email, message}}
2. IF (validate required fields)
   - Check: name AND email exist
3. Set (map/transform fields)
   - Format message for Slack
4. Slack (post to #notifications)
   - Text: "New submission from {{$json.body.name}}"
5. Respond to Webhook
   - Status: 200, Body: {success: true}
```

**Error Handling**:
```
Main Flow → [Success Path]
         └→ [Error Trigger → Slack Alert]
```

**Key Considerations**:
- Webhook data is under `.body`
- Set reasonable timeout for response
- Use "Respond to Webhook" for synchronous responses
- Validate payload before processing

---

## Pattern 2: HTTP API Integration

**Use when**: Fetching data from REST APIs, synchronizing with third-party services, data pipelines.

**Architecture**:
```
Trigger → HTTP Request → Transform → Action → Error Handler
```

**Example: GitHub Issues to Jira**
```
1. Manual Trigger (for testing) / Schedule (for sync)
2. HTTP Request (GET /repos/owner/repo/issues)
   - Authentication: GitHub credential
3. Code (filter and transform)
   - Filter open issues
   - Map to Jira format
4. Split In Batches (process 50 at a time)
5. HTTP Request (POST /jira/rest/api/2/issue)
   - Create Jira ticket for each
6. Loop (back to batch processing)
```

**Pagination Handling**:
```javascript
// In Code node — fetch all pages
const allResults = [];
let page = 1;
let hasMore = true;

while (hasMore && page <= 10) {
  const response = await $helpers.httpRequest({
    method: 'GET',
    url: `https://api.example.com/items?page=${page}`,
    headers: {Authorization: 'Bearer ' + $env.API_TOKEN}
  });

  allResults.push(...response.data);
  hasMore = response.data.length === 100; // Page size
  page++;
}

return allResults.map(item => ({json: item}));
```

---

## Pattern 3: Database Operations

**Use when**: Syncing between databases, scheduled queries, ETL workflows.

**Architecture**:
```
Schedule → Query → Transform → Write → Verify
```

**Example: Postgres to MySQL Sync**
```
1. Schedule (every 15 minutes)
2. Postgres (executeQuery)
   - Query: SELECT * FROM users WHERE updated_at > $last_sync
3. IF (check if records found)
   - Condition: $json.length > 0
4. MySQL (insert/update)
   - Upsert records
5. Postgres (update sync timestamp)
   - UPDATE sync_log SET last_sync = NOW()
```

**Batch Processing**:
```
1. Postgres (query 1000 records)
2. Split In Batches (100 per batch)
3. MySQL (insert batch)
4. Loop (until all batches processed)
```

**Transaction Pattern**:
```
1. Start Transaction
2. Operation A
3. Operation B
4. IF (both succeeded) → Commit
   ELSE → Rollback → Alert
```

---

## Pattern 4: AI Agent Workflow

**Use when**: Building conversational AI, multi-step reasoning, AI with tool access.

**Architecture**:
```
Trigger → AI Agent (Model + Tools + Memory) → Output
```

**Example: Customer Support AI**
```
1. Webhook (receive chat message)
2. AI Agent
   ├─ OpenAI Chat Model (ai_languageModel)
   │   - System prompt: "You are a helpful support agent"
   ├─ HTTP Request Tool (ai_tool)
   │   - Search knowledge base
   ├─ Database Tool (ai_tool)
   │   - Query user history
   └─ Window Buffer Memory (ai_memory)
       - Maintain conversation context
3. Respond to Webhook (send AI reply)
```

**AI Node Types**:
- `ai_languageModel` — LLM (OpenAI, Anthropic, etc.)
- `ai_tool` — Tools the AI can call
- `ai_memory` — Conversation memory (window buffer, vector store)
- `ai_chain` — LangChain chains
- `ai_outputParser` — Structured output parsing

**Memory Options**:
- **Window Buffer Memory**: Keeps last N messages
- **Vector Store Memory**: Semantic search over conversation history
- **No Memory**: Stateless (cheaper, faster)

---

## Pattern 5: Scheduled Tasks

**Use when**: Recurring reports, periodic data fetching, maintenance tasks.

**Architecture**:
```
Schedule → Fetch → Process → Deliver → Log
```

**Example: Daily Analytics Report**
```
1. Schedule (daily at 9:00 AM)
2. HTTP Request (fetch analytics)
   - Date range: yesterday
3. Code (aggregate and format)
   - Calculate metrics
   - Format as HTML/markdown
4. Email (send report)
   - To: team@example.com
   - Subject: Daily Analytics — {{$now.format('yyyy-MM-dd')}}
5. Slack (backup notification)
   - "Daily report sent to {{$json.recipientCount}} recipients"
6. Error Trigger → Slack (alert on failure)
```

**Cron Expression Examples**:
```
Every minute:        * * * * *
Every hour:          0 * * * *
Daily at 9 AM:       0 9 * * *
Weekly on Monday:    0 9 * * 1
Monthly 1st:         0 9 1 * *
Every 15 minutes:    */15 * * * *
```

---

## Data Flow Patterns

### Linear Flow
```
Trigger → Transform → Action → End
```
**Use**: Simple workflows with single path.

### Branching Flow
```
Trigger → IF → [True Path] → Action A
             └→ [False Path] → Action B
```
**Use**: Different actions based on conditions.

### Parallel Processing
```
Trigger → [Branch 1] → Merge
       └→ [Branch 2] ↗
```
**Use**: Independent operations that can run simultaneously.

### Loop Pattern
```
Trigger → Split In Batches → Process → Loop (until done)
```
**Use**: Processing large datasets in chunks.

### Error Handler Pattern
```
Main Flow → [Success Path]
         └→ [Error Trigger → Error Handler]
```
**Use**: Separate error handling workflow.

---

## Common Workflow Components

### Triggers
| Trigger | Use Case | Timing |
|---------|----------|--------|
| Webhook | External events | Instant |
| Schedule | Recurring tasks | Cron-based |
| Manual | Testing/admin | On-demand |
| Polling | Check for changes | Interval |

### Transformation Nodes
| Node | Use Case |
|------|----------|
| Set | Field mapping, simple transforms |
| Code | Complex logic, custom operations |
| IF | Conditional routing |
| Switch | Multi-condition routing |
| Merge | Combine data streams |
| Split In Batches | Process in chunks |

### Output Nodes
| Node | Use Case |
|------|----------|
| HTTP Request | Call APIs |
| Database | Write data |
| Email | Notifications |
| Slack | Team alerts |
| Respond to Webhook | Synchronous response |

---

## Error Handling Strategies

### Strategy 1: Per-Node Continue On Fail

Set node to continue on failure:
```
Node Settings → On Error → Continue
```

### Strategy 2: IF Node Error Check

```
HTTP Request → IF (status === 200)
              ├→ [Success] → Process
              └→ [Error] → Alert
```

### Strategy 3: Error Trigger Workflow

Create separate workflow:
```
Error Trigger → Slack (alert)
             → Email (notification)
             → Log (record error)
```

### Strategy 4: Try-Catch in Code Node

```javascript
try {
  const result = await riskyOperation();
  return [{json: {success: true, result}}];
} catch (error) {
  return [{json: {success: false, error: error.message}}];
}
```

---

## Workflow Statistics

**Most Common Triggers**:
1. Webhook — 35%
2. Schedule — 28%
3. Manual — 22%
4. Service triggers — 15%

**Most Common Transformations**:
1. Set — 68%
2. Code — 42%
3. IF — 38%
4. Switch — 18%

**Average Complexity**:
- Simple (3-5 nodes): 42%
- Medium (6-10 nodes): 38%
- Complex (11+ nodes): 20%

---

## Best Practices

### Planning
- Identify pattern before building
- List required nodes
- Understand data flow
- Plan error handling

### Building
- Start simple, add complexity
- Use descriptive node names
- Configure credentials properly
- Add notes to complex nodes

### Testing
- Test with sample data
- Handle empty data cases
- Test error scenarios
- Validate before activation

### Deployment
- Review execution settings
- Activate manually in UI
- Monitor first executions
- Document workflow purpose
