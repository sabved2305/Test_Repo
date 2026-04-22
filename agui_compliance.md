# AG-UI Compliance — Quick Reference

> A one-page summary of [`agui-protocol-compliance.md`](./agui-protocol-compliance.md).
> Read the full doc only if you are fixing one of the issues below or touching the adapter.

**What it is.** An audit of our custom AG-UI adapter (`LangGraphAgentAdapter.ts`) against `@ag-ui/core` v0.0.39 and `@ag-ui/client` v0.0.40.

> [!TIP]
> **TL;DR** — SSE transport, event naming, lifecycle order, and the core event shapes are correct. The adapter has **12 ranked deviations**. Only **four** are worth touching in the near term (1 Critical + 3 High); everything else is cosmetic or deferred.

---

## 1. Verdict at a glance

| Layer | Verdict | Why |
|---|---|---|
| SSE transport (`data: JSON\n\n`) | Correct | Matches SSE spec |
| Event type casing (`UPPERCASE_SNAKE_CASE`) | Correct | — |
| Run lifecycle order (`RUN_STARTED → … → RUN_FINISHED\|RUN_ERROR`) | Correct | — |
| Text-message streaming (`START → CONTENT* → END`, `delta`, `messageId`) | Correct | — |
| Tool-call events (`START → ARGS → END → RESULT`, `toolCallName`, `toolCallId`) | Correct | — |
| Step events (`stepName`, sequence) | Correct | — |
| State events | Partial | `STATE_SNAPSHOT` emits tuple on interrupt; `MESSAGES_SNAPSHOT` never emitted |
| Reasoning events | Wrong | Custom `REASONING_ENCRYPTED_VALUE` has wrong field names; `THINKING_*` unused |
| Lifecycle extensions | Non-spec | Extra fields on `RUN_STARTED` and `RUN_FINISHED` |
| `@ag-ui/client` package | Unused | Declared in `package.json` but frontend writes its own SSE parser |

---

## 2. The four issues that matter

| # | Issue | Severity | Location |
|---|---|---|---|
| 1 | `REASONING_ENCRYPTED_VALUE` — `subtype: 'thought' \| 'reasoning'` (spec: `'message' \| 'tool-call'`), field `value` (spec: `encryptedValue`), stores **plain text**, not encrypted. Event type isn't even in `@ag-ui/core` v0.0.39's `EventType` enum — only in docs. | **Critical** | `LangGraphAgentAdapter.ts:1012-1031`, `ag-ui.types.ts:134` |
| 2 | `@ag-ui/client` declared but never used. No Zod schema validation, no `HttpAgent`/`AbstractAgent`, no automatic reconnection, no `MessagesSnapshot` state tracking. | **High** | `frontend/package.json` |
| 3 | `MESSAGES_SNAPSHOT` never emitted. Spec designates it as the canonical way to sync conversation history; we rely on `STATE_SNAPSHOT` only. | **High** | — (gap) |
| 4 | `STATE_SNAPSHOT` emits LangGraph tuple `[[], capturedInterruptState]` on interrupt, instead of the plain state object. Frontend has to know to unwrap it. | **High** | `LangGraphAgentAdapter.ts:476-483` |

### 2.1 Why these four — severity and impact

- **#1 is critical** because external AG-UI clients receiving this event would fail Zod validation outright (wrong field name and wrong enum values) and because the field is misleadingly named "encrypted" while carrying plaintext.
- **#2 is high** because it means we pay a dependency cost without getting the library's benefits — validation would have caught #1 during development.
- **#3 is high** because spec-compliant consumers expecting `MESSAGES_SNAPSHOT` will never receive it.
- **#4 is high** because it's a shape violation a third-party `@ag-ui/client` would reject.

---

## 3. Medium-severity gap — `THINKING_*` events

The spec defines a full reasoning-streaming protocol that the codebase bypasses entirely:

- `THINKING_START`, `THINKING_END`
- `THINKING_TEXT_MESSAGE_START`, `THINKING_TEXT_MESSAGE_CONTENT`, `THINKING_TEXT_MESSAGE_END`

We use the custom `REASONING_ENCRYPTED_VALUE` instead. Fixing #1 can be done either by correcting that event's fields or by switching to `THINKING_*` for plaintext reasoning streams. The audit lists both as acceptable paths.

---

## 4. Low-severity deviations (safe to defer)

Non-spec cosmetic additions and edge cases. All rank **Low** in the audit.

