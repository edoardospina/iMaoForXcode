# Agent Mode Deep Dive

## 1. Ask Mode vs Agent Mode

### Mode Definitions

**Location**: `Tool/Sources/ConversationServiceProvider/LSPTypes.swift:75-123`

```swift
enum ChatMode: String {
    case Ask = "Ask"
    case Edit = "Edit"
    case Agent = "Agent"
}
```

### Key Differences

| Aspect | Ask Mode | Agent Mode |
|--------|----------|-----------|
| **Tools** | ❌ None | ✅ All enabled tools |
| **LSP Parameter** | `chatMode: nil` | `chatMode: "Agent"` |
| **Scope** | `.chatPanel` | `.agentPanel` |
| **MCP Servers** | ❌ Not available | ✅ Available |
| **Tool Confirmation** | N/A | Required |
| **Custom Agents** | ❌ Not supported | ✅ Supported via `.agent.md` files |
| **Subagents** | ❌ Not supported | ✅ If enabled |

### Request Structure

**Ask Mode**:
```swift
ConversationCreateParams {
    chatMode: nil,
    customChatModeId: nil,
    capabilities: { skills: [], allSkills: false }
}
```

**Agent Mode**:
```swift
ConversationCreateParams {
    chatMode: "Agent",
    customChatModeId: "Agent",  // or custom agent ID
    capabilities: {
        skills: ["run_in_terminal", "insert_edit_into_file", ...],
        allSkills: false
    }
}
```

### Tool Availability Logic

**File**: `Tool/Sources/ConversationServiceProvider/AgentModeToolHelpers.swift:10-32`

```swift
// Default Agent mode: Use tool's current status (enabled/disabled)
if selectedMode.isDefaultAgent {
    return currentStatus == .enabled
}

// Custom agents: Check if tool is in customTools list
guard let customTools = selectedMode.customTools else {
    return false  // No tools if customTools is nil
}
return customTools.contains(configurationKey)
```

**Built-in agents** (like "Plan") have preset tools that users cannot modify.

---

## 2. Async Tool Call Handling

### Execution Model

**Synchronous API, Asynchronous Execution**:

```swift
protocol ICopilotTool {
    func invokeTool(
        _ request: InvokeClientToolRequest,
        completion: @escaping (AnyJSONRPCResponse) -> Void,
        contextProvider: ToolContextProvider?
    ) -> Bool  // Returns immediately, execution happens async
}
```

**Pattern**:
```swift
func invokeTool(...) -> Bool {
    Task {
        // Async work here
        let result = await performLongOperation()
        completion(result)
    }
    return true  // Returns immediately
}
```

### Timeout Handling

**No Global Timeout** - Tools run indefinitely until completion.

**Per-Tool Timeouts**:
- `FetchWebPageTool`: 30 seconds (via WebContentExtractor)
- Others: No timeout

**For MCP Servers**:
- External API calls **will block** until complete
- No circuit breaker mechanism
- **Recommendation**: Implement timeouts in your MCP server

### Waiting Mechanism

**Event-Driven via Combine**:

1. Language Server sends `conversation/invokeClientTool`
2. `ServerRequestHandler` forwards to `ClientToolHandler`
3. `ClientToolHandler` emits event via `PassthroughSubject`
4. `ChatService` subscribes and executes tool
5. Tool calls completion handler when done
6. Response sent back to Language Server

**Conversation waits** for completion handler - no time limit.

### Parallel Execution

**Multiple tools can run concurrently**:
- Each tool gets its own `Task`
- No queuing or serialization
- Tools execute in parallel

**Single active conversation**:
- Only one conversation request active at a time
- Checked via `activeRequestId` guard

---

## 3. File Access & Workspace Scope

### Workspace Detection

**Methods**:
1. **Primary**: Xcode Accessibility APIs (monitors focused window)
2. **Fallback**: File system traversal (searches for `.xcworkspace`/`.xcodeproj`)
3. **Multi-project**: Parses `.xcworkspace` XML for subprojects

**Project Root**: Found by searching upward for:
- `.git` directory (preferred)
- Git worktree
- First valid directory
- Falls back to workspace URL

**File**: `Tool/Sources/XcodeInspector/XcodeWindowInspector.swift:183-210`

### What Files Are Accessible

**Two-Tier System**:

**Tier 1: Filespace (Active Files)**
- Only files **opened in Xcode**
- Tracked in `Workspace.filespaces: [URL: Filespace]`
- Must pass validation before registration

**Tier 2: Workspace Index (All Files)**
- All files in project discovered via directory enumeration
- Used for context skills and file watching
- Limited to 1,000,000 files per workspace

### Hidden Files

**❌ EXCLUDED** - The `.skipsHiddenFiles` option is enforced:

**File**: `Tool/Sources/Workspace/WorkspaceFile.swift:249`

