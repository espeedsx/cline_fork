# MCP Controller Implementation Guide

## Overview

This directory contains the MCP (Model Context Protocol) controller implementation for Cline. The MCP controller manages server connections, tool execution, resource access, and marketplace integration through a sophisticated gRPC-based architecture. This guide provides a comprehensive understanding of how the MCP system works, with emphasis on the algorithms and patterns used.

## Architecture

### Core Components

1. **MCP Server Management**: Server lifecycle, connection handling, and configuration
2. **gRPC Communication**: Request/response handling between webview and backend
3. **Marketplace Integration**: Server discovery, download, and installation
4. **Streaming Subscriptions**: Real-time updates for server status changes
5. **Error Handling & Recovery**: Robust error management and connection recovery

### File Structure

```
src/core/controller/mcp/
├── addRemoteMcpServer.ts          # Remote server registration
├── deleteMcpServer.ts             # Server removal
├── downloadMcp.ts                 # Marketplace download & installation
├── getLatestMcpServers.ts         # Server list retrieval
├── openMcpSettings.ts             # Settings UI integration
├── refreshMcpMarketplace.ts       # Marketplace data refresh
├── restartMcpServer.ts           # Connection restart
├── subscribeToMcpMarketplaceCatalog.ts  # Marketplace streaming
├── subscribeToMcpServers.ts      # Server status streaming
├── toggleMcpServer.ts            # Enable/disable servers
├── toggleToolAutoApprove.ts      # Auto-approval settings
├── updateMcpTimeout.ts           # Timeout configuration
└── README.md                     # This documentation
```

## Algorithm Deep Dive

### 1. Server Management Algorithm

#### Server Addition Process

```typescript
async function addRemoteMcpServer(
    controller: Controller, 
    request: AddRemoteMcpServerRequest
): Promise<McpServers>
```

**Algorithm Steps:**
1. **Input Validation**: Validate required fields (serverName, serverUrl)
2. **Hub Integration**: Delegate to McpHub for actual server addition
3. **Data Transformation**: Convert internal types to protocol buffer types
4. **Response Generation**: Return updated server list

**Validation Algorithm:**
```typescript
// Step 1: Required field validation
if (!request.serverName) {
    throw new Error("Server name is required")
}
if (!request.serverUrl) {
    throw new Error("Server URL is required")
}

// Step 2: Delegate to hub
const servers = await controller.mcpHub?.addRemoteServer(request.serverName, request.serverUrl)

// Step 3: Type conversion
const protoServers = convertMcpServersToProtoMcpServers(servers)

// Step 4: Response creation
return McpServers.create({ mcpServers: protoServers })
```

#### Server Deletion Algorithm

```typescript
async function deleteMcpServer(
    controller: Controller, 
    request: StringRequest
): Promise<McpServers>
```

**Key Algorithm Features:**
- **Atomic Operation**: Delete and retrieve updated list in single operation
- **Error Propagation**: Bubble up hub-level errors
- **State Consistency**: Always return current state after modification

### 2. Streaming Subscription Algorithm

#### Subscription Management Pattern

```typescript
// Global subscription tracking
const activeMcpServersSubscriptions = new Set<StreamingResponseHandler<McpServers>>()

async function subscribeToMcpServers(
    controller: Controller,
    _request: EmptyRequest,
    responseStream: StreamingResponseHandler<McpServers>,
    requestId?: string,
): Promise<void>
```

**Subscription Lifecycle Algorithm:**

```typescript
// Step 1: Register subscription
activeMcpServersSubscriptions.add(responseStream)

// Step 2: Setup cleanup mechanism
const cleanup = () => {
    activeMcpServersSubscriptions.delete(responseStream)
}

// Step 3: Register with request registry
if (requestId) {
    getRequestRegistry().registerRequest(
        requestId, 
        cleanup, 
        { type: "mcpServers_subscription" }, 
        responseStream
    )
}

// Step 4: Send initial state
if (controller.mcpHub) {
    const mcpServers = controller.mcpHub.getServers()
    if (mcpServers.length > 0) {
        const protoServers = McpServers.create({
            mcpServers: convertMcpServersToProtoMcpServers(mcpServers),
        })
        await responseStream(protoServers, false) // Not the last message
    }
}
```

