# MCP Tool Calling Implementation Guide

This document provides a comprehensive step-by-step guide to how Model Context Protocol (MCP) tool calling works in Cline, documenting the complete flow from AI assistant request to tool execution and response.

## Overview

MCP tool calling in Cline enables AI assistants to execute tools and access resources from external MCP servers. The system provides two primary tools:

1. **`use_mcp_tool`** - Execute tools provided by MCP servers
2. **`access_mcp_resource`** - Access resources provided by MCP servers

## Architecture Components

### Core Components
- **McpHub** (`src/services/mcp/McpHub.ts`) - Central MCP server management
- **ToolExecutor** (`src/core/task/ToolExecutor.ts`) - Tool execution orchestration  
- **Controller** (`src/core/controller/index.ts`) - Task coordination
- **MCP SDK Client** - Communication with MCP servers

### Data Flow
```
AI Assistant → ToolExecutor → McpHub → MCP Server → Tool Response → AI Assistant
```

## Step-by-Step MCP Tool Calling Process

### 1. AI Assistant Request Generation

**Location:** System prompts in `src/core/prompts/system-prompt/`

The AI assistant generates tool usage requests based on system prompts:

```xml
<use_mcp_tool>
<server_name>server-name</server_name>
<tool_name>tool-name</tool_name>
<arguments>{"param": "value"}</arguments>
</use_mcp_tool>
```

**System Prompt Definition:**
```typescript
// File: src/core/prompts/system-prompt/generic-system-prompt.ts:231
## use_mcp_tool

Use a tool from an MCP server to accomplish tasks or retrieve information.

<use_mcp_tool>
<server_name>The exact name of the MCP server to use</server_name>
<tool_name>The exact name of the tool to call</tool_name>
<arguments>JSON object with the tool arguments (if any)</arguments>
</use_mcp_tool>
```

### 2. Message Parsing and Validation

**Location:** `src/core/assistant-message/parse-assistant-message.ts`

The assistant's response is parsed into structured blocks:

```typescript
// Tool name validation
export const toolUseNames = [
    // ... other tools
    "use_mcp_tool",
    "access_mcp_resource",
] as const

// Parameter extraction and validation
const toolUse = {
    name: "use_mcp_tool",
    params: {
        server_name: "extracted-server-name",
        tool_name: "extracted-tool-name", 
        arguments: "extracted-json-arguments"
    }
}
```

### 3. Tool Execution Routing

**Location:** `src/core/task/ToolExecutor.ts:1306`

The `ToolExecutor` routes the parsed tool use to the appropriate handler:

```typescript
switch (block.name) {
    case "use_mcp_tool": {
        const server_name: string | undefined = block.params.server_name
        const tool_name: string | undefined = block.params.tool_name
        const mcp_arguments: string | undefined = block.params.arguments
        
        // Process tool execution...
    }
}
```

### 4. Parameter Validation and Processing

**Algorithm Steps:**

#### 4.1. Required Parameter Validation
```typescript
// File: src/core/task/ToolExecutor.ts:1329-1340
if (!server_name) {
    this.taskState.consecutiveMistakeCount++
    this.pushToolResult(await this.sayAndCreateMissingParamError("use_mcp_tool", "server_name"), block)
    await this.saveCheckpoint()
    break
}
if (!tool_name) {
    this.taskState.consecutiveMistakeCount++
    this.pushToolResult(await this.sayAndCreateMissingParamError("use_mcp_tool", "tool_name"), block)
    await this.saveCheckpoint()
    break
}
```

#### 4.2. Arguments JSON Parsing
```typescript
// File: src/core/task/ToolExecutor.ts:1347-1364
let parsedArguments: Record<string, unknown> | undefined
if (mcp_arguments) {
    try {
        parsedArguments = JSON.parse(mcp_arguments)
    } catch (_error) {
        this.taskState.consecutiveMistakeCount++
        await this.say("error", `Cline tried to use ${tool_name} with an invalid JSON argument. Retrying...`)
        this.pushToolResult(
            formatResponse.toolError(formatResponse.invalidMcpToolArgumentError(server_name, tool_name)),
            block,
        )
        await this.saveCheckpoint()
        break
    }
}
```