```swift
let enumerator = fileManager.enumerator(
    at: subproject,
    includingPropertiesForKeys: [...],
    options: [.skipsHiddenFiles]  // ← Hidden files skipped
)
```

**Cannot access**: `.gitignore`, `.env`, `.swiftformat`, `.github/` files, etc.

### File Format Limitations

**Whitelist** (25 extensions):

```swift
supportedFileExtensions: Set<String> = [
    // Code
    "swift", "m", "mm", "h", "cpp", "c",
    "js", "ts", "py", "rb", "java",

    // Scripts
    "applescript", "scpt",

    // Config
    "plist", "entitlements",

    // Documents
    "md", "json", "xml", "txt",
    "yaml", "yml", "html", "css"
]
```

**File**: `Tool/Sources/Workspace/WorkspaceFile.swift:7`

### Skip Patterns

**Always Excluded**:
```swift
skipPatterns = [
    ".git", ".svn", ".hg", "CVS",
    ".DS_Store", "Thumbs.db",
    "node_modules", "bower_components",
    "Preview Content", ".swiftpm"
]
```

Also skipped:
- `.xcworkspace`, `.xcodeproj` directories
- `.xcassets` directories
- `.app`, `.xcarchive` bundles

**File**: `Tool/Sources/Workspace/WorkspaceFile.swift:8-19`

### File Operations

**Available**:
- ✅ Read file content
- ✅ Monitor file changes (FSEvents)
- ✅ Track file saves
- ✅ Index up to 1M files

**Not Available**:
- ❌ Direct file writes (uses Xcode Accessibility API instead)
- ❌ Create files directly
- ❌ Delete files
- ❌ Access hidden files
- ❌ Access files outside workspace

---

## 4. Configuration Files

All files in `.github/` directory are **read by GitHub Copilot Language Server**, not directly by the app. The app only passes `workspaceFolders` parameter.

### `.github/copilot-instructions.md`

**Purpose**: Global instructions for ALL chat requests in workspace

**Format**: Plain Markdown (no frontmatter required)

**When Read**:
- Automatically by Language Server when `workspaceFolders` passed
- Also available as global setting (stored in UserDefaults)

**Applies To**: All LLM models (GitHub Copilot, BYOK, local)

**Example**:
```markdown
# Project Guidelines

Use 4-space indentation for Swift.
Follow Ray Wenderlich style guide.
Write unit tests for all new features.
```

**Location in Code**: `Tool/Sources/GitHubCopilotService/LanguageServer/GitHubCopilotRequest.swift:91-114`

---

### `.github/instructions/*.instructions.md`

**Purpose**: Scoped instructions for specific file patterns or tasks

**Format**:
```markdown
---
applyTo: "**/*.swift"
description: "Swift coding standards"
---
# Instructions here
Use guard statements for early returns.
Prefer value types over reference types.
```

**When Read**: Discovered via `conversation/templates` LSP request

**Applies To**: All LLM models via Language Server

**Auto-Applied**: Based on `applyTo` glob pattern

**Directory Structure**:
```
.github/
  instructions/
    swift-coding.instructions.md
    testing.instructions.md
    docs.instructions.md
```

---

### `.github/prompts/*.prompt.md`

**Purpose**: Reusable prompts triggered with `/` commands

**Format**:
```markdown
---
description: 'Generate unit tests for current file'
---
Create comprehensive unit tests for the current file.
Include edge cases and error conditions.
Use XCTest framework conventions.
```

**When Read**: Via `conversation/templates` when feature flag enabled

**Feature Flag**: `isEditorPreviewEnabled` must be true

**Usage**: Type `/prompt-name` in chat

**Applies To**: All LLM models

**Can Reference Files**:
```markdown
Review the implementation in [App.swift](../App.swift)
```

**Directory Structure**:
```
.github/
  prompts/
    generate-tests.prompt.md
    review-code.prompt.md
    create-docs.prompt.md
```

**Experimental**: Currently behind preview flag

---

### `.github/agents/*.agent.md`

**Purpose**: Define custom autonomous agents with specific tools/capabilities

**Format**:
```markdown
---
description: 'Code reviewer that checks style and logic'
tools: ['read_file', 'get_errors']
handoffs:
  - label: Start Implementation
    agent: Agent
    prompt: Begin implementation
    send: true
---
# Code Review Agent

You are a code reviewer. Analyze code for:
- Style consistency
- Logic errors
- Performance issues
- Security vulnerabilities

Never modify code. Only provide feedback.
```

**When Read**: Via `conversation/modes` LSP request

**Feature Flags Required**:
1. `isEditorPreviewEnabled` = true
2. `isCustomAgentEnabled` = true (policy check)

**Applies To**: All LLM models

**UI Integration**: Appears in agent mode dropdown

**Supports**:
- Custom tool lists
- Handoffs to other agents
- Subagent capabilities (if enabled)