- `RUN_STARTED` carries non-spec `isResume`.
- `RUN_FINISHED` carries non-spec `outcome`, `interrupt`, `nodeMetadata`. Audit recommends moving these to a separate `CUSTOM { name: 'INTERRUPT_DETECTED', value: ... }` event (Fix 3).
- `STEP_FINISHED` carries non-spec `result`, `metadata`.
- `TOOL_CALL_RESULT` carries non-spec `name` (tool name is already on `TOOL_CALL_START`).
- `STATE_DELTA` is received with the correct shape (`delta: any[]`) but the frontend treats it as a no-op (`RunPanelContent.tsx:2241`) — RFC 6902 patches are never applied; only `STATE_SNAPSHOT` is consumed.
- PV widgets (`PV_PRE_RENDER` / `PV_TEMPLATE` / `PV_POST_RENDER`) emit their own non-spec event types instead of wrapping in the standard `CUSTOM { name, value }` envelope. Standard `RAW` event is not implemented either.
- `streamMode: 'minimal'` silently drops standard events (`TEXT_MESSAGE_*`, `TOOL_CALL_*`, `STEP_*`, `STATE_*`) unless the LangGraph node carries `STREAM_RESPONSE`. Spec-compliant consumers would see an incomplete stream.
- SSE `event:` field is ignored by the frontend parser. Works because AG-UI puts `type` inside the JSON, but it's non-standard SSE behaviour.

---

## 5. Known-good list (don't worry about these)

The audit explicitly validates these twelve points as correct:

- `UPPERCASE_SNAKE_CASE` on every event type string.
- `TEXT_MESSAGE_CONTENT` uses `delta` (not `content`).
- `TOOL_CALL_START` uses `toolCallName` (not `toolName`).
- `STATE_SNAPSHOT` uses `snapshot` (not `state`) — normal path only; interrupt path is issue #4.
- `RUN_ERROR` uses `message` (not `error`).
- All events carry `messageId` / `toolCallId` (never a bare `id`).
- SSE transport is `data: JSON\n\n`.
- Lifecycle: `RUN_STARTED` first, `RUN_FINISHED`/`RUN_ERROR` last, always.
- Text sequence: `START → CONTENT* → END`.
- Tool-call sequence: `START → ARGS → END → RESULT`.
- `rawEvent` field used as the correct extension point for implementation-specific metadata.
- `timestamp` present on all events.

If a reviewer asks "is X spec-compliant?" and X is in this list, the answer is yes.

---

## 6. Relationship to Invoke V2 (ADR-007)

Invoke V2 consumes the same AG-UI event stream `/stream` uses. Not every deviation above affects V2:

| Issue | Affects V2 response? |
|---|---|
| `REASONING_ENCRYPTED_VALUE` field names | No — the collector ignores reasoning events. |
| `@ag-ui/client` unused | No — V2 runs server-side, no client SSE parsing. |
| `MESSAGES_SNAPSHOT` missing | No — V2 reads `messages` from `RUN_FINISHED.result.output`. |
| `STATE_SNAPSHOT` tuple on interrupt | No — V2 reads the interrupt payload from `CUSTOM / on_interrupt`, not `STATE_SNAPSHOT`. |
| Non-spec fields on `RUN_FINISHED` | Partial — V2 reads `result.output`; it ignores the non-spec fields. Fixing Fix 3 later would require collector to also handle a new `CUSTOM / INTERRUPT_DETECTED`. |
| PV events not in standard `CUSTOM` shape | No — collector keys off `PV_PRE_RENDER` / `PV_TEMPLATE` / `PV_POST_RENDER` names directly. |
| `streamMode: 'minimal'` drops events | N/A — V2 forces `streamMode: 'verbose'`. |

**In short: V2's response contract is unaffected by these compliance issues.** A separate ADR would cover fixing them. This also means adopting Fix 3 later is a coordinated change — the collector needs to learn about `CUSTOM / INTERRUPT_DETECTED` on the same release.

---

## 7. When to read the full doc

- You are implementing one of the five recommended fixes (Fix 1–5 in the audit).
- You are changing `LangGraphAgentAdapter.ts` emission logic or `frontend/src/types/ag-ui.types.ts`.
- You are onboarding a third-party AG-UI client and need the exact event shapes.
- You are writing a follow-up ADR proposing spec alignment.

Otherwise, this summary is enough.
