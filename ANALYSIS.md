# GitHub Copilot for Xcode - LLM Model Analysis

## 1. Traditional Completion vs NES (Next Edit Suggestions)

### Traditional Completion (`textDocument/inlineCompletion`)

**LSP Method**: `textDocument/inlineCompletion`
**Request Parameters**:
```swift
{
    textDocument: { uri, version },
    position: { line, character },
    formattingOptions: { tabSize, insertSpaces },
    context: { triggerKind: .invoked }
}
```

**Response Structure**:
```swift
InlineCompletionItem {
    insertText: String          // Text to insert
    filterText: String?         // Optional filter text
    range: Range?              // Optional range (start, end positions)
    command: Command?          // Telemetry command with UUID
}
```

**Behavior**:
- Forward-only completions (appends text after cursor)
- Multiple suggestions returned as array
- Typically completes current line or adds new lines
- Used for standard autocomplete scenarios
- Sends accept/reject telemetry via `notifyAccepted`/`notifyRejected`

**File**: `Tool/Sources/GitHubCopilotService/LanguageServer/GitHubCopilotService.swift:523-625`

---

### NES - Next Edit Suggestions (`textDocument/copilotInlineEdit`)

**LSP Method**: `textDocument/copilotInlineEdit`
**Request Parameters**:
```swift
CopilotInlineEditsParams {
    textDocument: { uri, version },
    position: { line, character }
}
```

**Response Structure**:
```swift
CopilotInlineEdit {
    text: String                          // New text for edit
    textDocument: VersionedTextDocumentIdentifier
    range: CursorRange                    // Range to replace (start, end)
    command: Command?                     // Telemetry command
}
```

**Behavior**:
- Can replace arbitrary ranges (not just append)
- Edits existing code, not just completions
- More powerful than traditional - can modify/refactor
- Used for inline editing scenarios
- Sends accept telemetry via `github.copilot.didAcceptNextEditSuggestionItem` command

**File**: `Tool/Sources/GitHubCopilotService/LanguageServer/GitHubCopilotService.swift:627-666`

---

### Key Differences

| Feature | Traditional Completion | NES (Inline Edit) |
|---------|----------------------|-------------------|
| **Operation** | Insert/append only | Replace ranges |
| **Use Case** | Autocomplete, code continuation | Code modification, refactoring |
| **Range** | After cursor position | Arbitrary range in document |
| **Complexity** | Simple forward completion | Context-aware editing |
| **Request** | Includes formatting options | Minimal parameters |
| **Telemetry** | `notifyAccepted`/`notifyRejected` | `didAcceptNextEditSuggestionItem` |

---

## 2. Model Control for Autocomplete

### Current State

**❌ NO CONTROL OVER AUTOCOMPLETE MODEL**

**Facts**:
- No `model` parameter in `textDocument/inlineCompletion` or `textDocument/copilotInlineEdit` requests
- Model selection is **server-side** - controlled by GitHub Copilot's language server
- Cannot specify model via API
- Cannot query which model is currently being used
- Cannot guarantee knowledge cutoff date
- No way to use local models for autocomplete
- No way to use alternative APIs (OpenAI, Anthropic, etc.) for autocomplete

**Evidence**:
```swift
// Traditional completion request - NO model parameter
GitHubCopilotRequest.InlineCompletion(doc: .init(
    textDocument: .init(uri: fileURL.absoluteString, version: 1),
    position: cursorPosition,
    formattingOptions: .init(tabSize: tabSize, insertSpaces: !usesTabsForIndentation),
    context: .init(triggerKind: .invoked)
))

// NES request - NO model parameter
GitHubCopilotRequest.CopilotInlineEdit(
    params: CopilotInlineEditsParams(
        textDocument: .init(uri: fileURL.absoluteString, version: 1),
        position: cursorPosition
    )
)
```

**Files**:
- Request definition: `Tool/Sources/GitHubCopilotService/LanguageServer/GitHubCopilotRequest.swift:229-277`
- NES request: `Tool/Sources/GitHubCopilotService/LanguageServer/GitHubCopilotRequest.swift:295-307`

---

### Why This Limitation Exists

