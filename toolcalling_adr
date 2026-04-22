ADR for tool calling:

**Abstract**
This document describes the tool calling architecture in the agentic workflow engine. The system enables LLM agents to invoke external tools (MCP, handoff, knowledge, Quantum MCP) within LangGraph-based workflows, handling schema filtering, parameter resolution, tool execution, and result processing. It serves as a reference for developers working on or extending the tool calling pipeline.

**1. Context and Problem Statement**

**Context:**

The system uses LangChain/LangGraph to build agentic workflows with ReAct-style agents
Agents have embedded tools configured via a visual canvas UI, where each tool has a JSON Schema and a config.parameters map for user-configured values
Users can set some parameters as static (fixed values or template variables) and leave others for the LLM to generate ("Let agent decide")
Tools can be of multiple types: mcp, mcp-resource, mcp-prompt, handoff, and quantummcp. Knowledge tools exist but follow a separate instantiation path (not in the standard tool type registry).
Multiple LLM providers (OpenAI, Azure OpenAI, Gemini) are supported, each with different schema requirements

**Problem**
- **Static vs. dynamic parameters:** The LLM shouldn't generate every parameter — some are pre-configured at design time and must be injected at runtime without LLM involvement.
- **Nested partial configuration:** A single parameter can be a deeply nested object where some sub-fields are static and others must be LLM-generated.
- **Multi-provider schema compatibility:** Each LLM provider (OpenAI, Azure, Gemini) has different schema requirements that tool definitions must accommodate.
- **Agent handoff with state preservation:** Agents must route control mid-conversation while preserving thread history and workflow state.
- **Selective output filtering:** Tool outputs may need to be trimmed before returning to the LLM to reduce token usage, while remaining fully available for state updates and debugging.
- **Full parameter visibility:** Full parameter visibility: Resolved parameters (static + dynamic) are stored in _toolDebug for Langfuse trace visibility, without polluting the LLM conversation history.

**Business Value**

**Fine-grained control**: workflow designers can mix static and dynamic values even within nested parameters\
**Reduced LLM costs**: the LLM only sees fields it needs to generate\
**Predictable behavior**: user-configured values are never overridden by the LLM\
**Safe concurrency**: fresh tool instances prevent parameter leakage between workflow runs\
**Provider transparency:** tool schemas automatically adapt to the agent's LLM provider without any per-tool configuration\
**Debugging transparency**: resolved parameters visible in Langfuse traces via _toolDebug; tool outputs available via toolResults

#### Goals and Requirements

| Goal | Description | Priority |
|------|-------------|----------|
| Unified parameter resolution | Merge static and LLM-generated sub-fields within the same parameter via recursive deep merge (`deepMergeObjects`) | High |
| Accurate LLM schema exposure | Hide fully static parameters; for partially configured nested objects, expose only unconfigured sub-fields (`prepareSchemaForLLM`) | High |
| Full parameter visibility | Resolved parameters stored in _toolDebug (Langfuse traces). Never enters LLM conversation history. | High
| Consistent behavior across tool types | Same resolution and filtering for all tool types (MCP, Handoff, Knowledge) and both paths (agent tools, standalone `ToolNode`) | High |
| Centralized schema filtering | Single shared utility (`schema.utils.ts`) for all recursive filtering, preventing duplication | Medium |
| Arbitrary nesting support | Handle objects within objects, arrays of objects, and nullable types at any depth | High |
| Prototype pollution protection | Skip `__proto__`, `constructor`, `prototype` keys in schema filtering and deep merge | High |
| Fresh instances per invocation | New `BaseTool` subclass per call to prevent shared mutable state | Medium |
| Tool call ID linkage | Correct `tool_call_id` mapping between `AIMessage` and `ToolMessage` for Azure OpenAI compliance | Medium |

**2.Architecture Overview**

High-Level Design\
The tool calling system is organized into four layers, each with a clear responsibility:

| Layer | What it does | When it runs |
|-------|-------------|--------------|
| Definition | Converts UI tool configs into runtime `ToolDefinition` objects and maps tool types to their classes | Once at workflow load |
| Schema | Filters the tool's JSON Schema to show only dynamic fields to the LLM, with provider-specific adjustments | Once at agent initialization |
| Execution | Resolves templates, deep-merges static + dynamic params, and runs the tool's core logic | Every tool invocation |
| Response | Extracts tool calls and results from agent state, persists agent thread | After agent completes
For standalone Tool Nodes, only the Execution layer is used — there is no LLM, so schema filtering and response processing are not applicable.
### High-Level Design

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                              AgentNode                                       │
│                                                                              │
│  ┌──────────────┐    ┌───────────────┐    ┌──────────────┐                  │
│  │ beforeAgent   │───▶│ wrapModelCall  │───▶│  afterAgent   │                 │
│  │ (msg setup)   │    │ (prompt resolve)│    │ (thread save) │                 │
│  └──────────────┘    └───────┬───────┘    └──────────────┘                  │
│                              │ (repeats per LLM call)                        │
│                              ▼                                               │
│                     ┌────────────────┐                                       │
│                     │   LLM (ReAct)  │                                       │
│                     │ generates calls │                                       │
│                     └───────┬────────┘                                       │
│                             │                                                │
│                             ▼                                                │
│               ┌─────────────────────────┐                                    │
│               │  processAgentResponse() │                                    │
│               │  (UI output + thread)   │                                    │
│               └─────────────────────────┘                                    │
└──────────────────────────────────────────────────────────────────────────────┘
                              │
              LLM tool_calls  │
                              ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                           ToolManager                                        │
│                                                                              │
│  ┌─────────────────────┐    ┌──────────────────────┐                        │
│  │ convertAgentTool     │───▶│ LangChainToolFactory  │                       │
│  │ ToDefinition()       │    │                        │                       │
│  │ (config → definition)│    │ ┌──────────────────┐  │                       │
│  └─────────────────────┘    │ │getLLMVisibleSchema│  │                       │
│                              │ │(schema filtering) │  │                       │
│                              │ └────────┬─────────┘  │                       │
│                              │          │             │                       │
│                              │          ▼             │                       │
│                              │ ┌──────────────────┐  │                       │
│                              │ │createLangchain    │  │                       │
│                              │ │ToolFunction()     │  │                       │
│                              │ │(execution wrapper)│  │                       │
│                              │ └────────┬─────────┘  │                       │
│                              └──────────┼────────────┘                       │
└─────────────────────────────────────────┼────────────────────────────────────┘
                                          │
                              calls       │
                                          ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                        schema.utils.ts                                       │
│                                                                              │
│  ┌──────────────────┐    ┌──────────────────┐    ┌──────────────────┐       │
│  │ prepareSchemaFor │───▶│drillParamSchema │───▶│     hasType      │       │
│  │ LLM()             │    │  (recurse nested)  │    │ (nullable check) │       │
│  │ (entry point)     │◀───│                    │    │                  │       │
│  └──────────────────┘    └──────────────────┘    └──────────────────┘       │
└──────────────────────────────────────────────────────────────────────────────┘
                                          │
                         filtered schema  │
                                          ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                            BaseTool                                          │
│                                                                              │
│  ┌──────────────────┐    ┌──────────────────┐    ┌──────────────────┐       │
│  │    invoke()       │───▶│resolveParameters()│───▶│    process()     │       │
│  │ (lifecycle mgr)   │    │ (template resolve │    │ (subclass logic) │       │
│  │                   │    │  + deep merge)    │    │                  │       │
│  └──────────────────┘    └──────────────────┘    └────────┬─────────┘       │
│                                                           │                  │
│  Subclasses:                                              │                  │
│  ┌─────────┐ ┌───────────┐ ┌───────────┐ ┌────────────┐ │                  │
│  │ MCPTool │ │HandoffTool│ │MCPResource│ │KnowledgeTool│ │                  │
│  └─────────┘ └───────────┘ └───────────┘ └────────────┘ │                  │
└──────────────────────────────────────────┬───────────────────────────────────┘
                                           │
                           returns         │
                                           ▼
                              ┌─────────────────────────┐
                              │     Return Paths         │
                              │                          │
                              │  string    → simple tool │
                              │  ToolMessage → metadata  │
                              │  Command   → handoff /   │
                              │              state update│
                              └─────────────────────────┘
```

### Component Interaction

**Setup** (once at agent init)

```
AgentNode.initializeReactAgent()
  │
  ├─▶ ToolManager.createAgentTools()
  │     ├── AgentToolConfig → ToolDefinition (convert + embed provider config)
  │     └── LangChainToolFactory.buildLangchainTools()
  │           ├── getLLMVisibleSchema() → prepareSchemaForLLM() (hide static, expose dynamic)
  │           ├── Inject thoughts/reasoning
  │           └── Wrap with coreTool() → LangChain tool
  │
  ├─▶ Create middleware (beforeAgent, wrapModelCall, afterAgent)
  └─▶ createAgent({ model, tools, systemPrompt, middleware })
