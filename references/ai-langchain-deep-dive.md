# AI & LangChain — Deep Dive Reference

Comprehensive guide to n8n's AI and LangChain integration for building intelligent workflows.

---

## Overview

n8n includes **50+ AI nodes** powered by LangChain, enabling:
- Conversational AI agents with tool access
- LLM integrations (OpenAI, Anthropic, Google, etc.)
- Structured output parsing
- Memory-powered conversations
- Custom AI tools

---

## AI Node Architecture

### Node Type Hierarchy

```
AI Agent (root)
├── Language Model (brain)
│   ├── OpenAI Chat Model
│   ├── Anthropic Chat Model
│   ├── Google Gemini
│   └── Azure OpenAI
├── Memory (context)
│   ├── Window Buffer Memory
│   ├── Vector Store Memory
│   └── No Memory
├── Tools (capabilities)
│   ├── HTTP Request Tool
│   ├── Database Tool
│   ├── Code Tool
│   └── Custom Workflow Tool
└── Output Parser (structure)
    ├── JSON Output Parser
    ├── CSV Output Parser
    └── Custom Format Parser
```

---

## Language Models

### Supported Providers

| Provider | Node Type | Credential Type | Best For |
|----------|-----------|-----------------|----------|
| **OpenAI** | `ai_languageModel` | OpenAI API | General purpose, GPT-4 |
| **Anthropic** | `ai_languageModel` | Anthropic API | Long context, Claude |
| **Google** | `ai_languageModel` | Google AI | Gemini models |
| **Azure** | `ai_languageModel` | Azure OpenAI | Enterprise, compliance |
| **Local** | `ai_languageModel` | Ollama | Privacy, cost control |

### Configuration

```javascript
// OpenAI Chat Model node
{
  "model": "gpt-4",              // or gpt-3.5-turbo, gpt-4-turbo
  "options": {
    "temperature": 0.7,          // 0 = deterministic, 1 = creative
    "maxTokens": 2000,           // Max response length
    "topP": 1,                   // Nucleus sampling
    "frequencyPenalty": 0,       // Reduce repetition
    "presencePenalty": 0         // Encourage new topics
  }
}
```

**Temperature guide**:
- `0.0-0.3`: Data extraction, classification, structured output
- `0.4-0.7`: General conversation, Q&A
- `0.8-1.0`: Creative writing, brainstorming

---

## Memory Systems

### Window Buffer Memory

Keeps last N messages in conversation.

```javascript
// Configuration
{
  "contextWindowLength": 10      // Number of messages to remember
}

// How it works:
// Message 1: User asks question
// Message 2: AI responds
// Message 3: User asks follow-up
// ... (up to 10 messages)
// Message 11: Oldest message dropped
```

**Best for**: Simple chatbots, short conversations, cost-sensitive applications.

### Vector Store Memory

Stores conversation history as embeddings for semantic search.

```javascript
// Configuration
{
  "vectorStore": "pinecone",     // or supabase, in-memory
  "embeddingModel": "text-embedding-ada-002",
  "maxRetrievedDocs": 5          // How many past messages to retrieve
}

// How it works:
// 1. Each message is converted to embedding vector
// 2. Stored in vector database
// 3. When user asks question, query is embedded
// 4. Most semantically similar past messages retrieved
// 5. Retrieved context added to prompt
```

**Best for**: Long conversations, knowledge retrieval, complex multi-turn tasks.

### No Memory

Stateless — each request is independent.

**Best for**: One-shot tasks, data extraction, classification, cost-sensitive batch processing.

---

## AI Tools

### Built-in Tools

| Tool | Purpose | Configuration |
|------|---------|---------------|
| **HTTP Request Tool** | Call APIs from AI | Method, URL, headers, body |
| **Database Tool** | Query databases | Connection, SQL query |
| **Code Tool** | Execute code | Language, code snippet |
| **Calculator Tool** | Math operations | Expression |

### Custom Workflow Tools

Create reusable tools as sub-workflows:

