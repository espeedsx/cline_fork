# Detailed Conversation Loop Elaboration

This document provides an in-depth, step-by-step elaboration of the conversation loop algorithm shown in `README_LOOP.md`, documenting the exact code locations, algorithms, and implementation details for each step.

## Flowchart Reference

```
START
  ↓
Initialize Task
  ↓
Parse User Input ──────────┐
  ↓                        │
Load Context               │
  ↓                        │
Build System Prompt        │
  ↓                        │
Make API Request           │
  ↓                        │
Stream Response ───────────┤
  ↓                        │
Parse Assistant Message    │
  ↓                        │
Execute Tools              │
  ↓                        │
Present Results            │
  ↓                        │
Check Completion           │
  ↓                        │
More Tools? ───YES─────────┘
  ↓ NO
Task Complete/User Input
  ↓
END or LOOP BACK
```

## Step-by-Step Implementation Details

### 1. START → Initialize Task

**File:** `src/core/controller/index.ts:158`  
**Entry Point:** `Controller.initTask()`

```typescript
async initTask(task?: string, images?: string[], files?: string[], historyItem?: HistoryItem): Promise<void>
```

**Algorithm:**
1. **Task Creation** - Instantiate new `Task` object with dependencies
2. **State Initialization** - Set up task state, MCP hub, message handlers
3. **Context Setup** - Initialize working directory, API configuration
4. **History Management** - Handle task history and resume logic

**Code Location:** `src/core/controller/index.ts:158-200`

**Key Steps:**
- Create `Task` instance with controller reference, MCP hub, and callbacks
- Set up task history item if resuming
- Configure API handler and model settings
- Initialize state synchronization with webview

**State Changes:**
- `Controller.task` becomes non-null
- Task initialization begins

---

### 2. Initialize Task → Parse User Input

**File:** `src/core/task/index.ts:967`  
**Entry Point:** `Task.startTask()`

```typescript
private async startTask(task?: string, images?: string[], files?: string[]): Promise<void>
```

**Algorithm:**
1. **Ignore Controller Setup** - Initialize `.cline-ignore` controller for file filtering
2. **State Reset** - Clear previous conversation history and messages
3. **Content Building** - Create user content blocks from task text, images, files
4. **File Processing** - Convert file paths to text content using `processFilesIntoText()`

**Code Location:** `src/core/task/index.ts:967-1006`

**Detailed Process:**
```typescript
// Step 1: Initialize ignore controller
await this.clineIgnoreController.initialize()

// Step 2: Reset message state
this.messageStateHandler.setClineMessages([])
this.messageStateHandler.setApiConversationHistory([])

// Step 3: Build user content
const userContent: UserContent = [
    {
        type: "text",
        text: `<task>\n${task}\n</task>`,
    },
    ...imageBlocks,
]

// Step 4: Process files into text
if (files && files.length > 0) {
    const fileContentString = await processFilesIntoText(files)
    userContent.push({
        type: "text",
        text: fileContentString,
    })
}
```

**State Changes:**
- `taskState.isInitialized = true`
- Message history cleared
- User content prepared for API request

---

### 3. Parse User Input → Load Context

**File:** `src/core/task/index.ts:1192`  
**Entry Point:** `Task.initiateTaskLoop()`

```typescript
private async initiateTaskLoop(userContent: UserContent): Promise<void>
```

**Algorithm:**
1. **Loop Initialization** - Set up main conversation loop with user content
2. **File Details Flag** - Determine if file details should be included (first iteration only)
3. **Abort Handling** - Check for task abortion throughout the loop
4. **Request Delegation** - Call recursive request handler

**Code Location:** `src/core/task/index.ts:1192-1221`

**Loop Structure:**
```typescript
let nextUserContent = userContent
let includeFileDetails = true
while (!this.taskState.abort) {
    const didEndLoop = await this.recursivelyMakeClineRequests(
        nextUserContent, 
        includeFileDetails
    )
    includeFileDetails = false // Only first time
    
    if (didEndLoop) break
    
    // Continue with no-tools-used message
    nextUserContent = [{
        type: "text",
        text: formatResponse.noToolsUsed(),
    }]
}
```

