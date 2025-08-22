# Task System Implementation Guide

## Table of Contents
1. [Overview](#overview)
2. [Core Architecture](#core-architecture)
3. [Task State Management](#task-state-management)
4. [Plan/Act Mode Algorithm](#planact-mode-algorithm)
5. [Tool Execution System](#tool-execution-system)
6. [Message State Handling](#message-state-handling)
7. [Focus Chain Management](#focus-chain-management)
8. [Deep Planning Algorithm](#deep-planning-algorithm)
9. [Key Algorithms and Data Flows](#key-algorithms-and-data-flows)
10. [Implementation Patterns](#implementation-patterns)

## Overview

The Cline task system is a sophisticated AI assistant architecture that manages conversations, tool execution, and context across multiple interaction modes. It implements a dual-mode system (Plan/Act) with comprehensive state management, tool restrictions, and real-time progress tracking.

### Key Design Principles
- **Separation of Concerns**: Clear boundaries between planning and execution
- **State Management**: Comprehensive tracking of conversation and execution state
- **Tool Safety**: Restricted tool access based on current mode
- **Context Preservation**: Maintains conversation history and file context
- **Real-time Updates**: Live progress tracking through Focus Chain

## Core Architecture

### Primary Components

The task system consists of several interconnected components:

```typescript
export class Task {
    // Core identification
    readonly taskId: string
    readonly ulid: string
    
    // State management
    taskState: TaskState
    messageStateHandler: MessageStateHandler
    
    // Execution engines
    api: ApiHandler
    toolExecutor: ToolExecutor
    
    // Context management
    contextManager: ContextManager
    fileContextTracker: FileContextTracker
    
    // Mode-specific features
    mode: Mode  // "plan" | "act"
    FocusChainManager?: FocusChainManager
}
```

### Component Responsibilities

1. **Task**: Main orchestrator, handles high-level workflow
2. **TaskState**: Manages execution state and flags
3. **ToolExecutor**: Executes and validates tool calls
4. **MessageStateHandler**: Manages conversation history
5. **FocusChainManager**: Tracks progress and todo lists

## Task State Management

The `TaskState` class implements a comprehensive state machine with multiple categories of state:

### State Categories and Algorithms

#### 1. Streaming State Management
```typescript
class TaskState {
    // Streaming control
    isStreaming = false
    isWaitingForFirstChunk = false
    didCompleteReadingStream = false
    currentStreamingContentIndex = 0
}
```

**Algorithm: Stream State Transition**
```
Initial State: { isStreaming: false, isWaitingForFirstChunk: false }
    ↓
Start Stream: { isStreaming: true, isWaitingForFirstChunk: true }
    ↓
First Chunk: { isStreaming: true, isWaitingForFirstChunk: false }
    ↓
Stream Complete: { isStreaming: false, didCompleteReadingStream: true }
```

#### 2. Plan Mode State Machine
```typescript
// Plan mode specific state
isAwaitingPlanResponse = false
didRespondToPlanAskBySwitchingMode = false
```

**Algorithm: Plan Mode Response Flow**
```
1. plan_mode_respond tool called
2. isAwaitingPlanResponse = true
3. Wait for user response OR mode switch
4. If mode switch:
   - didRespondToPlanAskBySwitchingMode = true
   - Continue with execution context
5. If user response:
   - Process feedback
   - Continue planning loop
```

#### 3. Error Recovery State
```typescript
// Error tracking and recovery
consecutiveMistakeCount: number = 0
didAutomaticallyRetryFailedApiRequest = false
```

**Algorithm: Error Recovery**
```
On Tool Error:
1. consecutiveMistakeCount++
2. If consecutiveMistakeCount > threshold:
   - Request user feedback
   - Reset counter on user input
3. If API error and !didAutomaticallyRetryFailedApiRequest:
   - Retry once
   - Set didAutomaticallyRetryFailedApiRequest = true
```

## Plan/Act Mode Algorithm

The Plan/Act mode system implements a sophisticated state machine that controls tool availability and behavior.

### Mode Switching Algorithm

```typescript
async togglePlanActMode(modeToSwitchTo: Mode, chatContent?: ChatContent): Promise<boolean> {
    // 1. Update global state
    this.cacheService.setGlobalState("mode", modeToSwitchTo)
    
    // 2. Update API handler (different models for different modes)
    if (this.task) {
        const apiConfiguration = this.cacheService.getApiConfiguration()
        this.task.api = buildApiHandler(
            { ...apiConfiguration, ulid: this.task.ulid }, 
            modeToSwitchTo
        )
    }
    
    // 3. Propagate mode change to all components
    this.task.updateMode(modeToSwitchTo)
    
    // 4. Handle special case: switching from plan to act during plan response
    if (this.task.taskState.isAwaitingPlanResponse && modeToSwitchTo === "act") {
        this.task.taskState.didRespondToPlanAskBySwitchingMode = true
        await this.task.handleWebviewAskResponse(
            "messageResponse",
            chatContent?.message || "PLAN_MODE_TOGGLE_RESPONSE",
            chatContent?.images || [],
            chatContent?.files || []
        )
        return true
    }
}
```

### Tool Restriction Algorithm

```typescript
private isPlanModeToolRestricted(toolName: ToolUseName): boolean {
    const planModeRestrictedTools: ToolUseName[] = ["write_to_file", "replace_in_file"]
    return planModeRestrictedTools.includes(toolName)
}

// In tool execution:
if (this.strictPlanModeEnabled && this.mode === "plan" && this.isPlanModeToolRestricted(block.name)) {
    const errorMessage = `Tool '${block.name}' is not available in PLAN MODE`
    await this.say("error", errorMessage)
    return // Block execution
}
```

**Mode-Specific Behavior Matrix:**

| Tool | Plan Mode | Act Mode | Rationale |
|------|-----------|----------|-----------|
| read_file | ✅ | ✅ | Information gathering |
| list_files | ✅ | ✅ | Exploration |
| execute_command | ✅ | ✅ | Investigation |
| write_to_file | ❌ | ✅ | Prevents premature implementation |
| replace_in_file | ❌ | ✅ | Prevents premature changes |
| plan_mode_respond | ✅ | ❌ | Plan-specific communication |

## Tool Execution System

The `ToolExecutor` class implements a comprehensive tool execution pipeline with validation, approval, and error handling.

### Tool Execution Algorithm

```typescript
async executeToolUse(block: ToolUse): Promise<void> {
    // 1. Validation Phase
    if (this.didAlreadyUseTool(block.name)) {
        await this.say("error", "Tool already used in this request")
        return
    }
    
    // 2. Plan Mode Restriction Check
    if (this.strictPlanModeEnabled && this.mode === "plan" && this.isPlanModeToolRestricted(block.name)) {
        await this.say("error", `Tool '${block.name}' is not available in PLAN MODE`)
        return
    }
    
    // 3. Parameter Validation
    if (!this.validateToolParameters(block)) {
        await this.sayAndCreateMissingParamError(block.name, missingParam)
        return
    }
    
    // 4. Auto-Approval Check
    const shouldAutoApprove = await this.shouldAutoApproveToolWithPath(block.name, path)
    
    // 5. Tool Execution
    try {
        const result = await this.executeTool(block)
        
        if (shouldAutoApprove) {
            await this.say("tool", result)
            this.taskState.consecutiveAutoApprovedRequestsCount++
        } else {
            const { response } = await this.ask("tool", result)
            if (response !== "yesButtonClicked") {
                this.taskState.didRejectTool = true
                return
            }
        }
    } catch (error) {
        await this.handleError("tool execution", error, block)
    }
}
```

### Auto-Approval Algorithm

The auto-approval system uses a pattern-matching algorithm to determine if tools should be automatically approved:

```typescript
class AutoApprove {
    shouldAutoApproveTool(toolName: ToolUseName): boolean | [boolean, boolean] {
        const alwaysApproved = ["read_file", "list_files", "search_files"]
        const neverApproved = ["execute_command"]
        const conditionallyApproved = ["write_to_file", "replace_in_file"]
        
        if (alwaysApproved.includes(toolName)) return true
        if (neverApproved.includes(toolName)) return false
        if (conditionallyApproved.includes(toolName)) return [true, false] // Check path
        
        return false
    }
    
    async shouldAutoApproveToolWithPath(toolName: ToolUseName, path: string): Promise<boolean> {
        if (!this.settings.enabled) return false
        
        // Path-based approval logic
        const approvedPatterns = this.settings.approvedPaths || []
        return approvedPatterns.some(pattern => this.matchesPattern(path, pattern))
    }
}
```

## Message State Handling

The `MessageStateHandler` manages conversation history with sophisticated merging and persistence algorithms.

### Message History Algorithm

```typescript
class MessageStateHandler {
    private apiConversationHistory: Anthropic.MessageParam[] = []
    private clineMessages: ClineMessage[] = []
    
    async saveClineMessagesAndUpdateHistory(): Promise<void> {
        // 1. Persist messages to disk
        await saveClineMessages(this.context, this.taskId, this.clineMessages)
        
        // 2. Calculate metrics
        const apiMetrics = getApiMetrics(combineApiRequests(combineCommandSequences(this.clineMessages.slice(1))))
        
        // 3. Update history item
        const historyItem: HistoryItem = {
            id: this.taskId,
            ulid: this.ulid,
            ts: lastMessage.ts,
            task: taskMessage.text,
            tokensIn: apiMetrics.totalTokensIn,
            tokensOut: apiMetrics.totalTokensOut,
            totalCost: apiMetrics.totalCost,
            // ... other metrics
        }
        
        // 4. Update task history
        await this.updateTaskHistory(historyItem)
    }
}
```

### Context Window Management

The system implements automatic context condensation when the window becomes too large:

```typescript
async handleContextWindowExceededError(): Promise<void> {
    const shouldCondense = this.useAutoCondense
    
    if (shouldCondense) {
        // Automatic condensation
        const summary = await this.condenseConversation()
        this.replaceHistoryWithSummary(summary)
    } else {
        // Manual user intervention
        await this.ask("context_exceeded", "Context window exceeded. Please condense or continue.")
    }
}
```

## Focus Chain Management

The Focus Chain system implements real-time progress tracking with file-based persistence and UI synchronization.

### Focus Chain Algorithm

```typescript
class FocusChainManager {
    async updateFCListFromToolResponse(taskProgress: string | undefined): Promise<void> {
        if (!taskProgress || !this.focusChainSettings.enabled) return
        
        // 1. Parse task progress from tool response
        const focusChainItems = extractFocusChainItemsFromText(taskProgress)
        
        // 2. Update internal state
        const markdownContent = createFocusChainMarkdownContent(focusChainItems)
        this.taskState.currentFocusChainChecklist = markdownContent
        
        // 3. Persist to disk
        await this.saveFocusChainToDisk(markdownContent)
        
        // 4. Update UI
        await this.postStateToWebview()
        
        // 5. Track telemetry
        const counts = parseFocusChainListCounts(markdownContent)
        telemetryService.captureFocusChainUpdate(this.taskId, counts.total, counts.completed)
    }
}
```

### File Watching Algorithm

The Focus Chain implements a file watcher with debouncing to handle external edits:

```typescript
async setupFocusChainFileWatcher() {
    const focusChainFilePath = getFocusChainFilePath(taskDir, this.taskId)
    
    this.focusChainFileWatcherCancel = HostProvider.watch.subscribeToFile({
        path: focusChainFilePath
    }, {
        onResponse: async (response) => {
            switch (response.type) {
                case "CHANGED":
                case "CREATED":
                    await this.updateFCListFromMarkdownFileAndNotifyUI()
                    break
                case "DELETED":
                    this.taskState.currentFocusChainChecklist = null
                    await this.postStateToWebview()
                    break
            }
        }
    })
}

private async updateFCListFromMarkdownFileAndNotifyUI() {
    // Debounce to prevent excessive updates
    if (this.fileUpdateDebounceTimer) {
        clearTimeout(this.fileUpdateDebounceTimer)
    }
    
    this.fileUpdateDebounceTimer = setTimeout(async () => {
        const markdownTodoList = await this.readFocusChainFromDisk()
        if (markdownTodoList && markdownTodoList !== this.taskState.currentFocusChainChecklist) {
            this.taskState.currentFocusChainChecklist = markdownTodoList
            this.taskState.todoListWasUpdatedByUser = true
            await this.postStateToWebview()
        }
    }, 300) // 300ms debounce
}
```

## Deep Planning Algorithm

The deep planning system implements a 4-phase structured approach to complex task planning.

### Deep Planning State Machine

```
Phase 1: Silent Investigation
    ↓
Phase 2: Discussion and Questions  
    ↓
Phase 3: Implementation Plan Document
    ↓
Phase 4: Implementation Task Creation
```

### Phase 1: Silent Investigation Algorithm

```typescript
// Triggered by /deep-planning slash command
const deepPlanningInstructions = `
## STEP 1: Silent Investigation

Execute these research commands:
1. find . -type f -name "*.{py,js,ts,java,cpp,go}" | head -30
2. grep -r "class|function|def|interface" --include="*.{py,js,ts}" .
3. grep -r "import|from|require|#include" . | sort | uniq
4. find . -name "requirements*.txt" -o -name "package.json" | xargs cat
5. grep -r "TODO|FIXME|XXX|HACK" --include="*.{py,js,ts}" .

// Perform research without commentary until complete understanding achieved
`
```

### Phase 2: Discussion Algorithm

The system uses targeted questioning to clarify implementation details:

```typescript
const questioningPhase = {
    // Only ask essential questions
    categories: [
        "Clarifying ambiguous requirements",
        "Choosing between valid approaches", 
        "Confirming system behavior assumptions",
        "Understanding technical preferences"
    ],
    
    // Keep questions direct and specific
    format: "Brief, targeted questions only when necessary"
}
```

### Phase 3: Plan Document Generation

```typescript
const planStructure = {
    sections: [
        "Overview",      // Goal and high-level approach
        "Types",         // Complete type definitions
        "Files",         // Exact files to create/modify/delete
        "Functions",     // New and modified functions with signatures
        "Classes",       // Class modifications and inheritance
        "Dependencies",  // Package requirements and versions
        "Testing",       // Validation strategies
        "Implementation Order" // Step-by-step execution sequence
    ]
}
```

### Phase 4: Task Creation Algorithm

```typescript
async createImplementationTask(planDocument: string): Promise<void> {
    // 1. Generate implementation steps from plan
    const steps = extractImplementationSteps(planDocument)
    
    // 2. Create new task with context
    const taskContext = `
Refer to @${planDocumentPath} for complete breakdown.

task_progress Items:
${steps.map((step, i) => `- [ ] Step ${i+1}: ${step}`).join('\n')}

Request: Switch to "act mode" for implementation.
    `
    
    // 3. Use new_task tool
    await this.executeToolUse({
        name: "new_task",
        params: { context: taskContext }
    })
}
```

## Key Algorithms and Data Flows

### 1. Message Processing Pipeline

```
User Input → Parse Mentions → Parse Slash Commands → Mode Validation → Tool Execution → Response Generation → UI Update
```

### 2. Context Management Flow

```
New Message → Check Context Window → Auto-Condense (if enabled) → Add to History → Update Metrics → Persist to Disk
```

### 3. Tool Approval Flow

```
Tool Call → Parameter Validation → Mode Restriction Check → Auto-Approval Check → User Approval (if needed) → Execution → Result Processing
```

### 4. State Synchronization

```
State Change → Update TaskState → Notify Components → Update UI → Persist Changes → Telemetry
```

## Implementation Patterns

### 1. Dependency Injection Pattern

The Task class uses constructor injection for all dependencies:

```typescript
constructor(
    controller: Controller,
    mcpHub: McpHub,
    updateTaskHistory: (historyItem: HistoryItem) => Promise<HistoryItem[]>,
    // ... other dependencies
) {
    // Initialize with injected dependencies
}
```

### 2. Observer Pattern

Components observe state changes and react accordingly:

```typescript
// TaskState changes trigger UI updates
async postStateToWebview(): Promise<void> {
    // Notify all observers of state change
}

// File changes trigger Focus Chain updates
onFileChange: async (response) => {
    await this.updateFCListFromMarkdownFileAndNotifyUI()
}
```

### 3. Strategy Pattern

Different behaviors based on mode:

```typescript
// Different API providers for different modes
const currentProvider = this.mode === "plan" 
    ? apiConfiguration.planModeApiProvider 
    : apiConfiguration.actModeApiProvider

// Different tool restrictions per mode
const isRestricted = this.mode === "plan" && this.isPlanModeToolRestricted(toolName)
```

### 4. State Machine Pattern

TaskState implements multiple state machines:

```typescript
// Streaming state machine
enum StreamingState {
    IDLE = "idle",
    WAITING_FOR_FIRST_CHUNK = "waiting_for_first_chunk", 
    STREAMING = "streaming",
    COMPLETE = "complete"
}

// Plan response state machine
enum PlanResponseState {
    NOT_AWAITING = "not_awaiting",
    AWAITING_RESPONSE = "awaiting_response",
    SWITCHING_MODE = "switching_mode"
}
```

### 5. Command Pattern

Tool execution implements command pattern:

```typescript
interface ToolCommand {
    name: string
    params: Record<string, any>
    execute(): Promise<ToolResponse>
    validate(): boolean
    requiresApproval(): boolean
}
```

This comprehensive task system provides a robust foundation for AI assistant interactions with sophisticated state management, tool safety, and user experience optimization. The algorithms ensure consistency, safety, and performance across all interaction modes.