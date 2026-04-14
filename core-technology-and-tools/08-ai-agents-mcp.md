# AI Agents & Model Context Protocol (MCP) - Study Material

> Based on skills from [ganeshavhad.com](https://ganeshavhad.com) — Software Developer Specialist
> *"Developed AI Agents and custom MCP servers, enabling seamless integration with language models and automating complex development workflows"*
> *"Demonstrated expertise in AI-powered development tools including GitHub Copilot and Cursor"*

---

## 1. AI-Powered Development Tools

### GitHub Copilot
An AI pair programmer that provides code suggestions, completions, and chat-based assistance.

| Feature | Description |
|---------|-------------|
| **Code Completion** | Inline suggestions as you type |
| **Chat** | Ask questions, explain code, generate code |
| **Agents** | Autonomous task execution (Copilot Agent Mode) |
| **Custom Instructions** | `.github/copilot-instructions.md` for repo-specific guidance |
| **MCP Integration** | Connect external tools and data sources |

### Effective Prompting for AI Tools
```
✅ Good Prompts:
- "Create a NestJS service that handles user registration with email validation and password hashing using bcrypt"
- "Refactor this function to use async/await instead of callbacks"
- "Add error handling for network failures with retry logic"

❌ Poor Prompts:
- "Write code"
- "Fix this"
- "Make it better"
```

---

## 2. What is Model Context Protocol (MCP)?

MCP is an **open standard** (by Anthropic) that provides a uniform way to connect AI models to external data sources and tools.

### Architecture

```
┌──────────────────────────────────────────────┐
│              AI Application                   │
│         (VS Code, Claude, etc.)              │
│                                              │
│  ┌─────────────┐  ┌─────────────┐           │
│  │ MCP Client  │  │ MCP Client  │  ...       │
│  └──────┬──────┘  └──────┬──────┘           │
└─────────┼────────────────┼──────────────────┘
          │                │
    ┌─────▼─────┐    ┌────▼──────┐
    │MCP Server │    │MCP Server │
    │(Database) │    │(Git/API)  │
    │           │    │           │
    │ Tools     │    │ Tools     │
    │ Resources │    │ Resources │
    │ Prompts   │    │ Prompts   │
    └───────────┘    └───────────┘
```

### Core MCP Concepts

| Concept | Description | Example |
|---------|-------------|---------|
| **Tools** | Functions the AI can call | `query_database`, `create_file` |
| **Resources** | Data the AI can read | Database schemas, file contents |
| **Prompts** | Reusable prompt templates | Code review checklist |
| **Sampling** | Server can request AI completions | Agentic workflows |

---

## 3. Building a Custom MCP Server (TypeScript)

### Setup

```bash
npm init -y
npm install @modelcontextprotocol/sdk zod
npm install -D typescript @types/node
```

### Basic MCP Server

```typescript
// src/index.ts
import { McpServer } from '@modelcontextprotocol/sdk/server/mcp.js';
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js';
import { z } from 'zod';

const server = new McpServer({
  name: 'project-tools',
  version: '1.0.0'
});

// Define a Tool
server.tool(
  'search_codebase',
  'Search the codebase for a specific pattern',
  {
    query: z.string().describe('The search pattern or keyword'),
    fileType: z.string().optional().describe('File extension filter (e.g., "ts", "js")'),
    maxResults: z.number().default(10).describe('Maximum results to return')
  },
  async ({ query, fileType, maxResults }) => {
    // Implementation
    const results = await performSearch(query, fileType, maxResults);

    return {
      content: [{
        type: 'text',
        text: JSON.stringify(results, null, 2)
      }]
    };
  }
);

// Define a Resource
server.resource(
  'project-config',
  'project://config',
  async (uri) => {
    const config = await readProjectConfig();
    return {
      contents: [{
        uri: uri.href,
        mimeType: 'application/json',
        text: JSON.stringify(config, null, 2)
      }]
    };
  }
);

// Define a Prompt
server.prompt(
  'code-review',
  'Generate a code review checklist for a pull request',
  { language: z.string().describe('Programming language') },
  ({ language }) => ({
    messages: [{
      role: 'user',
      content: {
        type: 'text',
        text: `Review the following ${language} code for:
1. Security vulnerabilities (OWASP Top 10)
2. Performance issues
3. Error handling
4. Code style and best practices
5. Test coverage gaps`
      }
    }]
  })
);

// Start server
const transport = new StdioServerTransport();
await server.connect(transport);
```

### Database Query MCP Server

```typescript
import { McpServer } from '@modelcontextprotocol/sdk/server/mcp.js';
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js';
import { Pool } from 'pg';
import { z } from 'zod';

const pool = new Pool({ connectionString: process.env.DATABASE_URL });

const server = new McpServer({
  name: 'database-tools',
  version: '1.0.0'
});

// List tables
server.tool(
  'list_tables',
  'List all tables in the database',
  {},
  async () => {
    const result = await pool.query(`
      SELECT table_name, table_type 
      FROM information_schema.tables 
      WHERE table_schema = 'public'
      ORDER BY table_name
    `);

    return {
      content: [{
        type: 'text',
        text: JSON.stringify(result.rows, null, 2)
      }]
    };
  }
);

// Describe table
server.tool(
  'describe_table',
  'Get column details for a specific table',
  {
    tableName: z.string().describe('Name of the table')
  },
  async ({ tableName }) => {
    // Validate table name to prevent SQL injection
    const validName = /^[a-zA-Z_][a-zA-Z0-9_]*$/.test(tableName);
    if (!validName) {
      return {
        content: [{ type: 'text', text: 'Invalid table name' }],
        isError: true
      };
    }

    const result = await pool.query(`
      SELECT column_name, data_type, is_nullable, column_default
      FROM information_schema.columns
      WHERE table_name = $1 AND table_schema = 'public'
      ORDER BY ordinal_position
    `, [tableName]);

    return {
      content: [{
        type: 'text',
        text: JSON.stringify(result.rows, null, 2)
      }]
    };
  }
);

// Read-only query (safe for AI usage)
server.tool(
  'run_query',
  'Execute a read-only SQL query',
  {
    query: z.string().describe('SQL SELECT query to execute')
  },
  async ({ query }) => {
    // Only allow SELECT statements
    const normalized = query.trim().toUpperCase();
    if (!normalized.startsWith('SELECT')) {
      return {
        content: [{ type: 'text', text: 'Only SELECT queries are allowed' }],
        isError: true
      };
    }

    try {
      const result = await pool.query(query);
      return {
        content: [{
          type: 'text',
          text: JSON.stringify({
            rowCount: result.rowCount,
            rows: result.rows.slice(0, 100) // Limit results
          }, null, 2)
        }]
      };
    } catch (error) {
      return {
        content: [{ type: 'text', text: `Query error: ${error.message}` }],
        isError: true
      };
    }
  }
);

const transport = new StdioServerTransport();
await server.connect(transport);
```

---

## 4. Configuring MCP in VS Code

```json
// .vscode/mcp.json
{
  "servers": {
    "database-tools": {
      "command": "node",
      "args": ["./mcp-servers/database/dist/index.js"],
      "env": {
        "DATABASE_URL": "postgresql://user:pass@localhost:5432/mydb"
      }
    },
    "project-tools": {
      "command": "npx",
      "args": ["-y", "my-mcp-server"],
      "env": {
        "API_KEY": "${env:API_KEY}"
      }
    }
  }
}
```

---

## 5. AI Agent Patterns

### ReAct Pattern (Reason + Act)
```
Loop:
  1. THINK: Analyze the current state and decide next action
  2. ACT:   Execute a tool call
  3. OBSERVE: Process the result
  4. Repeat until task is complete
```

### Tool-Use Agent
```typescript
// Simplified agent loop concept
async function agentLoop(task: string, tools: Tool[]) {
  const messages = [{ role: 'user', content: task }];

  while (true) {
    const response = await llm.chat(messages, { tools });

    if (response.finishReason === 'stop') {
      return response.content; // Task complete
    }

    if (response.finishReason === 'tool_use') {
      for (const toolCall of response.toolCalls) {
        const tool = tools.find(t => t.name === toolCall.name);
        const result = await tool.execute(toolCall.arguments);

        messages.push({
          role: 'tool',
          toolCallId: toolCall.id,
          content: JSON.stringify(result)
        });
      }
    }
  }
}
```

### Multi-Agent Architecture
```
┌─────────────┐
│ Orchestrator│
│   Agent     │
└──────┬──────┘
       │
  ┌────┼────────────┐
  │    │             │
  ▼    ▼             ▼
┌────┐ ┌─────┐  ┌────────┐
│Code│ │Test │  │Deploy  │
│Gen │ │Agent│  │Agent   │
│Agent│ │     │  │        │
└────┘ └─────┘  └────────┘
```

---

## 6. Phishing Simulation Platform

> *"Designed and Built an End-to-End Phishing Simulation and Training Platform"*

### Architecture Concepts
```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│  Campaign    │    │   Email      │    │  Tracking    │
│  Manager     │───►│   Sender     │───►│  Server      │
│              │    │              │    │              │
│ - Templates  │    │ - SMTP       │    │ - Pixel      │
│ - Targets    │    │ - Scheduling │    │ - Clicks     │
│ - Rules      │    │ - Throttle   │    │ - Reports    │
└──────────────┘    └──────────────┘    └──────┬───────┘
                                               │
                                        ┌──────▼───────┐
                                        │  Training    │
                                        │  Module      │
                                        │              │
                                        │ - Auto-assign│
                                        │ - Courses    │
                                        │ - Progress   │
                                        └──────────────┘
```

---

## 7. Best Practices for AI-Assisted Development

| Practice | Description |
|----------|-------------|
| **Review AI output** | Always review generated code for correctness and security |
| **Provide context** | Give AI enough context via instructions, comments, types |
| **Iterative refinement** | Start broad, refine with follow-up prompts |
| **Custom instructions** | Use `.github/copilot-instructions.md` for project standards |
| **Security scanning** | Run Snyk/SAST on AI-generated code |
| **Test AI code** | Write tests for all AI-generated functions |
| **Version control** | Commit frequently when working with AI |

---

## 8. Interview Questions

1. **What is MCP?** — Model Context Protocol. Open standard for connecting AI models to external tools and data sources via a client-server architecture.
2. **MCP Tools vs Resources?** — Tools: functions AI can execute (side effects). Resources: read-only data AI can access.
3. **What is an AI Agent?** — Autonomous system that uses LLMs + tools to accomplish tasks through iterative reasoning and action.
4. **How to secure MCP servers?** — Validate inputs, restrict to read-only operations, sanitize outputs, use least-privilege access.
5. **ReAct pattern?** — Reasoning + Acting loop. AI thinks, acts (tool call), observes result, repeats.
6. **How does GitHub Copilot work?** — LLM trained on code. Uses context (open files, project structure) to generate relevant suggestions.
7. **What is prompt engineering?** — Crafting inputs to AI models to get desired outputs. Includes system prompts, few-shot examples, chain-of-thought.
8. **How to evaluate AI code quality?** — Code review, automated testing, security scanning, performance benchmarks.

---

## 9. Practice Exercises

1. Build a custom MCP server that queries a PostgreSQL database
2. Create an MCP server exposing a REST API as tools
3. Configure MCP servers in VS Code for a project
4. Build a simple ReAct agent loop with tool calling
5. Create `.github/copilot-instructions.md` with project-specific coding standards