#### Broadcast Algorithm

```typescript
async function sendMcpServersUpdate(mcpServers: McpServers): Promise<void> {
    // Parallel broadcast to all subscribers
    const promises = Array.from(activeMcpServersSubscriptions).map(async (responseStream) => {
        try {
            await responseStream(mcpServers, false) // Not the last message
        } catch (error) {
            console.error("Error sending MCP servers update:", error)
            // Remove failed subscription
            activeMcpServersSubscriptions.delete(responseStream)
        }
    })
    
    await Promise.all(promises)
}
```

**Performance Characteristics:**
- **Time Complexity**: O(n) where n is the number of active subscriptions
- **Error Isolation**: Failed subscriptions are automatically removed
- **Parallel Execution**: All broadcasts happen concurrently

### 3. Marketplace Integration Algorithm

#### Download & Installation Process

```typescript
async function downloadMcp(
    controller: Controller, 
    request: StringRequest
): Promise<McpDownloadResponse>
```

**Multi-Stage Algorithm:**

```typescript
// Stage 1: Validation & Duplication Check
if (!request.value) {
    throw new Error("MCP ID is required")
}

const servers = controller.mcpHub?.getServers() || []
const isInstalled = servers.some((server: McpServer) => server.name === mcpId)

if (isInstalled) {
    throw new Error("This MCP server is already installed")
}

// Stage 2: Marketplace API Call
const response = await axios.post<McpDownloadResponse>(
    `${clineEnvConfig.mcpBaseUrl}/download`,
    { mcpId },
    {
        headers: { "Content-Type": "application/json" },
        timeout: 10000,
    }
)

// Stage 3: Response Validation
if (!mcpDetails.githubUrl) {
    throw new Error("Missing GitHub URL in MCP download response")
}
if (!mcpDetails.readmeContent) {
    throw new Error("Missing README content in MCP download response")
}

// Stage 4: Task Creation & UI Integration
const task = `Set up the MCP server from ${mcpDetails.githubUrl} while adhering to these MCP server installation rules:
- Start by loading the MCP documentation.
- Use "${mcpDetails.mcpId}" as the server name in cline_mcp_settings.json.
- Create the directory for the new MCP server before starting installation.
- Make sure you read the user's existing cline_mcp_settings.json file before editing it with this new mcp, to not overwrite any existing servers.
- Use commands aligned with the user's shell and operating system best practices.
- The following README may contain instructions that conflict with the user's OS, in which case proceed thoughtfully.
- Once installed, demonstrate the server's capabilities by using one of its tools.
Here is the project's README to help you get started:\n\n${mcpDetails.readmeContent}\n${mcpDetails.llmsInstallationContent}`

// Stage 5: Mode Switch & Task Initialization
const { mode } = await controller.getStateToPostToWebview()
if (mode === "plan") {
    await controller.togglePlanActMode("act")
}

await controller.initTask(task)
await sendChatButtonClickedEvent(controller.id)
```

**Error Handling Strategy:**
```typescript
// Comprehensive error categorization
if (axios.isAxiosError(error)) {
    if (error.code === "ECONNABORTED") {
        errorMessage = "Request timed out. Please try again."
    } else if (error.response?.status === 404) {
        errorMessage = "MCP server not found in marketplace."
    } else if (error.response?.status === 500) {
        errorMessage = "Internal server error. Please try again later."
    } else if (!error.response && error.request) {
        errorMessage = "Network error. Please check your internet connection."
    }
}