**State Changes:**
- Loop begins with initial user content
- `includeFileDetails` flag managed per iteration

---

### 4. Load Context → Build System Prompt

**File:** `src/core/task/index.ts:1655`  
**Entry Point:** `Task.attemptApiRequest()`

**Algorithm:**
1. **MCP Server Wait** - Wait for MCP servers to connect (10s timeout)
2. **Browser Settings** - Check browser tool availability and model support
3. **System Prompt Building** - Call `buildSystemPrompt()` with configuration
4. **Language Preferences** - Add preferred language instructions
5. **Rules Integration** - Load global/local Cline rules, Cursor rules, Windsurf rules
6. **Context Management** - Get conversation context and metadata

**Code Location:** `src/core/task/index.ts:1655-1734`

**Context Loading Process:**
```typescript
// Wait for MCP servers
await pWaitFor(() => this.mcpHub.isConnecting !== true, { timeout: 10_000 })

// Build system prompt
let systemPrompt = await buildSystemPrompt(
    this.cwd,
    supportsBrowserUse,
    this.mcpHub,
    this.browserSettings,
    this.api.getModel(),
    this.focusChainSettings,
)

// Add user instructions from rules
const { globalToggles, localToggles } = await refreshClineRulesToggles(this.controller, this.cwd)
const globalClineRulesFileInstructions = await getGlobalClineRules(globalClineRulesFilePath, globalToggles)
```

**Dependencies:**
- `src/core/prompts/system-prompt/build-system-prompt.ts` - System prompt construction
- `src/core/rules/` - Rules management system
- `src/core/context/context-management/ContextManager.ts` - Context handling

---

### 5. Build System Prompt → Make API Request

**File:** `src/core/task/index.ts:1734`  
**Continuation of:** `Task.attemptApiRequest()`

**Algorithm:**
1. **Context Metadata** - Get conversation context and management metadata
2. **Message Preparation** - Combine system prompt with conversation history
3. **Token Validation** - Check token limits and context window
4. **API Stream Creation** - Create streaming API request

**Code Location:** `src/core/task/index.ts:1734-1800`

**Request Building:**
```typescript
// Get context metadata
const contextManagementMetadata = await this.contextManager.getNewContextMessagesAndMetadata(
    this.messageStateHandler.getApiConversationHistory(),
    this.messageStateHandler.getClineMessages(),
    this.api,
    this.taskState.conversationHistoryDeletedRange,
    previousApiReqIndex,
    await ensureTaskDirectoryExists(this.getContext(), this.taskId),
    this.useAutoCondense,
)

// Create API stream
const stream = this.api.createMessage(
    systemPrompt,
    contextManagementMetadata.apiConversationHistory,
    {
        maxTokens: this.api.getModel().info.maxTokens,
        onFirstChunk: () => {
            this.taskState.isWaitingForFirstChunk = false
        },
        abortSignal: abortController.signal,
    }
)
```

**State Changes:**
- `taskState.isWaitingForFirstChunk = true`
- API request stream created

---

### 6. Make API Request → Stream Response

**File:** `src/core/task/index.ts:1957`  
**Entry Point:** `Task.recursivelyMakeClineRequests()`

**Algorithm:**
1. **Stream Iteration** - Process each chunk from API stream
2. **Chunk Type Handling** - Handle different chunk types (text, tool_use, etc.)
3. **Content Accumulation** - Build assistant message content
4. **Real-time Updates** - Update UI during streaming

**Code Location:** `src/core/task/index.ts:1957-2400`

**Stream Processing:**
```typescript
const stream = this.attemptApiRequest(previousApiReqIndex)
let assistantMessage = ""

for await (const { chunk, type } of stream) {
    if (type === "text") {
        assistantMessage += chunk.text
        this.updateAssistantMessageContent(chunk)
        this.presentAssistantMessage()
    } else if (type === "tool_use") {
        // Handle tool use chunk
        const parsedContent = parseAssistantMessageV2(assistantMessage)
        this.taskState.assistantMessageContent = parsedContent
        this.presentAssistantMessage()
    }
}
```