### 5. Auto-Approval and User Permission

**Location:** `src/core/task/ToolExecutor.ts:1373-1393`

#### 5.1. Auto-Approval Check Algorithm
```typescript
const isToolAutoApproved = this.mcpHub.connections
    ?.find((conn) => conn.server.name === server_name)
    ?.server.tools?.find((tool) => tool.name === tool_name)?.autoApprove

if (this.shouldAutoApproveTool(block.name) && isToolAutoApproved) {
    // Auto-approve path
    this.removeLastPartialMessageIfExistsWithType("ask", "use_mcp_server")
    await this.say("use_mcp_server", completeMessage, undefined, undefined, false)
    this.taskState.consecutiveAutoApprovedRequestsCount++
} else {
    // Manual approval path
    showNotificationForApprovalIfAutoApprovalEnabled(
        `Cline wants to use ${tool_name} on ${server_name}`,
        this.autoApprovalSettings.enabled,
        this.autoApprovalSettings.enableNotifications,
    )
    const didApprove = await this.askApproval("use_mcp_server", block, completeMessage)
    if (!didApprove) {
        await this.saveCheckpoint()
        break
    }
}
```

#### 5.2. Auto-Approval Configuration
**Location:** `src/services/mcp/McpHub.ts:452-455`

```typescript
// Mark tools as always allowed based on settings
const tools = (response?.tools || []).map((tool) => ({
    ...tool,
    autoApprove: autoApproveConfig.includes(tool.name),
}))
```

### 6. MCP Server Communication

**Location:** `src/core/task/ToolExecutor.ts:1396-1404`

#### 6.1. Pre-execution Setup
```typescript
// Signal tool execution start
await this.say("mcp_server_request_started")

// Check for any pending notifications before the tool call
const notificationsBefore = this.mcpHub.getPendingNotifications()
for (const notification of notificationsBefore) {
    await this.say("mcp_notification", `[${notification.serverName}] ${notification.message}`)
}
```

#### 6.2. Tool Execution via McpHub
```typescript
// File: src/core/task/ToolExecutor.ts:1404
const toolResult = await this.mcpHub.callTool(server_name, tool_name, parsedArguments, this.ulid)
```

### 7. McpHub Tool Execution Implementation

**Location:** `src/services/mcp/McpHub.ts:815-890`

#### 7.1. Connection and Server Validation
```typescript
async callTool(
    serverName: string,
    toolName: string, 
    toolArguments: Record<string, unknown> | undefined,
    ulid: string,
): Promise<McpToolCallResponse> {
    const connection = this.connections.find((conn) => conn.server.name === serverName)
    if (!connection) {
        throw new Error(
            `No connection found for server: ${serverName}. Please make sure to use MCP servers available under 'Connected MCP Servers'.`,
        )
    }

    if (connection.server.disabled) {
        throw new Error(`Server "${serverName}" is disabled and cannot be used`)
    }
```

#### 7.2. Timeout Configuration
```typescript
// File: src/services/mcp/McpHub.ts:832-841
let timeout = secondsToMs(DEFAULT_MCP_TIMEOUT_SECONDS) // sdk expects ms

try {
    const config = JSON.parse(connection.server.config)
    const parsedConfig = ServerConfigSchema.parse(config)
    timeout = secondsToMs(parsedConfig.timeout)
} catch (error) {
    console.error(`Failed to parse timeout configuration for server ${serverName}: ${error}`)
}
```

