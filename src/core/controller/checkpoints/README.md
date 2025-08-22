# Checkpoints Controller Implementation Guide

## Overview

This directory contains the checkpoint controller implementation for Cline's version control system. The checkpoint system provides time-travel functionality, allowing users to view differences between states and restore previous versions of their workspace without interfering with their main Git repository. This guide provides a comprehensive understanding of how the checkpoint system works, with emphasis on the algorithms and patterns used.

## Architecture

### Core Components

1. **Checkpoint Diff Controller**: Displays file differences between checkpoints
2. **Checkpoint Restore Controller**: Restores workspace to previous states
3. **Shadow Git Repository**: Isolated version control system
4. **Multi-file Diff Engine**: Advanced diff visualization
5. **State Management**: Task and workspace state coordination

### File Structure

```
src/core/controller/checkpoints/
├── checkpointDiff.ts      # Diff visualization controller
├── checkpointRestore.ts   # State restoration controller
└── README.md             # This documentation
```

## Algorithm Deep Dive

### 1. Checkpoint Diff Algorithm

#### Entry Point Function

```typescript
export async function checkpointDiff(
    controller: Controller, 
    request: Int64Request
): Promise<Empty>
```

**Algorithm Flow:**
```typescript
// Step 1: Validate request
if (request.value) {
    // Step 2: Delegate to task's multifile diff system
    await controller.task?.presentMultifileDiff(request.value, false)
}

// Step 3: Return empty response (operation is async UI update)
return Empty
```

#### Multi-file Diff Presentation Algorithm

Located in `src/core/task/multifile-diff.ts`:

```typescript
export async function showChangedFilesDiff(
    messageStateHandler: MessageStateHandler,
    checkpointTracker: CheckpointTracker,
    messageTs: number,
    seeNewChangesSinceLastTaskCompletion: boolean,
)
```

**Core Algorithm Steps:**

```typescript
// Step 1: Message Resolution
const clineMessages = messageStateHandler.getClineMessages()
const messageIndex = clineMessages.findIndex((m) => m.ts === messageTs)
const message = clineMessages[messageIndex]

// Step 2: Checkpoint Hash Validation
const lastCheckpointHash = message.lastCheckpointHash
if (!lastCheckpointHash) {
    console.error("No checkpoint hash found")
    return
}

// Step 3: Changed Files Retrieval
const changedFiles = await getChangedFiles(
    messageStateHandler,
    checkpointTracker,
    seeNewChangesSinceLastTaskCompletion,
    messageIndex,
    lastCheckpointHash,
)

// Step 4: Diff Generation & Display
const title = seeNewChangesSinceLastTaskCompletion ? "New changes" : "Changes since snapshot"
const diffs = changedFiles.map((file) => ({
    filePath: file.absolutePath,
    leftContent: file.before,
    rightContent: file.after,
}))
HostProvider.diff.openMultiFileDiff({ title, diffs })
```

#### Changed Files Detection Algorithm

```typescript
async function getChangedFiles(
    messageStateHandler: MessageStateHandler,
    checkpointTracker: CheckpointTracker,
    changesSinceLastTaskCompletion: boolean,
    messageIndex: number,
    lastCheckpointHash: string,
): Promise<ChangedFile[]>
```

**Branching Logic:**
```typescript
let changedFiles
if (changesSinceLastTaskCompletion) {
    // Algorithm: Find changes since last task completion
    changedFiles = await getChangesSinceLastTaskCompletion(
        messageStateHandler,
        checkpointTracker,
        messageIndex,
        lastCheckpointHash,
    )
} else {
    // Algorithm: Direct diff between current state and checkpoint
    changedFiles = await checkpointTracker.getDiffSet(lastCheckpointHash)
}
```

#### Task Completion Detection Algorithm

```typescript
async function getChangesSinceLastTaskCompletion(
    messageStateHandler: MessageStateHandler,
    checkpointTracker: CheckpointTracker,
    messageIndex: number,
    lastCheckpointHash: string,
): Promise<ChangedFile[]>
```

**Search Algorithm:**
```typescript
// Step 1: Find last completion message
const lastTaskCompletedMessageCheckpointHash = findLast(
    messageStateHandler.getClineMessages().slice(0, messageIndex),
    (m) => m.say === "completion_result",
)?.lastCheckpointHash

// Step 2: Determine comparison point
if (lastTaskCompletedMessageCheckpointHash) {
    // Compare against last completion
    return await checkpointTracker.getDiffSet(
        lastCheckpointHash,
        lastTaskCompletedMessageCheckpointHash
    )
} else {
    // No previous completion - compare against initial state
    return await checkpointTracker.getDiffSet(lastCheckpointHash)
}
```

