# Subagents Deep Dive

Assumes you've read `AGENT_MODE.md`. This focuses on subagent-specific mechanics.

---

## 1. How Subagents Are Called

### Decision Authority

**GitHub Copilot Language Server decides** when to spawn subagents, NOT the Xcode app.

**Decision Process**:
1. User sends request to main agent
2. Main agent (running in Language Server) analyzes task
3. If task benefits from specialization, Language Server spawns subagent
4. Subagent executes with its own conversation context
5. Results merge back to main agent
6. Main agent continues with subagent's output

**You cannot directly invoke subagents** - the Language Server's orchestration logic determines when delegation is appropriate.

---

## 2. User Influence on Subagent Invocation

### Indirect Request Methods

**Users can SUGGEST, not COMMAND**:

✅ **Effective**:
```
"Create a detailed plan for implementing authentication, then execute it"
→ May trigger Plan subagent, then implementation agent
```

```
"Review the code for security issues"
→ May trigger CVE_Remediator if CVEs detected
```

✅ **Via Handoffs** (in agent definitions):
- Plan agent has "Start Implementation" handoff
- User can click handoff button in UI
- Triggers explicit delegation to specified agent

❌ **Does NOT Work**:
```
"@plan create a plan"  // No @ mention support
"Use the Plan subagent"  // Direct invocation not supported
"/plan make a plan"  // Not a slash command
```

### Handoff Mechanism

**Defined in `.agent.md` frontmatter**:

```markdown
---
name: Plan
handoffs:
  - label: Start Implementation
    agent: Agent
    prompt: Start implementation
    send: true
---
```

**Handoff Structure** (`Tool/Sources/ConversationServiceProvider/LSPTypes.swift:125-137`):
```swift
struct HandOff {
    let agent: String        // Target agent ID (e.g., "Agent", "CVE_Remediator")
    let label: String        // Button label shown in UI
    let prompt: String       // Auto-sent message to new agent
    let send: Bool?          // If true, sends prompt automatically
}
```

**User Experience**:
- Handoff appears as button in chat UI
- Click button → Language Server spawns specified agent
- Optional auto-send of prompt message

---

## 3. Agent Definition Requirements

### Must Be Defined Beforehand

**YES** - Subagents MUST exist as either:

1. **Built-in Agents** (in Language Server):
   - `Plan` - Research and planning
   - `CVE_Remediator` - Security vulnerability fixes
   - Located: `Server/node_modules/@github/copilot-language-server/dist/assets/agents/`

2. **Custom Agents** (in workspace):
   - `.github/agents/*.agent.md` files
   - Discovered via `conversation/modes` LSP request
   - Requires `enableSubagent` preference + `isCustomAgentEnabled` policy

**Cannot spawn undefined agents** - Language Server only knows about registered agent definitions.

### Built-in Agent: Plan

**File**: `Server/.../agents/Plan.agent.md`

**Capabilities**:
- Tools: `read_file`, `list_dir`, `semantic_search`, `grep_search`, `file_search`, `get_errors`
- **Read-only** - Cannot modify files
- Iterative workflow: gather context → draft plan → user feedback loop
- Handoffs to "Agent" for implementation

**Key Constraint**:
```
STOP IMMEDIATELY if you consider starting implementation
Plans describe steps for the USER or another agent to execute later.
```

### Built-in Agent: CVE_Remediator

**File**: `Server/.../agents/CVE_Remediator.agent.md`

**Purpose**: Detect and fix security vulnerabilities

**Policy Control**:
```swift
struct CopilotPolicy {
    var cveRemediatorAgentEnabled: Bool = true
}
```

Organizations can disable this agent centrally.

---

## 4. Separate LLM Call & Context

### Each Subagent = New Conversation

**YES** - Subagents are separate LLM calls with:

**Separate**:
- `conversationId` - Unique per subagent
- `turnId` - Independent turn tracking
- Context window - Fresh conversation context
- System prompt - From `.agent.md` definition
- Tool access - Only tools specified in agent definition

**Linked**:
- `parentTurnId` - References parent agent's turn
- Tracked in `turnParentMap` for relationship management

**Data Structure** (`Tool/Sources/ConversationServiceProvider/LSPTypes+AgentRound.swift:6-18`):
```swift
struct AgentRound {
    let roundId: Int                    // Unique round identifier
    var reply: String                   // Agent's text response
    var toolCalls: [AgentToolCall]?     // Tools used
    var subAgentRounds: [AgentRound]?   // ← Nested subagent executions
}
```

**Tracking** (`Core/Sources/ChatService/ChatService.swift:697-706`):
```swift
struct ConversationTurnTrackingState {
    var turnParentMap: [String: String]       // turnId → parentTurnId
    var validConversationIds: Set<String>     // All valid IDs (main + subs)
}

// Track relationship
if let parentTurnId = progress.parentTurnId {
    conversationTurnTracking.turnParentMap[turnId] = parentTurnId
}
```

---

## 5. Context Merging: Subagent → Parent

### What Gets Added Back

**Merged into parent's `subAgentRounds[]`** (`Tool/Sources/ChatAPIService/Memory/ChatMemory.swift:23-68`):

