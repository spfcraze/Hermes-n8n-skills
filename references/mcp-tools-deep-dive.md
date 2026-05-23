# MCP Tools — Deep Dive Reference

Complete guide to n8n-mcp tools for AI-assisted workflow building.

---

## Overview

n8n-mcp provides **20+ tools** organized into categories:

1. **Node Discovery** — Find and understand nodes
2. **Configuration Validation** — Check node and workflow configs
3. **Workflow Management** — Create, edit, and manage workflows
4. **Template Library** — Search and deploy 2,700+ templates
5. **Documentation** — Get tool and node documentation

---

## Tool Availability

### Always Available (no n8n API needed)

- `search_nodes` — Find nodes by keyword
- `get_node` — Get node information
- `validate_node` — Validate node configuration
- `validate_workflow` — Validate workflow JSON
- `search_templates` — Search template library
- `get_template` — Get template details
- `tools_documentation` — Get tool documentation
- `n8n_health_check` — Check connectivity

### Requires n8n API (N8N_API_URL + N8N_API_KEY)

- `n8n_create_workflow` — Create workflow
- `n8n_update_partial_workflow` — Edit workflow incrementally
- `n8n_update_full_workflow` — Replace entire workflow
- `n8n_get_workflow` — Get workflow by ID
- `n8n_list_workflows` — List all workflows
- `n8n_delete_workflow` — Delete workflow
- `n8n_validate_workflow` — Validate by ID
- `n8n_autofix_workflow` — Auto-fix common issues
- `n8n_test_workflow` — Test/trigger workflow
- `n8n_executions` — Manage executions
- `n8n_workflow_versions` — Version history
- `n8n_deploy_template` — Deploy template

---

## Node Discovery Tools

### search_nodes

Find nodes by keyword.

```javascript
search_nodes({
  query: "slack",
  mode: "OR",           // "OR", "AND", "FUZZY"
  limit: 20,
  source: "all",        // "all", "core", "community", "verified"
  includeExamples: true
})
```

**Returns**:
```javascript
{
  nodeType: "nodes-base.slack",           // For get_node/validate_node
  workflowNodeType: "n8n-nodes-base.slack", // For workflow tools
  name: "Slack",
  description: "..."
}
```

### get_node

Get detailed node information.

```javascript
get_node({
  nodeType: "nodes-base.slack",
  detail: "standard",        // "minimal", "standard" (default), "full"
  mode: "info",              // "info", "docs", "search_properties", "versions"
  includeExamples: true
})
```

**Detail levels**:
- `minimal` — Basic info only (~1KB)
- `standard` — Operations, properties, examples (~5KB) — **RECOMMENDED**
- `full` — Complete schema (~100KB+) — Use sparingly

**Modes**:
- `info` — Structured data (default)
- `docs` — Human-readable documentation
- `search_properties` — Find specific properties
- `versions` — Version history

---

## Validation Tools

### validate_node

Check node configuration.

```javascript
validate_node({
  nodeType: "nodes-base.slack",
  config: {
    resource: "message",
    operation: "post",
    channel: "#general",
    text: "Hello!"
  },
  mode: "full",              // "minimal", "full"
  profile: "runtime"         // "minimal", "runtime", "ai-friendly", "strict"
})
```

### validate_workflow

Check workflow JSON structure.

```javascript
validate_workflow({
  workflow: {
    name: "My Workflow",
    nodes: [...],
    connections: {...}
  },
  options: {
    profile: "runtime"
  }
})
```

---

## Workflow Management Tools

### n8n_create_workflow

Create new workflow.

```javascript
n8n_create_workflow({
  name: "New Workflow",
  nodes: [
    {
      id: "node-1",
      name: "Webhook",
      type: "n8n-nodes-base.webhook",
      position: [250, 300],
      parameters: {
        path: "my-webhook",
        responseMode: "responseNode"
      }
    }
  ],
  connections: {}
})
```

### n8n_update_partial_workflow

**Most used workflow editing tool.** Supports 15+ operation types.

```javascript
n8n_update_partial_workflow({
  id: "workflow-id",
  operations: [
    // Add node
    {type: "addNode", node: {...}},

    // Update node
    {type: "updateNode", name: "Node Name", updates: {...}},

    // Remove node
    {type: "removeNode", name: "Node Name"},

    // Add connection
    {type: "addConnection", source: "Node A", target: "Node B"},

    // Remove connection
    {type: "removeConnection", source: "Node A", target: "Node B"},

    // Rename node
    {type: "renameNode", oldName: "Old", newName: "New"},

    // Clean stale connections
    {type: "cleanStaleConnections"}
  ],
  validateOnly: false,       // Set true to preview changes
  continueOnError: false
})
```

### Smart Parameters

For multi-output nodes (IF, Switch), use semantic parameters:

```javascript
// IF node — semantic branch names
{type: "addConnection", source: "IF", target: "True Handler", branch: "true"}
{type: "addConnection", source: "IF", target: "False Handler", branch: "false"}

// Switch node — semantic case numbers
{type: "addConnection", source: "Switch", target: "Handler A", case: 0}
{type: "addConnection", source: "Switch", target: "Handler B", case: 1}
```

### n8n_autofix_workflow

Automatically fix common issues.

```javascript
n8n_autofix_workflow({
  id: "workflow-id",
  applyFixes: true,          // Actually apply (false = preview)
  fixTypes: ["operators", "metadata"],
  confidenceThreshold: 0.8
})
```

