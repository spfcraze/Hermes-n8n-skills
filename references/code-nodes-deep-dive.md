# Code Nodes — Deep Dive Reference

Complete guide to JavaScript and Python Code nodes in n8n.

---

## JavaScript Code Node

### Execution Modes

#### Run Once for All Items (Recommended — 95% of cases)

Code executes **once** regardless of input count.

```javascript
// Get all items
const allItems = $input.all();

// Process
const processed = allItems.map(item => ({
  json: {
    id: item.json.id,
    name: item.json.name,
    processed: true
  }
}));

return processed;
```

**Best for**: Aggregation, filtering, batch processing, transformations, API calls with all data.

#### Run Once for Each Item

Code executes **separately** for each input item.

```javascript
// Get current item
const item = $input.item;

return [{
  json: {
    ...item.json,
    processed: true,
    processedAt: new Date().toISOString()
  }
}];
```

**Best for**: Per-item API calls, independent operations, per-item validation.

---

### Data Access Patterns

#### $input.all() — Most Common

```javascript
const items = $input.all();

// Filter
const active = items.filter(item => item.json.status === 'active');

// Map
const mapped = items.map(item => ({
  json: {
    id: item.json.id,
    upperName: item.json.name.toUpperCase()
  }
}));

// Reduce
const total = items.reduce((sum, item) => sum + (item.json.amount || 0), 0);

return [{json: {total, count: items.length}}];
```

#### $input.first() — Single Item

```javascript
const firstItem = $input.first();
const data = firstItem.json;

return [{
  json: {
    result: processData(data),
    processedAt: new Date().toISOString()
  }
}];
```

#### $input.item — Each Item Mode

```javascript
// Only available in "Run Once for Each Item" mode
const currentItem = $input.item;

return [{
  json: {
    ...currentItem.json,
    itemProcessed: true
  }
}];
```

#### $node — Reference Other Nodes

```javascript
// Get output from specific nodes
const webhookData = $node["Webhook"].json;
const httpData = $node["HTTP Request"].json;

return [{
  json: {
    combined: {
      webhook: webhookData.body,
      api: httpData.data
    }
  }
}];
```

---

### Return Format (CRITICAL)

**Must return array of objects with `json` property:**

```javascript
// CORRECT — Single result
return [{json: {result: 'success'}}];

// CORRECT — Multiple results
return [
  {json: {id: 1, data: 'first'}},
  {json: {id: 2, data: 'second'}}
];

// CORRECT — Transformed array
const transformed = $input.all().map(item => ({
  json: {id: item.json.id, processed: true}
}));
return transformed;

// CORRECT — Empty result
return [];

// CORRECT — Conditional
if (shouldProcess) {
  return [{json: processedData}];
} else {
  return [];
}
```

```javascript
// INCORRECT — Missing array wrapper
return {json: {result: 'success'}};

// INCORRECT — Missing json property
return [{result: 'success'}];

// INCORRECT — Plain string
return "processed";

// INCORRECT — Raw input
return $input.all();
```

---

### Built-in Functions

#### $helpers.httpRequest()

Make HTTP requests from within code:

```javascript
const response = await $helpers.httpRequest({
  method: 'POST',
  url: 'https://api.example.com/data',
  headers: {
    'Authorization': 'Bearer ' + $env.API_TOKEN,
    'Content-Type': 'application/json'
  },
  body: {
    name: $json.body.name,
    email: $json.body.email
  }
});

return [{json: {data: response}}];
```

**Options**:
- `method`: GET, POST, PUT, PATCH, DELETE
- `url`: Endpoint URL
- `headers`: Header object
- `body`: Request body (object, auto-JSON-stringified)
- `query`: Query parameters object

#### DateTime (Luxon)

```javascript
const now = DateTime.now();
const formatted = now.toFormat('yyyy-MM-dd');
const iso = now.toISO();
const tomorrow = now.plus({days: 1});
const lastWeek = now.minus({weeks: 1});

return [{
  json: {
    today: formatted,
    tomorrow: tomorrow.toFormat('yyyy-MM-dd'),
    iso: iso
  }
}];
```

#### $jmespath()

Query JSON structures:

```javascript
const data = $input.first().json;
const names = $jmespath(data, 'users[*].name');
const firstEmail = $jmespath(data, 'users[0].email');

return [{json: {names, firstEmail}}];
```

#### $getWorkflowStaticData()

Persistent storage across executions:

```javascript
const staticData = $getWorkflowStaticData('global');

// Read
let counter = staticData.counter || 0;

// Update
counter++;
staticData.counter = counter;

return [{json: {counter}}];
```

---

### Production Patterns

#### Pattern 1: Multi-Source Aggregation

```javascript
const allItems = $input.all();
const results = [];

for (const item of allItems) {
  const source = item.json.source;
  if (source === 'api1' && item.json.data) {
    results.push({
      json: {
        title: item.json.data.title,
        source: 'API1'
      }
    });
  } else if (source === 'api2' && item.json.result) {
    results.push({
      json: {
        title: item.json.result.name,
        source: 'API2'
      }
    });
  }
}

return results;
```

#### Pattern 2: Regex Filtering