1. **Text Reply** (`reply: String`)
   - Subagent's reasoning and responses
   - Appended to existing reply if round already exists

2. **Tool Calls** (`toolCalls: [AgentToolCall]`)
   - All tools invoked by subagent
   - Includes status, parameters, results, errors
   - Merged by tool call ID to avoid duplicates

3. **Nested Sub-Subagents** (recursive)
   - If subagent spawns its own subagents
   - Nested `subAgentRounds` maintain full hierarchy

**Merging Algorithm**:
```swift
// Parent round structure
parentRounds[lastParentRoundIndex].subAgentRounds = [
    AgentRound(
        roundId: 1,
        reply: "Subagent analyzed the code...",
        toolCalls: [
            AgentToolCall(name: "read_file", status: .completed, ...),
            AgentToolCall(name: "grep_search", status: .completed, ...)
        ],
        subAgentRounds: nil  // or further nesting
    )
]
```

### What Does NOT Transfer

**Not merged**:
- Subagent's internal context window
- Intermediate reasoning steps (unless in reply)
- Failed/cancelled tool calls (unless recorded)
- Conversation metadata (turnId, conversationId remain separate)

**Parent sees**:
- Final output (reply text)
- Tools executed (with results)
- Nested subagent activity (if any)

---

## 6. Limits on Subagent Count

### No Explicit Hard Limit

**Findings from codebase search**:
- ❌ No `maxSubagents` configuration
- ❌ No limit on `subAgentRounds[]` array size
- ❌ No depth limit on nesting

**Practical Limits**:
- **LLM context window** - Too many subagents exhaust context
- **Token costs** - Each subagent = separate LLM call
- **Response time** - Sequential execution increases latency
- **Language Server heuristics** - Likely has internal limits (not exposed)

**Recursive Nesting**:
```swift
AgentRound {
    subAgentRounds: [
        AgentRound {
            subAgentRounds: [  // Sub-subagent
                AgentRound { ... }
            ]
        }
    ]
}
```

**Theoretically unlimited depth** - but impractical beyond 2-3 levels.

---

## 7. Multiple Instances of Same Subagent

### YES - Tracked by `roundId`

**Each invocation gets unique `roundId`**:

```swift
subAgentRounds: [
    AgentRound(roundId: 1, reply: "Plan agent - first call"),
    AgentRound(roundId: 2, reply: "Plan agent - second call"),
    AgentRound(roundId: 3, reply: "Plan agent - third call")
]
```

**Use Case Example**:
```
User: "Create plans for auth, payments, and notifications"
→ Main agent spawns Plan subagent 3 times
→ Each produces separate plan
→ All merged into parent's subAgentRounds[]
```

**Merging Logic** (`Tool/Sources/ChatAPIService/Memory/ChatMemory.swift:188-225`):
- Matches by `roundId` to update existing
- Appends new if `roundId` not found
- Allows parallel/sequential same-agent invocations

---

## 8. Pre-defined Subagents

### Two Built-in Agents

#### **Plan Agent**

**Location**: `Server/.../agents/Plan.agent.md`

**Purpose**: Research and outline multi-step plans

**Tools**:
- `read_file` - Read workspace files
- `list_dir` - List directory contents
- `semantic_search` - Semantic code search
- `grep_search` - Pattern-based search
- `file_search` - File name search
- `get_errors` - Get Xcode errors

**Workflow**:
1. Gather context comprehensively
2. Present draft plan for review
3. Iterate based on user feedback
4. **Never** starts implementation

**Handoffs**:
- "Start Implementation" → Agent (with auto-send)
- "Open in Editor" → Agent (saves plan to `.prompt.md` file)

**Key Behavior**:
```
STOPPING_RULES: STOP if considering implementation
Plans are for USER or another agent to execute later
```

---

#### **CVE_Remediator Agent**

**Location**: `Server/.../agents/CVE_Remediator.agent.md`

**Purpose**: Detect and fix security vulnerabilities

**Mission**:
1. Identify CVEs in dependencies (by severity)
2. Upgrade to patched versions
3. Resolve build errors from upgrades
4. Verify no new CVEs introduced

**Policy Control**:
```swift
// Tool/Sources/GitHubCopilotService/Services/CopilotPolicyNotifier.swift
struct CopilotPolicy {
    var cveRemediatorAgentEnabled: Bool = true
}
```

**Enable/Disable**: Via organization policy (not user setting)

---

## 9. LLM API Call Management

### Subagent = Separate `conversation/create`

**Each subagent invocation triggers**:
1. New `conversation/create` LSP request
2. With `parentTurnId` parameter set
3. Separate conversation context
4. Independent from parent

### Example: Azure API (localhost via workaround)

**Parent Agent Request**:
```http
POST http://localhost:1234/v1/chat/completions
Content-Type: application/json

{
  "model": "llama-3.1-8b",
  "messages": [
    {"role": "system", "content": "You are Agent..."},
    {"role": "user", "content": "Create a plan then implement it"}
  ],
  "tools": [
    {"type": "function", "function": {"name": "read_file", ...}},
    {"type": "function", "function": {"name": "insert_edit_into_file", ...}}
  ]
}
```

