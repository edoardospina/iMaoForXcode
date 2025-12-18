# Tools Implementation Guide for GitHub Copilot for Xcode

## Executive Summary

**Critical Facts for Local LLM Usage:**

1. **Tools ONLY work in Chat mode** - NOT available in autocomplete or NES
2. **GitHub Copilot Language Server handles ALL tool translation** - You don't adapt tool formats per model
3. **Your local LLM MUST support function/tool calling** - Non-negotiable requirement
4. **Edit file tool uses macOS Accessibility APIs** - Not LLM-dependent, works universally
5. **Tool definitions use universal JSON Schema** - Language server translates to model-specific formats

---

## 1. Tool Availability by Mode

### ❌ Autocomplete (`textDocument/inlineCompletion`)

**NO TOOLS AVAILABLE**

Request structure:
```swift
{
    textDocument: { uri, version },
    position: { line, character },
    formattingOptions: { tabSize, insertSpaces },
    context: { triggerKind }
}
```

Pure completion - no tool access.

---

### ❌ NES - Next Edit Suggestions (`textDocument/copilotInlineEdit`)

**NO TOOLS AVAILABLE**

Request structure:
```swift
{
    textDocument: { uri, version },
    position: { line, character }
}
```

Pure inline editing - no tool access.

---

### ✅ Chat Mode (`conversation/create`, `conversation/turn`)

**FULL TOOL ACCESS**

Request structure:
```swift
{
    workDoneToken: string,
    turns: [TurnSchema],
    capabilities: {
        skills: [string],        // ← Tool names
        allSkills: bool
    },
    model: string,               // ← Your local model ID
    modelProviderName: string,   // ← e.g., "Azure" (for localhost)
    chatMode: "Agent",           // ← Enables tools
    needToolCallConfirmation: bool,
    ignoredSkills: [string]      // ← Disable specific tools
}
```

**Key Insight**: Tools are not passed in requests. They're pre-registered with the Language Server.

---

## 2. Tool Types

### Native/Client Tools (Built-in)

**Location**: `Core/Sources/ChatService/ToolCalls/*.swift`

**Registry**: `Tool/Sources/GitHubCopilotService/LanguageServer/ClientToolRegistry.swift`

**6 Native Tools**:

1. **`run_in_terminal`** - Execute shell commands
2. **`get_terminal_output`** - Retrieve terminal output
3. **`get_errors`** - Get Xcode compiler errors
4. **`insert_edit_into_file`** - Modify existing files (KEY TOOL)
5. **`create_file`** - Create new files
6. **`fetch_webpage`** - Fetch web content

**Execution**: Local Swift implementations

---

### MCP Tools (External)

**Configuration**: UserDefaults JSON → Language Server
**Discovery**: Language Server connects to MCP servers
**Execution**: RPC calls to external MCP servers
**Management**: `CopilotMCPToolManager`

**Example MCP Tools**: Filesystem operations, database queries, API calls

---

### Comparison

| Aspect | Native Tools | MCP Tools |
|--------|-------------|-----------|
| **Implementation** | Swift code in app | External servers |
| **Registration** | `conversation/registerTools` | MCP server discovery |
| **Invocation** | `conversation/invokeClientTool` | Via Language Server → MCP server |
| **Execution** | Local (app process) | Remote (MCP server process) |
| **Configuration** | Hardcoded at startup | JSON config in UserDefaults |

---

## 3. Tool Registration & Lifecycle

### Step 1: Tool Definition

**File**: `Tool/Sources/GitHubCopilotService/LanguageServer/ClientToolRegistry.swift`

```swift
func registerClientTools(server: GitHubCopilotConversationServiceType) async -> [LanguageModelTool] {
    var tools: [LanguageModelToolInformation] = []

    let insertEditIntoFileTool: LanguageModelToolInformation = .init(
        name: "insert_edit_into_file",
        description: "Insert new code into an existing file...",
        inputSchema: .init(
            type: "object",
            properties: [
                "filePath": .init(
                    type: "string",
                    description: "Absolute path to file"
                ),
                "code": .init(
                    type: "string",
                    description: "Code change to apply"
                ),
                "explanation": .init(
                    type: "string",
                    description: "Short explanation"
                )
            ],
            required: ["filePath", "code", "explanation"]
        ),
        confirmationMessages: .init(
            title: "Edit File",
            message: "Allow editing {filePath}?"
        )
    )

    tools.append(insertEditIntoFileTool)
    // ... other tools

    return try? await server.registerTools(tools: tools) ?? []
}
```