**Stream Types (from `src/core/api/transform/stream.ts`):**
- `ApiStreamTextChunk` - Text content
- `ApiStreamToolUseChunk` - Tool usage
- `ApiStreamFinalChunk` - End of stream

**State Changes:**
- `taskState.isStreaming = true`
- `assistantMessageContent` updated incrementally

---

### 7. Stream Response → Parse Assistant Message

**File:** `src/core/assistant-message/parse-assistant-message.ts`  
**Entry Point:** `parseAssistantMessageV2()`

```typescript
export function parseAssistantMessageV2(content: string): AssistantMessageContent[]
```

**Algorithm:**
1. **Content Parsing** - Parse raw assistant response into structured blocks
2. **Tool Extraction** - Identify and extract tool use blocks
3. **Parameter Validation** - Validate tool parameters
4. **Block Creation** - Create typed content blocks

**Code Location:** `src/core/assistant-message/parse-assistant-message.ts:1-100`

**Parsing Process:**
```typescript
// Parse into blocks
const blocks: AssistantMessageContent[] = []

// Extract text blocks
const textBlocks = extractTextBlocks(content)
blocks.push(...textBlocks)

// Extract tool use blocks
const toolBlocks = extractToolUseBlocks(content)
for (const tool of toolBlocks) {
    // Validate parameters
    const validatedParams = validateToolParameters(tool.name, tool.parameters)
    blocks.push({
        type: "tool_use",
        name: tool.name,
        parameters: validatedParams
    })
}
```

**Output Types:**
- `TextBlock` - Plain text content
- `ToolUseBlock` - Tool usage with validated parameters

**State Changes:**
- `taskState.assistantMessageContent` populated with parsed blocks

---

### 8. Parse Assistant Message → Execute Tools

**File:** `src/core/task/ToolExecutor.ts:303`  
**Entry Point:** `ToolExecutor.executeTool()`

```typescript
async executeTool(toolUse: ToolUse): Promise<[boolean, ToolResponse]>
```

**Algorithm:**
1. **Auto-Approval Check** - Determine if tool should be auto-approved
2. **User Approval** - Request user approval if needed
3. **Tool Dispatch** - Route to specific tool execution method
4. **Error Handling** - Handle execution errors and user rejection

**Code Location:** `src/core/task/ToolExecutor.ts:303-400`

**Execution Flow:**
```typescript
// Check auto-approval
const shouldAutoApprove = this.shouldAutoApproveTool(toolUse.name)

// Get user approval if needed
if (!shouldAutoApprove) {
    const approval = await this.getToolApproval(toolUse)
    if (!approval) return [false, "Tool execution rejected"]
}

// Execute tool
switch (toolUse.name) {
    case "read_file":
        return await this.executeReadFile(toolUse.parameters)
    case "write_to_file":
        return await this.executeWriteFile(toolUse.parameters)
    case "execute_command":
        return await this.executeCommand(toolUse.parameters)
    // ... other tools
}
```

**Auto-Approval Logic (src/core/task/tools/autoApprove.ts):**
```typescript
shouldAutoApproveTool(toolName: ToolUseName): boolean | [boolean, boolean] {
    const settings = this.autoApprovalSettings
    
    switch (toolName) {
        case "read_file":
            return settings.readFiles
        case "write_to_file":
            return [settings.writeFiles, settings.showNotifications]
        case "execute_command":
            return settings.executeCommands
    }
}
```

**State Changes:**
- `taskState.didAlreadyUseTool = true`
- `taskState.didRejectTool = true` (if rejected)

---

### 9. Execute Tools → Present Results

**File:** `src/core/task/index.ts:1841`  
**Entry Point:** `Task.presentAssistantMessage()`

```typescript
private async presentAssistantMessage(): Promise<void>
```

**Algorithm:**
1. **Lock Management** - Prevent concurrent presentation updates
2. **Content Streaming** - Update UI with current assistant message content
3. **Tool Results** - Include tool execution results
4. **State Synchronization** - Sync state with webview