```

**Execution** (per agent run, ReAct loop)

```
AgentNode.process()
        │
        ├──▶ prepareAgentInput()
        │         └── Load thread history from state
        │
        └──▶ executeAgent() → starts the ReAct loop
                    │
                    ▼
          ┌─────────────────────────┐
          │  [beforeAgent] (once)   │
          │  Build messages from    │
          │  prompt template        │
          └───────────┬─────────────┘
                      │
                      ▼
          ┌─────────────────────────┐◀──────────────────────┐
          │  [wrapModelCall]        │                        │
          │  Resolve system prompt  │                        │
          │  Set modelSettings      │                        │
          │  Send to LLM            │                        │
          └───────────┬─────────────┘                        │
                      │                                      │
                      ▼                                      │
              LLM responds with                              │
              AIMessage                                      │
                      │                                      │
                ┌─────┴──────┐                               │
                │            │                               │
           Has tool_calls?   No tool_calls                   │
                │            │                               │
                ▼            ▼                               │
         Execute tool   Agent done ──▶ [afterAgent]          │
                │                       Save thread          │
                ▼                                            │
    LangChainToolFactory                                     │
    .createLangchainToolFunction()                           │
                │                                            │
                ├── Parse & sanitize LLM args                │
                ├── Strip thoughts/reasoning                  │
                ├── instantiateToolFromDefinition()           │
                │       (fresh BaseTool instance)             │
                │                                            │
                ├──▶ BaseTool.invoke()                       │
                │         ├── resolveParameters()            │
                │         │     ├── Resolve templates        │
                │         │     └── deepMergeObjects()       │
                │         │         (static + dynamic)       │
                │         │                                  │
                │         └── process()                      │
                │               (MCPTool / HandoffTool /     │
                │                KnowledgeTool / etc.)       │
                │                                            │             
                │                                            │
                └── Return path:                             │
                      │                                      │
                ┌─────┼─────────┬──────────────┐             │
                │     │         │              │             │
            Handoff  Command  ToolMessage    String          │
            (goto)   (state   (metadata)    (simple)         │
                │    update)       │            │            │
                │       │         │            │            │
                ▼       └─────────┴────────────┘            │
            Exits                  │                         │
            agent          Back to LLM ──────────────────────┘
```

**Response**

          ┌─────────────────────────────────┐
          │  Was it a handoff Command?       │
          │  (agentResponse has 'goto')      │
          └──────────┬──────────┬────────────┘
                     │          │
                    Yes         No
                     │          │
                     ▼          ▼
             Return Command   processAgentResponse()
             directly to           │
             LangGraph             ├── Extract text from last AIMessage
             (skip response        |
              processing)          ├── Extract toolCalls from AIMessage (call.args)
                                   ├── Extract toolResults from agent state
                                   ├── Detect returnDirect interrupt
                                   └── Return { text, toolCalls, toolResults,
                                                 threadUpdate, agentFinalVariables }
                                              │
                                              ▼
                                          UI display




**Standalone Tool Node** (no LLM, separate path)

```
Graph routing reaches ToolNode
        │
        ▼
ToolNode.process()
        │
        ├── getToolDefinition() from node config
        │
        ├── ToolManager.createToolInstance()
        │       (fresh BaseTool instance)
        │
        └──▶ BaseTool.invoke(state, {}, config)
                    │           ▲
                    │      empty {} = no LLM args
                    │      (all params are static)
                    │
                    ├── resolveParameters()
                    │     └── Resolve templates only, no merge needed
                    │
                    └── process() → subclass logic
                          │
                          ▼
                    Return ProcessOutput
                    (or throw Command for handoff)