**Key Structure**:
```swift
struct LanguageModelToolInformation {
    name: String                          // Tool identifier
    description: String                   // Instructions for LLM
    inputSchema: LanguageModelToolSchema  // JSON Schema for parameters
    confirmationMessages: ToolConfirmationMessages?  // Optional user confirmation
}

struct LanguageModelToolSchema {
    type: "object"
    properties: [String: ToolInputPropertySchema]
    required: [String]
}
```

---

### Step 2: Registration with Language Server

**LSP Request**: `conversation/registerTools`

**File**: `Tool/Sources/GitHubCopilotService/LanguageServer/GitHubCopilotRequest.swift`

```swift
struct RegisterTools: GitHubCopilotRequestType {
    typealias Response = Array<LanguageModelTool>

    var params: RegisterToolsParams

    var request: ClientRequest {
        .custom("conversation/registerTools", encodedParams, NullHandler)
    }
}
```

**When**: On Language Server initialization, before any conversations

**Returns**: Registered tools with status (`enabled`/`disabled`)

```swift
struct LanguageModelTool {
    id: String
    type: ToolType                // .client, .mcp, .shared
    toolProvider: ToolProvider
    nameForModel: String          // How LLM sees it
    name: String                  // Internal name
    displayName: String?
    description: String?
    displayDescription: String
    inputSchema: [String: AnyCodable]?
    annotations: ToolAnnotations?
    status: ToolStatus            // .enabled or .disabled
}
```

---

### Step 3: Tool Availability in Conversations

**Chat Request** (`conversation/create`):
```swift
{
    capabilities: {
        skills: ["run_in_terminal", "insert_edit_into_file", ...],
        allSkills: false
    },
    chatMode: "Agent",  // Enables tool use
    model: "llama-3.1-8b",
    modelProviderName: "Azure"  // Your localhost setup
}
```

**Critical**: Tools are NOT sent in the request. The Language Server knows which tools are available and translates them to the model's format.

---

## 4. How Tool Calling Works (The Magic)

### The Translation Layer

```
┌─────────────────────────────────────────────────────────────┐
│ Universal Tool Definition (JSON Schema)                     │
│ {                                                            │
│   name: "insert_edit_into_file",                           │
│   description: "Insert code into file...",                 │
│   inputSchema: { type: "object", properties: {...} }      │
│ }                                                            │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│ GitHub Copilot Language Server (Node.js)                    │
│                                                              │
│ Translates to model-specific format:                        │
│                                                              │
│ ┌────────────────┐  ┌──────────────────┐  ┌──────────────┐ │
│ │ OpenAI         │  │ Anthropic        │  │ Google       │ │
│ │ function_call  │  │ tool_use         │  │ function_    │ │
│ │ format         │  │ format           │  │ calling      │ │
│ └────────────────┘  └──────────────────┘  └──────────────┘ │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
                    LLM Model
                  (any provider)
```

**YOU DON'T HANDLE THIS** - The GitHub Copilot Language Server does ALL translation.

---

### Model-Specific Formats (Handled by Language Server)

**OpenAI Format**:
```json
{
  "functions": [{
    "name": "insert_edit_into_file",
    "description": "Insert code into file...",
    "parameters": {
      "type": "object",
      "properties": {
        "filePath": { "type": "string", "description": "..." },
        "code": { "type": "string", "description": "..." }
      },
      "required": ["filePath", "code"]
    }
  }],
  "function_call": "auto"
}
```

**Anthropic Format**:
```json
{
  "tools": [{
    "name": "insert_edit_into_file",
    "description": "Insert code into file...",
    "input_schema": {
      "type": "object",
      "properties": {
        "filePath": { "type": "string", "description": "..." },
        "code": { "type": "string", "description": "..." }
      },
      "required": ["filePath", "code"]
    }
  }],
  "tool_choice": { "type": "auto" }
}
```

**The Language Server knows your `modelProviderName` and translates accordingly.**

---

## 5. Tool Invocation Flow

### Complete Execution Path

#### **Phase 1: LLM Decides to Use Tool**