**Directory Structure**:
```
.github/
  agents/
    code-reviewer.agent.md
    deployment-helper.agent.md
    test-generator.agent.md
```

---

### How Configuration Files Are Loaded

**Flow**:
```
App passes workspaceFolders[]
         ↓
GitHub Copilot Language Server
         ↓
Scans .github/ directories
         ↓
Loads matching config files
         ↓
Injects into conversation context
```

**File Watching**:
- Language Server watches for changes
- App notifies via `workspace/didChangeWatchedFiles`
- Changes applied automatically

**Cross-Platform**:
- Same `.github/` structure works in VS Code, Visual Studio, GitHub.com
- Files are portable across IDEs

---

## 5. Enable Subagent

### What It Does

Allows Agent mode to invoke **custom agents as subagents** for task delegation.

**Example Flow**:
```
User: "Review and fix the authentication code"
    ↓
Main Agent: Analyzes request
    ↓
Spawns Code Reviewer Subagent
    ↓
Subagent: Reviews code, finds issues
    ↓
Main Agent: Receives feedback
    ↓
Spawns Implementation Subagent
    ↓
Subagent: Fixes issues
    ↓
Main Agent: Returns final result
```

### Technical Implementation

**Data Structure**:

```swift
struct AgentRound {
    let roundId: Int
    var reply: String
    var toolCalls: [AgentToolCall]?
    var subAgentRounds: [AgentRound]?  // ← Nested subagent execution
}

struct ConversationProgressBegin {
    let conversationId: String
    let turnId: String
    let parentTurnId: String?  // ← Links subagent to parent
}
```

**Tracking**:
```swift
struct ConversationTurnTrackingState {
    var turnParentMap: [String: String]  // Maps subturn ID → parent ID
    var validConversationIds: Set<String>  // All valid IDs (main + subs)
}
```

**File**: `Core/Sources/ChatService/ChatService.swift:693-706`

### Configuration

**Setting**: Advanced tab → "Enable Subagent" toggle

**Preference Key**: `enableSubagent` (default: false)

**Requires Restart**: Must restart app to take effect

**Sent to Language Server**:
```swift
"copilotCapabilities": [
    "subAgent": JSONValue(booleanLiteral: enableSubagent)
]
```

**File**: `Tool/Sources/GitHubCopilotService/LanguageServer/GitHubCopilotService.swift:214-295`

### Organization Policy Control

**Policy Check**:
```swift
struct CopilotPolicy {
    var subagentEnabled: Bool = true
}
```

Organizations can disable subagents centrally.

**File**: `Tool/Sources/GitHubCopilotService/Services/CopilotPolicyNotifier.swift`

### Model Selection

**Subagents inherit parent model** - No separate model selection.

The same model context is used for parent and subagent turns.

### System Prompts

**Subagents use custom `.agent.md` definitions**:

```markdown
---
description: 'What this subagent does'
tools: ['tool1', 'tool2']
---
Define subagent behavior, constraints, inputs/outputs.
```

Each subagent has its own instructions from its `.agent.md` file.

### How Subagents Are Spawned

**Requirements**:
1. ✅ Agent mode active (`chatMode: "Agent"`)
2. ✅ `enableSubagent` preference true
3. ✅ Custom agents exist (`.agent.md` files)
4. ✅ Organization policy allows (`subagentEnabled: true`)
5. ✅ Parent agent decides to delegate

**Decision**: Language Server determines when to spawn subagents based on task analysis.

**Progress Notifications**:
- `ConversationProgressBegin` with `parentTurnId` set
- Subagent rounds merged into parent's `subAgentRounds[]`
- UI displays nested execution

### Differences from Main Agent

| Aspect | Main Agent | Subagent |
|--------|-----------|----------|
| **Hierarchy** | Top-level | Has `parentTurnId` |
| **Conversation ID** | Own ID | Own ID (separate) |
| **Tool Access** | All enabled tools | Tools from `.agent.md` |
| **UI Display** | Standard | Nested with special styling |
| **Cancellation** | Cancels all | Cancelled with parent |
| **Message Merging** | N/A | Merged into parent's rounds |

---

## 6. Logging

### Log Locations

**Main Logs**:
```
~/Library/Logs/GitHubCopilot/github-copilot-for-xcode.log
```

**Rotated Logs**:
```
~/Library/Logs/GitHubCopilot/github-copilot-for-xcode-YYYYMMDDHHMMSS.log
```

**MCP Runtime Logs**:
```
~/Library/Logs/GitHubCopilot/MCPRuntimeLogs/{workspace}.log
```

**Conversation History** (SQLite):
```
~/.config/github-copilot/xcode/{username}/conversations/{workspace}.db
```

### What Is Logged