#### 7.3. Telemetry and MCP SDK Call
```typescript
// File: src/services/mcp/McpHub.ts:842-864
this.telemetryService.captureMcpToolCall(
    ulid, serverName, toolName, "started", undefined,
    toolArguments ? Object.keys(toolArguments) : undefined,
)

try {
    const result = await connection.client.request(
        {
            method: "tools/call",
            params: {
                name: toolName,
                arguments: toolArguments,
            },
        },
        CallToolResultSchema,
        {
            timeout,
        },
    )

    this.telemetryService.captureMcpToolCall(
        ulid, serverName, toolName, "success", undefined,
        toolArguments ? Object.keys(toolArguments) : undefined,
    )

    return {
        ...result,
        content: result.content ?? [],
    }
} catch (error) {
    this.telemetryService.captureMcpToolCall(
        ulid, serverName, toolName, "error",
        error instanceof Error ? error.message : String(error),
        toolArguments ? Object.keys(toolArguments) : undefined,
    )
    throw error
}
```

### 8. MCP Server Communication Protocol

The MCP SDK handles the actual communication with the MCP server:

#### 8.1. Request Structure
```json
{
    "method": "tools/call",
    "params": {
        "name": "tool-name",
        "arguments": {
            "param1": "value1",
            "param2": "value2"
        }
    }
}
```

#### 8.2. Transport Types
**Location:** `src/services/mcp/types.ts`

```typescript
export type Transport = StdioClientTransport | SSEClientTransport | StreamableHTTPClientTransport

export type McpConnection = {
    server: McpServer
    client: Client
    transport: Transport
}
```

#### 8.3. Connection Types
- **stdio**: Local process communication
- **sse**: Server-Sent Events over HTTP
- **streamableHttp**: HTTP streaming

### 9. Response Processing and Formatting

**Location:** `src/core/task/ToolExecutor.ts:1406-1447`

#### 9.1. Post-execution Notifications
```typescript
// Check for any pending notifications after the tool call
const notificationsAfter = this.mcpHub.getPendingNotifications()
for (const notification of notificationsAfter) {
    await this.say("mcp_notification", `[${notification.serverName}] ${notification.message}`)
}
```

#### 9.2. Response Content Processing
```typescript
// Extract images from response
const toolResultImages =
    toolResult?.content
        .filter((item) => item.type === "image")
        .map((item) => `data:${item.mimeType};base64,${item.data}`) || []

// Extract text content
let toolResultText =
    (toolResult?.isError ? "Error:\n" : "") +
        toolResult?.content
            .map((item) => {
                if (item.type === "text") {
                    return item.text
                }
                if (item.type === "resource") {
                    const { blob, ...rest } = item.resource
                    return JSON.stringify(rest, null, 2)
                }
                return ""
            })
            .filter(Boolean)
            .join("\n\n") || "(No response)"
```

#### 9.3. Model Capability Handling
```typescript
// Handle image support based on model capabilities
const supportsImages = this.api.getModel().info.supportsImages ?? false
if (toolResultImages.length > 0 && !supportsImages) {
    toolResultText += `\n\n[${toolResultImages.length} images were provided in the response, and while they are displayed to the user, you do not have the ability to view them.]`
}
```

### 10. Response Delivery to AI Assistant

#### 10.1. UI Update
```typescript
// File: src/core/task/ToolExecutor.ts:1434-1435
const toolResultToDisplay = toolResultText + toolResultImages?.map((image) => `\n\n${image}`).join("")
await this.say("mcp_server_response", toolResultToDisplay)
```

#### 10.2. Tool Result Formatting
```typescript
// File: src/core/task/ToolExecutor.ts:1444-1447
this.pushToolResult(
    formatResponse.toolResult(toolResultText, supportsImages ? toolResultImages : undefined),
    block,
)
```

#### 10.3. State Persistence
```typescript
await this.saveCheckpoint()
```

## Access MCP Resource Flow

The `access_mcp_resource` tool follows a similar pattern with key differences:

### Resource Access Implementation

**Location:** `src/core/task/ToolExecutor.ts:1459-1525`

```typescript
case "access_mcp_resource": {
    const server_name: string | undefined = block.params.server_name
    const uri: string | undefined = block.params.uri
    
    // Validation, approval, and execution similar to use_mcp_tool
    const resourceResult = await this.mcpHub.readResource(server_name, uri)
}
```

### McpHub Resource Reading

**Location:** `src/services/mcp/McpHub.ts:795-813`