LLM response includes tool call:
```json
{
  "tool_calls": [{
    "id": "call_123",
    "type": "function",
    "function": {
      "name": "insert_edit_into_file",
      "arguments": "{\"filePath\": \"/path/to/file.swift\", \"code\": \"...\", \"explanation\": \"...\"}"
    }
  }]
}
```

---

#### **Phase 2: Language Server → App**

**LSP Request**: `conversation/invokeClientTool`

```swift
struct InvokeClientToolRequest {
    id: string
    method: "conversation/invokeClientTool"
    params: InvokeClientToolParams {
        conversationId: string
        turnId: string
        roundId: string
        toolCallId: string
        name: string              // "insert_edit_into_file"
        input: [String: AnyCodable]  // Tool arguments
    }
}
```

**File**: `Tool/Sources/GitHubCopilotService/LanguageServer/ServerRequestHandler.swift`

```swift
case "conversation/invokeClientTool":
    let invokeParams = try JSONDecoder().decode(
        InvokeClientToolParams.self,
        from: paramsData
    )
    ClientToolHandlerImpl.shared.invokeClientTool(
        InvokeClientToolRequest(id: id, method: method, params: invokeParams),
        completion: responseHandler
    )
```

---

#### **Phase 3: Event Broadcasting**

**File**: `Tool/Sources/GitHubCopilotService/LanguageServer/ClientToolHandler.swift`

```swift
public func invokeClientTool(
    _ request: InvokeClientToolRequest,
    completion: @escaping (AnyJSONRPCResponse) -> Void
) {
    onClientToolInvokeEvent.send((request, completion))
}
```

Uses **Combine** to publish event to all subscribers.

---

#### **Phase 4: ChatService Receives & Routes**

**File**: `Core/Sources/ChatService/ChatService.swift`

```swift
private func subscribeToClientToolInvokeEvent() {
    ClientToolHandlerImpl.shared.onClientToolInvokeEvent
        .sink { [weak self] (request, completion) in
            guard let params = request.params else { return }

            // Validate conversation ID
            guard let validIds = self?.conversationTurnTracking.validConversationIds,
                  validIds.contains(params.conversationId) else {
                return
            }

            // Get tool implementation
            guard let tool = CopilotToolRegistry.shared.getTool(name: params.name) else {
                completion(errorResponse("Tool not found"))
                return
            }

            // Execute tool
            _ = tool.invokeTool(request, completion: completion, contextProvider: self)
        }
        .store(in: &cancellables)
}
```

---

#### **Phase 5: Tool Execution**

**File**: `Core/Sources/ChatService/ToolCalls/InsertEditIntoFileTool.swift`

```swift
public class InsertEditIntoFileTool: ICopilotTool {
    public static let name = ToolName.insertEditIntoFile

    public func invokeTool(
        _ request: InvokeClientToolRequest,
        completion: @escaping (AnyJSONRPCResponse) -> Void,
        contextProvider: ToolContextProvider?
    ) -> Bool {
        // Extract parameters
        guard let input = request.params?.input,
              let code = input["code"]?.value as? String,
              let filePath = input["filePath"]?.value as? String,
              let contextProvider
        else {
            completeResponse(request, status: .error,
                           response: "Invalid parameters",
                           completion: completion)
            return true
        }

        let fileURL = URL(fileURLWithPath: filePath)
        let originalContent = try String(contentsOf: fileURL, encoding: .utf8)

        // Apply edit (see Phase 6)
        InsertEditIntoFileTool.applyEdit(
            for: fileURL,
            content: code,
            contextProvider: contextProvider
        ) { newContent, error in
            if let error = error {
                self.completeResponse(request, status: .error,
                                    response: error.localizedDescription,
                                    completion: completion)
                return
            }

            // Track edit for undo
            let fileEdit = FileEdit(
                fileURL: fileURL,
                originalContent: originalContent,
                modifiedContent: code,
                toolName: InsertEditIntoFileTool.name
            )
            contextProvider.updateFileEdits(by: fileEdit)

            // Return success
            self.completeResponse(request, response: newContent,
                                completion: completion)
        }

        return true
    }
}
```

---

#### **Phase 6: Edit Application (macOS Accessibility API)**

**THIS IS THE CRITICAL PART FOR UNDERSTANDING**