```

| Decision | Rationale |
| --- | --- |
| LLM sees only dynamic fields; tools receive everything | Saves tokens, prevents LLM from overriding user-configured values |
| Recursive schema filtering for nested objects | Flat filtering hides entire objects; recursion exposes only dynamic sub-fields at any depth |
| Fresh tool instances per invocation | Prevents parameter leakage between concurrent workflows |
| Three return paths (string / ToolMessage / Command) | Different tools need simple output, metadata, or state updates + routing |
| Handoff bypasses response processing | Command exits agent immediately; HandoffTool persists thread and state explicitly |
| State explicitly preserved in every Command | `createAgent` replaces state fields — without spreading existing values, they'd be wiped |


## 4. Detailed Design

### 4.1 Tool Definition Conversion

**File:** `ToolManager.ts`

Converts UI-facing `AgentToolConfig` to runtime `ToolDefinition`. Generates unique IDs (`{agentId}_{toolName}`), embeds MCP provider config to avoid runtime lookups, and passes through all tool configuration.

| Config passed through | Purpose |
| --- | --- |
| `parameters` | Static values set by user in UI |
| `schema` | JSON Schema defining all tool params |
| `captureRawResponse` | Persist tool output to state for cross-tool references |
| `llmProjection` | Filter tool output before sending to LLM |
| `returnDirect` | Interrupt workflow after tool execution |
| `includeThoughts` | Inject thoughts/reasoning params into schema |
| `variableUpdates` | Declarative state updates after execution |

**Tool type registry** — adding a new tool type requires one map entry and one class:

| Type | Class |
| --- | --- |
| `mcp` | MCPTool |
| `mcp-resource` | MCPResource |
| `mcp-prompt` | MCPPrompt |
| `handoff` | HandoffTool |
| `quantummcp` | QuantumMCPTool |

---

### 4.2 Schema Filtering

**File:** `schema.utils.ts`

| Function | What it does |
| --- | --- |
| `prepareSchemaForLLM()` | Entry point. Expose dynamic fields, hide static primitives, delegate partial objects to `drillParamSchema()` |
| `drillParamSchema()` | Recursively processes nested objects (`properties`) and arrays (`items`). Returns filtered node or `null` if fully static |
| `hasType()` | Handles nullable types like `["object", "null"]` that a simple string check would miss |

Includes prototype pollution protection — skips `__proto__`, `constructor`, `prototype`.

`getLLMVisibleSchema()` in `LangChainToolFactory.ts` calls `prepareSchemaForLLM()` once, applies provider-specific adjustments (Gemini: strip date formats, handle nullable types), and injects `thoughts`/`reasoning` if enabled.

---
### 4.3 Schema Building & Tool Wrapping
**File:** `LangChainToolFactory.ts`
| Method | What it does |
| --- | --- |
| `getLLMVisibleSchema()` | Calls `prepareSchemaForLLM()` once, applies Gemini adjustments (strip date formats, handle nullable types) |
| `buildLLMSchema()` | Adds `thoughts`/`reasoning` if enabled, sets `additionalProperties` per provider |
| `createLangChainTool()` | Wraps with `coreTool()` using filtered schema. Sets alias as LLM name, attaches metadata |
| `createHandoffTool()` | Same wrapping but does NOT set `returnDirect` (Command handles exit) |
---

### 4.4 Tool Execution

**Files:** `LangChainToolFactory.ts` → `BaseTool.ts`

When the LLM calls a tool, `createLangchainToolFunction()` executes:

| Step | What happens |
| --- | --- |
| 1. Parse & sanitize | Handle input, strip `thoughts`/`reasoning` |
| 2. Fresh instance | `instantiateToolFromDefinition()` — no singletons |
| 3. Invoke | `BaseTool.invoke()` → `resolveParameters()` → `process()` |
| 4. Debug | Store resolvedParameters in _toolDebug for Langfuse trace visibility |
| 5. Return | Based on config (see below) |

**Return path decision:**

| Condition | Returns | Use case |
| --- | --- | --- |
| Handoff tool | Command with `goto` | Routes to another agent |
| `captureRawResponse` / `variableUpdates` / `toolType` | Command with state updates + ToolMessage | Persist data, update variables |
| Special `toolType` only (e.g., partial-view) | ToolMessage with metadata | Middleware detection |
| None of the above | Plain string | Simple tool output |

---

`applyLLMProjection()` filters tool responses before sending back to the LLM using include/exclude rules — reduces token usage while full response is still captured for state and UI.

---

### 4.5 Parameter Resolution

**File:** `BaseTool.ts`

| Step | What happens |
| --- | --- |
| 1. Read | `config.parameters` (static values from UI) |
| 2. Resolve | Template variables via `StateManager` (`{{system.tenantId}}` → actual value) |
| 3. Deep merge | `deepMergeObjects(resolvedStatic, llmArguments)` — LLM overrides at every nesting level |

If no config params exist, LLM arguments are returned as-is.

---

### 4.6 Handoff Tool

**File:** `HandoffTool.ts`

Creates `Command({ goto: targetNodeId, graph: Command.PARENT })` for routing to another agent.

| Responsibility | How |
| --- | --- |
| Tool call ID | From `runtimeConfig.metadata`; fallback: infer from state. Required for Azure OpenAI. |
| State persistence | Spreads existing variables + applies `variableUpdates`. Stores resolved params in `toolResults`. |
| Thread persistence | Saves agent conversation in Command (since `afterAgent` is bypassed) |
| UI metadata | Attaches `node_metadata` to ToolMessage. Stores resolvedParameters in _toolDebug. |

---

### 4.7 Agent Middleware

**File:** `AgentNode.ts`

| Hook | Runs | What it does |
| --- | --- | --- |
| `beforeAgent` | Once | Load thread history, resolve prompt template, build message array |
| `wrapModelCall` | Every LLM call | Resolve system prompt, set `parallel_tool_calls`, set `strict: false` |
| `afterAgent` | Once | Filter system messages, save conversation to per-agent thread |

---

### 4.8 Response Processing

**File:** `AgentNode.ts`

| Step | What it extracts |
| --- | --- |
| Text | From last AIMessage content |
| Tool calls | From AIMessage tool_calls (LLM-generated parameters via call.args)|
| Tool results | From `response.variables.nodes[nodeId].toolResults` |
| Interrupt | Detected if last message is ToolMessage from a `returnDirect` tool |
| Variables | `agentFinalVariables` propagated to parent graph |

Handoff Commands bypass this entirely — returned directly before `processAgentResponse` is called. Standalone Tool Nodes (`ToolNode.ts`) call `BaseTool.invoke(state, {}, config)` with empty LLM arguments — all params static, no schema filtering or response processing needed. Tool output is always returned directly in ProcessOutput.data.result — visible in UI and state without captureRawResponse.

## 5. Implementation
## Implementation

### Components

| File | Layer | Purpose |
| --- | --- | --- |
| `AgentNode.ts` | Orchestration | Agent lifecycle, middleware (beforeAgent, wrapModelCall, afterAgent), response processing, handoff Command handling |
| `ToolManager.ts` | Definition | Tool type registry, `AgentToolConfig` → `ToolDefinition` conversion, validation, factory orchestration |
| `LangChainToolFactory.ts` | Schema / Execution | Schema building (`getLLMVisibleSchema`), LangChain tool wrapping (`coreTool`), tool execution function, output projection |
| `schema.utils.ts` | Schema | Recursive schema filtering (`prepareSchemaForLLM`, `drillParamSchema`, `hasType`). Shared across factory and BaseTool paths. |
| `BaseTool.ts` | Execution | Abstract base class. Parameter resolution (`resolveParameters`, `deepMergeObjects`), invoke lifecycle, variable updates, PV event dispatching |
| `HandoffTool.ts` | Execution | Handoff routing via `Command`. Tool call ID management, thread persistence, state propagation |
| `MCPTool.ts` | Execution | MCP server tool invocation |
| `MCPResource.ts` | Execution | MCP resource fetching |
| `MCPPrompt.ts` | Execution | MCP prompt execution |
| `QuantumMCPTool.ts` | Execution | Quantum MCP endpoint invocation |
| `KnowledgeTool.ts` | Execution | Knowledge base queries (class exists but not yet registered in tool type registry)
| `ToolNode.ts` | Orchestration | Standalone tool execution node (no LLM). Calls `BaseTool.invoke()` with empty LLM args |
| `tool.types.ts` | Types | Core interfaces: `ToolDefinition`, `ToolResult`, `ExecutionContext`, `HandoffToolSettings` |
| `SchemaParametersSection.tsx` | Frontend | Tool parameter form. Flattens/unflattens nested params, handles auto/manual toggle |
| `SchemaParameterField.tsx` | Frontend | Individual parameter field. Auto-state persistence, type display (`array<object>`), nested rendering |
| `name.utils.ts` | Utility | `normalizeToolName()` — ensures tool names are valid LangChain identifiers |
| `projection.utils.ts` | Utility | `applyProjection()` — implements include/exclude rules for `llmProjection` output filtering |
| `LangChainMCPClient.ts` | Connectivity | MCP connection pooling. Fresh tool instances reuse shared connections. |
| `knowledge-tool-utils.ts` | Utility | Generates file-type-specific schemas for KnowledgeTool |

### Dependencies

| Package | Used by | Purpose |
| --- | --- | --- |
| `@langchain/core` | Factory, BaseTool, HandoffTool, AgentNode | Core primitives: `tool()`, `ToolMessage`, `AIMessage`, `RunnableConfig`, `ToolRuntime`, `dispatchCustomEvent` |
| `@langchain/langgraph` | Factory, HandoffTool, AgentNode | Graph runtime: `Command`, `LangGraphRunnableConfig`, `getCurrentTaskInput` |
| `langchain` | AgentNode | Agent creation: `createAgent`, `createMiddleware` |
| `@langchain/mcp-adapters` | LangChainMCPClient | MCP server connectivity: `MultiServerMCPClient` |
| `@nestjs/common` | Factory, BaseTool, ToolManager | Logging: `Logger` |


## 6. Quality Attributes

### Performance

| Concern | Mitigation |
| --- | --- |
| Fresh tool instance per invocation | Lightweight object construction; MCP connections pooled separately |
| Recursive schema filtering | Runs once at agent init, not per tool call. Depth bounded by schema. |
| `deepMergeObjects` on every tool call | Typical nesting is 2-3 levels deep — negligible cost |

### Reliability

| Concern | Mitigation |
| --- | --- |
| Template resolution fails (`{{flow.missing}}`) | Catches per-key, falls back to raw value, logs warning |
| Tool execution throws | Returns `{ ok: false }`, LangGraph passes error to LLM for retry |
| Handoff bypasses `afterAgent` | HandoffTool persists thread and state explicitly in its Command |
| State wiped by Command updates | Every Command spreads existing variables before applying updates |
| LLM enters infinite tool-calling loop | `recursionLimit` from agent config caps ReAct iterations |

### Security

| Concern | Mitigation |
| --- | --- |
| Prototype pollution via parameter keys | `prepareSchemaForLLM` and `deepMergeObjects` skip `__proto__`, `constructor`, `prototype` |
| Sensitive data exposed to LLM | Static params hidden from LLM schema — only dynamic fields visible |

### Observability

| Concern | Mitigation |
| --- | --- |
| What params did the tool actually receive? | _toolDebug in Command updates — visible per-tool in Langfuse traces. Doesn't enter LLM conversation. |
| Static params polluting Langfuse traces | Never enter `AIMessage.tool_calls[].args` — traces reflect actual LLM behavior |
| Tool execution lifecycle | `BaseTool.invoke()` logs start/success/error with tool name and duration |

### Scalability

| Concern | Mitigation |
| --- | --- |
| Concurrent workflows sharing state | Fresh instances per invocation — no shared mutable state |
| MCP connection exhaustion | Connections pooled via `LangChainMCPClient`, independent of tool instances |
| Thread history grows unbounded | System messages filtered out in `afterAgent` — only meaningful messages persisted |

## 7. Usage Examples

### Partially static nested object

```
Schema:        result_data: { name (string), age (number), emp (number), type (string) }
Static params: result_data: { emp: 50, type: "mnc" }
LLM sees:      result_data: { name, age }              ← only dynamic sub-fields
LLM generates: { result_data: { name: "Google", age: 26 } }
Tool receives: { result_data: { name: "Google", age: 26, emp: 50, type: "mnc" } }
```

### Fully static parameter

```
Schema:        query (string)
Static params: query: "info about {{flow.company_name}}"
LLM sees:      (query hidden from schema)
At runtime:    flow.company_name = "Microsoft"
Tool receives: { query: "info about Microsoft" }
```

### All dynamic parameters

```
Schema:        query (string), count (number)
Static params: (none)
LLM sees:      { query, count, thoughts, reasoning }
LLM generates: { query: "AI news", count: 5, thoughts: "...", reasoning: "..." }
Tool receives: { query: "AI news", count: 5 }           ← thoughts/reasoning stripped
```

### Cross-tool reference via captureRawResponse

```
Tool A config: captureRawResponse: true
Tool A runs:   returns { summary: "Google is a tech company", revenue: "..." }
Persisted to:  variables.nodes.agent_1.toolResults.toolA