**Code Location:** `src/core/task/index.ts:1841-1900`

**Presentation Process:**
```typescript
// Prevent concurrent updates
if (this.taskState.presentAssistantMessageLocked) {
    this.taskState.presentAssistantMessageHasPendingUpdates = true
    return
}

this.taskState.presentAssistantMessageLocked = true

try {
    // Update message content
    await this.updateMessageContent()
    
    // Sync state with webview
    await this.postStateToWebview()
    
    // Handle pending updates
    if (this.taskState.presentAssistantMessageHasPendingUpdates) {
        this.taskState.presentAssistantMessageHasPendingUpdates = false
        await this.presentAssistantMessage() // Recursive call for pending updates
    }
} finally {
    this.taskState.presentAssistantMessageLocked = false
}
```

**State Changes:**
- UI updated with latest content
- `presentAssistantMessageLocked` managed for concurrency

---

### 10. Present Results → Check Completion

**File:** `src/core/task/index.ts:1957` (within `recursivelyMakeClineRequests`)  
**Completion Check Algorithm:**

**Algorithm:**
1. **Tool Usage Check** - Determine if any tools were used in the response
2. **Completion Detection** - Look for `attempt_completion` tool usage
3. **Request Limit Check** - Check against `MAX_REQUESTS_PER_TASK`
4. **User Decision** - Ask user about continuing if limits reached

**Code Location:** `src/core/task/index.ts:2350-2450`

**Completion Logic:**
```typescript
// Check if any tools were used
const didUseTools = this.taskState.didAlreadyUseTool

// Check for completion attempt
const hasCompletionAttempt = this.taskState.assistantMessageContent.some(
    block => block.type === "tool_use" && block.name === "attempt_completion"
)

// Check request limits
const requestCount = this.messageStateHandler.getApiConversationHistory().length
const maxRequests = this.getMaxRequestsPerTask()

if (hasCompletionAttempt) {
    return true // End loop
} else if (!didUseTools) {
    // No tools used, continue with prompt
    return false
} else if (requestCount >= maxRequests) {
    // Max requests reached, ask user
    const shouldContinue = await this.askUserToContinue()
    return !shouldContinue
}

return false // Continue loop
```

**State Changes:**
- `taskState.consecutiveMistakeCount` incremented if no tools used
- Loop continuation decision made

---

### 11. Check Completion → More Tools? (Decision Point)

**File:** `src/core/task/index.ts:1192` (within `initiateTaskLoop`)

**Decision Algorithm:**
1. **Loop Continuation Check** - Based on `didEndLoop` return value
2. **Content Preparation** - Prepare next user content if continuing
3. **Error Recovery** - Handle consecutive mistakes

**Code Location:** `src/core/task/index.ts:1196-1220`

**Decision Logic:**
```typescript
const didEndLoop = await this.recursivelyMakeClineRequests(nextUserContent, includeFileDetails)

if (didEndLoop) {
    // Task completed or max requests reached
    break
} else {
    // Continue with "no tools used" prompt
    nextUserContent = [
        {
            type: "text",
            text: formatResponse.noToolsUsed(),
        },
    ]
    this.taskState.consecutiveMistakeCount++
}
```

**Branch Outcomes:**
- **YES (More Tools)** → Loop back to "Parse User Input"
- **NO** → Proceed to "Task Complete/User Input"

---

### 12. More Tools? → END or LOOP BACK

**Two Possible Paths:**

#### Path A: Task Complete/User Input
**File:** `src/core/task/index.ts:1203-1206`

**Algorithm:**
1. **Task Completion** - Handle successful task completion
2. **Resource Cleanup** - Clean up resources and sessions
3. **State Finalization** - Finalize task state

**Code Location:** `src/core/task/index.ts:1203-1206`

```typescript
if (didEndLoop) {
    // Task completed
    break
}
```

#### Path B: LOOP BACK
**File:** `src/core/task/index.ts:1212-1219`

**Algorithm:**
1. **Content Reset** - Prepare new user content for next iteration
2. **State Reset** - Reset iteration-specific state
3. **Loop Continuation** - Continue to next iteration

