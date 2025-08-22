# Prompts System Implementation Guide

## Table of Contents
1. [Overview](#overview)
2. [System Architecture](#system-architecture)
3. [System Prompt Building Algorithm](#system-prompt-building-algorithm)
4. [Command Processing System](#command-processing-system)
5. [Context Management and Summarization](#context-management-and-summarization)
6. [Response Formatting Engine](#response-formatting-engine)
7. [User Instructions Integration](#user-instructions-integration)
8. [Model Family Strategies](#model-family-strategies)
9. [Slash Command Processing](#slash-command-processing)
10. [Deep Planning Algorithm](#deep-planning-algorithm)
11. [Key Algorithms and Data Flows](#key-algorithms-and-data-flows)
12. [Implementation Patterns](#implementation-patterns)

## Overview

The Cline prompts system is a sophisticated prompt engineering framework that dynamically constructs context-aware system prompts, processes user commands, manages conversation state, and formats responses. It implements advanced algorithms for context summarization, command processing, and model-specific prompt optimization.

### Key Design Principles
- **Dynamic Prompt Construction**: Context-aware system prompt building
- **Model-Specific Optimization**: Different strategies for different AI model families
- **Command Processing Pipeline**: Sophisticated slash command and mention parsing
- **Context Preservation**: Advanced summarization and context management
- **Response Formatting**: Structured formatting for tool results and user feedback

## System Architecture

### Core Components

The prompts system consists of several interconnected modules:

```typescript
// Primary Components
src/core/prompts/
├── build-system-prompt.ts          // Main prompt building orchestrator
├── contextManagement.ts            // Context summarization algorithms
├── responses.ts                    // Response formatting functions
├── commands.md                     // Slash command prompt templates
├── loadMcpDocumentation.ts         // MCP integration prompts
└── system-prompt/
    ├── generic-system-prompt.ts    // Standard model prompts
    ├── families/
    │   └── next-gen-models/
    │       └── next-gen-system-prompt.ts  // Advanced model prompts
    ├── user-instructions/
    │   └── addUserInstructions.ts  // User rule integration
    └── utils.ts                    // Model family detection
```

### Component Responsibilities

1. **build-system-prompt.ts**: Main orchestrator that selects appropriate prompt strategy
2. **contextManagement.ts**: Implements summarization and continuation algorithms
3. **responses.ts**: Formats tool results, errors, and user feedback
4. **commands.md**: Contains templates for all slash command behaviors
5. **Model Family Detection**: Determines optimal prompting strategy per model

## System Prompt Building Algorithm

The system prompt building process implements a sophisticated decision tree that adapts to different model capabilities and user contexts.

### Main Building Algorithm

```typescript
export const buildSystemPrompt = async (
    cwd: string,
    supportsBrowserUse: boolean,
    mcpHub: McpHub,
    browserSettings: BrowserSettings,
    apiHandlerModel: ApiHandlerModel,
    focusChainSettings: FocusChainSettings,
) => {
    // 1. Model Family Detection
    if (isNextGenModelFamily(apiHandlerModel)) {
        return SYSTEM_PROMPT_NEXT_GEN(cwd, supportsBrowserUse, mcpHub, browserSettings, focusChainSettings)
    } else {
        return SYSTEM_PROMPT_GENERIC(cwd, supportsBrowserUse, mcpHub, browserSettings, focusChainSettings)
    }
}
```

### Model Family Detection Algorithm

```typescript
// Hierarchical model family classification
export function isNextGenModelFamily(apiHandlerModel: ApiHandlerModel): boolean {
    return (
        isClaude4ModelFamily(apiHandlerModel) ||
        isGemini2dot5ModelFamily(apiHandlerModel) ||
        isGrok4ModelFamily(apiHandlerModel) ||
        isGPT5ModelFamily(apiHandlerModel)
    )
}

// Pattern matching algorithms for each family
export function isClaude4ModelFamily(apiHandlerModel: ApiHandlerModel): boolean {
    const modelId = apiHandlerModel.id.toLowerCase()
    return (
        modelId.includes("sonnet-4") || 
        modelId.includes("opus-4") || 
        modelId.includes("4-sonnet") || 
        modelId.includes("4-opus")
    )
}
```

### Prompt Construction Pipeline

```
User Context → Model Detection → Family Strategy → Tool Definitions → User Instructions → Final Prompt
```

**Algorithm: Prompt Assembly**
```
1. Initialize base prompt template
2. Detect model family (Next-Gen vs Generic)
3. Add tool definitions based on capabilities
4. Integrate MCP documentation if available
5. Add browser tools if supported
6. Append focus chain tools if enabled
7. Merge user custom instructions
8. Apply model-specific optimizations
9. Return final assembled prompt
```

## Command Processing System

The command processing system implements a multi-stage parsing pipeline that handles slash commands, mentions, and special instructions.

### Command Processing Pipeline

```typescript
// Located in src/core/slash-commands/index.ts
export async function parseSlashCommands(
    text: string,
    localWorkflowToggles: ClineRulesToggles,
    globalWorkflowToggles: ClineRulesToggles,
    ulid: string,
): Promise<{ processedText: string; needsClinerulesFileCheck: boolean }>
```

### Slash Command Processing Algorithm

```
Input Text → Pattern Matching → Command Validation → Template Injection → Workflow Resolution → Final Processing
```

**Algorithm: Command Pattern Matching**
```typescript
const tagPatterns = [
    { tag: "task", regex: /<task>(\s*\/([a-zA-Z0-9_.-]+))(\s+.+?)?\s*<\/task>/is },
    { tag: "feedback", regex: /<feedback>(\s*\/([a-zA-Z0-9_.-]+))(\s+.+?)?\s*<\/feedback>/is },
    { tag: "answer", regex: /<answer>(\s*\/([a-zA-Z0-9_.-]+))(\s+.+?)?\s*<\/answer>/is },
    { tag: "user_message", regex: /<user_message>(\s*\/([a-zA-Z0-9_.-]+))(\s+.+?)?\s*<\/user_message>/is },
]

// Processing algorithm:
1. For each tag pattern:
   a. Execute regex match against input text
   b. Extract command name and parameters
   c. Check against supported commands list
   d. If built-in command: apply template replacement
   e. If custom workflow: load from file system
   f. Replace original command with expanded template
```

### Built-in Command Templates

```typescript
const commandReplacements: Record<string, string> = {
    newtask: newTaskToolResponse(),
    smol: condenseToolResponse(),
    compact: condenseToolResponse(),
    newrule: newRuleToolResponse(),
    reportbug: reportBugToolResponse(),
    "deep-planning": deepPlanningToolResponse(),
}
```

### Command Priority Algorithm

```
1. Check if command matches SUPPORTED_DEFAULT_COMMANDS
2. If yes: Apply built-in template replacement
3. If no: Search local workflow files
4. If found: Load custom workflow content
5. If not found: Search global workflow files
6. If found: Load global workflow content
7. If none found: Return original text unchanged
```

## Context Management and Summarization

The context management system implements sophisticated algorithms for conversation summarization and context preservation when approaching token limits.

### Summarization Algorithm

```typescript
export const summarizeTask = (focusChainEnabled: boolean) => `
<explicit_instructions type="summarize_task">
// Comprehensive 8-section summarization template
1. Primary Request and Intent
2. Key Technical Concepts  
3. Files and Code Sections
4. Problem Solving
5. Pending Tasks
6. Current Work
7. Optional Next Step
8. Focus Chain Progress (if enabled)
</explicit_instructions>
```

### Context Summarization Process

```
Context Overflow Detection → Analysis Phase → Summarization → Continuation Setup → History Replacement
```

**Algorithm: Context Window Management**
```
1. Monitor conversation token count
2. When approaching limit:
   a. Trigger summarization prompt
   b. AI analyzes conversation chronologically
   c. Extract key technical details
   d. Preserve current work state
   e. Identify next steps
   f. Generate structured summary
3. Replace conversation history with summary
4. Add continuation prompt
5. Resume conversation with preserved context
```

### Continuation Algorithm

```typescript
export const continuationPrompt = (summaryText: string) => `
This session is being continued from a previous conversation that ran out of context. 
The conversation is summarized below: ${summaryText}.

Please continue the conversation from where we left it off without asking the user any further questions.
Pay special attention to the most recent user message when responding.
`
```

## Response Formatting Engine

The response formatting system provides sophisticated formatting for tool results, errors, and user interactions.

### Response Formatting Categories

#### 1. Tool Result Formatting
```typescript
toolResult: (text: string, images?: string[], fileString?: string) => {
    // Algorithm: Multi-modal response construction
    const toolResultOutput = []
    
    // Add text content
    const textBlock: Anthropic.TextBlockParam = { type: "text", text }
    toolResultOutput.push(textBlock)
    
    // Add image blocks if present
    if (images && images.length > 0) {
        const imageBlocks = formatImagesIntoBlocks(images)
        toolResultOutput.push(...imageBlocks)
    }
    
    // Add file content if present
    if (fileString) {
        const fileBlock: Anthropic.TextBlockParam = { type: "text", text: fileString }
        toolResultOutput.push(fileBlock)
    }
    
    return toolResultOutput
}
```

#### 2. Error Formatting Algorithm
```typescript
// Context-aware error messaging
diffError: (relPath: string, originalContent: string | undefined) => {
    // Algorithm: Error context reconstruction
    return `
    Error Analysis:
    1. SEARCH block content mismatch detected
    2. File reverted to original state
    3. Provide current file content
    4. Suggest corrective action
    5. Fallback recommendation
    `
}
```

#### 3. File Edit Response Algorithm
```typescript
fileEditWithUserChanges: (
    relPath: string,
    userEdits: string,
    autoFormattingEdits: string | undefined,
    finalContent: string | undefined,
    newProblemsMessage: string | undefined,
) => {
    // Algorithm: Multi-stage edit response construction
    return `
    1. Acknowledge user edits: ${userEdits}
    2. Document auto-formatting: ${autoFormattingEdits}
    3. Show final content: ${finalContent}
    4. Provide guidance for future edits
    5. Include problem diagnostics: ${newProblemsMessage}
    `
}
```

### Response Processing Pipeline

```
Tool Result → Content Analysis → Format Selection → Multi-modal Assembly → Context Addition → Final Response
```

## User Instructions Integration

The user instructions system implements a hierarchical merging algorithm that combines multiple sources of user customization.

### Instruction Sources Hierarchy

```typescript
export function addUserInstructions(
    globalClineRulesFileInstructions?: string,     // Lowest priority
    localClineRulesFileInstructions?: string,      // Medium priority  
    localCursorRulesFileInstructions?: string,     // Medium priority
    localCursorRulesDirInstructions?: string,      // Medium priority
    localWindsurfRulesFileInstructions?: string,   // Medium priority
    clineIgnoreInstructions?: string,              // Highest priority
    preferredLanguageInstructions?: string,        // Highest priority
)
```

### Instruction Merging Algorithm

```
Language Preferences → Global Rules → Local Rules → External Tool Rules → Ignore Rules → Final Instructions
```

**Algorithm: Instruction Assembly**
```typescript
let customInstructions = ""

// 1. Add language preferences (highest priority)
if (preferredLanguageInstructions) {
    customInstructions += preferredLanguageInstructions + "\n\n"
}

// 2. Add global cline rules
if (globalClineRulesFileInstructions) {
    customInstructions += globalClineRulesFileInstructions + "\n\n"
}

// 3. Add local project rules (override global)
if (localClineRulesFileInstructions) {
    customInstructions += localClineRulesFileInstructions + "\n\n"
}

// 4. Add external tool integrations
[localCursorRulesFileInstructions, localCursorRulesDirInstructions, localWindsurfRulesFileInstructions]
    .forEach(instructions => {
        if (instructions) customInstructions += instructions + "\n\n"
    })

// 5. Add ignore instructions (final override)
if (clineIgnoreInstructions) {
    customInstructions += clineIgnoreInstructions
}

return formatInstructions(customInstructions)
```

## Model Family Strategies

The system implements different prompting strategies optimized for different AI model families.

### Strategy Selection Algorithm

```typescript
// Model family detection and strategy selection
if (isNextGenModelFamily(apiHandlerModel)) {
    // Advanced models: More sophisticated instructions
    return SYSTEM_PROMPT_NEXT_GEN(...)
} else {
    // Standard models: Conservative instructions  
    return SYSTEM_PROMPT_GENERIC(...)
}
```

### Next-Generation Model Optimizations

```typescript
// Enhanced capabilities for advanced models
const SYSTEM_PROMPT_NEXT_GEN = async (...) => {
    return `
    // Advanced reasoning instructions
    // More complex tool combinations
    // Enhanced context understanding
    // Sophisticated error recovery
    `
}
```

### Generic Model Compatibility

```typescript
// Broader compatibility for standard models
const SYSTEM_PROMPT_GENERIC = async (...) => {
    return `
    // Conservative instructions
    // Step-by-step guidance
    // Clear constraint boundaries
    // Simple tool usage patterns
    `
}
```

## Slash Command Processing

The slash command system implements a sophisticated template-based processing pipeline with support for both built-in and custom commands.

### Deep Planning Command Algorithm

The `/deep-planning` command implements a 4-phase structured planning process:

```typescript
export const deepPlanningToolResponse = () => `
<explicit_instructions type="deep-planning">
## STEP 1: Silent Investigation
- Execute research commands without commentary
- Build comprehensive codebase understanding
- Gather technical debt and dependency information

## STEP 2: Discussion and Questions
- Ask targeted clarification questions
- Choose between valid implementation approaches
- Confirm system behavior assumptions

## STEP 3: Create Implementation Plan Document
- Generate structured markdown document
- Include 8 comprehensive sections:
  [Overview] → [Types] → [Files] → [Functions] → [Classes] → [Dependencies] → [Testing] → [Implementation Order]

## STEP 4: Create Implementation Task
- Use new_task tool with task_progress checklist
- Include navigation commands for plan document
- Request mode switch to "act mode"
</explicit_instructions>
`
```

### Command Template Processing

```typescript
// Template injection algorithm
const processedText = commandReplacements[commandName] + textWithoutSlashCommand

// Workflow file loading algorithm  
const workflowContent = await fs.readFile(matchingWorkflow.fullPath, "utf8")
const processedText = `<explicit_instructions type="${matchingWorkflow.fileName}">
${workflowContent}
</explicit_instructions>` + textWithoutSlashCommand
```

## Deep Planning Algorithm

The deep planning system implements a comprehensive 4-phase algorithm for complex task planning.

### Phase 1: Silent Investigation Algorithm

```
Codebase Discovery → Pattern Analysis → Dependency Mapping → Technical Debt Assessment → Context Building
```

**Investigation Commands:**
```bash
# 1. Project structure discovery
find . -type f -name "*.{py,js,ts,java,cpp,go}" | head -30

# 2. Code pattern analysis  
grep -r "class|function|def|interface|struct" --include="*.{py,js,ts}" .

# 3. Dependency analysis
grep -r "import|from|require|#include" . | sort | uniq

# 4. Technical debt identification
grep -r "TODO|FIXME|XXX|HACK|NOTE" --include="*.{py,js,ts}" .
```

### Phase 2: Discussion Algorithm

```
Question Generation → Priority Assessment → User Interaction → Clarification Capture → Decision Documentation
```

**Question Categories:**
- Ambiguous requirement clarification
- Implementation approach selection
- System behavior confirmation  
- Technical preference understanding

### Phase 3: Plan Document Generation

```
Analysis Synthesis → Structure Definition → Section Population → Technical Specification → Validation
```

**Document Structure:**
```markdown
# Implementation Plan

[Overview] - Goal and high-level approach
[Types] - Complete type definitions and data structures  
[Files] - Exact files to create/modify/delete
[Functions] - New and modified functions with signatures
[Classes] - Class modifications and inheritance details
[Dependencies] - Package requirements and versions
[Testing] - Validation strategies and test requirements
[Implementation Order] - Step-by-step execution sequence
```

### Phase 4: Task Creation Algorithm

```
Plan Analysis → Step Extraction → Progress Checklist Generation → Navigation Commands → Task Assembly
```

## Key Algorithms and Data Flows

### 1. Prompt Construction Flow

```
Context Gathering → Model Detection → Strategy Selection → Template Loading → Instruction Merging → Final Assembly
```

### 2. Command Processing Flow

```
Text Input → Pattern Matching → Command Extraction → Template Lookup → Content Injection → Text Replacement
```

### 3. Context Management Flow

```
Token Monitoring → Overflow Detection → Summarization Trigger → Content Analysis → History Replacement → Continuation Setup
```

### 4. Response Formatting Flow

```
Raw Output → Content Analysis → Format Selection → Multi-modal Assembly → Context Enhancement → Final Response
```

## Implementation Patterns

### 1. Template Pattern

All system prompts use template-based construction:

```typescript
const SYSTEM_PROMPT_TEMPLATE = (context: PromptContext) => `
Base instructions with dynamic ${context.variable} injection
${context.conditionalSection ? 'Conditional content' : ''}
`
```

### 2. Strategy Pattern

Different prompting strategies based on model capabilities:

```typescript
interface PromptStrategy {
    buildPrompt(context: PromptContext): string
    formatTools(tools: Tool[]): string
    handleErrors(error: Error): string
}

class NextGenStrategy implements PromptStrategy { /* ... */ }
class GenericStrategy implements PromptStrategy { /* ... */ }
```

### 3. Pipeline Pattern

Multi-stage processing pipelines:

```typescript
class CommandProcessor {
    private stages: ProcessingStage[] = [
        new PatternMatchingStage(),
        new ValidationStage(), 
        new TemplateInjectionStage(),
        new WorkflowResolutionStage()
    ]
    
    process(input: string): string {
        return this.stages.reduce((text, stage) => stage.process(text), input)
    }
}
```

### 4. Builder Pattern

Complex prompt construction:

```typescript
class SystemPromptBuilder {
    private prompt = ""
    
    addBaseInstructions(): this { /* ... */ return this }
    addToolDefinitions(tools: Tool[]): this { /* ... */ return this }
    addUserInstructions(instructions: string): this { /* ... */ return this }
    addModelOptimizations(model: ModelFamily): this { /* ... */ return this }
    
    build(): string { return this.prompt }
}
```

### 5. Observer Pattern

Context change notifications:

```typescript
interface ContextObserver {
    onContextChange(newContext: Context): void
}

class PromptManager {
    private observers: ContextObserver[] = []
    
    updateContext(context: Context) {
        this.observers.forEach(observer => observer.onContextChange(context))
    }
}
```

This comprehensive prompts system provides sophisticated context management, dynamic prompt construction, and intelligent command processing that adapts to different AI models and user preferences while maintaining consistency and performance across all interaction modes.