**Response** (Language Server decides to spawn Plan subagent):
```json
{
  "choices": [{
    "message": {
      "role": "assistant",
      "content": "I'll create a plan first...",
      "tool_calls": []
    }
  }]
}
```

**Subagent Request** (NEW conversation):
```http
POST http://localhost:1234/v1/chat/completions
Content-Type: application/json

{
  "model": "llama-3.1-8b",
  "messages": [
    {"role": "system", "content": "You are a PLANNING AGENT..."},
    {"role": "user", "content": "Create a plan for implementing auth"}
  ],
  "tools": [
    {"type": "function", "function": {"name": "read_file", ...}},
    {"type": "function", "function": {"name": "semantic_search", ...}}
    // NO insert_edit_into_file - Plan is read-only
  ]
}
```

**Subagent Response**:
```json
{
  "choices": [{
    "message": {
      "role": "assistant",
      "content": "## Plan: Authentication Implementation\n\n1. Create auth models...",
      "tool_calls": [
        {"id": "call_1", "function": {"name": "read_file", "arguments": "{...}"}}
      ]
    }
  }]
}
```

**After Subagent Completes** - Parent continues:
```http
POST http://localhost:1234/v1/chat/completions
Content-Type: application/json

{
  "model": "llama-3.1-8b",
  "messages": [
    {"role": "system", "content": "You are Agent..."},
    {"role": "user", "content": "Create a plan then implement it"},
    {"role": "assistant", "content": "I'll create a plan first..."},
    {"role": "assistant", "content": "[Subagent Plan]: ## Plan: Authentication...\n\n1. Create auth models..."},
    {"role": "assistant", "content": "Now I'll implement the plan..."}
  ],
  "tools": [...]
}
```

---

### Example: OpenAI API

**Parent Agent**:
```http
POST https://api.openai.com/v1/chat/completions
Authorization: Bearer sk-...

{
  "model": "gpt-4",
  "messages": [...],
  "functions": [
    {"name": "read_file", "description": "...", "parameters": {...}},
    {"name": "insert_edit_into_file", ...}
  ]
}
```

**Subagent (Plan)**:
```http
POST https://api.openai.com/v1/chat/completions
Authorization: Bearer sk-...

{
  "model": "gpt-4",
  "messages": [
    {"role": "system", "content": "You are a PLANNING AGENT..."},
    {"role": "user", "content": "Create implementation plan"}
  ],
  "functions": [
    {"name": "read_file", ...},
    {"name": "semantic_search", ...}
    // Different function set - no editing tools
  ]
}
```

---

### Key API Patterns

**Each Subagent Call**:
1. **NEW API request** - Not continuation of parent
2. **Separate system prompt** - From `.agent.md`
3. **Different tool set** - Specified in agent definition
4. **Independent context** - Fresh conversation
5. **Parent receives summary** - Via `subAgentRounds[]` merge

**Token Costs**:
- Parent conversation: Full context window
- Subagent #1: Full context window (separate)
- Subagent #2: Full context window (separate)
- **Total**: N+1 full conversations (N = number of subagents)

**Latency**:
- Sequential execution (one at a time)
- Each subagent blocks parent until completion
- No parallel subagent execution (currently)

---

## Summary: Critical Points

### Invocation
- ✅ Language Server decides when to spawn
- ✅ Users can suggest via prompt phrasing
- ✅ Handoffs provide explicit delegation
- ❌ No direct "@subagent" invocation

### Definition
- ✅ Must be pre-defined (built-in or `.agent.md`)
- ✅ Plan and CVE_Remediator built-in
- ✅ Custom agents require `enableSubagent` + policy

### Context
- ✅ Separate LLM call with own `conversationId`
- ✅ Fresh context window per subagent
- ✅ Results merge into parent's `subAgentRounds[]`
- ✅ Parent sees: reply, toolCalls, nested subagents

### Limits
- ❌ No hard limit on count
- ✅ Practical limits: context, cost, latency
- ✅ Multiple instances of same agent allowed
- ✅ Unlimited nesting depth (theoretically)

### API Calls
- ✅ Each subagent = separate `conversation/create`
- ✅ Different system prompts and tool sets
- ✅ Sequential execution (blocks parent)
- ✅ Works with all providers (Azure, OpenAI, local)

---

## File References

**Subagent Data Structures**:
- `Tool/Sources/ConversationServiceProvider/LSPTypes+AgentRound.swift`
- `Tool/Sources/ConversationServiceProvider/LSPTypes.swift:81-137` (HandOff)

**Context Merging**:
- `Tool/Sources/ChatAPIService/Memory/ChatMemory.swift`

**Tracking**:
- `Core/Sources/ChatService/ChatService.swift:690-739`

**Built-in Agents**:
- `Server/.../agents/Plan.agent.md`
- `Server/.../agents/CVE_Remediator.agent.md`

**UI**:
- `Core/Sources/ConversationTab/ModeAndModelPicker/ModePicker/ChatModePicker.swift`
- `Core/Sources/ConversationTab/Views/ConversationAgentProgressView/`