```javascript
// Tool definition
{
  "name": "SearchOrders",
  "description": "Search customer orders by email or order ID. Returns order details including status, items, and tracking.",
  "workflow": "sub-workflow-search-orders"  // Workflow ID
}

// Tool input (from AI)
{
  "email": "customer@example.com",
  "orderId": "ORD-12345"
}

// Tool output (to AI)
{
  "orders": [
    {"id": "ORD-12345", "status": "shipped", "items": [...]}
  ]
}
```

**Critical**: Tool descriptions must be detailed — the AI uses them to decide which tool to use.

### Tool Best Practices

1. **Name clearly**: `SearchOrders` not `Tool1`
2. **Describe thoroughly**: What it does, what inputs it needs, what it returns
3. **Return structured data**: JSON is easier for AI to parse than plain text
4. **Handle errors**: Return `{error: "..."}` instead of crashing
5. **Limit tools**: 3-5 tools per agent is optimal. Too many = confusion.

---

## AI Agent Configuration

### Basic Agent Setup

```javascript
// AI Agent node
{
  "options": {
    "systemMessage": "You are a helpful customer support agent. Be concise and friendly.",
    "maxIterations": 10,         // Max tool calls per request
    "returnIntermediateSteps": false  // Include tool calls in output?
  }
}
```

### System Message Engineering

```javascript
// Good system message
"You are a technical support specialist for Acme Corp.
- Always verify the user's account before providing sensitive info
- If you don't know something, use the SearchDocs tool
- Keep responses under 3 sentences unless asked for detail
- Escalate billing issues to the Billing tool"

// Bad system message
"You are helpful."  // Too vague
```

### Agent Types

| Type | Behavior | Best For |
|------|----------|----------|
| **Conversational** | Chat-like, uses memory | Customer support, Q&A |
| **OpenAI Functions** | Structured tool calling | Reliable tool use |
| **ReAct** | Reasoning + Acting | Complex multi-step tasks |
| **Plan and Execute** | Plans first, then executes | Structured workflows |

---

## Output Parsers

### JSON Output Parser

Force AI to return valid JSON:

```javascript
// Output Parser node
{
  "schema": {
    "type": "object",
    "properties": {
      "sentiment": {"type": "string", "enum": ["positive", "negative", "neutral"]},
      "confidence": {"type": "number", "minimum": 0, "maximum": 1},
      "topics": {"type": "array", "items": {"type": "string"}}
    },
    "required": ["sentiment", "confidence"]
  }
}
```

### CSV Output Parser

```javascript
{
  "schema": {
    "type": "csv",
    "columns": ["name", "email", "role"],
    "delimiter": ","
  }
}
```

### Custom Format Parser

```javascript
{
  "format": "Name: {name}\nEmail: {email}\nRole: {role}",
  "variables": ["name", "email", "role"]
}
```

---

## Expression Variables for AI

### Input Variables

```javascript
{{$json.message}}              // User's input message
{{$json.history}}              // Conversation history (if memory enabled)
{{$json.context}}              // Additional context passed to agent
```

### Output Variables

```javascript
{{$json.response}}             // AI's text response
{{$json.output}}               // Structured output (with parser)
{{$json.tool_calls}}           // Tools the AI invoked
{{$json.intermediateSteps}}    // Step-by-step reasoning (if enabled)
```

### Metadata Variables

```javascript
{{$json.metadata.model}}       // Model used
{{$json.metadata.tokens}}      // Token usage {prompt, completion, total}
{{$json.metadata.finishReason}} // Why generation stopped
```

---

## Common AI Patterns

### Pattern 1: Customer Support Bot

```
Webhook (chat message)
  → AI Agent
    ├─ System: "You are a support agent..."
    ├─ Memory: Window Buffer (last 10 messages)
    ├─ Tools:
    │   ├─ SearchKnowledgeBase (HTTP Request)
    │   ├─ CheckOrderStatus (Database)
    │   └─ CreateTicket (HTTP Request)
    └─ Output: JSON Parser {response, escalation_needed}
  → IF (escalation_needed)
    ├─ True: Send to human (Slack alert)
    └─ False: Respond to webhook
```

### Pattern 2: Data Extraction

```
Webhook (document upload)
  → Read File (get text)
  → AI Agent (No Memory)
    ├─ System: "Extract key information from this document"
    ├─ Model: GPT-4 (temperature 0.1)
    └─ Output: JSON Parser
       {invoice_number, date, total, items: [{name, qty, price}]}
  → Database (store extracted data)
```

