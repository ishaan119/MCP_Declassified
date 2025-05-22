# MCP Demystified: How Exactly It Works

## Introduction

You've probably heard about Model Context Protocol (MCP) - a standardized way for LLMs to interact with external systems. But how does this actually work under the hood?

Most engineers understand REST APIs, GraphQL, and HTTP. MCP is different, and if you're like me, you might be wondering: "What's the transport? How does discovery work? What are the core primitives?"

This article will demystify MCP by explaining exactly how it works at the protocol level, with practical examples you can run today.

## What MCP Actually Is

**Think of MCP as JSON-RPC over stdio designed specifically for AI interactions.** Your MCP server acts as a protocol bridge between Claude and your existing systems.

```
Claude Desktop ←--stdio--→ MCP Server ←--HTTP--→ Your REST API ←--→ Database
   (Client)              (Protocol Bridge)      (Business Logic)    (Data)
```

### Key Insights

**Process Communication**: MCP servers are spawned as child processes and communicate via stdin/stdout. This eliminates network configuration while allowing your server to make normal HTTP calls to existing APIs.

**No Backend Changes**: You don't need to modify your existing backend systems. Your REST API, database schema, and business logic stay exactly the same. MCP just adds a thin translation layer.

## The Three Core Primitives

MCP defines exactly three types of capabilities that map to familiar API patterns:

### 1. Tools (Function Calls)
- **Purpose**: Actions Claude can execute
- **Control**: Claude autonomously decides when to call them
- **API Analogy**: POST/PUT/DELETE endpoints that change state
- **When to use**: Creating records, sending emails, triggering workflows

```python
@mcp.tool()
async def create_support_ticket(title: str, description: str, priority: str) -> str:
    """Create a new support ticket in our system"""
    async with httpx.AsyncClient() as client:
        response = await client.post(f"{API_BASE}/tickets", 
            headers={"Authorization": f"Bearer {API_TOKEN}"},
            json={
                "title": title,
                "description": description, 
                "priority": priority,
                "status": "open"
            })
        ticket = response.json()
        return f"Created ticket #{ticket['id']}: {ticket['title']}"
```

### 2. Resources (Data Access)
- **Purpose**: Information Claude can read for context
- **Control**: Claude requests when it needs background information
- **API Analogy**: GET endpoints that provide read-only data
- **When to use**: User profiles, configuration data, documentation

```python
@mcp.resource("tickets://recent")
async def get_recent_tickets() -> str:
    """Get recent support tickets for context"""
    async with httpx.AsyncClient() as client:
        response = await client.get(f"{API_BASE}/tickets/recent",
            headers={"Authorization": f"Bearer {API_TOKEN}"})
        tickets = response.json()
        
        # Format for Claude's consumption
        summary = "Recent Support Tickets:\n"
        for ticket in tickets:
            summary += f"- #{ticket['id']}: {ticket['title']} ({ticket['status']})\n"
        return summary
```

### 3. Prompts (Templates)
- **Purpose**: Structured interaction patterns with your data
- **Control**: User explicitly invokes them via Claude's UI
- **API Analogy**: Parameterized report generators
- **When to use**: Analysis workflows, report templates, guided interactions

```python
@mcp.prompt()
async def customer_health_check(customer_id: str) -> str:
    """Generate a comprehensive customer health analysis"""
    async with httpx.AsyncClient() as client:
        # Gather data from multiple endpoints
        customer = await client.get(f"{API_BASE}/customers/{customer_id}")
        tickets = await client.get(f"{API_BASE}/customers/{customer_id}/tickets")
        usage = await client.get(f"{API_BASE}/customers/{customer_id}/usage")
        
        return f"""Analyze this customer's health:

**Customer Profile:**
{json.dumps(customer.json(), indent=2)}

**Recent Support Activity:**
{json.dumps(tickets.json(), indent=2)}

**Usage Metrics:**
{json.dumps(usage.json(), indent=2)}

Please provide a health score (1-10) and actionable recommendations."""
```

## How the Protocol Works

### Architecture Stack

```
┌─────────────────────────────────────┐
│   Your Business Logic & HTTP APIs   │  ← Unchanged existing systems
├─────────────────────────────────────┤
│   MCP Methods (tools, resources)    │  ← Your server implementation
├─────────────────────────────────────┤
│   JSON-RPC 2.0 Message Format       │  ← Protocol specification
├─────────────────────────────────────┤
│   stdio Transport (pipes)           │  ← Process communication
└─────────────────────────────────────┘
```

- **Transport Layer**: stdin/stdout pipes between Claude and your server process
- **Protocol Layer**: JSON-RPC 2.0 for structured request/response messages
- **Application Layer**: Your server translates MCP calls to HTTP API calls
- **Business Layer**: Your existing REST APIs and database remain unchanged

### Discovery Protocol: The Exact Sequence

Here's what happens when Claude starts your MCP server:

**Step 1: Process Spawning**
```bash
Claude executes: python /path/to/your/server.py
```

**Step 2: Handshake**
```
- Claude: "I'm Claude Desktop v1.0, what can you do?"

- Your server: "I'm WeatherAPI Bridge v1.0, I can call weather tools and provide config data"
```

**Step 3: Tool Discovery**
```
- Claude: "What functions do you have?"

- Your server: "I have get_weather(city) and send_alert(message, urgency)"
```

**Step 4: Resource Discovery**
```
- Claude: "What data can you provide?"

- Your server: "I can provide current config at config://app and user data at users://active"
```

**Step 5: Runtime Usage**
```
- Claude: "Execute get_weather with city=Tokyo"

- Your server: Makes HTTP call to `POST /api/weather` → Returns formatted result
```


## Why This Design Works

**Process Isolation**: Each MCP server runs in its own process with controlled environment

**Simple Transport**: stdio eliminates network configuration and security concerns

**Familiar Patterns**: JSON-RPC is a well-established standard with robust tooling

**Minimal Changes**: Your existing APIs and infrastructure remain completely unchanged

**Secure by Default**: No exposed ports, authentication handled at the API level

## Getting Started

The beauty of MCP is that **your backend doesn't change**. Your REST API, database, and business logic remain exactly the same. You just add a thin MCP bridge that translates between Claude's requests and your API calls.

This design lets you expose any existing system to Claude without architectural changes - just wrap your APIs with MCP tools and resources.

## Resources

- [MCP Specification](https://spec.modelcontextprotocol.io/)
- [Official Python SDK](https://github.com/modelcontextprotocol/python-sdk)
- [Complete bridge examples](https://github.com/your-repo/mcp-api-bridges)
