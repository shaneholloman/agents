# MCP Server Development Guide

## Overview

To create high-quality MCP (Model Context Protocol) servers that enable LLMs to effectively interact with external services, use this guide. An MCP server provides tools that allow LLMs to access external services and APIs. The quality of an MCP server is measured by how well it enables LLMs to accomplish real-world tasks using the tools provided.

---

## Table of Contents

1. [High-Level Workflow](#high-level-workflow)
2. [Phase 1: Deep Research and Planning](#phase-1-deep-research-and-planning)
3. [Phase 2: Design and Architecture](#phase-2-design-and-architecture)
4. [Phase 3: Implementation](#phase-3-implementation)
5. [Phase 4: Testing and Validation](#phase-4-testing-and-validation)
6. [Best Practices](#best-practices)
7. [Common Patterns](#common-patterns)
8. [Examples](#examples)
9. [Troubleshooting](#troubleshooting)

---

## High-Level Workflow

Creating a high-quality MCP server involves four main phases:

1. **Deep Research and Planning** - Understand the service, user needs, and design principles
2. **Design and Architecture** - Create tool schemas and plan implementation
3. **Implementation** - Build the MCP server with best practices
4. **Testing and Validation** - Ensure quality and usability

---

## Phase 1: Deep Research and Planning

### 1.1 Understand Agent-Centric Design Principles

Before diving into implementation, understand how to design tools for AI agents:

#### Build for Workflows, Not Just API Endpoints

- **Don't simply wrap existing API endpoints** - build thoughtful, high-impact workflow tools
- **Consolidate related operations** (e.g., `schedule_event` that both checks availability and creates event)
- **Focus on tools that enable complete tasks**, not just individual API calls
- **Consider what workflows agents actually need to accomplish**

**Example:**
```typescript
// ❌ Bad: Just wrapping API endpoints
check_calendar_availability()
create_calendar_event()
send_event_invitation()

// ✅ Good: Workflow-oriented tool
schedule_meeting({
  title: "Team Sync",
  attendees: ["alice@example.com", "bob@example.com"],
  duration_minutes: 30,
  preferred_time_slots: ["2025-01-20T14:00:00Z", "2025-01-20T15:00:00Z"]
})
// Returns: Automatically finds availability, creates event, and sends invites
```

#### Optimize for Limited Context

Agents have constrained context windows - make every token count:

- **Return high-signal information**, not exhaustive data dumps
- **Provide concise vs detailed response format options**
- **Default to human-readable identifiers** over technical codes (names over IDs)
- **Consider the agent's context budget as a scarce resource**

**Example:**
```typescript
// ❌ Bad: Verbose, low-signal response
{
  "id": "msg_abc123xyz",
  "created_at": "2025-01-18T10:30:45.123Z",
  "updated_at": "2025-01-18T10:30:45.123Z",
  "deleted_at": null,
  "version": 1,
  "metadata": {},
  "content": "Hello",
  "author_id": "usr_456def",
  "author_name": "Alice",
  "author_email": "alice@example.com",
  "author_avatar": "https://...",
  "channel_id": "ch_789ghi",
  "channel_name": "general",
  "reactions": [],
  "thread_count": 0
}

// ✅ Good: Concise, high-signal response
{
  "message": "Alice: Hello",
  "channel": "general",
  "timestamp": "10:30 AM",
  "id": "msg_abc123xyz"  // Include ID for follow-up actions
}
```

#### Design Actionable Error Messages

Error messages should guide agents toward correct usage:

- **Suggest specific next steps**: "Try X instead" or "First call Y"
- **Include corrective examples** showing the right approach
- **Provide context about why the error occurred**
- **Make errors recoverable** when possible

**Example:**
```typescript
// ❌ Bad: Vague error
"Invalid request"

// ✅ Good: Actionable error
{
  "error": "Cannot schedule meeting: No available time slots found",
  "suggestion": "Try expanding the time range or reducing the duration",
  "next_steps": [
    "Call find_availability() with a wider date range",
    "Check attendee calendars individually with get_calendar()"
  ],
  "example": {
    "tool": "find_availability",
    "args": {
      "attendees": ["alice@example.com"],
      "start_date": "2025-01-20",
      "end_date": "2025-01-27",  // Expanded range
      "duration_minutes": 30
    }
  }
}
```

#### Make Tools Composable

- **Each tool should do one thing well** and work with other tools
- **Use consistent data formats** across tools
- **Return IDs/references** that can be used in subsequent calls
- **Document tool combinations** in descriptions

**Example:**
```typescript
// Tools work together in a natural flow:
// 1. Search for files
const results = search_files({ query: "budget 2025" })
// Returns: [{ id: "file_123", name: "budget_2025.xlsx", ... }]

// 2. Get specific file
const file = get_file({ file_id: "file_123" })

// 3. Share file
share_file({
  file_id: "file_123",
  users: ["alice@example.com"],
  permission: "edit"
})
```

#### Enable Progressive Disclosure

- **Start simple, allow complexity** when needed
- **Provide sensible defaults** for optional parameters
- **Support both quick actions and detailed control**
- **Use optional parameters** for advanced features

**Example:**
```typescript
// Simple use case - sensible defaults
send_email({
  to: "alice@example.com",
  subject: "Quick question",
  body: "Can we meet tomorrow?"
})

// Advanced use case - full control when needed
send_email({
  to: ["alice@example.com", "bob@example.com"],
  cc: ["manager@example.com"],
  subject: "Q4 Planning Meeting",
  body: "...",
  attachments: ["file_123"],
  schedule_send: "2025-01-20T09:00:00Z",
  priority: "high",
  request_read_receipt: true,
  tracking_enabled: false
})
```

### 1.2 Research the Service

Thoroughly understand the service you're building for:

#### Study the Official Documentation

- **Read the complete API documentation**
- **Understand authentication methods** (API keys, OAuth, tokens)
- **Note rate limits and quotas**
- **Review error codes and handling**
- **Check for webhooks or real-time features**

#### Analyze Common Use Cases

- **What do users typically do** with this service?
- **What are the most frequent workflows?**
- **What integrations exist?**
- **What pain points do users have?**

#### Research Existing Integrations

- **Study existing SDKs and libraries**
- **Look at competitor integrations**
- **Review community tools and wrappers**
- **Identify gaps and opportunities**

### 1.3 Identify User Needs

Understand who will use your MCP server and what they need:

#### Define Your Target Users

- **Who will use this MCP server?** (Developers, business users, analysts?)
- **What are their technical skill levels?**
- **What problems are they trying to solve?**

#### Map User Journeys

- **What tasks do users want to accomplish?**
- **What is the typical workflow?**
- **What information do they need?**
- **What actions must they take?**

**Example User Journey - Project Management:**
```
User Goal: Track project progress

Journey:
1. List active projects
2. Get project details and tasks
3. Check task status and assignees
4. Update task progress
5. Add comments or notes
6. Generate status report

MCP Tools Needed:
- list_projects()
- get_project_details({ project_id })
- update_task_status({ task_id, status })
- add_task_comment({ task_id, comment })
- generate_project_report({ project_id, format })
```

### 1.4 Plan Tool Inventory

Create a comprehensive list of tools your server will provide:

#### Categorize Tools

Group tools by functionality:
- **Read operations** (get, list, search, fetch)
- **Write operations** (create, update, delete)
- **Action operations** (send, publish, execute)
- **Analysis operations** (analyze, calculate, summarize)

#### Prioritize Tools

Not all tools are equally important:

1. **Critical tools** - Essential for core workflows (implement first)
2. **Important tools** - Enhance functionality (implement second)
3. **Nice-to-have tools** - Additional convenience (implement if time permits)

**Example Tool Inventory - Email Service:**

| Priority | Tool | Category | Purpose |
|----------|------|----------|---------|
| Critical | `send_email` | Write | Send emails |
| Critical | `list_emails` | Read | List inbox messages |
| Critical | `read_email` | Read | Get email content |
| Important | `search_emails` | Read | Find specific emails |
| Important | `create_draft` | Write | Save draft email |
| Important | `reply_to_email` | Write | Reply to message |
| Nice-to-have | `archive_email` | Action | Archive message |
| Nice-to-have | `label_email` | Action | Add labels |

---

## Phase 2: Design and Architecture

### 2.1 Design Tool Schemas

Each tool needs a well-designed schema:

#### Tool Schema Components

```typescript
{
  name: string,           // Unique identifier
  description: string,    // What the tool does
  inputSchema: {         // JSON Schema for parameters
    type: "object",
    properties: { ... },
    required: [ ... ]
  }
}
```

#### Schema Design Best Practices

**1. Use Clear, Descriptive Names**

```typescript
// ❌ Bad
get_data({ id: "123" })

// ✅ Good
get_project_details({ project_id: "123" })
```

**2. Write Comprehensive Descriptions**

Include in your description:
- What the tool does
- When to use it
- Key parameters
- Example use cases
- Related tools

```typescript
{
  name: "search_emails",
  description: `Search emails in the user's mailbox using various criteria.

Usage: Find specific emails by sender, subject, date range, or content.

Parameters:
- query: Search text (searches subject and body)
- from: Filter by sender email
- date_after: Only emails after this date (ISO 8601)
- date_before: Only emails before this date (ISO 8601)
- limit: Maximum results to return (default: 20, max: 100)

Examples:
- Find emails from boss: { from: "boss@example.com" }
- Find recent budget emails: { query: "budget", date_after: "2025-01-01" }
- Find specific subject: { query: "subject:Q4 Report" }

Related tools: read_email, list_emails`,

  inputSchema: { ... }
}
```

**3. Design Intuitive Parameters**

```typescript
// ❌ Bad: Too many required parameters
{
  name: "create_task",
  inputSchema: {
    type: "object",
    properties: {
      title: { type: "string" },
      description: { type: "string" },
      assignee_id: { type: "string" },
      priority: { type: "number" },
      due_date: { type: "string" },
      project_id: { type: "string" },
      status: { type: "string" },
      tags: { type: "array" },
      estimated_hours: { type: "number" }
    },
    required: ["title", "description", "assignee_id", "priority",
               "due_date", "project_id", "status"]
  }
}

// ✅ Good: Minimal required, sensible defaults
{
  name: "create_task",
  inputSchema: {
    type: "object",
    properties: {
      title: {
        type: "string",
        description: "Task title"
      },
      description: {
        type: "string",
        description: "Detailed task description (optional)"
      },
      assignee: {
        type: "string",
        description: "Email of person to assign to (optional)"
      },
      due_date: {
        type: "string",
        format: "date",
        description: "Due date in YYYY-MM-DD format (optional)"
      },
      priority: {
        type: "string",
        enum: ["low", "medium", "high"],
        default: "medium",
        description: "Task priority"
      }
    },
    required: ["title"]
  }
}
```

**4. Use Enums for Fixed Choices**

```typescript
{
  priority: {
    type: "string",
    enum: ["low", "medium", "high", "urgent"],
    description: "Task priority level"
  },
  status: {
    type: "string",
    enum: ["open", "in_progress", "blocked", "completed"],
    description: "Current task status"
  }
}
```

**5. Support Flexible Inputs**

```typescript
{
  recipients: {
    oneOf: [
      {
        type: "string",
        description: "Single email address"
      },
      {
        type: "array",
        items: { type: "string" },
        description: "Multiple email addresses"
      }
    ]
  }
}
```

### 2.2 Design Response Formats

Responses should be consistent and informative:

#### Standard Response Structure

```typescript
// Success response
{
  success: true,
  data: { ... },      // The actual result
  metadata?: { ... }  // Optional metadata
}

// Error response
{
  success: false,
  error: {
    code: string,
    message: string,
    details?: any,
    suggestions?: string[]
  }
}
```

#### Response Design Principles

**1. Be Consistent**

All tools should follow the same response pattern:

```typescript
// ✅ Consistent responses
list_projects() → { success: true, data: { projects: [...] } }
get_project() → { success: true, data: { project: {...} } }
create_task() → { success: true, data: { task: {...} } }
```

**2. Include Relevant Context**

```typescript
{
  success: true,
  data: {
    projects: [
      { id: "proj_1", name: "Website Redesign", status: "active" },
      { id: "proj_2", name: "Mobile App", status: "planning" }
    ],
    total: 2,
    page: 1,
    has_more: false
  },
  metadata: {
    timestamp: "2025-01-18T10:30:00Z",
    user: "alice@example.com"
  }
}
```

**3. Provide Actionable Next Steps**

```typescript
{
  success: true,
  data: {
    draft: {
      id: "draft_123",
      subject: "Project Update",
      status: "draft"
    }
  },
  next_actions: [
    "Edit draft with update_draft({ draft_id: 'draft_123', ... })",
    "Send email with send_email({ draft_id: 'draft_123' })",
    "Delete draft with delete_draft({ draft_id: 'draft_123' })"
  ]
}
```

### 2.3 Plan Error Handling

Design comprehensive error handling:

#### Error Categories

```typescript
enum ErrorCode {
  // Client errors (4xx)
  INVALID_INPUT = "invalid_input",
  UNAUTHORIZED = "unauthorized",
  FORBIDDEN = "forbidden",
  NOT_FOUND = "not_found",
  RATE_LIMIT = "rate_limit_exceeded",

  // Server errors (5xx)
  SERVICE_ERROR = "service_error",
  TIMEOUT = "timeout",
  UNAVAILABLE = "service_unavailable",

  // Integration errors
  AUTH_FAILED = "authentication_failed",
  API_ERROR = "api_error"
}
```

#### Error Response Examples

```typescript
// Invalid input
{
  success: false,
  error: {
    code: "invalid_input",
    message: "Invalid email address format",
    details: {
      field: "recipients",
      value: "not-an-email",
      expected: "Valid email address (e.g., user@example.com)"
    },
    suggestions: [
      "Check the email address format",
      "Use the format: user@domain.com"
    ]
  }
}

// Rate limit
{
  success: false,
  error: {
    code: "rate_limit_exceeded",
    message: "API rate limit exceeded",
    details: {
      limit: 100,
      used: 100,
      reset_at: "2025-01-18T11:00:00Z",
      retry_after_seconds: 300
    },
    suggestions: [
      "Wait 5 minutes before retrying",
      "Reduce the frequency of requests",
      "Contact support to increase rate limit"
    ]
  }
}

// Service unavailable
{
  success: false,
  error: {
    code: "service_unavailable",
    message: "The email service is temporarily unavailable",
    details: {
      service: "email-api",
      status: "degraded",
      estimated_recovery: "2025-01-18T10:45:00Z"
    },
    suggestions: [
      "Retry in a few minutes",
      "Check service status at https://status.example.com"
    ]
  }
}
```

---

## Phase 3: Implementation

### 3.1 Set Up Project Structure

Create a well-organized project:

```
mcp-server-myservice/
├── src/
│   ├── index.ts              # Main entry point
│   ├── server.ts             # MCP server setup
│   ├── tools/                # Tool implementations
│   │   ├── index.ts          # Tool registry
│   │   ├── read-tools.ts     # Read operations
│   │   ├── write-tools.ts    # Write operations
│   │   └── action-tools.ts   # Action operations
│   ├── api/                  # API client
│   │   ├── client.ts         # API client class
│   │   ├── auth.ts           # Authentication
│   │   └── types.ts          # Type definitions
│   ├── utils/                # Utilities
│   │   ├── validation.ts     # Input validation
│   │   ├── formatting.ts     # Response formatting
│   │   └── errors.ts         # Error handling
│   └── config/               # Configuration
│       └── constants.ts      # Constants and defaults
├── tests/                    # Test files
├── package.json
├── tsconfig.json
└── README.md
```

### 3.2 Implement Core Server

Basic MCP server implementation:

```typescript
// src/server.ts
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import {
  CallToolRequestSchema,
  ListToolsRequestSchema,
} from "@modelcontextprotocol/sdk/types.js";
import { tools } from "./tools/index.js";
import { handleToolCall } from "./tools/handler.js";

export class MyServiceServer {
  private server: Server;

  constructor() {
    this.server = new Server(
      {
        name: "myservice-mcp-server",
        version: "1.0.0",
      },
      {
        capabilities: {
          tools: {},
        },
      }
    );

    this.setupHandlers();
  }

  private setupHandlers() {
    // List available tools
    this.server.setRequestHandler(ListToolsRequestSchema, async () => ({
      tools,
    }));

    // Handle tool calls
    this.server.setRequestHandler(CallToolRequestSchema, async (request) => {
      try {
        return await handleToolCall(request);
      } catch (error) {
        return {
          content: [
            {
              type: "text",
              text: JSON.stringify({
                success: false,
                error: {
                  code: "internal_error",
                  message: error.message,
                },
              }),
            },
          ],
        };
      }
    });
  }

  async run() {
    const transport = new StdioServerTransport();
    await this.server.connect(transport);
    console.error("MyService MCP server running on stdio");
  }
}
```

### 3.3 Implement Tools

Example tool implementation:

```typescript
// src/tools/read-tools.ts
import { Tool } from "@modelcontextprotocol/sdk/types.js";
import { apiClient } from "../api/client.js";
import { formatResponse, formatError } from "../utils/formatting.js";
import { validateInput } from "../utils/validation.js";

export const listProjectsTool: Tool = {
  name: "list_projects",
  description: `List all projects in your workspace.

Returns: Array of projects with basic information (id, name, status, created date).

Parameters:
- status: Filter by project status (optional)
- limit: Maximum number of projects to return (default: 50, max: 100)
- sort: Sort order - "recent" or "name" (default: "recent")

Example: { "status": "active", "limit": 10 }`,

  inputSchema: {
    type: "object",
    properties: {
      status: {
        type: "string",
        enum: ["active", "archived", "completed", "all"],
        description: "Filter projects by status",
      },
      limit: {
        type: "number",
        minimum: 1,
        maximum: 100,
        default: 50,
        description: "Maximum number of projects to return",
      },
      sort: {
        type: "string",
        enum: ["recent", "name"],
        default: "recent",
        description: "Sort order",
      },
    },
  },
};

export async function listProjects(args: any) {
  try {
    // Validate input
    const validation = validateInput(args, listProjectsTool.inputSchema);
    if (!validation.valid) {
      return formatError({
        code: "invalid_input",
        message: "Invalid input parameters",
        details: validation.errors,
        suggestions: [
          "Check the parameter types and values",
          "Refer to the tool description for valid options",
        ],
      });
    }

    // Set defaults
    const { status = "all", limit = 50, sort = "recent" } = args;

    // Call API
    const response = await apiClient.listProjects({
      status: status === "all" ? undefined : status,
      limit,
      sort,
    });

    // Format response
    return formatResponse({
      projects: response.data.map((project) => ({
        id: project.id,
        name: project.name,
        status: project.status,
        created: project.created_at,
        task_count: project.tasks?.length || 0,
      })),
      total: response.total,
      showing: response.data.length,
    });
  } catch (error) {
    if (error.response?.status === 401) {
      return formatError({
        code: "unauthorized",
        message: "Authentication failed. Please check your API key.",
        suggestions: [
          "Verify your API key is correct",
          "Check if your API key has expired",
          "Generate a new API key if needed",
        ],
      });
    }

    if (error.response?.status === 429) {
      return formatError({
        code: "rate_limit_exceeded",
        message: "Rate limit exceeded",
        details: {
          retry_after: error.response.headers["retry-after"],
        },
        suggestions: [
          `Wait ${error.response.headers["retry-after"]} seconds before retrying`,
          "Reduce request frequency",
        ],
      });
    }

    return formatError({
      code: "api_error",
      message: "Failed to list projects",
      details: error.message,
    });
  }
}
```

### 3.4 Implement API Client

Create a robust API client:

```typescript
// src/api/client.ts
import axios, { AxiosInstance, AxiosError } from "axios";
import { getApiKey } from "./auth.js";

class APIClient {
  private client: AxiosInstance;
  private baseURL: string;

  constructor() {
    this.baseURL = process.env.MYSERVICE_API_URL || "https://api.example.com";
    this.client = axios.create({
      baseURL: this.baseURL,
      timeout: 30000,
      headers: {
        "Content-Type": "application/json",
      },
    });

    this.setupInterceptors();
  }

  private setupInterceptors() {
    // Request interceptor - add auth
    this.client.interceptors.request.use(
      (config) => {
        const apiKey = getApiKey();
        if (apiKey) {
          config.headers.Authorization = `Bearer ${apiKey}`;
        }
        return config;
      },
      (error) => Promise.reject(error)
    );

    // Response interceptor - handle errors
    this.client.interceptors.response.use(
      (response) => response,
      (error: AxiosError) => {
        if (error.response) {
          // Server responded with error
          console.error(
            `API Error: ${error.response.status} - ${JSON.stringify(error.response.data)}`
          );
        } else if (error.request) {
          // Request made but no response
          console.error("API Error: No response received");
        } else {
          // Error setting up request
          console.error(`API Error: ${error.message}`);
        }
        return Promise.reject(error);
      }
    );
  }

  async listProjects(params: {
    status?: string;
    limit?: number;
    sort?: string;
  }) {
    const response = await this.client.get("/projects", { params });
    return response.data;
  }

  async getProject(projectId: string) {
    const response = await this.client.get(`/projects/${projectId}`);
    return response.data;
  }

  async createProject(data: { name: string; description?: string }) {
    const response = await this.client.post("/projects", data);
    return response.data;
  }

  async updateProject(projectId: string, data: any) {
    const response = await this.client.patch(`/projects/${projectId}`, data);
    return response.data;
  }

  async deleteProject(projectId: string) {
    const response = await this.client.delete(`/projects/${projectId}`);
    return response.data;
  }
}

export const apiClient = new APIClient();
```

### 3.5 Implement Utilities

Utility functions for validation and formatting:

```typescript
// src/utils/validation.ts
export function validateInput(input: any, schema: any): {
  valid: boolean;
  errors?: string[];
} {
  const errors: string[] = [];

  // Check required fields
  if (schema.required) {
    for (const field of schema.required) {
      if (!(field in input)) {
        errors.push(`Missing required field: ${field}`);
      }
    }
  }

  // Validate types and constraints
  if (schema.properties) {
    for (const [key, prop] of Object.entries(schema.properties as any)) {
      if (key in input) {
        const value = input[key];

        // Type check
        if (prop.type === "string" && typeof value !== "string") {
          errors.push(`Field '${key}' must be a string`);
        }
        if (prop.type === "number" && typeof value !== "number") {
          errors.push(`Field '${key}' must be a number`);
        }
        if (prop.type === "boolean" && typeof value !== "boolean") {
          errors.push(`Field '${key}' must be a boolean`);
        }
        if (prop.type === "array" && !Array.isArray(value)) {
          errors.push(`Field '${key}' must be an array`);
        }

        // Enum check
        if (prop.enum && !prop.enum.includes(value)) {
          errors.push(
            `Field '${key}' must be one of: ${prop.enum.join(", ")}`
          );
        }

        // Number constraints
        if (prop.minimum !== undefined && value < prop.minimum) {
          errors.push(`Field '${key}' must be >= ${prop.minimum}`);
        }
        if (prop.maximum !== undefined && value > prop.maximum) {
          errors.push(`Field '${key}' must be <= ${prop.maximum}`);
        }
      }
    }
  }

  return {
    valid: errors.length === 0,
    errors: errors.length > 0 ? errors : undefined,
  };
}

// src/utils/formatting.ts
export function formatResponse(data: any) {
  return {
    content: [
      {
        type: "text",
        text: JSON.stringify(
          {
            success: true,
            data,
          },
          null,
          2
        ),
      },
    ],
  };
}

export function formatError(error: {
  code: string;
  message: string;
  details?: any;
  suggestions?: string[];
}) {
  return {
    content: [
      {
        type: "text",
        text: JSON.stringify(
          {
            success: false,
            error,
          },
          null,
          2
        ),
      },
    ],
    isError: true,
  };
}
```

---

## Phase 4: Testing and Validation

### 4.1 Unit Testing

Test individual tool functions:

```typescript
// tests/tools/read-tools.test.ts
import { describe, it, expect, beforeEach, vi } from "vitest";
import { listProjects } from "../../src/tools/read-tools.js";
import { apiClient } from "../../src/api/client.js";

vi.mock("../../src/api/client.js");

describe("listProjects", () => {
  beforeEach(() => {
    vi.clearAllMocks();
  });

  it("should list projects successfully", async () => {
    const mockProjects = [
      { id: "1", name: "Project A", status: "active", created_at: "2025-01-01" },
      { id: "2", name: "Project B", status: "active", created_at: "2025-01-02" },
    ];

    vi.mocked(apiClient.listProjects).mockResolvedValue({
      data: mockProjects,
      total: 2,
    });

    const result = await listProjects({ status: "active" });
    const response = JSON.parse(result.content[0].text);

    expect(response.success).toBe(true);
    expect(response.data.projects).toHaveLength(2);
    expect(response.data.total).toBe(2);
  });

  it("should handle invalid input", async () => {
    const result = await listProjects({ limit: 1000 }); // exceeds max
    const response = JSON.parse(result.content[0].text);

    expect(response.success).toBe(false);
    expect(response.error.code).toBe("invalid_input");
  });

  it("should handle API errors", async () => {
    vi.mocked(apiClient.listProjects).mockRejectedValue(
      new Error("API Error")
    );

    const result = await listProjects({});
    const response = JSON.parse(result.content[0].text);

    expect(response.success).toBe(false);
    expect(response.error.code).toBe("api_error");
  });

  it("should handle rate limiting", async () => {
    const error: any = new Error("Rate limited");
    error.response = {
      status: 429,
      headers: { "retry-after": "60" },
    };

    vi.mocked(apiClient.listProjects).mockRejectedValue(error);

    const result = await listProjects({});
    const response = JSON.parse(result.content[0].text);

    expect(response.success).toBe(false);
    expect(response.error.code).toBe("rate_limit_exceeded");
    expect(response.error.details.retry_after).toBe("60");
  });
});
```

### 4.2 Integration Testing

Test with actual MCP client:

```typescript
// tests/integration/server.test.ts
import { Client } from "@modelcontextprotocol/sdk/client/index.js";
import { StdioClientTransport } from "@modelcontextprotocol/sdk/client/stdio.js";
import { spawn } from "child_process";

describe("MCP Server Integration", () => {
  let client: Client;
  let serverProcess: any;

  beforeAll(async () => {
    // Start server
    serverProcess = spawn("node", ["dist/index.js"]);

    // Create client
    const transport = new StdioClientTransport({
      command: "node",
      args: ["dist/index.js"],
    });

    client = new Client(
      {
        name: "test-client",
        version: "1.0.0",
      },
      {
        capabilities: {},
      }
    );

    await client.connect(transport);
  });

  afterAll(async () => {
    await client.close();
    serverProcess.kill();
  });

  it("should list available tools", async () => {
    const response = await client.listTools();

    expect(response.tools).toBeDefined();
    expect(response.tools.length).toBeGreaterThan(0);

    const listProjectsTool = response.tools.find(
      (t) => t.name === "list_projects"
    );
    expect(listProjectsTool).toBeDefined();
  });

  it("should execute list_projects tool", async () => {
    const response = await client.callTool({
      name: "list_projects",
      arguments: { status: "active" },
    });

    expect(response.content).toBeDefined();
    expect(response.content[0].type).toBe("text");

    const result = JSON.parse(response.content[0].text);
    expect(result.success).toBe(true);
    expect(result.data.projects).toBeDefined();
  });
});
```

### 4.3 Manual Testing

Test with Claude Desktop or other MCP clients:

#### 1. Configure MCP Client

Add to Claude Desktop config:

```json
{
  "mcpServers": {
    "myservice": {
      "command": "node",
      "args": ["/path/to/mcp-server-myservice/dist/index.js"],
      "env": {
        "MYSERVICE_API_KEY": "your-api-key-here"
      }
    }
  }
}
```

#### 2. Test Common Workflows

Create a test checklist:

```markdown
## Manual Testing Checklist

### Basic Functionality
- [ ] Server starts without errors
- [ ] Tools are listed correctly
- [ ] Tool descriptions are clear and helpful

### Read Operations
- [ ] List projects returns data
- [ ] Get project details works
- [ ] Search returns relevant results
- [ ] Pagination works correctly

### Write Operations
- [ ] Create project succeeds
- [ ] Update project modifies data
- [ ] Delete project removes data
- [ ] Changes are reflected in subsequent reads

### Error Handling
- [ ] Invalid input shows helpful errors
- [ ] Missing required fields are caught
- [ ] API errors are handled gracefully
- [ ] Rate limiting provides retry guidance
- [ ] Authentication failures are clear

### Edge Cases
- [ ] Empty results handled well
- [ ] Large datasets don't overflow context
- [ ] Special characters in input work
- [ ] Concurrent requests work correctly

### User Experience
- [ ] Responses are concise and relevant
- [ ] Error messages are actionable
- [ ] Tool combinations work naturally
- [ ] Common workflows feel smooth
```

#### 3. Test with Real Scenarios

Test realistic user workflows:

```
Scenario 1: Project Status Review
1. List all active projects
2. Get details for top 3 projects
3. Check tasks for each project
4. Generate summary

Scenario 2: Create and Update
1. Create new project
2. Add tasks to project
3. Assign tasks to team members
4. Update task statuses

Scenario 3: Search and Report
1. Search for projects by keyword
2. Get project metrics
3. Export report
```

### 4.4 Performance Testing

Test performance and optimize:

#### Monitor Response Times

```typescript
// Add timing to tool handlers
export async function listProjects(args: any) {
  const startTime = Date.now();

  try {
    const result = await actualListProjects(args);

    const duration = Date.now() - startTime;
    console.error(`listProjects completed in ${duration}ms`);

    return result;
  } catch (error) {
    const duration = Date.now() - startTime;
    console.error(`listProjects failed after ${duration}ms`);
    throw error;
  }
}
```

#### Test with Large Datasets

```typescript
// Test pagination with large results
const hugeList = await client.callTool({
  name: "list_emails",
  arguments: { limit: 100 },
});

// Measure response size
const responseSize = JSON.stringify(hugeList).length;
console.log(`Response size: ${responseSize} characters`);

// Should be < 10KB for good context usage
expect(responseSize).toBeLessThan(10000);
```

#### Optimize Response Formats

```typescript
// ❌ Bad: Returns full objects (large)
{
  projects: [
    {
      id: "proj_123",
      name: "Website Redesign",
      description: "Complete overhaul of company website with new branding...",
      created_at: "2025-01-01T00:00:00Z",
      updated_at: "2025-01-18T10:30:00Z",
      owner: { id: "usr_456", name: "Alice", email: "alice@example.com" },
      team: [/* full team member objects */],
      tasks: [/* full task objects */],
      metadata: { /* lots of metadata */ }
    },
    // ... more projects
  ]
}

// ✅ Good: Returns summary (compact)
{
  projects: [
    {
      id: "proj_123",
      name: "Website Redesign",
      status: "active",
      owner: "Alice",
      tasks: { total: 12, completed: 7, in_progress: 3, blocked: 2 }
    },
    // ... more projects
  ],
  note: "Use get_project_details(id) for full information"
}
```

---

## Best Practices

### Security

**1. Secure API Keys**

```typescript
// ❌ Bad: Hardcoded key
const apiKey = "sk_live_abc123xyz";

// ✅ Good: Environment variable
const apiKey = process.env.MYSERVICE_API_KEY;
if (!apiKey) {
  throw new Error("MYSERVICE_API_KEY environment variable is required");
}
```

**2. Validate All Inputs**

```typescript
// Always validate before using
function deleteProject(args: any) {
  if (!args.project_id || typeof args.project_id !== "string") {
    throw new Error("Invalid project_id");
  }

  // Additional validation
  if (!args.project_id.match(/^proj_[a-z0-9]+$/)) {
    throw new Error("Invalid project_id format");
  }

  // Safe to use
  return apiClient.deleteProject(args.project_id);
}
```

**3. Handle Sensitive Data**

```typescript
// Don't log sensitive information
console.error("Making API call", {
  endpoint: "/users",
  // ❌ Don't log: apiKey, passwords, tokens
  // ✅ Do log: non-sensitive parameters
  userId: args.user_id,
});

// Redact sensitive data in responses
function formatUser(user: any) {
  return {
    id: user.id,
    name: user.name,
    email: user.email,
    // ❌ Don't include: password, api_key, ssn
    // Don't include internal IDs or sensitive metadata
  };
}
```

### Error Handling

**1. Catch and Handle All Errors**

```typescript
try {
  return await apiClient.doSomething(args);
} catch (error) {
  // Specific error types
  if (error.response?.status === 404) {
    return formatError({
      code: "not_found",
      message: "Resource not found",
      suggestions: ["Check the ID is correct", "Try listing resources first"],
    });
  }

  if (error.response?.status === 429) {
    return formatError({
      code: "rate_limit",
      message: "Too many requests",
      details: { retry_after: error.response.headers["retry-after"] },
    });
  }

  // Generic fallback
  return formatError({
    code: "api_error",
    message: "An unexpected error occurred",
    details: error.message,
  });
}
```

**2. Provide Context in Errors**

```typescript
{
  error: {
    code: "not_found",
    message: "Project 'proj_123' not found",
    context: {
      project_id: "proj_123",
      user: "alice@example.com",
      action: "get_project_details"
    },
    suggestions: [
      "Verify the project ID is correct",
      "Check if you have access to this project",
      "List available projects with list_projects()"
    ]
  }
}
```

### Performance

**1. Implement Caching**

```typescript
// Simple in-memory cache
const cache = new Map();
const CACHE_TTL = 5 * 60 * 1000; // 5 minutes

async function listProjects(args: any) {
  const cacheKey = JSON.stringify(args);
  const cached = cache.get(cacheKey);

  if (cached && Date.now() - cached.timestamp < CACHE_TTL) {
    console.error("Returning cached result");
    return cached.result;
  }

  const result = await apiClient.listProjects(args);
  cache.set(cacheKey, { result, timestamp: Date.now() });

  return result;
}
```

**2. Use Pagination**

```typescript
// Always paginate large results
{
  name: "list_emails",
  inputSchema: {
    properties: {
      limit: {
        type: "number",
        default: 20,
        maximum: 100,  // Cap at reasonable size
      },
      cursor: {
        type: "string",
        description: "Pagination cursor from previous response",
      },
    },
  },
}

// Return pagination info
{
  data: {
    emails: [...],  // Limited batch
    pagination: {
      cursor: "next_page_token",
      has_more: true,
      total: 1250
    }
  }
}
```

**3. Optimize Response Size**

```typescript
// Provide verbosity options
{
  name: "get_project",
  inputSchema: {
    properties: {
      project_id: { type: "string" },
      include: {
        type: "array",
        items: {
          enum: ["tasks", "team", "comments", "attachments"],
        },
        description: "Optional related data to include",
      },
    },
  },
}

// Only return what's requested
async function getProject(args: any) {
  const project = await apiClient.getProject(args.project_id);

  const result: any = {
    id: project.id,
    name: project.name,
    status: project.status,
    // ... basic fields
  };

  // Only include if requested
  if (args.include?.includes("tasks")) {
    result.tasks = project.tasks;
  }
  if (args.include?.includes("team")) {
    result.team = project.team;
  }

  return result;
}
```

### Documentation

**1. Write Comprehensive README**

```markdown
# MCP Server - MyService

Connect Claude to MyService for project management.

## Installation

\`\`\`bash
npm install -g mcp-server-myservice
\`\`\`

## Configuration

Add to your MCP settings:

\`\`\`json
{
  "mcpServers": {
    "myservice": {
      "command": "mcp-server-myservice",
      "env": {
        "MYSERVICE_API_KEY": "your-api-key"
      }
    }
  }
}
\`\`\`

## Getting an API Key

1. Go to https://myservice.com/settings/api
2. Click "Generate API Key"
3. Copy the key and add to your MCP config

## Available Tools

### list_projects
List all projects in your workspace.

**Usage:** "Show me my active projects"

### get_project_details
Get detailed information about a specific project.

**Usage:** "What are the details of project proj_123?"

## Examples

See [EXAMPLES.md](./EXAMPLES.md) for detailed usage examples.

## Troubleshooting

See [TROUBLESHOOTING.md](./TROUBLESHOOTING.md) for common issues.
```

**2. Provide Usage Examples**

```markdown
# Examples

## Project Management

### List active projects
> Show me all active projects

\`\`\`json
{
  "tool": "list_projects",
  "args": { "status": "active" }
}
\`\`\`

### Create a new project
> Create a new project called "Website Redesign"

\`\`\`json
{
  "tool": "create_project",
  "args": {
    "name": "Website Redesign",
    "description": "Complete overhaul of company website"
  }
}
\`\`\`

### Update project status
> Mark project proj_123 as completed

\`\`\`json
{
  "tool": "update_project",
  "args": {
    "project_id": "proj_123",
    "status": "completed"
  }
}
\`\`\`

## Task Management

### Add tasks to project
> Add a task "Design homepage" to project proj_123

\`\`\`json
{
  "tool": "create_task",
  "args": {
    "project_id": "proj_123",
    "title": "Design homepage",
    "assignee": "alice@example.com"
  }
}
\`\`\`
```

**3. Document Tool Combinations**

```markdown
# Common Workflows

## Weekly Status Report

Combine multiple tools to generate a comprehensive status report:

1. List active projects
\`\`\`
list_projects({ status: "active" })
\`\`\`

2. For each project, get details
\`\`\`
get_project_details({ project_id: "proj_1" })
get_project_details({ project_id: "proj_2" })
\`\`\`

3. Get tasks for each project
\`\`\`
list_tasks({ project_id: "proj_1", status: "all" })
\`\`\`

4. Generate summary
Analyze the data and create a report showing:
- Active projects count
- Completed vs in-progress tasks
- Blocked tasks requiring attention
- Overall progress
```

---

## Common Patterns

### Pattern 1: List and Details

Provide a two-step pattern for exploring resources:

```typescript
// Step 1: List with summary info
list_projects() → {
  projects: [
    { id: "proj_1", name: "Website Redesign", task_count: 12 },
    { id: "proj_2", name: "Mobile App", task_count: 8 }
  ]
}

// Step 2: Get details for specific item
get_project_details({ project_id: "proj_1" }) → {
  project: {
    id: "proj_1",
    name: "Website Redesign",
    description: "...",
    tasks: [...],  // Full task list
    team: [...],   // Team members
    // ... complete information
  }
}
```

### Pattern 2: Search and Filter

Support multiple ways to find resources:

```typescript
// By keyword
search_projects({ query: "redesign" })

// By filters
list_projects({
  status: "active",
  owner: "alice@example.com",
  created_after: "2025-01-01"
})

// By tags
list_projects({ tags: ["urgent", "client-facing"] })
```

### Pattern 3: Create and Update

Separate creation from updates:

```typescript
// Create with minimal required fields
create_project({
  name: "New Project"
}) → { project: { id: "proj_123", ... } }

// Update specific fields
update_project({
  project_id: "proj_123",
  description: "Updated description",
  status: "active"
})

// Use PUT for replacing, PATCH for updating
replace_project({ ... })  // Replaces entire object
update_project({ ... })   // Updates specific fields
```

### Pattern 4: Batch Operations

Support operating on multiple items:

```typescript
// Batch create
create_tasks({
  project_id: "proj_123",
  tasks: [
    { title: "Task 1", assignee: "alice@example.com" },
    { title: "Task 2", assignee: "bob@example.com" }
  ]
}) → {
  created: 2,
  tasks: [...]
}

// Batch update
update_tasks({
  task_ids: ["task_1", "task_2", "task_3"],
  status: "completed"
}) → {
  updated: 3,
  results: [...]
}
```

### Pattern 5: Hierarchical Resources

Support parent-child relationships:

```typescript
// Get parent
get_project({ project_id: "proj_123" })

// List children
list_tasks({ project_id: "proj_123" })

// Get specific child
get_task({ task_id: "task_456" })
// Also accepts: get_task({ project_id: "proj_123", task_id: "task_456" })

// Create child
create_task({
  project_id: "proj_123",  // Parent reference
  title: "New task"
})
```

---

## Examples

### Example 1: GitHub MCP Server

A well-designed GitHub MCP server:

```typescript
// High-level workflow tools
{
  name: "create_pull_request",
  description: "Create a pull request with commits from a branch",
  inputSchema: {
    properties: {
      repo: { type: "string", description: "Repository (owner/repo)" },
      title: { type: "string" },
      body: { type: "string" },
      head: { type: "string", description: "Branch with changes" },
      base: { type: "string", default: "main", description: "Target branch" },
    },
  },
}

// Combines multiple API calls:
// 1. Validate branch exists
// 2. Check for conflicts
// 3. Create PR
// 4. Add labels if specified
// 5. Request reviews if specified

// Returns concise, actionable response
{
  pull_request: {
    number: 123,
    url: "https://github.com/owner/repo/pull/123",
    status: "open",
    checks: "pending"
  },
  next_actions: [
    "Monitor checks with get_pr_status({ repo, pr_number: 123 })",
    "Add reviewers with request_pr_review({ ... })"
  ]
}
```

### Example 2: Slack MCP Server

Workflow-oriented messaging tools:

```typescript
{
  name: "send_message_to_channel",
  description: "Send a message to a Slack channel by name or ID",
  inputSchema: {
    properties: {
      channel: {
        type: "string",
        description: "Channel name (e.g., 'general') or ID"
      },
      message: { type: "string" },
      thread_ts: {
        type: "string",
        description: "Reply to specific message (optional)"
      },
    },
  },
}

// Smart features:
// - Accepts channel name OR ID (looks up if needed)
// - Automatically handles formatting
// - Supports threading
// - Returns minimal response

{
  message_sent: true,
  channel: "general",
  timestamp: "1642521600.123456",
  permalink: "https://workspace.slack.com/archives/C123/p1642521600123456"
}
```

### Example 3: Calendar MCP Server

Intelligent scheduling:

```typescript
{
  name: "schedule_meeting",
  description: "Find available time and schedule a meeting with attendees",
  inputSchema: {
    properties: {
      attendees: {
        type: "array",
        items: { type: "string" },
        description: "Email addresses of attendees"
      },
      title: { type: "string" },
      duration_minutes: { type: "number", default: 30 },
      preferred_times: {
        type: "array",
        items: { type: "string", format: "date-time" },
        description: "Preferred start times (will use first available)"
      },
      date_range: {
        type: "object",
        properties: {
          start: { type: "string", format: "date" },
          end: { type: "string", format: "date" }
        },
        description: "Search for availability in this range"
      }
    },
  },
}

// Workflow: Checks availability → Creates event → Sends invites
// Returns: Event details + what was done

{
  meeting: {
    id: "evt_123",
    title: "Team Sync",
    start: "2025-01-20T14:00:00Z",
    end: "2025-01-20T14:30:00Z",
    attendees: ["alice@example.com", "bob@example.com"],
    all_accepted: false,
    calendar_link: "https://calendar.google.com/event?eid=..."
  },
  actions_taken: [
    "Found availability at 2:00 PM",
    "Created calendar event",
    "Sent invitations to 2 attendees"
  ]
}
```

---

## Troubleshooting

### Common Issues

#### 1. Authentication Failures

**Problem:** "Authentication failed" errors

**Solutions:**
- Check API key is correct
- Verify API key has required permissions
- Check if API key has expired
- Ensure environment variable is set correctly

```bash
# Check if env var is set
echo $MYSERVICE_API_KEY

# Test API key directly
curl -H "Authorization: Bearer $MYSERVICE_API_KEY" \
  https://api.example.com/test
```

#### 2. Rate Limiting

**Problem:** "Rate limit exceeded" errors

**Solutions:**
- Implement request throttling
- Add caching to reduce API calls
- Use batch operations where possible
- Request rate limit increase from service

```typescript
// Add rate limiting
import { RateLimiter } from "limiter";

const limiter = new RateLimiter({
  tokensPerInterval: 100,
  interval: "minute"
});

async function makeAPICall() {
  await limiter.removeTokens(1);
  return apiClient.doSomething();
}
```

#### 3. Tool Not Found

**Problem:** "Tool 'xyz' not found" errors

**Solutions:**
- Check tool is registered in tools array
- Verify tool name matches exactly (case-sensitive)
- Restart MCP server after code changes
- Check server logs for registration errors

```typescript
// Ensure tool is exported and registered
export const tools = [
  listProjectsTool,
  getProjectTool,
  createProjectTool,
  // ... all tools must be listed here
];
```

#### 4. Large Response Sizes

**Problem:** Responses exceed context limits

**Solutions:**
- Implement pagination
- Return summaries instead of full objects
- Add verbosity options
- Trim unnecessary fields

```typescript
// Add pagination
{
  name: "list_items",
  inputSchema: {
    properties: {
      limit: { type: "number", default: 20, maximum: 100 },
      cursor: { type: "string" }
    }
  }
}

// Return summary by default
function formatProject(project: any, detailed: boolean = false) {
  if (detailed) {
    return project;  // Full object
  }

  // Summary only
  return {
    id: project.id,
    name: project.name,
    status: project.status,
    task_count: project.tasks?.length || 0
  };
}
```

#### 5. Slow Performance

**Problem:** Tool calls take too long

**Solutions:**
- Add caching
- Optimize API calls (batch, parallel)
- Reduce response sizes
- Add timeout handling

```typescript
// Parallel API calls
async function getProjectSummary(projectIds: string[]) {
  const results = await Promise.all(
    projectIds.map(id => apiClient.getProject(id))
  );
  return results;
}

// Timeout handling
async function withTimeout<T>(
  promise: Promise<T>,
  timeoutMs: number
): Promise<T> {
  const timeout = new Promise((_, reject) =>
    setTimeout(() => reject(new Error("Timeout")), timeoutMs)
  );
  return Promise.race([promise, timeout]) as Promise<T>;
}
```

### Debugging Tips

#### Enable Debug Logging

```typescript
// Add debug logging
const DEBUG = process.env.DEBUG === "true";

function log(...args: any[]) {
  if (DEBUG) {
    console.error("[DEBUG]", ...args);
  }
}

// Use throughout code
log("Calling API:", endpoint, params);
log("Response:", response.status, response.data);
```

#### Test Tools Individually

```bash
# Test MCP server manually
echo '{"jsonrpc":"2.0","id":1,"method":"tools/list"}' | \
  node dist/index.js

# Test specific tool
echo '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"list_projects","arguments":{}}}' | \
  node dist/index.js
```

#### Monitor Performance

```typescript
// Add performance monitoring
function withTiming(name: string, fn: Function) {
  return async (...args: any[]) => {
    const start = Date.now();
    try {
      const result = await fn(...args);
      console.error(`${name} completed in ${Date.now() - start}ms`);
      return result;
    } catch (error) {
      console.error(`${name} failed after ${Date.now() - start}ms`);
      throw error;
    }
  };
}

// Wrap tool functions
export const listProjects = withTiming(
  "listProjects",
  actualListProjects
);
```

---

## Conclusion

Building a high-quality MCP server requires:

1. **Deep understanding** of agent-centric design principles
2. **Thorough research** of the service and user needs
3. **Thoughtful design** of workflows, tools, and responses
4. **Robust implementation** with proper error handling
5. **Comprehensive testing** and validation
6. **Clear documentation** and examples

By following this guide, you'll create MCP servers that enable LLMs to effectively accomplish real-world tasks with external services.

---

## Additional Resources

### MCP Documentation
- [Official MCP Specification](https://spec.modelcontextprotocol.io/)
- [MCP TypeScript SDK](https://github.com/modelcontextprotocol/typescript-sdk)
- [MCP Python SDK](https://github.com/modelcontextprotocol/python-sdk)

### Example MCP Servers
- [Official MCP Servers](https://github.com/modelcontextprotocol/servers)
- [Community MCP Servers](https://github.com/topics/mcp-server)

### Related Topics
- [JSON Schema Documentation](https://json-schema.org/)
- [REST API Design Best Practices](https://restfulapi.net/)
- [OpenAPI Specification](https://swagger.io/specification/)

---

## Version History

- **1.0.0** (2025-01-18) - Initial comprehensive guide