**Categories**:
1. **Service** - ExtensionService lifecycle
2. **UI** - User interface events
3. **Client** - XPC client communications, file operations
4. **GitHubCopilot** - API interactions, Language Server messages
5. **WorkspacePool** - Workspace management, file watching
6. **MCP** - Model Context Protocol operations (separate file)

**Information**:
- Service lifecycle events
- Error messages with stack traces
- GitHub Copilot API requests/responses
- Tool invocations and completions
- File operations
- Workspace changes
- Authentication status
- XPC communication errors

**File**: `Tool/Sources/Logger/Logger.swift`

### Log Levels

1. **DEBUG** - Detailed debugging (includes file, line, function)
2. **INFO** - General operational messages
3. **ERROR** - Error conditions (sent to telemetry)

### Log Rotation

**Configuration**:
- Max size: 5 MB per file
- Max files: 10 archived logs
- Overflow limit: 10 MB (stops logging temporarily)
- Max lock time: 1 hour

**File**: `Tool/Sources/Logger/FileLogger.swift`

### Accessing Logs

**Method 1: Direct**
```bash
tail -f ~/Library/Logs/GitHubCopilot/github-copilot-for-xcode.log
```

**Method 2: Settings UI**
- Advanced → Logging → "Open Copilot Log Folder"

**Method 3: Console.app**
- Filter: Process = "GitHub Copilot for Xcode"
- Subsystem: `com.github.CopilotForXcode`
- Search: "Tool", "invoke", "edit", "MCP"

### Verbose Logging

**Setting**: Advanced tab → "Verbose Logging"

**Effect**: Sets `GH_COPILOT_VERBOSE=true` for Language Server

**Requires Restart**: Yes

### MCP Logs

**Separate Directory**: `~/Library/Logs/GitHubCopilot/MCPRuntimeLogs/`

**Per-Workspace**: Each workspace gets its own log file

**Format**:
```
[ISO8601_TIMESTAMP] [LEVEL] [SERVER-TOOL] MESSAGE
```

**NOT in main log** - MCP category excluded from main log file

**File**: `Tool/Sources/Logger/MCPRuntimeLogger.swift`

---

## Summary Checklist

### Using Agent Mode

**Prerequisites**:
- ✅ Select "Agent" from mode picker (not "Ask")
- ✅ Enable desired tools in settings
- ✅ Configure `.github/` files if needed

**For Custom Agents**:
- ✅ Create `.agent.md` files in `.github/agents/`
- ✅ Enable preview features
- ✅ Check organization policy allows custom agents

**For Subagents**:
- ✅ Enable "Enable Subagent" in Advanced settings
- ✅ Restart app
- ✅ Create custom agents
- ✅ Verify organization policy allows subagents

### File Access

**Agent CAN access**:
- ✅ Source code files (25 supported extensions)
- ✅ Files within workspace boundaries
- ✅ Open files in Xcode
- ✅ Indexed workspace files (up to 1M)

**Agent CANNOT access**:
- ❌ Hidden files (dotfiles)
- ❌ Files in skip patterns (node_modules, .git, etc.)
- ❌ Unsupported file formats
- ❌ Files outside workspace
- ❌ `.xcodeproj` internals

### Configuration Files

**Apply to ALL models** (including local):
- ✅ `.github/copilot-instructions.md`
- ✅ `.github/instructions/*.instructions.md`
- ✅ `.github/prompts/*.prompt.md`
- ✅ `.github/agents/*.agent.md`

**How they work**:
- Read by GitHub Copilot Language Server
- App only passes `workspaceFolders` parameter
- Language Server handles injection into context

### Logging

**Quick access**:
```bash
# View live log
tail -f ~/Library/Logs/GitHubCopilot/github-copilot-for-xcode.log

# View MCP logs
ls ~/Library/Logs/GitHubCopilot/MCPRuntimeLogs/

# Open in Finder
open ~/Library/Logs/GitHubCopilot/
```

**Tool invocation logs**:
- Search for: "Tool", "invoke", "insert_edit_into_file"
- Category: Client, Service, GitHubCopilot

---

## File References

**Mode Selection**: `Core/Sources/ConversationTab/ModeAndModelPicker/ModePicker/ChatModePicker.swift`

**Tool Helpers**: `Tool/Sources/ConversationServiceProvider/AgentModeToolHelpers.swift`

**Tool Execution**: `Core/Sources/ChatService/ChatService.swift:179-204`

**Workspace**: `Tool/Sources/Workspace/Workspace.swift`

**File Discovery**: `Tool/Sources/Workspace/WorkspaceFile.swift`

**Config Files**: `Tool/Sources/ConversationServiceProvider/PromptType.swift`

**Subagents**: `Tool/Sources/ConversationServiceProvider/LSPTypes+AgentRound.swift`

**Logging**: `Tool/Sources/Logger/Logger.swift`, `Tool/Sources/Logger/FileLogger.swift`