### Pattern 3: Content Generation

```
Schedule (daily at 9 AM)
  → HTTP Request (fetch trending topics)
  → AI Agent (No Memory)
    ├─ System: "Write a blog post about this topic"
    ├─ Model: GPT-4 (temperature 0.8)
    └─ Output: Plain text
  → Email (send draft to editor)
```

### Pattern 4: Classification & Routing

```
Webhook (incoming message)
  → AI Agent (No Memory)
    ├─ System: "Classify this message into: sales, support, billing, spam"
    ├─ Model: GPT-3.5 (temperature 0)
    └─ Output: JSON Parser {category, confidence}
  → Switch (route by category)
    ├─ sales → CRM node
    ├─ support → Ticket system
    ├─ billing → Accounting
    └─ spam → Discard
```

---

## Error Handling for AI Workflows

### Common AI Errors

| Error | Cause | Fix |
|-------|-------|-----|
| "Model not found" | Wrong model name | Check model identifier |
| "Rate limit exceeded" | Too many requests | Add delay, reduce batch size |
| "Context length exceeded" | Input too long | Truncate, summarize, or use vector memory |
| "Invalid JSON" | Output parser failed | Lower temperature, improve schema description |
| "Tool not found" | AI tried non-existent tool | Check tool names and descriptions |
| "Max iterations reached" | AI looping | Increase maxIterations or simplify task |

### Retry Strategy

```javascript
// In AI Agent node settings:
{
  "retryOnFail": true,
  "maxRetries": 3,
  "waitBetweenRetries": 1000
}
```

### Fallback Pattern

```
AI Agent (primary: GPT-4)
  → IF (error OR timeout)
    → AI Agent (fallback: GPT-3.5)
      → IF (still error)
        → Static response: "I'm having trouble. Please try again."
```

---

## Cost Optimization

### Model Selection by Task

| Task | Recommended Model | Cost Level |
|------|------------------|------------|
| Classification | GPT-3.5-turbo | $ |
| Data extraction | GPT-3.5-turbo | $ |
| General chat | GPT-3.5-turbo | $ |
| Complex reasoning | GPT-4 | $$ |
| Creative writing | GPT-4 | $$ |
| Code generation | GPT-4-turbo | $$ |
| Long documents | Claude (100K context) | $$ |

### Token Management

```javascript
// Estimate tokens (rough: 1 token ≈ 4 chars)
const estimateTokens = (text) => Math.ceil(text.length / 4);

// Monitor usage
console.log("Prompt tokens:", $json.metadata.tokens.prompt);
console.log("Completion tokens:", $json.metadata.tokens.completion);
console.log("Total cost:", ($json.metadata.tokens.total / 1000) * 0.002); // $0.002 per 1K tokens for GPT-3.5
```

### Caching Responses

```javascript
// Use $getWorkflowStaticData for simple caching
const staticData = $getWorkflowStaticData('global');
const cacheKey = JSON.stringify($json.message);

if (staticData[cacheKey]) {
  return [{json: {cached: true, response: staticData[cacheKey]}}];
}

// ... call AI ...
staticData[cacheKey] = aiResponse;
return [{json: {cached: false, response: aiResponse}}];
```

---

## Security Considerations

- **Never expose API keys** in workflow JSON or expressions
- **Use credentials system** for all LLM providers
- **Sanitize user input** before sending to AI (prompt injection risk)
- **Review AI outputs** before acting on them (hallucinations)
- **Log AI interactions** for audit trails
- **Set maxTokens** to prevent runaway generation
- **Use output parsers** to constrain AI responses to expected formats

---

## Quick Reference

```
AI WORKFLOW CHECKLIST:
□ Choose appropriate model (cost vs capability)
□ Set temperature (0 for structured, 0.7+ for creative)
□ Configure memory (Window for short, Vector for long)
□ Define tools with clear descriptions
□ Set maxIterations (prevent infinite loops)
□ Add output parser for structured data
□ Test with edge cases (long inputs, ambiguous queries)
□ Monitor token usage and costs
□ Add error handling and fallback
□ Review AI outputs before production deployment
```