```swift
public static func applyEdit(
    for fileURL: URL,
    content: String,
    contextProvider: ToolContextProvider,
    xcodeInstance: AppInstanceInspector
) throws -> String {
    // Get Xcode's source editor element via Accessibility API
    guard let editorElement = getEditorElement(by: xcodeInstance, for: fileURL)
    else {
        throw InsertEditError.missingEditorElement(file: fileURL)
    }

    // Read current content
    let value = try editorElement.copyValue(key: kAXValueAttribute)
    let lines = value.components(separatedBy: .newlines)

    // Verify correct file is open
    try checkOpenedFileURL(for: fileURL, xcodeInstance: xcodeInstance)

    // Apply edit using Accessibility API
    try AXHelper().injectUpdatedCodeWithAccessibilityAPI(
        .init(
            content: content,
            newSelection: nil,
            modifications: [
                // Delete entire file content
                .deletedSelection(.init(
                    start: .init(line: 0, character: 0),
                    end: .init(line: lines.count - 1, character: lines.last?.count ?? 0)
                )),
                // Insert new content
                .inserted(0, [content])
            ]
        ),
        focusElement: editorElement
    )

    // Verify content was applied
    return try getCurrentEditorContent(for: fileURL, by: xcodeInstance)
}
```

**Key Points**:
- Uses **macOS Accessibility APIs** (`kAXValueAttribute`, `AXUIElement`)
- Directly manipulates Xcode's text editor
- **NOT dependent on LLM** - works with any model
- Requires file to be open in Xcode
- Performs full file replacement (delete → insert)

---

#### **Phase 7: Response to LLM**

```swift
func completeResponse(
    _ request: InvokeClientToolRequest,
    status: ToolInvocationStatus = .success,  // or .error, .cancelled
    response: String = "",
    completion: @escaping (AnyJSONRPCResponse) -> Void
) {
    let toolResult = LanguageModelToolResult(
        status: status,
        content: [
            LanguageModelToolResult.Content(value: response)
        ]
    )

    let jsonResult = try? JSONEncoder().encode(toolResult)
    let jsonValue = try? JSONDecoder().decode(JSONValue.self, from: jsonResult ?? Data())

    completion(AnyJSONRPCResponse(
        id: request.id,
        result: JSONValue.array([jsonValue ?? .null, JSONValue.null])
    ))
}
```

Response flows back:
```
App → Language Server → LLM Model
```

LLM receives tool result and can continue conversation.

---

## 6. The Edit File Tool: Deep Dive

### Why This Tool Is Special

**Most complex tool** because:
1. Must understand minimal code hints (comments like `// ...existing code...`)
2. Must preserve existing code while applying changes
3. Requires direct IDE manipulation (not file system writes)
4. Needs retry logic (Xcode may not be ready)
5. Must verify file URL matches focused file

---

### LLM Instructions (from Tool Description)

```
Insert new code into an existing file in the workspace.

The system is very smart and can understand how to apply your edits to the files,
you just need to provide minimal hints.

Avoid repeating existing code, instead use comments to represent regions of
unchanged code. Be as concise as possible.

Example:
// ...existing code...
{ changed code }
// ...existing code...
{ changed code }
// ...existing code...

Example edit to Person class:
class Person {
    // ...existing code...
    age: number;
    // ...existing code...
    getAge() {
        return this.age;
    }
}
```

**Critical**: The LLM should provide **minimal hints**, not full file content.

---

### How "Smart System" Works

**NOT LLM-POWERED** - Uses macOS Accessibility API:

1. **Open file in Xcode** (`NSWorkspace.openFileInXcode`)
2. **Get focused editor element** (`AppInstanceInspector → AXUIElement`)
3. **Verify file URL** (compare `realtimeDocumentURL` with target)
4. **Read current content** (`editorElement.copyValue(kAXValueAttribute)`)
5. **Replace entire content** (delete all → insert new)
6. **Verify success** (read back content)

**The "smart" part**: Full file replacement means the app EXPECTS the LLM to provide complete modified content, not diffs.

---

### Requirements for LLMs

**What the LLM must do**:
1. Read current file content (via context or prior tool call)
2. Generate complete modified file content
3. Follow the format convention (use comments for unchanged regions)
4. Call tool with: `filePath`, `code` (full new content), `explanation`

**What the LLM does NOT need to do**:
- Calculate diffs
- Apply patches
- Understand Accessibility APIs
- Verify file state

---