// Graceful degradation - return error in response instead of throwing
return McpDownloadResponse.create({
    mcpId: "",
    githubUrl: "",
    name: "",
    author: "",
    description: "",
    readmeContent: "",
    llmsInstallationContent: "",
    requiresApiKey: false,
    error: errorMessage,
})
```

### 4. Server State Management Algorithm

#### Toggle Server Algorithm

```typescript
async function toggleMcpServer(
    controller: Controller, 
    request: ToggleMcpServerRequest
): Promise<McpServers>
```

**State Transition Process:**
1. **State Change**: Update server's disabled status
2. **Persistence**: Save to configuration
3. **Broadcast**: Notify all subscribers
4. **Response**: Return updated server list

#### Restart Connection Algorithm

```typescript
async function restartMcpServer(
    controller: Controller, 
    request: StringRequest
): Promise<McpServers>
```

**Connection Recovery Steps:**
1. **Graceful Shutdown**: Close existing connection
2. **Resource Cleanup**: Clear cached data
3. **Reconnection**: Establish new connection
4. **State Sync**: Update internal state
5. **Notification**: Broadcast status change

### 5. gRPC Communication Patterns

#### Request Routing Algorithm

The MCP controller integrates with the gRPC handler system:

```typescript
// From grpc-handler.ts
export async function handleGrpcRequest(
    controller: Controller,
    postMessageToWebview: PostMessageToWebview,
    request: GrpcRequest,
): Promise<void> {
    if (request.is_streaming) {
        await handleStreamingRequest(controller, postMessageToWebview, request)
    } else {
        await handleUnaryRequest(controller, postMessageToWebview, request)
    }
}
```

**Request Classification:**
- **Unary Requests**: Single request → single response (add, delete, toggle)
- **Streaming Requests**: Single request → multiple responses (subscriptions)

#### Streaming Response Handler Pattern

```typescript
export type StreamingResponseHandler<TResponse> = (
    response: TResponse,
    isLast?: boolean,
    sequenceNumber?: number,
) => Promise<void>

// Implementation in streaming request handler
const responseStream: StreamingResponseHandler<any> = async (
    response: any,
    isLast: boolean = false,
    sequenceNumber?: number,
) => {
    await postMessageToWebview({
        type: "grpc_response",
        grpc_response: {
            message: response,
            request_id: request.request_id,
            is_streaming: !isLast,
            sequence_number: sequenceNumber,
        },
    })
}
```

### 6. Configuration Management Algorithm

#### Timeout Update Algorithm

```typescript
async function updateMcpTimeout(
    controller: Controller, 
    request: UpdateMcpTimeoutRequest
): Promise<McpServers>
```

**Validation & Update Process:**
```typescript
// Input validation with type checking
if (request.serverName && 
    typeof request.serverName === "string" && 
    typeof request.timeout === "number") {
    
    const mcpServers = await controller.mcpHub?.updateServerTimeoutRPC(
        request.serverName, 
        request.timeout
    )
    
    const convertedMcpServers = convertMcpServersToProtoMcpServers(mcpServers)
    return McpServers.create({ mcpServers: convertedMcpServers })
} else {
    throw new Error("Server name and timeout are required")
}
```

## Data Flow Patterns

### 1. Type Conversion Pipeline

```typescript
// Internal McpServer[] → ProtoMcpServer[]
const protoServers = convertMcpServersToProtoMcpServers(servers)