```javascript
const pattern = /\b([A-Z]{2,5})\b/g;
const matches = {};

for (const item of $input.all()) {
  const text = item.json.text || '';
  const found = text.match(pattern);
  if (found) {
    found.forEach(match => {
      matches[match] = (matches[match] || 0) + 1;
    });
  }
}

return [{json: {matches}}];
```

#### Pattern 3: Data Transformation

```javascript
const items = $input.all();

return items.map(item => {
  const data = item.json;
  const nameParts = (data.name || '').split(' ');

  return {
    json: {
      first_name: nameParts[0] || '',
      last_name: nameParts.slice(1).join(' ') || '',
      email: data.email?.toLowerCase(),
      created_at: new Date().toISOString()
    }
  };
});
```

#### Pattern 4: Top N Filtering

```javascript
const items = $input.all();

const topItems = items
  .sort((a, b) => (b.json.score || 0) - (a.json.score || 0))
  .slice(0, 10);

return topItems.map(item => ({json: item.json}));
```

#### Pattern 5: Aggregation & Reporting

```javascript
const items = $input.all();
const total = items.reduce((sum, item) => sum + (item.json.amount || 0), 0);

return [{
  json: {
    total,
    count: items.length,
    average: items.length ? total / items.length : 0,
    timestamp: new Date().toISOString()
  }
}];
```

---

## Python Code Node (Beta)

### Important: JavaScript First

**Recommendation**: Use JavaScript for 95% of use cases. Python has significant limitations.

**When to use Python**:
- Need specific Python standard library functions
- Significantly more comfortable with Python
- Data transformations better suited to Python

**Why JavaScript is preferred**:
- Full n8n helper functions ($helpers.httpRequest)
- Luxon DateTime library included
- No external library limitations
- Better documentation and community support

---

### Python Modes

#### Python (Beta) — Recommended

```python
items = _input.all()
now = _now  # Built-in datetime object

return [{
    "json": {
        "count": len(items),
        "timestamp": now.isoformat()
    }
}]
```

#### Python (Native) — Limited

```python
processed = []
for item in _items:
    processed.append({
        "json": {
            "id": item["json"].get("id"),
            "processed": True
        }
    })
return processed
```

---

### Python Data Access

| Pattern | Code |
|---------|------|
| All items | `items = _input.all()` |
| First item | `first = _input.first()` |
| Current item | `item = _input.item` |
| Other node | `data = _node["Node Name"]["json"]` |
| Webhook body | `body = _json.get("body", {})` |

### Python Return Format

```python
# CORRECT
return [{"json": {"result": "success"}}]
return [
    {"json": {"id": 1}},
    {"json": {"id": 2}}
]

# INCORRECT
return {"json": {"result": "success"}}  # Missing list wrapper
return [{"result": "success"}]           # Missing json key
```

### Python Standard Library (Available)

```python
import json        # JSON parsing
import datetime    # Date/time
import re          # Regular expressions
import base64      # Encoding/decoding
import hashlib     # Hashing
import urllib.parse # URL parsing
import math        # Math functions
import random      # Random numbers
import statistics  # Statistical functions
```

### Python Limitations

```python
# NOT AVAILABLE — will raise ModuleNotFoundError
import requests   # No
import pandas     # No
import numpy      # No
import scipy      # No
from bs4 import BeautifulSoup  # No
```

**Workarounds**:
- HTTP requests → Use HTTP Request node or switch to JavaScript
- Data analysis → Use statistics module or JavaScript
- Web scraping → Use HTTP Request + HTML Extract nodes

---

## Error Prevention

### Top 5 JavaScript Code Node Errors

| Rank | Error | Cause | Prevention |
|------|-------|-------|------------|
| 1 | Empty code / missing return | Forgot return | Always end with return |
| 2 | Expression syntax confusion | Using `{{}}` in code | Use direct JS access |
| 3 | Incorrect return wrapper | Missing array/json | Follow `[{json: ...}]` format |
| 4 | Missing null checks | Crashes on undefined | Use optional chaining `?.` |
| 5 | Webhook body nesting | `$json.field` vs `$json.body.field` | Always use `.body` |

### Top 5 Python Code Node Errors

| Rank | Error | Cause | Prevention |
|------|-------|-------|------------|
| 1 | ModuleNotFoundError | Importing external lib | Use stdlib only |
| 2 | Empty code / missing return | Forgot return | Always return list |
| 3 | KeyError | Dict access without .get() | Use `.get()` with defaults |
| 4 | IndexError | List access without check | Check length first |
| 5 | Incorrect return format | Missing list/json | Follow `[{"json": ...}]` |

---

## Debugging Tips

1. **Use console.log() / print()** for intermediate values
2. **Test with empty input** to catch edge cases
3. **Check data structure** with `console.log(JSON.stringify(data, null, 2))`
4. **Use try-catch** for risky operations:

```javascript
try {
  const result = riskyOperation();
  return [{json: {success: true, result}}];
} catch (error) {
  return [{json: {success: false, error: error.message}}];
}
```

5. **Validate output format** before returning:

```javascript
const result = processData();
if (!Array.isArray(result)) {
  throw new Error('Result must be an array');
}
return result;
```