### Error Handling

```swift
public enum InsertEditError: LocalizedError {
    case missingEditorElement(file: URL)
    case openingApplicationUnavailable
    case fileNotOpenedInXcode
    case fileURLMismatch(expected: URL, actual: URL?)
}
```

**Common failures**:
- File not open in Xcode
- Wrong file focused
- Xcode not responding
- Accessibility permissions denied

**Retry logic**: 6 attempts with 0.5s delay to get editor element

---

## 7. Requirements for Local LLMs

### Mandatory Capabilities

#### **1. Function/Tool Calling Support**

Your local LLM **MUST** support one of:
- OpenAI function calling format
- Anthropic tool use format
- Google function calling format
- Any format the GitHub Copilot Language Server understands

**Models that work**:
- ✅ Llama 3.1+ (tool calling)
- ✅ Mistral 7B+ with Instruct tuning
- ✅ Qwen 2.5+ (function calling)
- ✅ Command R/R+ (tool use)
- ✅ Gemma 2 with tool training
- ❌ Base models without tool training (Llama 2, GPT-2, etc.)

**Check your model's documentation for tool calling support.**

---

#### **2. Instruction Following**

Must accurately:
- Parse tool descriptions
- Extract JSON schemas
- Generate valid JSON arguments
- Follow formatting conventions

**Test**: Ask model to call a simple function. If it can't, **it won't work**.

---

#### **3. Context Window**

Minimum **8K tokens** (tools + conversation + code)

Tool definitions consume ~2-3K tokens. File context can be 5K+.

**Recommended**: 16K+ context window

---

#### **4. JSON Output**

Must reliably output:
```json
{
  "tool_calls": [{
    "name": "insert_edit_into_file",
    "arguments": {
      "filePath": "/path/to/file",
      "code": "...",
      "explanation": "..."
    }
  }]
}
```

Models with poor JSON formatting will fail.

---

### LMStudio Configuration

**For localhost:1234 setup:**

1. **Model Selection**: Choose model with tool calling (e.g., `llama-3.1-8b-instruct`)
2. **API Format**: OpenAI-compatible
3. **Endpoint**: `http://localhost:1234/v1/chat/completions`
4. **Context Length**: Set to model's max (e.g., 128K for Llama 3.1)
5. **Temperature**: 0.3-0.7 (too high = unreliable tool calls)

**Test tool calling**:
```bash
curl http://localhost:1234/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "llama-3.1-8b-instruct",
    "messages": [{"role": "user", "content": "Edit the file at /test.txt"}],
    "tools": [{
      "type": "function",
      "function": {
        "name": "insert_edit_into_file",
        "description": "Edit a file",
        "parameters": {
          "type": "object",
          "properties": {
            "filePath": {"type": "string"},
            "code": {"type": "string"}
          },
          "required": ["filePath", "code"]
        }
      }
    }]
  }'
```

If response includes `tool_calls`, you're good.

---

### Chat Format Requirements

**System Prompt Must Include**:
- Tool descriptions
- JSON schema for each tool
- Instructions on when to use tools
- Format examples

The GitHub Copilot Language Server handles this automatically.

---

### Thinking Models (o1, o3-mini, etc.)

**Special Considerations**:

**o1-style models**:
- May not support traditional tool calling
- Use "reasoning tokens" before output
- May require different prompting

**GitHub Copilot's handling**:
- Language Server has special logic for reasoning models
- May disable certain tools
- May adjust prompting

**Your responsibility**: Configure `modelProviderName` correctly (e.g., "OpenAI" for o1)

---

## 8. Model-Specific Adaptations

### Who Handles What

**GitHub Copilot Language Server handles**:
- ✅ Tool format translation (OpenAI ↔ Anthropic ↔ Google)
- ✅ Model-specific prompting
- ✅ Reasoning model adaptations
- ✅ Context window management
- ✅ Error handling for tool parsing

**You handle**:
- ❌ Nothing tool-related
- ✅ Model selection (`model`, `modelProviderName`)
- ✅ LMStudio/local server configuration
- ✅ Ensuring model supports tool calling

---

### Model Provider Configuration

**In conversation request**:
```swift
{
    model: "llama-3.1-8b",           // Model ID
    modelProviderName: "Azure"       // Provider (use for localhost)
}
```