Tool B config: static param data = "{{nodes.agent_1.toolResults.toolA.summary}}"
Tool B gets:   { data: "Google is a tech company" }
```

### Handoff tool

```
Schema:        reason (string), context (string)
Static params: (none — all dynamic)
LLM generates: { reason: "needs billing info", context: "user asked about invoice" }
HandoffTool:   Command({ goto: "billing_agent" })
               ToolMessage with node_metadata
               State: handoffActive: true, toolResults: { handoff: { from, to, reason, context } }
```

### Standalone Tool Node (no LLM)

```
Schema:        query (string), tenantId (string)
Static params: query: "monthly report", tenantId: "{{system.tenantId}}"
LLM args:      {} (empty — no LLM involved)
At runtime:    system.tenantId = "tenant_abc"
Tool receives: { query: "monthly report", tenantId: "tenant_abc" }
```

## 8. Consequences

### Positive

- LLM only generates what it needs — saves tokens and prevents overriding user-configured values
- Nested objects work at arbitrary depth with recursive filtering and deep merge
- No cross-workflow parameter leakage — fresh instances per invocation
- Langfuse traces stay clean — static params never enter LLM conversation history. Resolved params visible per-tool via _toolDebug in Langfuse.
- Same execution path for all tool types — consistent behavior across MCP, handoff, knowledge tools
- Template variables enable dynamic parameter values that resolve at runtime from workflow state
- Cross-tool data passing via `captureRawResponse` + template references

### Negative

- `createAgent` uses replacement for state fields — every new Command must spread existing variables or data is silently lost. Already implemented throughout, but must be followed in future code.
- Three middleware hooks add complexity — developers must understand which hook runs when and what it affects

### Neutral

- Handoff tools bypass `processAgentResponse()` — data travels in the Command instead. Future logic added to `processAgentResponse` won't apply to handoff flows.
- Fresh tool instances use slightly more memory per invocation, but MCP connections are pooled separately so the real cost is minimal
- `thoughts`/`reasoning` injected and stripped per call — small overhead, enables AG-UI streaming without polluting tool inputs
- Provider-specific adjustments (Gemini date formats, nullable types) add conditional branches but are isolated in `getLLMVisibleSchema`
- Schema filtering runs once at init, not per call — no runtime impact but adds initialization time

## 9. Risks and Mitigation

| Risk | Impact | Likelihood | Mitigation |
| --- | --- | --- | --- |
| Template resolution fails at runtime | Tool executes with unresolved `{{variable}}` string instead of actual value | Medium | `resolveParameters` catches per-key, falls back to raw value, and logs a warning. Tool still executes — developer must check logs for resolution failures. |


## 10. Constraints & Assumptions

### Constraints

- `createAgent` does not support custom reducers — every Command must explicitly spread existing state variables to avoid data loss.
- Azure OpenAI requires `ToolMessage.tool_call_id` to exactly match `AIMessage.tool_calls[].id`.
- Gemini and OpenAI have conflicting schema requirements (`additionalProperties`, nullable types, date formats) — handled per-provider in `getLLMVisibleSchema`.
- Handoff tools cannot use `returnDirect: true` — causes a re-entry bug.

### Assumptions

- Fields set to "Let agent decide" are absent from `config.parameters` — the frontend deletes them.
- The LLM respects `additionalProperties: false` and does not generate extra fields — no server-side enforcement exists.
- MCP connections are pooled externally — tool instances do not manage connections.
- Template variable paths reference valid state keys — invalid paths fall back to raw string with a warning.
- 
## 11. Scope and Non-Goals

### In Scope

- Full tool calling lifecycle: definition, schema filtering, execution, and response processing
- All tool types (MCP, handoff, knowledge, Quantum MCP, MCP-resource, MCP-prompt)
- Recursive schema filtering for nested objects, arrays, and nullable types
- Static + dynamic parameter merging with template variable resolution
- Provider-specific schema handling (Gemini, OpenAI, Azure)
- Tool output control (`llmProjection`, `captureRawResponse`, `variableUpdates`)
- Handoff routing with state and thread persistence
- Agent middleware and thread management
- Resolved parameter visibility via _toolDebug (Langfuse)

### Out of Scope (Deferred)

- Server-side stripping of LLM-hallucinated fields
- Nested array item editing in frontend UI (uses JSON editor)
- LLM-generated parameter validation (type/range checking)
- Runtime schema modification after agent initialization
- MCP connection pooling, workflow graph routing, structured output compliance (managed by other modules)