1. **Architecture**: GitHub Copilot Language Server is a Node.js process that wraps GitHub's proprietary inference infrastructure
2. **LSP Protocol**: Standard LSP `textDocument/inlineCompletion` doesn't include model parameters
3. **Performance**: Autocomplete requires <100ms latency - GitHub's infrastructure is optimized for this
4. **GitHub Control**: Model selection, A/B testing, and optimization are managed by GitHub

---

### Attempted Workarounds (All Fail)

**Workaround 1: Use BYOK for suggestions**
- ❌ BYOK only works for chat (`conversation/create`, `conversation/turn`)
- Suggestion providers don't support BYOK models
- File: `Tool/Sources/GitHubCopilotService/Services/GitHubCopilotSuggestionService.swift`

**Workaround 2: Modify request to add model parameter**
- ❌ GitHub's language server ignores unknown parameters
- Server-side model routing is hardcoded

**Workaround 3: Replace GitHub Copilot LSP server**
- ❌ Would require rewriting entire suggestion system
- Would lose GitHub Copilot authentication, telemetry, and infrastructure
- Massive engineering effort

---

### Model Information for Autocomplete

**Unknown Variables**:
- Which model(s) GitHub uses for autocomplete
- Whether it's the same model for all users
- Knowledge cutoff date of autocomplete model
- Whether GitHub A/B tests different models
- Model version or update frequency

**What We Know**:
- Model is likely optimized for low-latency code completion
- Likely smaller/faster than chat models
- May be different from GitHub Copilot Chat models
- GitHub controls updates and rollouts

**Official Channels**:
- Check GitHub Copilot settings in GitHub.com account
- GitHub Copilot documentation may specify models
- Contact GitHub support for model information
- Monitor GitHub Copilot changelog for model updates

---

## 3. Model Control for Chat

### Current Architecture

**✅ FULL CONTROL OVER CHAT MODELS**

**Chat API Parameters**:
```swift
// conversation/create and conversation/turn include:
model: String?                  // Model ID (e.g., "gpt-4", "claude-3-5-sonnet")
modelProviderName: String?      // Provider (e.g., "OpenAI", "Anthropic", "Azure")
```

**Files**:
- Request: `Tool/Sources/GitHubCopilotService/LanguageServer/GitHubCopilotService.swift:669-720`
- Service: `Tool/Sources/GitHubCopilotService/Services/GitHubCopilotConversationService.swift:43-86`

---

### BYOK (Bring Your Own Key) System

**Supported Providers** (`Tool/Sources/GitHubCopilotService/LanguageServer/GitHubCopilotRequestTypes/BYOK.swift:3-14`):
- OpenAI
- Anthropic
- Azure
- Gemini (Google)
- Groq
- OpenRouter

**Authentication Types**:
1. **GlobalApiKey**: Single API key for all models
   - Used by: OpenAI, Anthropic, Gemini, Groq, OpenRouter
   - Key stored globally per provider

2. **PerModelDeployment**: API key + deployment URL per model
   - Used by: Azure
   - Allows custom endpoints (localhost support!)

**Configuration** (`Core/Sources/HostApp/BYOKSettings/BYOKObservable.swift:206-241`):
```swift
enum BYOKAuthType {
    case GlobalApiKey              // Single key for provider
    case PerModelDeployment        // Key + URL per model
}
```

---

## 4. Adding Local LLM Support to Chat

### Option 1: Use Azure Provider (Workaround - No Code Changes)

**Steps**:
1. Open app → Settings → BYOK Settings
2. Select **Azure** as provider
3. Click "Add Model"
4. Configure:
   - **Deployment Name**: Your model name (e.g., "llama-3.1-8b")
   - **Target URI**: `http://localhost:1234/v1` (LMStudio default)
   - **API Key**: Any string (use "local" if no validation)
   - **Display Name**: "Local LLaMA 3.1" (optional)
   - **Tool Calling**: Enable if model supports it
   - **Vision**: Enable if model supports it
5. Enable the model
6. Select it in chat

**Requirements**:
- LMStudio (or similar) must expose OpenAI-compatible API
- Endpoint: `http://localhost:1234/v1/chat/completions`
- API key validation may be bypassed (test with dummy key)

**Limitations**:
- Must use "Azure" provider (confusing UX)
- No localhost-specific UI
- May show Azure-related help text

**File**: `Core/Sources/HostApp/BYOKSettings/ModelSheet.swift:5-171`

---

### Option 2: Add "Custom" Provider (Proper Solution - Minimal Code Changes)