```typescript
async readResource(serverName: string, uri: string): Promise<McpResourceResponse> {
    const connection = this.connections.find((conn) => conn.server.name === serverName)
    if (!connection) {
        throw new Error(`No connection found for server: ${serverName}`)
    }
    if (connection.server.disabled) {
        throw new Error(`Server "${serverName}" is disabled`)
    }

    return await connection.client.request(
        {
            method: "resources/read",
            params: {
                uri,
            },
        },
        ReadResourceResultSchema,
    )
}
```

## Error Handling and Recovery

### 1. Connection Errors
```typescript
// File: src/services/mcp/McpHub.ts:823-825
if (!connection) {
    throw new Error(
        `No connection found for server: ${serverName}. Please make sure to use MCP servers available under 'Connected MCP Servers'.`,
    )
}
```

### 2. Server Disabled
```typescript
if (connection.server.disabled) {
    throw new Error(`Server "${serverName}" is disabled and cannot be used`)
}
```

### 3. Timeout Handling
```typescript
const result = await connection.client.request(
    requestPayload,
    CallToolResultSchema,
    {
        timeout, // Configurable timeout per server
    },
)
```

### 4. General Error Recovery
```typescript
// File: src/core/task/ToolExecutor.ts:1453-1457
} catch (error) {
    await this.handleError("executing MCP tool", error, block)
    await this.saveCheckpoint()
    break
}
```

## Performance Considerations

### 1. Connection Pooling
- MCP connections are maintained in `McpHub.connections`
- Connections are reused across multiple tool calls
- Automatic reconnection on connection failures

### 2. Timeout Management
```typescript
// Configurable timeouts per server
timeout = secondsToMs(parsedConfig.timeout)
```

### 3. Telemetry and Monitoring
```typescript
this.telemetryService.captureMcpToolCall(
    ulid, serverName, toolName, "started|success|error",
    errorMessage, argumentKeys
)
```

## Security Considerations

### 1. Server Validation
- Only connected servers can be used
- Disabled servers are rejected
- Server name validation prevents injection

### 2. Argument Sanitization
```typescript
// JSON parsing with error handling
try {
    parsedArguments = JSON.parse(mcp_arguments)
} catch (_error) {
    // Handle invalid JSON gracefully
}
```

### 3. Auto-Approval Controls
- Tool-level auto-approval configuration
- User notification system
- Manual approval fallback

## Debugging and Troubleshooting

### 1. Common Issues
- **Server not found**: Check MCP server configuration and connection status
- **Tool not found**: Verify tool name matches server's exposed tools
- **Invalid arguments**: Check JSON formatting and parameter requirements
- **Timeout errors**: Adjust server timeout configuration

### 2. Debug Logging Points
```typescript
console.log(`[downloadMcp] Response from download API`, { response })
console.error(`Failed to execute MCP tool ${tool_name}:`, error)
```

### 3. State Inspection
- Check `this.mcpHub.connections` for server status
- Monitor `this.taskState.consecutiveMistakeCount` for error patterns
- Review `this.taskState.consecutiveAutoApprovedRequestsCount` for auto-approval behavior

## Integration Points

### 1. System Prompt Integration
- Tools are documented in system prompts
- Examples provided for proper usage
- Connected servers are listed dynamically

### 2. UI Integration
- Real-time tool execution display
- User approval interfaces
- Error message presentation

### 3. State Management
- Tool results integrated into conversation history
- Checkpoint saving for recovery
- Progress tracking and telemetry

## Extension and Customization

### 1. Adding New MCP Features
- Extend `McpHub` for new capabilities
- Update system prompts with new tools
- Add UI components for new interactions

### 2. Custom Tool Handling
- Extend `ToolExecutor` switch statement
- Add validation for new parameter types
- Implement specialized error handling

### 3. Enhanced Auto-Approval
- Add rule-based auto-approval logic
- Implement risk scoring for tools
- Add granular permission controls

This comprehensive guide covers the complete MCP tool calling implementation in Cline, from AI assistant request generation through tool execution and response delivery, providing a foundation for understanding, debugging, and extending the MCP integration system.