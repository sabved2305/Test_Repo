# ADR-007: Invoke V2 — A2A Message Response for Agents-as-APIs

| Field             | Value      |
| ----------------- | ---------- |
| **Status**        | Accepted   |
| **Author(s)**     | [Name]     |
| **Created**       | 2026-04-21 |
| **Last Updated**  | 2026-04-21 |
| **Approvers**     | [TBD]      |
| **Supersedes**    | N/A        |
| **Superseded by** | N/A        |

---

## Abstract

The existing `/invoke` endpoint (V0/V1) returns a Leo-specific envelope (`GepReturnDto`) that drops widgets, has no ordering between text and widgets, leaks internal state, and does not validate inputs. It works as a chat-UI fallback, but not when an agent is called as an API — by another service, an automation, or a third-party integration.

**Invoke V2** replaces it with `POST /v2/workflow-engine/:id/invoke`, which returns an **A2A Message** (Google's open Agent-to-Agent protocol): an ordered `parts[]` of text and widgets, plus a clean `metadata` block.

> [!TIP]
> **In one sentence** — V2 runs the same streaming pipeline as `/stream`, and a subscriber (`A2AStreamCollector`) assembles the AG-UI event stream into an A2A Message. Zero new state variables. Zero schema changes on existing workflows.

---

## 1. Why this ADR exists

### 1.1 Two audiences, two transports

```mermaid
flowchart LR
    subgraph CHAT["Interactive Chat"]
        UI[Chat UI]
        CP[Copilots]
        EA[Embedded Assistants]
    end

    subgraph API["Agents as APIs"]
        BS[Backend Services]
        AU[Automations]
        A2[Agent-to-Agent]
        TP[3rd Party]
    end

    CHAT --> S["/stream<br/>SSE · AG-UI"]
    API  --> I["/v2/.../invoke<br/>A2A Message"]

    S --> R[Agent Runtime]
    I --> R

    classDef blue  fill:#e0f2fe,stroke:#0284c7,color:#0c4a6e
    classDef green fill:#dcfce7,stroke:#16a34a,color:#14532d
    classDef grey  fill:#f3f4f6,stroke:#6b7280,color:#111827
    class S blue
    class I green
    class R grey
```

The streaming path has long been feature-complete — tokens, tool calls, step events, widgets, HITL interrupts — all as AG-UI events. The invoke path was bolted on later as a simpler `compiledGraph.invoke()` call. It works for chat-UI fallbacks, but falls short when the caller *is* an API consumer.

### 1.2 What V0/V1 invoke returns today

```json
{
  "isSuccess": true,
  "returnValue": {
    "messages": [ "..." ],
    "selectedAgentId": "...",
    "state": { "flow": {}, "system": {}, "nodes": {} }
  }
}
```

### 1.3 Five problems with that

- **Widgets are lost.** PV widgets are dispatched as `CUSTOM` SSE events (`PV_PRE_RENDER` / `PV_TEMPLATE` / `PV_POST_RENDER`). The blocking `invoke()` path never sees them.
- **No content ordering.** If an agent produces `text → widget → text`, the response can't represent that sequence.
- **Proprietary format.** Every consumer must learn `GepReturnDto`. No open standard.
- **State leaks.** `flow` / `system` / `nodes` are exposed alongside the answer, with no separation between "the answer" and "internals".
- **No input validation.** A malformed request starts executing anyway; the LLM sees garbage and either hallucinates or crashes.

### 1.4 Why now

Product direction is moving toward **agents as APIs** — callable by other services, automations, agent-to-agent orchestrators, and external integrations. Those consumers want a structured, ordered, standards-compliant response. V1 does not deliver that.

### 1.5 Goals

| ID  | Goal                                                                           | Priority |
| --- | ------------------------------------------------------------------------------ | -------- |
| G1  | Return A2A `Message { message_id, role, parts[], metadata }`                   | P0       |
| G2  | Preserve order of text and widgets as the agent produced them                  | P0       |
| G3  | Include widgets (PV, A2UI) as first-class parts with `media_type`              | P0       |
| G4  | Clean separation — `parts[]` is the answer, `metadata` is side-channel         | P0       |
| G5  | No duplicated logic between stream and invoke                                  | P0       |
| G6  | HITL interrupts surface on invoke, not just stream                             | P0       |
| G7  | Validate inputs before execution starts                                        | P1       |
| G8  | Graceful fallback for workflows that do not emit events                        | P1       |
| G9  | `/stream` stays untouched                                                      | P0       |
| G10 | Works on existing workflows with zero schema changes                           | P1       |

---

## 2. The decision

> [!NOTE]
> **Chosen approach — stream-and-collect.**
> Invoke V2 runs the same streaming pipeline as `/stream`. A subscriber (`A2AStreamCollector`) turns the AG-UI event stream into an A2A Message.
>
> No new state variables. No output-node templating. No per-node wiring.
> **If a workflow streams correctly, it invokes correctly.**

### 2.1 High-level architecture

```mermaid
flowchart TB
    CLIENT[Client / Consumer]

    subgraph RUNTIME["Agent Runtime"]
        direction TB
        subgraph CTRL["WorkflowEngineController"]
            C1["stream()"]
            C2["invokeV2() @Version('2')"]
            C3["invoke() V0/V1"]
        end

        subgraph SVC["WorkflowEngineService"]
            S1["stream()"]
            S2["execute()"]
            S3["invoke()"]
        end

        subgraph EMIT["Shared emission layer"]
            ADAPT[LangGraphAgentAdapter<br/>.streamExecutionObservable]
            GRAPH[compiledGraph]
        end

        SSE[SSE serializer]
        COLL[A2AStreamCollector]
        BLOCK["compiledGraph.invoke"]
    end

    CLIENT --> C1 & C2 & C3
    C1 --> S1 --> ADAPT
    C2 --> S2 --> ADAPT
    C3 --> S3 --> BLOCK --> GRAPH
    ADAPT --> GRAPH
    ADAPT --> SSE --> CLIENT
    ADAPT --> COLL --> CLIENT
    BLOCK --> CLIENT

    classDef blue   fill:#e0f2fe,stroke:#0284c7,color:#0c4a6e
    classDef green  fill:#dcfce7,stroke:#16a34a,color:#14532d
    classDef red    fill:#fee2e2,stroke:#dc2626,color:#7f1d1d
    classDef shared fill:#f5f3ff,stroke:#7c3aed,color:#4c1d95

    class C1,S1,SSE blue
    class C2,S2,COLL green
    class C3,S3,BLOCK red
    class ADAPT,GRAPH,EMIT shared
```

The critical property: **`/stream` and `/v2/.../invoke` share the same `streamExecutionObservable(...)` emitter.** Only the subscriber differs.

### 2.2 Call flow

```mermaid
sequenceDiagram
    participant Client
    participant Ctrl as invokeV2()
    participant Svc as execute()
    participant Adapter as streamExecutionObservable()
    participant Coll as A2AStreamCollector

    Client->>Ctrl: POST /v2/.../invoke
    Ctrl->>Svc: execute(id, dto)
    Svc->>Adapter: stream (streamMode: 'verbose')

    loop AG-UI events
        Adapter-->>Coll: TEXT_MESSAGE_* · CUSTOM · ...
        Note over Coll: accumulates parts[] in order
    end

    Adapter-->>Coll: RUN_FINISHED (output)
    Coll-->>Svc: A2AMessage
    Svc-->>Ctrl: A2AMessage
    Ctrl-->>Client: 200 OK · A2A JSON
```

### 2.3 Why not the alternatives

```mermaid
flowchart TD
    Q{How to build parts[]?}

    Q --> A[A. Stream-and-collect<br/>Reuse stream events]
    Q --> B[B. Template parsing<br/>finalAnswer with placeholders]
    Q --> C[C. finalAnswer state accumulator<br/>Appended by every emitter node]

    A --> OK((Shipped))
    B --> NB[Parallel code path<br/>Silent drops on missing vars<br/>Fails G5, G10]
    C --> NC[Modifies every emitter node<br/>Duplicates thread.messages<br/>State bloat over long sessions]

    NB --> REJ((Rejected))
    NC --> REMOVED((Implemented then removed))

    classDef good fill:#dcfce7,stroke:#16a34a,color:#14532d
    classDef bad  fill:#fee2e2,stroke:#dc2626,color:#7f1d1d
    class A,OK good
    class B,C,NB,NC,REJ,REMOVED bad
```

> [!IMPORTANT]
> Three of the source documents (`a2a-invoke-response-proposal.md`, `invoke-api-approaches.md`, `invoke-v2-overview.md`) describe **Option C** as the winning approach. It was implemented and later **removed**. The header comment in `A2AStreamCollector.ts` preserves that history.

---

## 3. What consumers see

### 3.1 Happy path

**Request**

```http
POST /v2/workflow-engine/wf-abc-123/invoke
Content-Type: application/json

{
  "bpc": "BPC001",
  "environment": "DEV",
  "version": "1.0",
  "interface": {
    "inputs": { "message": "Show me my recent orders" }
  },
  "options": { "sessionId": "session-001" }
}
```

**Response**

```json
{
  "message_id": "msg_run_1709024400_abc123",
  "role": "agent",
  "parts": [
    { "text": "Here are your recent orders:" },
    {
      "data": {
        "preText": "Looking up order...",
        "template": "<OrderTracker orderId='12345' />",
        "postText": "Order found."
      },
      "media_type": "pv"
    },
    { "text": "Let me know if you need anything else." }
  ],
  "metadata": {
    "status": "COMPLETED",
    "runId": "run_1709024400_abc123",
    "sessionId": "session-001",
    "output": {
      "selectedAgentId": "wf-abc-123",
      "selectedAgentName": "Order Agent"
    }
  }
}
```

The client renders `parts` top to bottom. **`text → widget → text`, exactly as the agent produced them.**

### 3.2 Response anatomy

```mermaid
flowchart LR
    MSG[A2A Message]

    MSG --> MID[message_id]
    MSG --> ROLE["role: 'agent'"]
    MSG --> PARTS["parts · ordered"]
    MSG --> META[metadata]

    PARTS --> PT["{ text }"]
    PARTS --> PV["{ data, media_type: 'pv' }"]
    PARTS --> AU["{ data, media_type: 'a2ui' }"]

    META --> ST["status · COMPLETED / FAILED / INTERRUPTED"]
    META --> RID[runId]
    META --> SID[sessionId]
    META --> OUT["output · non-reserved keys"]

    classDef root fill:#f5f3ff,stroke:#7c3aed,color:#4c1d95
    classDef parts fill:#dcfce7,stroke:#16a34a,color:#14532d
    classDef meta fill:#e0f2fe,stroke:#0284c7,color:#0c4a6e
    class MSG root
    class PT,PV,AU,PARTS parts
    class ST,RID,SID,OUT,META meta
```

### 3.3 HITL interrupt

```json
{
  "message_id": "msg_run_xyz",
  "role": "agent",
  "parts": [
    { "text": "Please approve this refund before I continue." }
  ],
  "metadata": {
    "status": "INTERRUPTED",
    "runId": "run_xyz",
    "sessionId": "session-001",
    "output": {
      "interrupt": {
        "checkpointId": "...",
        "nodeId": "approval-node",
        "payload": { "message": "Please approve this refund before I continue." }
      }
    }
  }
}
```

Resuming: the consumer calls `/v2/.../invoke` again with `options.resume.metadata.checkpointId`. The service replays events from the checkpoint and returns another A2A Message (completed, interrupted again, or failed).

### 3.4 Run lifecycle

```mermaid
stateDiagram-v2
    [*] --> Running
    Running --> Completed: RUN_FINISHED
    Running --> Interrupted: CUSTOM / on_interrupt
    Running --> Failed: RUN_ERROR · validation error · stream error

    Interrupted --> Running: resume(checkpointId)

    Completed --> [*]
    Failed --> [*]
```

### 3.5 Error

```json
{
  "message_id": "msg_error_1709024400",
  "role": "agent",
  "parts": [{ "text": "Input validation failed: message: Required" }],
  "metadata": {
    "status": "FAILED",
    "errorCode": "EXECUTION_ERROR",
    "runId": "run_error_1709024400",
    "sessionId": "session-001"
  }
}
```

> Errors always look like A2A Messages — same shape, different `status`.

---

## 4. How it works

### 4.1 The `A2AMessage` type

Defined in `services/libs/shared/src/dtos/a2a/a2a.types.ts`.

```typescript
export interface A2APart {
  text?: string;
  data?: any;
  media_type?: string;   // "pv" | "a2ui" | future
  metadata?: Record<string, any>;
}

export interface A2AMessage {
  message_id: string;
  role: 'user' | 'agent';
  parts: A2APart[];
  metadata?: Record<string, any>;
}
```

Three part kinds show up in V2 responses:

- **Text** — `{ text: "..." }` — emitted when an agent or LLM node produces natural language.
- **PV widget** — `{ data: { preText, template, postText }, media_type: "pv" }` — emitted when a Partial-View tool runs.
- **A2UI widget** — `{ data: { surfaceId, rendererType, payload, ... }, media_type: "a2ui" }` — emitted when an A2UI-capable tool runs.

### 4.2 The collector — `A2AStreamCollector`

Located at `services/libs/agentic-engine/src/workflow-execution/A2AStreamCollector.ts`. One public method:

```typescript
A2AStreamCollector.collect(stream$, runId, sessionId): Promise<A2AMessage>
```

**State held while the stream runs**

- `parts: A2APart[]` — the ordered output being built.
- `textBuffer: string` — in-flight text between `TEXT_MESSAGE_START` and `TEXT_MESSAGE_END`.
- `pvBuffers: Map<toolCallId, { preText, template, postText }>` — in-flight PV widgets, keyed per tool call so concurrent tools do not mix.
- `hitlInterrupt: object | null` — latest `on_interrupt` payload, if any.

**Event → action**

```mermaid
flowchart LR
    E1[TEXT_MESSAGE_CONTENT] --> B1[(textBuffer)]
    E2[TEXT_MESSAGE_END] -.flush.-> PT[text part]
    B1 -.-> E2

    E3[CUSTOM PV_PRE_RENDER] --> B2[(pvBuffers)]
    E4[CUSTOM PV_TEMPLATE] --> B2
    E5[CUSTOM PV_POST_RENDER] -.flush text first.-> PT
    E5 -.then push.-> PV[PV widget part]

    E6[CUSTOM A2UI_MESSAGE] -.flush text first.-> PT
    E6 -.then push.-> AU[A2UI widget part]

    E7[CUSTOM on_interrupt] --> HIT[(hitlInterrupt)]

    E8[RUN_FINISHED] --> MD[metadata]
    E9[RUN_ERROR] --> FAIL["FAILED message · early resolve"]

    PT & PV & AU --> PARTS[(parts[])]

    classDef source fill:#fef3c7,stroke:#d97706,color:#78350f
    classDef buffer fill:#e0f2fe,stroke:#0284c7,color:#0c4a6e
    classDef output fill:#dcfce7,stroke:#16a34a,color:#14532d
    classDef fail   fill:#fee2e2,stroke:#dc2626,color:#7f1d1d

    class E1,E2,E3,E4,E5,E6,E7,E8,E9 source
    class B1,B2,HIT buffer
    class PT,PV,AU,PARTS,MD output
    class FAIL fail
```

> [!NOTE]
> **Why text is flushed before a widget push** — order preservation. If the LLM says *"here you go:"* and then a tool fires, the text must land in `parts` before the widget.

**On stream `complete`**

1. Flush any leftover `textBuffer`.
2. Build `metadata` from `RUN_FINISHED.result.output` (strip the reserved `messages` key).
3. If an interrupt was captured — set `metadata.status = 'INTERRUPTED'`, nest the payload under `metadata.output.interrupt`, push a text part with the interrupt message.
4. If `parts` is still empty — fall back to the last AI message from `output.messages` as a single text part.
5. Resolve `{ message_id: 'msg_' + runId, role: 'agent', parts, metadata }`.

**Ignored events** — `TOOL_CALL_*`, `STEP_*`, `STATE_*`, `REASONING_ENCRYPTED_VALUE`, `RUN_STARTED`, `TEXT_MESSAGE_START` (implicit).

### 4.3 Metadata

Top-level fields the collector always sets:

- `status` — `"COMPLETED"` / `"FAILED"` / `"INTERRUPTED"`
- `errorCode` — string, only on failure
- `runId` — `run_{timestamp}_{random}`
- `sessionId` — from the request, or auto-generated
- `output` — non-reserved output-node keys (omitted if empty)

> The only **reserved key** is `messages` (conversation history). Everything else on the output node — `selectedAgentId`, `selectedAgentName`, user keys like `relatedQuestions` — flows through to `metadata.output`.

### 4.4 Input validation (Start node)

The Start node (`services/libs/agentic-engine/src/nodes/StartNode.ts`) supports a JSON-Schema-flavoured config called `inputValidation`.

```json
{
  "inputValidation": {
    "properties": {
      "message":     { "type": "string", "minLength": 1 },
      "attachments": { "type": "array" },
      "count":       { "type": "number", "minimum": 1, "maximum": 100 }
    },
    "required": ["message"]
  }
}
```

At run start, `StartNode` builds a Zod schema from `properties` and runs `safeParse` on `interface.inputs`. On failure it throws, the graph emits `RUN_ERROR`, and the collector surfaces a `FAILED` A2A Message with a human-readable issue list.

**Supported types and validators**

| Type      | Extra validators                            |
| --------- | ------------------------------------------- |
| `string`  | `minLength`, `maxLength`, `pattern`, `enum` |
| `number`  | `minimum`, `maximum`                        |
| `boolean` | —                                           |
| `array`   | items are `z.any()`                         |
| `object`  | treated as `z.record(z.any())`              |

If `inputValidation` is not configured, validation is skipped — backward-compatible.

### 4.5 End-to-end code — the V2 path

**Controller** (`workflow-engine.controller.ts`)

```typescript
@Post(':id/invoke')
@Version('2')
@HttpCode(HttpStatus.OK)
async invokeV2(
  @Param('id') id: string,
  @Body() request: InvokeApiRequestV1,
): Promise<A2AMessage> {
  try {
    if (!request.options) { /* defaults */ }
    if (!request.options.sessionId) {
      request.options.sessionId = uuidv4();
    }
    const dto = this.dtoAdapter.v1ToInternal(request);
    return await this.workflowEngine.execute(id, dto);
  } catch (error) {
    return A2AResponseTransformer.buildErrorMessage(
      error.message ?? 'An unexpected error occurred',
      `run_error_${Date.now()}`,
      request.options?.sessionId ?? 'unknown',
      'EXECUTION_ERROR',
    );
  }
}
```

**Service** (`workflow-engine.service.ts`)

```typescript
async execute(id: string, invokeRequest: InvokeRequestDto): Promise<A2AMessage> {
  this.request.headers['RuntimeToken'] = invokeRequest.options?.runtimeToken ?? '';
  const headers = this.request.headers as Record<string, string>;

  // HITL resume path omitted for brevity

  const { compiledGraph, processedSchema } = await this.createGraph(id, invokeRequest);
  const langfuseCallback = this.createLangfuseCallback(id, invokeRequest /* ... */);

  const sessionId = invokeRequest.options?.sessionId || 'default';
  const runId = `run_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;

  const adapterRequest = WorkflowExecutionHelper.buildGraphRequest(
    id,
    invokeRequest,
    interfaceInputs,
    headers,
    [langfuseCallback],
    this.buildRuntimeContextForRequest(id, invokeRequest /* ... */),
  );
  adapterRequest.config = { ...adapterRequest.config, streamMode: 'verbose' };

  const stream$ = this.adapter.streamExecutionObservable(compiledGraph, adapterRequest);
  const tapped$ = await this.wrapV2WithSessionCapture(id, invokeRequest, sessionId, stream$);
  return A2AStreamCollector.collect(tapped$, runId, sessionId);
}
```

Three points worth noting:

- **`streamMode: 'verbose'` is forced.** `minimal` mode would drop PV widgets.
- **`wrapV2WithSessionCapture`** taps the stream for session capture; the collector subscribes to the tapped observable.
- **Runtime headers** — `RuntimeToken` and request headers are passed through `buildRuntimeContextForRequest(...)`, so tools that need header-based auth forwarding behave identically to the stream path.

### 4.6 HITL resume

Same endpoint. The caller sets `options.resume.metadata.checkpointId`. The service detects it, calls `handleResume(id, request, headers)` (shared with the stream path), unwraps `{ data, type }` wrappers into raw AG-UI events, and feeds them into `A2AStreamCollector.collect(...)`. Resume responses return the same A2A shape — completed, interrupted again, or failed.

### 4.7 Three versions coexist

| Version      | Endpoint                                | Returns                     |
| ------------ | --------------------------------------- | --------------------------- |
| V0 (default) | `POST /workflow-engine/:id/invoke`      | `GepReturnDto` — unchanged  |
| V1           | `POST /v1/workflow-engine/:id/invoke`   | `GepReturnDto` — unchanged  |
| **V2**       | `POST /v2/workflow-engine/:id/invoke`   | **`A2AMessage`** — this ADR |

V0/V1 stay live until internal consumers migrate. Deprecation will be a separate ADR.

---

## 5. Stream vs V2 invoke at a glance

| Aspect                   | `/stream`                 | `/v2/.../invoke`             |
| ------------------------ | ------------------------- | ---------------------------- |
| Transport                | SSE                       | HTTP JSON                    |
| Execution                | `streamExecutionObservable(...)` | Same, forced to `verbose` |
| Subscriber               | SSE serializer            | `A2AStreamCollector.collect` |
| Real-time events         | Yes                       | No (collected in-memory)     |
| Token streaming          | Yes                       | No (buffered per message)    |
| Widgets (PV, A2UI)       | Yes                       | Yes (in `parts[]`)           |
| HITL interrupts          | Yes                       | Yes (`status: INTERRUPTED`)  |
| Tool / step / state events | Yes                     | No (ignored)                 |
| Langfuse tracing         | Yes                       | Yes                          |
| Session capture          | Yes                       | Yes                          |

> [!TIP]
> Both paths share the same execution and event-emission code. **Anything the stream sees, the collector can see.** That is what makes feature parity free.

---

## 6. What's in the canvas

- **Endpoints panel** (`frontend/src/components/canvas/panel-content/EndpointsPanelContent.tsx`) — published vs development URL toggle, development token, code snippets in multiple languages, response schema and example.
- **Start node** — builders can add typed input fields with the validators in §4.4. `inputValidation` is optional.
- **Output node** — any key a builder adds (apart from the internal `messages`) appears in `metadata.output` of the V2 response.
- **Variables panel** — inspect Variables / Threads / Runtime state during a run.

---

## 7. Risks and trade-offs

| Risk                                                             | Impact               | Mitigation                                                                                   |
| ---------------------------------------------------------------- | -------------------- | -------------------------------------------------------------------------------------------- |
| Invoke runs the full streaming pipeline even without a client    | Minor overhead       | Acceptable — dominated by LLM latency. A separate path would drift from stream.              |
| Nodes that update state without emitting events are invisible    | Silent content loss  | Document in builder guidance; require `TEXT_MESSAGE_*` / `CUSTOM` emission for `parts[]`.     |
| Three invoke versions coexist                                    | Maintenance          | V0/V1 frozen; deprecation tracked separately.                                                 |
| Legacy `A2AResponseTransformer` is dead code in the success path | Reuse risk           | Header comment on `A2AStreamCollector.ts` documents the supersession; cleanup is tech debt.   |
| AG-UI event shapes change underneath us                          | Collector breaks     | Unit tests on the collector with representative event fixtures.                              |
| V0/V1 consumers assume `messages` is in the response             | Migration friction   | Called out in the Endpoints panel and migration notes.                                        |

---

## 8. Scope

### In scope

- `POST /v2/workflow-engine/:id/invoke` on both `core` and `portal-engine`
- `A2AStreamCollector` + `A2AMessage` / `A2APart` types
- HITL resume returning A2A
- Error A2A Messages (controller, execution, stream)
- Canvas Endpoints panel, Start-node validation, Variables panel
- Retaining legacy `A2AResponseTransformer.buildErrorMessage`

### Out of scope

| Feature                                                 | Why not now                                                      |
| ------------------------------------------------------- | ---------------------------------------------------------------- |
| Full A2A Task envelope (history + artifacts + JSON-RPC) | Consumers need only a Message today; Task can wrap Message later |
| Streaming A2A (A2A frames over SSE)                     | Stream already exists via AG-UI; add if consumers demand it      |
| Deprecation of V0/V1 invoke                             | Separate ADR once internal consumers have migrated               |
| AG-UI protocol compliance fixes                         | Separate ADR; unrelated to the V2 invoke contract                |

---

## 9. Files touched

### New

| File                                                                        | Purpose                                  |
| --------------------------------------------------------------------------- | ---------------------------------------- |
| `services/libs/shared/src/dtos/a2a/a2a.types.ts`                            | `A2APart`, `A2AMessage` types            |
| `services/libs/agentic-engine/src/workflow-execution/A2AStreamCollector.ts` | Subscribes to stream, builds A2A Message |

### Modified

| File                                                                                   | Change                                                 |
| -------------------------------------------------------------------------------------- | ------------------------------------------------------ |
| `services/apps/{core,portal-engine}/src/workflow-engine/workflow-engine.controller.ts` | `invokeV2()` route with `@Version('2')`                |
| `services/apps/{core,portal-engine}/src/workflow-engine/workflow-engine.service.ts`    | `execute()` method; HITL resume returns A2A            |
| `services/libs/agentic-engine/src/nodes/StartNode.ts`                                  | Zod-based `inputValidation`                            |
| `services/libs/agentic-engine/src/index.ts`                                            | Re-exports A2A types / collector / transformer         |
| `frontend/src/components/canvas/panel-content/EndpointsPanelContent.tsx`               | Endpoints panel UI                                     |

### Retained as legacy

| File                                                                            | Status                                                         |
| ------------------------------------------------------------------------------- | -------------------------------------------------------------- |
| `services/libs/agentic-engine/src/workflow-execution/A2AResponseTransformer.ts` | Success path is dead code; kept for `buildErrorMessage` helper |

No new runtime dependencies — `zod` is already in the workspace.

---

## 10. Testing strategy

| Test type                      | Coverage                                                                                          |
| ------------------------------ | ------------------------------------------------------------------------------------------------- |
| Unit — `A2AStreamCollector`    | Text-only, `text → widget → text` order, concurrent PV tools, A2UI, HITL, `RUN_ERROR`, fallback   |
| Unit — metadata                | `messages` excluded; custom output keys preserved                                                 |
| Unit — `StartNode` validation  | Required fields, type mismatches, `minLength` / `enum` / `pattern`                                |
| Integration — `invokeV2`       | End-to-end on both apps; includes HITL resume                                                     |
| Integration — error paths      | Controller DTO error, execution `RUN_ERROR`, stream subscription error                            |
| Regression                     | V0/V1 invoke and `/stream` continue to pass existing suites                                       |

---

## 11. Rollout

| Phase | Scope                                                                       | Status           |
| ----- | --------------------------------------------------------------------------- | ---------------- |
| **1** | V2 endpoint live alongside V0/V1; collector in production; Endpoints panel  | Complete         |
| **2** | Internal consumers migrate V0/V1 → V2                                       | In progress      |
| **3** | External / A2A-compatible consumers onboarded                               | Pending          |
| **4** | Deprecate V0/V1 invoke (separate ADR)                                       | Pending          |

**Feature flag** — none. V2 is additive.

**Rollback** — remove the `@Version('2')` decorator in both controllers. V0/V1 are untouched. No state-level rollback required because V2 introduces no state variables.

---

## 12. Consequences

**Positive**

- Open-standard response — any A2A client consumes our agents without custom code
- Content order preserved end-to-end (G2) with zero builder effort (G10)
- PV and A2UI widgets as first-class response citizens (G3)
- Clean metadata separation (G4)
- No code duplication between stream and invoke (G5)
- HITL works on invoke (G6)
- Errors look the same as successes — uniform A2A Messages

**Negative**

- Invoke pays the cost of the full streaming pipeline
- Three invoke versions coexist until deprecation
- Legacy `A2AResponseTransformer` is dead code in the success path
- Only streamed content reaches the response — state-only updates are invisible

**Neutral**

- `/stream` unchanged
- V0/V1 invoke unchanged
- No new runtime dependencies

---

## 13. Related documents

- [ADR-002: Workflow Node](./ADR-002-workflow-node.md)
- [ADR-005: HITL — Human in the Loop](./ADR-005-hitl-human-in-the-loop.md)
- [A2A Protocol Specification](https://a2a-protocol.org/latest/specification/)
- [AG-UI Protocol Documentation](https://docs.ag-ui.com/)

---

<details>
<summary><strong>Appendix A — AG-UI events and how V2 uses them</strong></summary>

| Event                               | Consumed?               |
| ----------------------------------- | ----------------------- |
| `RUN_STARTED`                       | No                      |
| `RUN_FINISHED`                      | Yes — `metadata.output` |
| `RUN_ERROR`                         | Yes — resolves FAILED   |
| `TEXT_MESSAGE_START`                | No (implicit)           |
| `TEXT_MESSAGE_CONTENT`              | Yes — buffer            |
| `TEXT_MESSAGE_END`                  | Yes — flush text part   |
| `TOOL_CALL_START / ARGS / END / RESULT` | No                  |
| `STEP_STARTED` / `STEP_FINISHED`    | No                      |
| `STATE_SNAPSHOT` / `STATE_DELTA`    | No                      |
| `REASONING_ENCRYPTED_VALUE`         | No                      |
| `CUSTOM / PV_PRE_RENDER`            | Yes — open PV buffer    |
| `CUSTOM / PV_TEMPLATE`              | Yes — set PV template   |
| `CUSTOM / PV_POST_RENDER`           | Yes — push PV part      |
| `CUSTOM / A2UI_MESSAGE`             | Yes — push A2UI part    |
| `CUSTOM / on_interrupt`             | Yes — mark INTERRUPTED  |

</details>

<details>
<summary><strong>Appendix B — End-to-end trace</strong></summary>

**Scenario** — *"Show me my recent orders"*

**Events emitted by the shared streaming pipeline**

1. `RUN_STARTED`
2. `TEXT_MESSAGE_START` → `TEXT_MESSAGE_CONTENT("Here are your recent orders:")` → `TEXT_MESSAGE_END`
3. `CUSTOM / PV_PRE_RENDER (toolCallId=t1, delta="Looking up order...")`
4. `CUSTOM / PV_TEMPLATE (toolCallId=t1, delta="<OrderTracker orderId='12345' />")`
5. `CUSTOM / PV_POST_RENDER (toolCallId=t1, delta="Order found.")`
6. `TEXT_MESSAGE_START` → `TEXT_MESSAGE_CONTENT("Let me know if you need anything else.")` → `TEXT_MESSAGE_END`
7. `RUN_FINISHED (result.output = { selectedAgentId, selectedAgentName, messages: [...] })`

**Collector state after each event**

| After event        | `parts` length | Buffers                                                                              |
| ------------------ | -------------- | ------------------------------------------------------------------------------------ |
| 2 — TEXT_END       | 1              | `textBuffer=""`, `pvBuffers={}`                                                      |
| 3 — PV_PRE_RENDER  | 1              | `pvBuffers={ t1: { preText: "Looking...", template: "", postText: "" } }`            |
| 4 — PV_TEMPLATE    | 1              | `pvBuffers={ t1: { preText: "Looking...", template: "<OrderTracker/>", postText: "" } }` |
| 5 — PV_POST_RENDER | 2              | `pvBuffers={}` (flushed into `parts`)                                                |
| 6 — TEXT_END       | 3              | `textBuffer=""`, `pvBuffers={}`                                                      |
| 7 — RUN_FINISHED   | 3              | Metadata built from `result.output`                                                  |

The final A2A Message is the one shown in §3.1.

</details>

---

## Changelog

| Date       | Author  | Change                                                                                                                                                                                                                                                                                                          |
| ---------- | ------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 2026-04-21 | [Name]  | Initial draft. Consolidates `invoke-v2-overview.md`, `a2a-invoke-response-proposal.md`, `invoke-api-approaches.md`, `stream-vs-invoke-analysis.md`. The `finalAnswer` accumulator described in those docs was implemented and later removed; its history is preserved in §2 to avoid re-proposing it. |