**Changes Required**:

#### **Step 1: Add Custom Provider Enum Value**

**File**: `Tool/Sources/GitHubCopilotService/LanguageServer/GitHubCopilotRequestTypes/BYOK.swift`

```swift
public enum BYOKProviderName: String, Codable, Equatable, Hashable, Comparable, CaseIterable {
    case Azure
    case Anthropic
    case Gemini
    case Groq
    case OpenAI
    case OpenRouter
    case Custom        // ADD THIS

    public static func < (lhs: BYOKProviderName, rhs: BYOKProviderName) -> Bool {
        return lhs.rawValue < rhs.rawValue
    }
}
```

#### **Step 2: Configure Auth Type**

**File**: `Core/Sources/HostApp/BYOKSettings/BYOKObservable.swift`

```swift
extension BYOKProviderName {
    var title: String {
        switch self {
        case .Azure: return "Azure"
        case .Anthropic: return "Anthropic"
        case .Gemini: return "Gemini"
        case .Groq: return "Groq"
        case .OpenAI: return "OpenAI"
        case .OpenRouter: return "OpenRouter"
        case .Custom: return "Custom (Local LLM)"  // ADD THIS
        }
    }

    var authType: BYOKAuthType {
        switch self {
        case .Anthropic, .Gemini, .Groq, .OpenAI, .OpenRouter:
            return .GlobalApiKey
        case .Azure, .Custom:  // ADD .Custom HERE
            return .PerModelDeployment
        }
    }
}
```

#### **Step 3: Update UI Labels (Optional)**

**File**: `Core/Sources/HostApp/BYOKSettings/ModelSheet.swift`

Update line 42-54 to show better labels for Custom provider:
```swift
TextField(
    isPerModelDeployment
        ? (provider == .Custom ? "Model Name" : "Deployment Name")
        : "Model ID",
    text: $modelId
)
```

Update line 54 help text:
```swift
TextField(
    provider == .Custom ? "Base URL (e.g., http://localhost:1234/v1)" : "Target URI",
    text: $deploymentUrl
)
```

**That's It!** The rest of the infrastructure is already in place.

---

### Architecture Diagram: BYOK Flow

```
┌─────────────────────────────────────────────────────┐
│ Chat UI (ConversationTab)                           │
│  - User selects model from dropdown                 │
│  - Includes GitHub Copilot native + BYOK models     │
└──────────────────────┬──────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────┐
│ ChatService                                          │
│  - Builds ConversationRequest with model params     │
│  - model: "llama-3.1-8b"                            │
│  - modelProviderName: "Custom"                      │
└──────────────────────┬──────────────────────────────┘
                       │ XPC
                       ▼
┌─────────────────────────────────────────────────────┐
│ GitHubCopilotConversationService                     │
│  - Forwards to GitHub Copilot Language Server       │
└──────────────────────┬──────────────────────────────┘
                       │ LSP
                       ▼
┌─────────────────────────────────────────────────────┐
│ GitHub Copilot Language Server (Node.js)            │
│  - Routes request based on modelProviderName        │
│  - For BYOK: Makes HTTP call to configured endpoint│
│  - deploymentUrl: http://localhost:1234/v1          │
│  - apiKey: Retrieved from BYOK manager              │
└──────────────────────┬──────────────────────────────┘
                       │ HTTP
                       ▼
┌─────────────────────────────────────────────────────┐
│ Local LLM Server (LMStudio, Ollama, etc.)           │
│  - OpenAI-compatible API                            │
│  - POST /v1/chat/completions                        │
│  - Returns streaming response                       │
└─────────────────────────────────────────────────────┘
```

---

## 5. Additional Model Controls to Consider

### A. Model Switching for Autocomplete (Requires GitHub Changes)

**What Would Be Needed**:
- GitHub Copilot Language Server update to accept model parameter
- GitHub infrastructure changes to support multiple models
- Model selection UI in app settings

**Likelihood**: Low - GitHub controls this tightly

---

### B. Model Temperature/Parameters for Chat

**Current State**: Not configurable
**What Could Be Added**:
- Temperature slider in chat UI
- Max tokens setting
- Top-p, frequency penalty controls

**Implementation**:
- Add parameters to `ConversationRequest`
- Update `conversation/create` LSP call
- Requires GitHub Copilot LSP support (may not accept custom params)