**Fixes applied**:
- Binary operator structure
- Unary operator structure
- IF/Switch metadata

### n8n_test_workflow

Test workflow with sample data.

```javascript
n8n_test_workflow({
  workflowId: "workflow-id",
  triggerType: "webhook",
  data: {
    body: {name: "Test", email: "test@example.com"}
  }
})
```

---

## Template Tools

### search_templates

Search 2,700+ workflow templates.

```javascript
search_templates({
  searchMode: "keyword",     // "keyword", "by_task", "by_nodes"
  query: "webhook slack",
  task: "ai_automation",     // For by_task mode
  nodeTypes: ["n8n-nodes-base.webhook"], // For by_nodes mode
  limit: 20
})
```

### get_template

Get template details.

```javascript
get_template({
  templateId: 2947,
  mode: "structure"          // "nodes_only", "structure", "full"
})
```

### n8n_deploy_template

Deploy template as new workflow.

```javascript
n8n_deploy_template({
  templateId: 2947,
  name: "My Weather Bot",
  autoFix: true
})
```

---

## System Tools

### tools_documentation

Get tool documentation.

```javascript
// Overview of all tools
tools_documentation()

// Specific tool
tools_documentation({topic: "search_nodes", depth: "full"})

// JavaScript Code node guide
tools_documentation({topic: "javascript_code_node_guide", depth: "full"})
```

### n8n_health_check

Verify connectivity.

```javascript
n8n_health_check({mode: "diagnostic"})
// Returns: status, features, API availability, version, tool availability
```

---

## Common Workflows

### Pattern 1: Node Discovery

```javascript
// Step 1: Search
const results = await search_nodes({query: "slack", limit: 20});

// Step 2: Get details
const details = await get_node({
  nodeType: results[0].nodeType,
  detail: "standard",
  includeExamples: true
});

// Step 3: Validate config
const validation = await validate_node({
  nodeType: details.nodeType,
  config: {resource: "message", operation: "post", channel: "#general", text: "Hello"},
  profile: "runtime"
});
```

### Pattern 2: Build Workflow Iteratively

```javascript
// Step 1: Create
const workflow = await n8n_create_workflow({
  name: "My Workflow",
  nodes: [triggerNode],
  connections: {}
});

// Step 2: Add nodes
await n8n_update_partial_workflow({
  id: workflow.id,
  operations: [{type: "addNode", node: transformNode}]
});

// Step 3: Connect
await n8n_update_partial_workflow({
  id: workflow.id,
  operations: [{type: "addConnection", source: "Trigger", target: "Transform"}]
});

// Step 4: Validate
await n8n_validate_workflow({id: workflow.id});

// Step 5: Auto-fix
await n8n_autofix_workflow({id: workflow.id, applyFixes: true});
```

### Pattern 3: Template-Based Start

```javascript
// Step 1: Search templates
const templates = await search_templates({searchMode: "keyword", query: "webhook slack"});

// Step 2: Get details
const template = await get_template({templateId: templates[0].id, mode: "structure"});

// Step 3: Deploy
const workflow = await n8n_deploy_template({
  templateId: template.id,
  name: "My Slack Bot",
  autoFix: true
});
```

---

## nodeType Format Reference

### Two Different Formats

| Context | Format | Example |
|---------|--------|---------|
| Discovery/Validation | `nodes-base.*` | `nodes-base.slack` |
| Workflow JSON | `n8n-nodes-base.*` | `n8n-nodes-base.slack` |
| LangChain | `@n8n/n8n-nodes-langchain.*` | `@n8n/n8n-nodes-langchain.agent` |

### Conversion

```javascript
// search_nodes returns BOTH formats
{
  nodeType: "nodes-base.slack",              // For get_node/validate_node
  workflowNodeType: "n8n-nodes-base.slack"   // For workflow tools
}
```

---

## Common Mistakes

### Mistake 1: Wrong nodeType Format

```javascript
// WRONG
get_node({nodeType: "slack"})                    // Missing prefix
get_node({nodeType: "n8n-nodes-base.slack"})     // Wrong prefix for get_node

// CORRECT
get_node({nodeType: "nodes-base.slack"})
```

### Mistake 2: Using detail="full" Unnecessarily

```javascript
// WRONG — Returns 100KB+ data
get_node({nodeType: "nodes-base.slack", detail: "full"})

// CORRECT — Returns ~5KB focused data
get_node({nodeType: "nodes-base.slack", detail: "standard"})
```

### Mistake 3: Not Using Validation Profiles

```javascript
// WRONG — Uses default
validate_node({nodeType, config})

// CORRECT — Explicit profile
validate_node({nodeType, config, mode: "full", profile: "runtime"})
```

### Mistake 4: One-Shot Workflow Building

```javascript
// WRONG — Trying to build everything at once
n8n_update_partial_workflow({id, operations: [addNode, addNode, addConnection, ...]})

// CORRECT — Iterative building
n8n_update_partial_workflow({id, operations: [addNode]})
n8n_update_partial_workflow({id, operations: [addConnection]})
n8n_validate_workflow({id})
```

---

## Best Practices

1. **Use `detail: "standard"`** for most get_node calls
2. **Use `profile: "runtime"`** for validation
3. **Build iteratively** — one operation at a time
4. **Validate after every change**
5. **Use smart parameters** for IF/Switch connections
6. **Search templates first** — don't reinvent
7. **Use auto-fix** before manual debugging
8. **Check health** if tools seem unresponsive