**Code Location:** `src/core/task/index.ts:1212-1219`

```typescript
nextUserContent = [
    {
        type: "text",
        text: formatResponse.noToolsUsed(),
    },
]
this.taskState.consecutiveMistakeCount++
// Loop continues...
```

---

## State Management Throughout the Loop

### Critical State Variables

**File:** `src/core/task/TaskState.ts`

```typescript
class TaskState {
    // Loop Control
    abort: boolean = false                           // Emergency stop
    isInitialized: boolean = false                   // Task ready
    
    // Stream Management
    isStreaming: boolean = false                     // Currently streaming
    isWaitingForFirstChunk: boolean = false         // Waiting for API response
    didCompleteReadingStream: boolean = false       // Stream complete
    
    // Content Management
    userMessageContentReady: boolean = false        // Ready for next request
    assistantMessageContent: AssistantMessageContent[] = []
    
    // Tool Execution
    didAlreadyUseTool: boolean = false              // Tools used this iteration
    didRejectTool: boolean = false                  // User rejected tools
    
    // Presentation
    presentAssistantMessageLocked: boolean = false  // Prevent concurrent updates
    presentAssistantMessageHasPendingUpdates: boolean = false
    
    // Error Handling
    consecutiveMistakeCount: number = 0             // Track consecutive no-tool responses
}
```

### State Transitions

```
IDLE (abort=false, isInitialized=false)
  ↓ initTask()
INITIALIZING (isInitialized=false)
  ↓ startTask()
READY (isInitialized=true, userMessageContentReady=true)
  ↓ attemptApiRequest()
STREAMING (isStreaming=true, isWaitingForFirstChunk=true)
  ↓ first chunk received
PROCESSING (isWaitingForFirstChunk=false)
  ↓ stream complete
TOOL_EXECUTION (didCompleteReadingStream=true)
  ↓ tools executed
PRESENTING (presentAssistantMessageLocked=true)
  ↓ presentation complete
AWAITING_DECISION (userMessageContentReady=true)
  ↓ decision made
READY (loop back) or COMPLETED (didEndLoop=true)
```

---

## Error Handling and Recovery

### Context Window Management
**File:** `src/core/context/context-management/ContextManager.ts`

When the conversation exceeds the model's context window:

1. **Auto-Condensation** - Automatically condense conversation history
2. **User Intervention** - Ask user to manually condense or reset
3. **Graceful Degradation** - Fall back to shorter context

### Stream Errors
**File:** `src/core/task/index.ts:1800-1850`

Stream interruption recovery:
1. **Retry Logic** - Automatic retry for transient errors
2. **User Notification** - Inform user of persistent errors
3. **State Recovery** - Reset stream state for retry

### Tool Execution Errors
**File:** `src/core/task/ToolExecutor.ts:350-400`

Tool failure handling:
1. **Error Classification** - Categorize error types
2. **Retry Strategy** - Retry with different parameters
3. **Fallback Options** - Suggest alternative approaches

---

## Performance Optimizations

### Prompt Caching
- System prompts are cached when possible
- Context reuse minimizes token usage

### Streaming Optimization
- Real-time UI updates during streaming
- Chunked content processing

### Memory Management
- Conversation history condensation
- Resource cleanup on task completion

---

## Extension Points

### Adding New Tools
1. **Tool Definition** - Add to `toolUseNames` array
2. **Parameter Schema** - Define parameter validation
3. **Execution Logic** - Implement in `ToolExecutor`
4. **Auto-Approval** - Configure approval settings

### Custom State Management
1. **State Extension** - Extend `TaskState` class
2. **Persistence** - Add to message state handler
3. **UI Synchronization** - Update webview state sync

### New API Providers
1. **API Handler** - Implement `ApiHandler` interface
2. **Stream Adapter** - Create stream transformation
3. **Model Configuration** - Add model-specific settings

---

This detailed elaboration provides the complete implementation pathway through Cline's conversation loop, enabling developers to understand, debug, and extend the system effectively.