**Time Complexity**: O(n) where n is the number of messages before the target message
**Space Complexity**: O(m) where m is the size of the changed files

### 2. Checkpoint Restore Algorithm

#### Main Restore Function

```typescript
export async function checkpointRestore(
    controller: Controller, 
    request: CheckpointRestoreRequest
): Promise<Empty>
```

**Multi-Stage Restoration Process:**

```typescript
// Stage 1: Task Cancellation & State Cleanup
await controller.cancelTask()

// Stage 2: Request Validation
if (request.number) {
    // Stage 3: Task Initialization Wait
    await pWaitFor(() => controller.task?.taskState.isInitialized === true, {
        timeout: 3_000,
    })
    
    // Stage 4: Checkpoint Restoration
    await controller.task?.restoreCheckpoint(
        request.number, 
        request.restoreType as ClineCheckpointRestore, 
        request.offset
    )
}

// Stage 5: Response
return Empty.create({})
```

#### Task Cancellation Algorithm

**Synchronization Strategy:**
```typescript
// Critical: Cancel active task to prevent state conflicts
await controller.cancelTask()

// Note: cancelTask awaits abortTask, which awaits diffViewProvider.revertChanges,
// which reverts any edited files, allowing us to reset to a checkpoint rather than 
// running into a state where the revertChanges function is called alongside or 
// after the checkpoint reset
```

#### Initialization Wait Algorithm

```typescript
await pWaitFor(() => controller.task?.taskState.isInitialized === true, {
    timeout: 3_000,
}).catch((error) => {
    console.log("Failed to init new Cline instance to restore checkpoint", error)
    HostProvider.window.showMessage({
        type: ShowMessageType.ERROR,
        message: "Failed to restore checkpoint",
    })
    throw error
})
```

**Polling Strategy:**
- **Polling Interval**: Default `pWaitFor` implementation
- **Timeout**: 3 seconds maximum wait
- **Error Handling**: User notification + exception propagation

#### Core Restoration Algorithm

Located in `src/core/task/index.ts`:

```typescript
async restoreCheckpoint(
    messageTs: number, 
    restoreType: ClineCheckpointRestore, 
    offset?: number
)
```

**Detailed Algorithm Implementation:**

```typescript
// Step 1: Message Index Resolution with Offset
const clineMessages = this.messageStateHandler.getClineMessages()
const messageIndex = clineMessages.findIndex((m) => m.ts === messageTs) - (offset || 0)

// Step 2: Checkpoint Hash Discovery
const lastHashIndex = findLastIndex(
    clineMessages.slice(0, messageIndex), 
    (m) => m.lastCheckpointHash !== undefined
)

// Step 3: Message Validation
const message = clineMessages[messageIndex]
const lastMessageWithHash = clineMessages[lastHashIndex]

if (!message) {
    console.error("Message not found", clineMessages)
    return
}

// Step 4: Hash Extraction & Validation
let hashToRestore: string
if (message.lastCheckpointHash) {
    hashToRestore = message.lastCheckpointHash
} else if (lastMessageWithHash?.lastCheckpointHash) {
    hashToRestore = lastMessageWithHash.lastCheckpointHash
} else {
    console.error("No checkpoint hash found for restoration")
    return
}

// Step 5: Restoration Type Processing
switch (restoreType) {
    case "task":
        await this.restoreTaskToCheckpoint(messageIndex, hashToRestore)
        break
    case "workspace":
        await this.restoreWorkspaceToCheckpoint(hashToRestore)
        break
    case "taskAndWorkspace":
        await this.restoreTaskToCheckpoint(messageIndex, hashToRestore)
        await this.restoreWorkspaceToCheckpoint(hashToRestore)
        break
}
```

### 3. Restoration Type Algorithms

#### Task Restoration Algorithm

```typescript
private async restoreTaskToCheckpoint(messageIndex: number, hashToRestore: string) {
    // Step 1: Message Truncation
    const clineMessages = this.messageStateHandler.getClineMessages()
    const truncatedMessages = clineMessages.slice(0, messageIndex + 1)
    
    // Step 2: State Update
    this.messageStateHandler.setClineMessages(truncatedMessages)
    
    // Step 3: UI Refresh
    await this.refreshUIWithUpdatedMessages()
}
```

#### Workspace Restoration Algorithm

```typescript
private async restoreWorkspaceToCheckpoint(hashToRestore: string) {
    // Step 1: Validation
    if (!this.checkpointTracker) {
        throw new Error("Checkpoint tracker not available")
    }
    
    // Step 2: Git Reset Operation
    await this.checkpointTracker.restoreToCheckpoint(hashToRestore)
    
    // Step 3: File System Sync
    await this.syncFileSystemWithGit()
}
```