**Language Server uses `modelProviderName` to**:
1. Determine tool calling format
2. Adjust prompting strategy
3. Configure error handling
4. Apply model-specific quirks

**For local LLMs via Azure workaround**:
- Set `modelProviderName: "Azure"`
- Language Server will use OpenAI-compatible format
- Your local model must accept OpenAI function calling format

---

### Known Model Behaviors

**GPT-4/GPT-4 Turbo**:
- Excellent tool calling
- Follows JSON schemas precisely
- Good at minimal hint format

**Claude 3.5 Sonnet/Opus**:
- Best tool calling accuracy
- Excellent instruction following
- Great with complex tool chains

**Gemini Pro**:
- Good tool calling
- Sometimes verbose arguments
- May need temperature tuning

**Llama 3.1 (local)**:
- Reliable tool calling if quantization ≥ Q5
- May struggle with complex schemas
- Works best with clear, simple tool descriptions

**Mistral/Mixtral (local)**:
- Good tool calling with instruct versions
- May need lower temperature (0.3-0.5)
- Sometimes generates extra text outside JSON

---

## 9. Debugging Tool Issues

### Common Problems

#### **Problem: Model doesn't call tools**

**Possible causes**:
1. Model lacks tool calling training
2. `chatMode` not set to "Agent"
3. Tools not registered
4. Model temperature too low (model too conservative)

**Fix**:
- Verify model supports tool calling
- Check conversation request includes `chatMode: "Agent"`
- Verify tools appear in `conversation/registerTools` response
- Increase temperature to 0.5-0.7

---

#### **Problem: Invalid tool arguments**

**Possible causes**:
1. Model doesn't follow JSON schema
2. Tool description unclear
3. Required parameters not specified

**Fix**:
- Test model with simpler schemas first
- Improve tool descriptions
- Add examples to tool descriptions

---

#### **Problem: Edit file tool fails**

**Possible causes**:
1. File not open in Xcode
2. Wrong file focused
3. Accessibility permissions denied
4. File path incorrect (relative vs absolute)

**Fix**:
- Open file in Xcode first
- Check file path is absolute
- Grant Accessibility permissions (System Preferences → Privacy → Accessibility)
- Check logs: `Logger.client.error("Failed to inject code...")`

---

#### **Problem: Tool calls hang**

**Possible causes**:
1. Local model server not responding
2. Timeout too short
3. Model generating infinite output

**Fix**:
- Check LMStudio server status
- Verify `http://localhost:1234/v1/chat/completions` responds
- Add max_tokens limit to model config
- Check Language Server logs

---

### Logging & Monitoring

**Key log locations**:
```swift
// Tool invocation
Logger.client.info("Invoking tool: \(toolName)")

// Tool completion
Logger.client.info("Tool \(toolName) completed with status: \(status)")

// Edit file errors
Logger.client.error("Failed to inject code: \(error)")

// MCP runtime logs
Logger.logMCPRuntime(level: level, message: message, server: server, tool: tool)
```

**Check logs**:
```bash
# macOS Console.app
# Filter: Process = "GitHub Copilot for Xcode"
# Search for "Tool" or "invoke" or "edit"
```

---

## 10. Advanced: Custom Tool Development

### Adding a New Native Tool

**Step 1: Define Tool**

**File**: `Tool/Sources/GitHubCopilotService/LanguageServer/ClientToolRegistry.swift`

```swift
let myCustomTool = LanguageModelToolInformation(
    name: "my_custom_tool",
    description: "Does something useful...",
    inputSchema: .init(
        type: "object",
        properties: [
            "param1": .init(type: "string", description: "..."),
            "param2": .init(type: "number", description: "...")
        ],
        required: ["param1"]
    )
)
tools.append(myCustomTool)
```

---

**Step 2: Add to ToolName Enum**

**File**: `Tool/Sources/GitHubCopilotService/LanguageServer/ToolNames.swift`

```swift
public enum ToolName: String, Codable {
    case runInTerminal = "run_in_terminal"
    case insertEditIntoFile = "insert_edit_into_file"
    case myCustomTool = "my_custom_tool"  // ADD THIS
}
```

---

**Step 3: Implement Tool**

**File**: `Core/Sources/ChatService/ToolCalls/MyCustomTool.swift`