**File to Modify**: `Tool/Sources/ConversationServiceProvider/Types.swift`

---

### C. Model Performance Monitoring

**What to Add**:
- Token usage tracking (input/output)
- Response latency metrics
- Model-specific analytics

**Implementation**:
- Capture LSP progress notifications
- Parse telemetry data
- Display in UI or logs

**Files**:
- `Tool/Sources/GitHubCopilotService/LanguageServer/GitHubCopilotService.swift` (LSP notifications)
- `Core/Sources/ChatService/ChatService.swift` (analytics)

---

### D. Model Selection Presets

**Feature**: Quick-switch profiles
- "Fast" - GPT-3.5 or smaller models
- "Balanced" - GPT-4 or Claude 3.5
- "Powerful" - Claude Opus or GPT-4 Turbo
- "Local" - Custom local models

**Implementation**:
- Add preset enum to preferences
- Map presets to model IDs
- UI toggle in chat panel

**Files**:
- `Tool/Sources/Preferences/Types/` (new preset type)
- `Core/Sources/ConversationTab/` (UI)

---

### E. Per-Workspace Model Configuration

**Feature**: Different models for different projects
- iOS project → GPT-4 (better Swift knowledge)
- Data science → Claude Opus (better analysis)
- Legacy code → Local model (privacy)

**Implementation**:
- Extend `Workspace` type to include model preferences
- Load workspace-specific settings on project open
- Override global model selection

**Files**:
- `Tool/Sources/Workspace/WorkspaceInfo.swift`
- `Tool/Sources/Preferences/` (workspace-specific prefs)

---

### F. Fallback Model Configuration

**Feature**: Automatic fallback if primary model fails
- Primary: Claude Opus → Fallback: GPT-4 → Fallback: Local model

**Implementation**:
- Retry logic in `GitHubCopilotConversationService`
- Model priority list in preferences
- Error handling for rate limits, timeouts

**Files**:
- `Tool/Sources/GitHubCopilotService/Services/GitHubCopilotConversationService.swift:43-86`

---

## 6. Summary & Action Items

### Autocomplete Model Control

**Status**: ❌ Not possible
**Reason**: GitHub Copilot Language Server controls model selection server-side
**Workaround**: None available
**To Check Model**: Contact GitHub Support or check GitHub.com Copilot settings

---

### Chat Model Control

**Status**: ✅ Fully supported
**Current Options**: 6 BYOK providers + GitHub Copilot native models
**Local LLM Support**:
- **Quick Fix**: Use Azure provider with localhost URL
- **Proper Fix**: Add 3 lines to create "Custom" provider (15 minutes of work)

---

### Recommended Next Steps

**Immediate (0 code changes)**:
1. Configure LMStudio with OpenAI-compatible API on port 1234
2. Use Azure provider workaround to connect
3. Test with local model

**Short-term (< 1 hour)**:
1. Add "Custom" provider enum value
2. Configure as PerModelDeployment auth type
3. Test and iterate

**Long-term (Future enhancements)**:
1. Add model performance monitoring
2. Implement model presets
3. Add per-workspace model configuration
4. Build fallback model system

---

## Files Reference

**Key Files for Understanding**:
- `Tool/Sources/GitHubCopilotService/LanguageServer/GitHubCopilotService.swift` - Core LSP client
- `Tool/Sources/GitHubCopilotService/Services/GitHubCopilotSuggestionService.swift` - Autocomplete
- `Tool/Sources/GitHubCopilotService/Services/GitHubCopilotConversationService.swift` - Chat
- `Tool/Sources/GitHubCopilotService/LanguageServer/GitHubCopilotRequestTypes/BYOK.swift` - BYOK types
- `Core/Sources/HostApp/BYOKSettings/BYOKObservable.swift` - BYOK provider config
- `Core/Sources/HostApp/BYOKSettings/ModelSheet.swift` - Model configuration UI

**Files to Modify for Local LLM**:
1. `Tool/Sources/GitHubCopilotService/LanguageServer/GitHubCopilotRequestTypes/BYOK.swift:3-14` (add enum)
2. `Core/Sources/HostApp/BYOKSettings/BYOKObservable.swift:221-241` (add title & auth type)
3. `Core/Sources/HostApp/BYOKSettings/ModelSheet.swift:42-54` (optional UI improvements)