#### Combined Restoration Algorithm

```typescript
private async restoreTaskAndWorkspace(messageIndex: number, hashToRestore: string) {
    // Sequential execution to maintain consistency
    await this.restoreTaskToCheckpoint(messageIndex, hashToRestore)
    await this.restoreWorkspaceToCheckpoint(hashToRestore)
    
    // Additional validation step
    await this.validateRestoreConsistency()
}
```

### 4. Shadow Git Operations

#### Shadow Git Path Algorithm

```typescript
// From CheckpointUtils.ts
export async function getShadowGitPath(
    globalStoragePath: string,
    taskId: string, 
    cwdHash: string
): Promise<string> {
    const shadowGitDir = path.join(
        globalStoragePath,
        "shadow-git",
        cwdHash,  // Workspace-specific directory
        taskId    // Task-specific subdirectory
    )
    
    await fs.mkdir(shadowGitDir, { recursive: true })
    return shadowGitDir
}
```

#### Working Directory Hash Algorithm

```typescript
export function hashWorkingDir(workingDir: string): string {
    // Create deterministic hash from absolute path
    const crypto = require('crypto')
    return crypto
        .createHash('sha256')
        .update(path.resolve(workingDir))
        .digest('hex')
        .substring(0, 16) // Use first 16 characters for readability
}
```

#### Git Initialization Algorithm

```typescript
// From GitOperations.ts
async initShadowGit(gitPath: string, workingDir: string, taskId: string) {
    // Step 1: Repository Initialization
    const git = simpleGit(gitPath)
    await git.init()
    
    // Step 2: Working Tree Configuration
    await git.raw(['config', 'core.worktree', workingDir])
    
    // Step 3: User Configuration
    await git.addConfig('user.name', 'Cline Checkpoint System')
    await git.addConfig('user.email', 'checkpoint@cline.dev')
    
    // Step 4: Branch Setup
    const branchName = `checkpoint-${taskId}`
    await git.checkoutLocalBranch(branchName)
    
    // Step 5: Initial Commit
    await this.createInitialCommit(git, taskId)
}
```

### 5. Diff Generation Algorithms

#### File Change Detection

```typescript
interface ChangedFile {
    relativePath: string
    absolutePath: string
    before: string  // File content at checkpoint
    after: string   // Current file content
}
```

#### Content Comparison Algorithm

```typescript
async function generateFileDiff(
    checkpointTracker: CheckpointTracker,
    filePath: string,
    checkpointHash: string
): Promise<ChangedFile | null> {
    
    // Step 1: Get current content
    const currentContent = await fs.readFile(filePath, 'utf8')
    
    // Step 2: Get checkpoint content
    const checkpointContent = await checkpointTracker.getFileAtCheckpoint(
        checkpointHash, 
        filePath
    )
    
    // Step 3: Content comparison
    if (currentContent === checkpointContent) {
        return null // No changes
    }
    
    // Step 4: Create diff object
    return {
        relativePath: path.relative(process.cwd(), filePath),
        absolutePath: filePath,
        before: checkpointContent,
        after: currentContent,
    }
}
```

#### Multi-file Diff Aggregation

```typescript
async function aggregateChangedFiles(
    checkpointTracker: CheckpointTracker,
    checkpointHash: string,
    fileList: string[]
): Promise<ChangedFile[]> {
    
    const changes: ChangedFile[] = []
    
    // Parallel processing for performance
    const diffPromises = fileList.map(filePath => 
        generateFileDiff(checkpointTracker, filePath, checkpointHash)
    )
    
    const diffResults = await Promise.all(diffPromises)
    
    // Filter out null results (unchanged files)
    return diffResults.filter((diff): diff is ChangedFile => diff !== null)
}
```

### 6. Error Handling Patterns

#### Graceful Degradation Strategy

```typescript
try {
    const changedFiles = await getChangedFiles(/* parameters */)
    if (!changedFiles.length) {
        HostProvider.window.showMessage({
            type: ShowMessageType.INFORMATION,
            message: "No changes found",
        })
    }
    return changedFiles
} catch (error) {
    const errorMessage = error instanceof Error ? error.message : "Unknown error"
    HostProvider.window.showMessage({
        type: ShowMessageType.ERROR,
        message: "Failed to retrieve diff set: " + errorMessage,
    })
    return [] // Return empty array to prevent cascading failures
}
```

#### Checkpoint Validation Algorithm