```swift
public class MyCustomTool: ICopilotTool {
    public static let name = ToolName.myCustomTool

    public func invokeTool(
        _ request: InvokeClientToolRequest,
        completion: @escaping (AnyJSONRPCResponse) -> Void,
        contextProvider: ToolContextProvider?
    ) -> Bool {
        guard let input = request.params?.input,
              let param1 = input["param1"]?.value as? String
        else {
            completeResponse(request, status: .error,
                           response: "Invalid parameters",
                           completion: completion)
            return true
        }

        // Do work
        let result = doSomething(param1)

        // Return result
        completeResponse(request, response: result, completion: completion)
        return true
    }

    private func doSomething(_ input: String) -> String {
        // Your implementation
        return "Result: \(input)"
    }
}
```

---

**Step 4: Register in Tool Registry**

**File**: `Tool/Sources/GitHubCopilotService/LanguageServer/CopilotToolRegistry.swift`

```swift
public class CopilotToolRegistry {
    private init() {
        tools[ToolName.myCustomTool.rawValue] = MyCustomTool()
        // ... existing tools
    }
}
```

---

**Step 5: Build & Test**

```bash
cd Script
sh localbuild-app.sh
```

Test in chat:
- Ask model to use your tool
- Verify tool appears in registered tools
- Check execution logs

---

## 11. Summary & Checklist

### For Local LLM Usage

**Prerequisites**:
- ✅ Model supports function/tool calling
- ✅ 8K+ context window
- ✅ Reliable JSON output
- ✅ LMStudio or similar serving OpenAI-compatible API

**Configuration**:
- ✅ Add model via Azure provider workaround
- ✅ Set `deploymentUrl: http://localhost:1234/v1`
- ✅ Enable model in BYOK settings
- ✅ Select model in chat dropdown

**In Chat**:
- ✅ Select "Agent" mode (enables tools)
- ✅ Verify tools appear (check UI or logs)
- ✅ Test simple tool first (e.g., `fetch_webpage`)
- ✅ Then test `insert_edit_into_file`

**Troubleshooting**:
- ✅ Check LMStudio server running
- ✅ Test tool calling with curl
- ✅ Verify Accessibility permissions
- ✅ Check Console.app logs
- ✅ Start with lower temperature (0.3)

---

### Key Takeaways

1. **Tools ONLY in chat mode** - Autocomplete/NES don't support tools
2. **Language Server handles translation** - You don't adapt per model
3. **Universal JSON Schema** - One tool definition works for all models
4. **Edit tool uses Accessibility APIs** - Not LLM-dependent
5. **Model must support tool calling** - Non-negotiable for local LLMs
6. **GitHub Copilot does the heavy lifting** - You just configure model & provider

---

## 12. File Reference

**Tool Definition & Registration**:
- `Tool/Sources/GitHubCopilotService/LanguageServer/ClientToolRegistry.swift`
- `Tool/Sources/GitHubCopilotService/LanguageServer/ToolNames.swift`
- `Tool/Sources/ConversationServiceProvider/LSPTypes.swift` (types)

**Tool Implementations**:
- `Core/Sources/ChatService/ToolCalls/InsertEditIntoFileTool.swift` ⭐
- `Core/Sources/ChatService/ToolCalls/CreateFileTool.swift`
- `Core/Sources/ChatService/ToolCalls/RunInTerminalTool.swift`
- `Core/Sources/ChatService/ToolCalls/GetErrorsTool.swift`
- `Core/Sources/ChatService/ToolCalls/FetchWebPageTool.swift`

**Tool Invocation Flow**:
- `Tool/Sources/GitHubCopilotService/LanguageServer/ServerRequestHandler.swift`
- `Tool/Sources/GitHubCopilotService/LanguageServer/ClientToolHandler.swift`
- `Core/Sources/ChatService/ChatService.swift` (subscriber)

**Tool Registry**:
- `Tool/Sources/GitHubCopilotService/LanguageServer/CopilotToolRegistry.swift`

**MCP Tools**:
- `Tool/Sources/GitHubCopilotService/LanguageServer/CopilotMCPToolManager.swift`
- `Tool/Sources/GitHubCopilotService/LanguageServer/GitHubCopilotRequestTypes/MCP.swift`

**Accessibility APIs** (for edit tool):
- `Tool/Sources/AXHelper/AXHelper.swift`
- `Tool/Sources/XcodeInspector/AppInstanceInspector.swift`