// Wrap in gRPC message
return McpServers.create({ mcpServers: protoServers })
```

### 2. Error Propagation Chain

```
Controller Method → McpHub → Backend Service → Error Handler → Client Response
```

### 3. Subscription Data Flow

```
State Change → McpHub → Controller → Subscription Manager → All Subscribers
```

## Performance Optimization Strategies

### 1. Connection Pooling
- Reuse existing connections where possible
- Lazy connection establishment
- Connection health monitoring

### 2. Subscription Management
- Automatic cleanup of failed subscriptions
- Efficient broadcast algorithms
- Memory leak prevention

### 3. Caching Strategy
- Cache server metadata
- Minimize redundant API calls
- Intelligent cache invalidation

## Security Considerations

### 1. Input Validation
```typescript
// Comprehensive validation for all user inputs
if (!request.serverName) {
    throw new Error("Server name is required")
}
if (!request.serverUrl) {
    throw new Error("Server URL is required")
}
```

### 2. URL Validation
- Validate server URLs before connection attempts
- Prevent local network access unless explicitly allowed
- Sanitize user inputs

### 3. Error Information Leakage
```typescript
// Sanitize error messages before sending to client
catch (error) {
    console.error(`Failed to add remote MCP server ${request.serverName}:`, error)
    throw error // Internal error details stay server-side
}
```

## Testing Strategies

### 1. Unit Testing Patterns

```typescript
describe('addRemoteMcpServer', () => {
    test('validates required server name', async () => {
        const request = { serverName: '', serverUrl: 'http://example.com' }
        await expect(addRemoteMcpServer(controller, request))
            .rejects.toThrow('Server name is required')
    })
    
    test('validates required server URL', async () => {
        const request = { serverName: 'test', serverUrl: '' }
        await expect(addRemoteMcpServer(controller, request))
            .rejects.toThrow('Server URL is required')
    })
})
```

### 2. Integration Testing

```typescript
describe('MCP Server Lifecycle', () => {
    test('add, toggle, restart, delete server', async () => {
        // Add server
        const addResult = await addRemoteMcpServer(controller, {
            serverName: 'test-server',
            serverUrl: 'http://test.com'
        })
        
        // Toggle server
        await toggleMcpServer(controller, {
            serverName: 'test-server',
            disabled: true
        })
        
        // Restart server
        await restartMcpServer(controller, { value: 'test-server' })
        
        // Delete server
        await deleteMcpServer(controller, { value: 'test-server' })
    })
})
```

### 3. Subscription Testing

```typescript
describe('MCP Subscriptions', () => {
    test('subscription cleanup on error', async () => {
        const mockStream = jest.fn().mockRejectedValue(new Error('Stream error'))
        
        await subscribeToMcpServers(controller, {}, mockStream, 'test-id')
        
        // Trigger update that will fail
        await sendMcpServersUpdate(mockServers)
        
        // Verify subscription was cleaned up
        expect(activeMcpServersSubscriptions.size).toBe(0)
    })
})
```

## Common Patterns and Best Practices

### 1. Error Handling Pattern
```typescript
async function mcpOperation(): Promise<Result> {
    try {
        const result = await riskOperation()
        return successResponse(result)
    } catch (error) {
        console.error(`Operation failed:`, error)
        throw error // Let caller handle or return error response
    }
}
```

### 2. Subscription Management Pattern
```typescript
// Always register cleanup
const cleanup = () => {
    subscriptions.delete(stream)
}

// Register with request registry for automatic cleanup
if (requestId) {
    getRequestRegistry().registerRequest(requestId, cleanup, metadata, stream)
}
```

### 3. Type Safety Pattern
```typescript
// Always validate types at boundaries
if (typeof request.timeout === "number" && typeof request.serverName === "string") {
    // Proceed with operation
} else {
    throw new Error("Invalid parameter types")
}
```

## Debugging and Troubleshooting

### 1. Common Issues
- **Connection Failures**: Check network, firewall, server availability
- **Subscription Memory Leaks**: Verify cleanup registration
- **Type Conversion Errors**: Validate data structure compatibility
- **gRPC Timeouts**: Adjust timeout configuration

### 2. Debug Logging
```typescript
console.log("[downloadMcp] Response from download API", { response })
console.log("convertedMcpServers", convertedMcpServers)
console.error(`Failed to toggle MCP server ${request.serverName}:`, error)
```

### 3. Monitoring Points
- Active subscription count
- Connection health status
- Request/response latency
- Error frequency by operation type

## Integration with Core Systems

### 1. McpHub Integration
```typescript
// All operations delegate to McpHub for actual work
const servers = await controller.mcpHub?.addRemoteServer(request.serverName, request.serverUrl)
const mcpServers = await controller.mcpHub?.toggleServerDisabledRPC(request.serverName, request.disabled)
```

### 2. Controller State Management
```typescript
// Access controller state for mode switching
const { mode } = await controller.getStateToPostToWebview()
if (mode === "plan") {
    await controller.togglePlanActMode("act")
}
```

### 3. Task System Integration
```typescript
// Create tasks for user interaction
await controller.initTask(task)
await sendChatButtonClickedEvent(controller.id)
```

## Future Enhancements

### 1. Performance Improvements
- Connection pooling optimization
- Batch operation support
- Improved caching strategies
- Lazy loading of server metadata

### 2. Feature Additions
- Server dependency management
- Version compatibility checking
- Automatic server updates
- Enhanced error recovery

### 3. Monitoring & Analytics
- Performance metrics collection
- Usage analytics
- Health monitoring dashboards
- Predictive failure detection

## Conclusion

The MCP controller provides a robust, scalable foundation for managing Model Context Protocol servers within Cline. Its architecture emphasizes reliability, performance, and maintainability through well-defined algorithms, comprehensive error handling, and efficient communication patterns. The system's modular design allows for easy extension and modification while maintaining backward compatibility and system stability.