```typescript
private validateCheckpointHash(hash: string | undefined): string | null {
    if (!hash) {
        console.error("No checkpoint hash provided")
        return null
    }
    
    // Clean legacy hash format
    const cleanHash = hash.startsWith("HEAD ") ? hash.slice(5) : hash
    
    // Validate hash format (Git SHA-1)
    const hashRegex = /^[a-f0-9]{40}$/i
    if (!hashRegex.test(cleanHash)) {
        console.error("Invalid checkpoint hash format:", cleanHash)
        return null
    }
    
    return cleanHash
}
```

#### State Consistency Validation

```typescript
private async validateRestoreConsistency(): Promise<boolean> {
    try {
        // Verify task state matches checkpoint
        const currentHash = await this.checkpointTracker?.getCurrentCommitHash()
        const expectedHash = this.getCurrentMessage()?.lastCheckpointHash
        
        if (currentHash !== expectedHash) {
            console.warn("State inconsistency detected after restore")
            return false
        }
        
        return true
    } catch (error) {
        console.error("Failed to validate restore consistency:", error)
        return false
    }
}
```

## Performance Characteristics

### Time Complexity Analysis

| Operation | Time Complexity | Space Complexity | Notes |
|-----------|----------------|------------------|-------|
| Message Index Lookup | O(n) | O(1) | Linear search through messages |
| Checkpoint Hash Discovery | O(n) | O(1) | Reverse search for hash |
| File Diff Generation | O(f × s) | O(s) | f=files, s=average file size |
| Git Operations | O(f × log f) | O(f) | Git's internal algorithms |
| UI Update | O(d) | O(d) | d=number of diffs to display |

### Memory Optimization Strategies

1. **Lazy Loading**: Only load file contents when needed
2. **Streaming Diffs**: Process large files in chunks
3. **Content Caching**: Cache frequently accessed checkpoint content
4. **Garbage Collection**: Clean up unused diff objects

### Performance Optimization Patterns

```typescript
// Batch file operations
const filePromises = changedFiles.map(file => 
    this.processFileAsync(file)
)
const results = await Promise.all(filePromises)

// Memory-efficient content comparison
const isChanged = await this.compareFileHashes(
    currentFile, 
    checkpointFile
) // Compare hashes before loading full content
```

## Security Considerations

### Path Validation

```typescript
private validateFilePath(filePath: string): boolean {
    // Prevent directory traversal
    const normalizedPath = path.normalize(filePath)
    if (normalizedPath.includes('..')) {
        throw new Error("Invalid file path: directory traversal detected")
    }
    
    // Ensure path is within workspace
    const workspacePath = this.getWorkspacePath()
    if (!normalizedPath.startsWith(workspacePath)) {
        throw new Error("File path outside workspace not allowed")
    }
    
    return true
}
```

### Content Sanitization

```typescript
private sanitizeFileContent(content: string): string {
    // Remove potential sensitive information
    return content
        .replace(/password\s*[:=]\s*[\'"]\w+[\'"]/gi, 'password: "[REDACTED]"')
        .replace(/api[_-]?key\s*[:=]\s*[\'"]\w+[\'"]/gi, 'api_key: "[REDACTED]"')
        .replace(/token\s*[:=]\s*[\'"]\w+[\'"]/gi, 'token: "[REDACTED]"')
}
```

## Testing Strategies

### Unit Testing Patterns

```typescript
describe('checkpointDiff', () => {
    test('handles valid timestamp request', async () => {
        const mockController = createMockController()
        const request = { value: BigInt(1234567890) }
        
        const result = await checkpointDiff(mockController, request)
        
        expect(mockController.task?.presentMultifileDiff)
            .toHaveBeenCalledWith(1234567890, false)
        expect(result).toEqual(Empty)
    })
    
    test('handles null timestamp gracefully', async () => {
        const mockController = createMockController()
        const request = { value: null }
        
        const result = await checkpointDiff(mockController, request)
        
        expect(mockController.task?.presentMultifileDiff)
            .not.toHaveBeenCalled()
        expect(result).toEqual(Empty)
    })
})
```

### Integration Testing

```typescript
describe('Checkpoint Restore Integration', () => {
    test('complete restore workflow', async () => {
        // Setup: Create checkpoint
        const checkpointHash = await createTestCheckpoint()
        
        // Setup: Modify workspace
        await modifyTestFiles()
        
        // Action: Restore checkpoint
        await checkpointRestore(controller, {
            number: checkpointHash,
            restoreType: "taskAndWorkspace"
        })
        
        // Verify: Workspace restored
        expect(await getFileContent('test.txt'))
            .toEqual(originalContent)
        
        // Verify: Task state restored
        expect(controller.task?.getCurrentMessageIndex())
            .toEqual(expectedIndex)
    })
})
```

### Error Scenario Testing

```typescript
describe('Error Handling', () => {
    test('handles corrupted checkpoint gracefully', async () => {
        const invalidRequest = { 
            number: "invalid-hash",
            restoreType: "workspace" 
        }
        
        await expect(checkpointRestore(controller, invalidRequest))
            .resolves.not.toThrow()
        
        expect(mockShowMessage).toHaveBeenCalledWith({
            type: ShowMessageType.ERROR,
            message: expect.stringContaining("Failed to restore")
        })
    })
})
```

## Common Patterns and Best Practices

### 1. Async Operation Pattern

```typescript
async function safeAsyncOperation<T>(
    operation: () => Promise<T>,
    fallback: T,
    errorMessage: string
): Promise<T> {
    try {
        return await operation()
    } catch (error) {
        console.error(errorMessage, error)
        HostProvider.window.showMessage({
            type: ShowMessageType.ERROR,
            message: errorMessage
        })
        return fallback
    }
}
```

### 2. State Validation Pattern

```typescript
private ensureValidState(): void {
    if (!this.task) {
        throw new Error("Task not initialized")
    }
    if (!this.checkpointTracker) {
        throw new Error("Checkpoint tracker not available")
    }
    if (!this.enableCheckpoints) {
        throw new Error("Checkpoints disabled in settings")
    }
}
```

### 3. Resource Cleanup Pattern

```typescript
async dispose(): Promise<void> {
    try {
        await this.checkpointTracker?.cleanup()
        this.messageStateHandler?.dispose()
        this.diffViewProvider?.dispose()
    } catch (error) {
        console.error("Error during cleanup:", error)
        // Continue cleanup despite errors
    }
}
```

## Debugging and Troubleshooting

### Common Issues

1. **Checkpoint Hash Not Found**
   - Verify message has `lastCheckpointHash` property
   - Check checkpoint tracker initialization
   - Validate shadow git repository exists

2. **Restore Timeout**
   - Increase `pWaitFor` timeout
   - Check task initialization logic
   - Verify no blocking operations

3. **File Diff Errors**
   - Validate file paths exist
   - Check file permissions
   - Ensure shadow git is properly configured

### Debug Logging

```typescript
console.debug("Checkpoint operation:", {
    messageTs,
    messageIndex,
    checkpointHash,
    restoreType,
    offset,
    currentWorkingDir: process.cwd()
})
```

### Monitoring Points

- Checkpoint creation frequency
- Restore operation success rate
- Diff generation performance
- Shadow git repository size
- Memory usage during operations

## Integration with Core Systems

### Task System Integration

```typescript
// Checkpoint system integrates with task lifecycle
class Task {
    async createCheckpoint(): Promise<string> {
        return await this.checkpointTracker?.createCheckpoint()
    }
    
    async attachCheckpointToMessage(messageTs: number, hash: string): void {
        const message = this.findMessage(messageTs)
        message.lastCheckpointHash = hash
        await this.persistMessages()
    }
}
```

### UI Integration

```typescript
// Diff viewer integration
HostProvider.diff.openMultiFileDiff({
    title: "Changes since checkpoint",
    diffs: formattedDiffs,
    showLineNumbers: true,
    enableSyntaxHighlighting: true
})
```

### File System Integration

```typescript
// Shadow git worktree configuration
await git.raw(['config', 'core.worktree', workspaceDir])
await git.raw(['config', 'core.excludesfile', excludeFilePath])
```

## Future Enhancements

### Performance Improvements

1. **Incremental Diffs**: Only compute diffs for changed files
2. **Diff Caching**: Cache computed diffs for frequently accessed checkpoints
3. **Parallel Processing**: Parallelize file operations across multiple workers
4. **Compression**: Compress checkpoint data to reduce storage

### Feature Additions

1. **Checkpoint Branching**: Support multiple checkpoint branches
2. **Selective Restore**: Restore only specific files from checkpoints
3. **Checkpoint Merging**: Merge changes from different checkpoints
4. **Automated Checkpoints**: Create checkpoints based on significant events

### User Experience

1. **Visual Timeline**: Interactive timeline of checkpoints
2. **Smart Suggestions**: Suggest optimal restore points
3. **Diff Analytics**: Show statistics about changes between checkpoints
4. **Export Capabilities**: Export diffs to various formats

This implementation provides a robust foundation for checkpoint management in Cline, with comprehensive error handling, performance optimization, and extensibility for future enhancements.