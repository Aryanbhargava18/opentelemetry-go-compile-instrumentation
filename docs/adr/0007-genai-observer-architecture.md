# 7. GenAI Observer Architecture

Date: 2026-07-30

## Status

PROPOSED

## Context

Instrumenting AI SDKs (OpenAI, Anthropic) requires parsing Server-Sent Events (SSE) to aggregate telemetry like input/output token usage and finish reasons. Additionally, capturing streaming message content (`gen_ai.content.completion`) poses a high OOM risk if unbounded string deltas are accumulated in memory.

Currently, this requires complex state machines inside the HTTP middleware of every SDK (e.g., `openai-go/streaming.go` is ~260 lines of custom `io.TeeReader` buffer management and content concatenation limits). Also, correctly handling premature stream aborts versus clean `[DONE]` events requires strict event-ordering logic to prevent dropped terminal states (e.g., overriding a parsed `finish_reason` if a subsequent transport error occurs). As we expand to support Gemini and Agent frameworks (LangChain), preventing duplicated buffer management and span lifecycle logic across every package will prevent memory leaks, inconsistent truncation limits, and semantic convention drift.

## Decision

Introduce `pkg/genai.Observer` to centralize telemetry mapping, span lifecycle, and memory bounding. 

```text
  build time
       ┌───────────────────────┬───────────────────────┐
       │                     otelc                     │
       │                  compile-time                 │
       └───────────────────────┬───────────────────────┘
                               │
                  [ Instrumented app binary ]
                 ( instrumented SDK calls ↓ )

     ┌────────────────────────┐   ┌────────────────────────┐
     │       openai-go        │   │    anthropic-sdk-go    │
     │ first architecture test│   │ week 6 - stress test   │
     └───────────┬────────────┘   └───────────┬────────────┘
                 │ (passes http.Body + ChunkExtractor)│
                 └────────────┬───────────────┘
                              │
  ────── no provider SDK types cross this boundary ─────────
                              │
            ┌─────────────────▼─────────────────┐
            │pkg/genai - proposed adapter layer │
            │                                   │
            │ ┌───────────────────────────────┐ │
            │ │         StreamAdapter         │ │
            │ │io.TeeReader · frames SSE bytes│ │
            │ └───────────────┬───────────────┘ │
            │                 │ invokes         │
            │     ┌───────────▼───────────┐     │
            │     │    ChunkExtractor     │     │
            │     │   (SDK JSON parser)   │     │
            │     └───────────┬───────────┘     │
            │                 │ returns         │
            │     ┌───────────▼───────────┐     │
            │     │     ExtractedData     │     │
            │     │  (provider-neutral)   │     │
            │     └───────────┬───────────┘     │
            │                 │ delegates       │
            │ ┌───────────────▼───────────────┐ │
            │ │        genai.Observer         │ │
            │ │span lifecycle · attrs · bounds│ │
            │ └───────────────────────────────┘ │
            │                                   │
            │ * hypothesis: revised if provider-│
            │   specific logic leaks upward     │
            └─────────────────┬─────────────────┘
                              │
             ┌────────────────▼────────────────┐
             │ OpenTelemetry GenAI Conventions │
             └────────────────┬────────────────┘
                              │
                  ┌───────────▼───────────┐
                  │  gen_ai.* telemetry   │
                  └───────────────────────┘

 ·········· under evaluation — week 12 (not implementation) ··········
      ┌──────────────────────────┐       ┌────────────────┐
      │      Non-HTTP SDKs       │       │      MCP       │
      │does observer generalize? │       │ opt. prototype │
      └──────────────────────────┘       └────────────────┘
```

### 1. The `StreamAdapter` for HTTP Middleware
For SDKs operating at the HTTP layer, the Observer provides a `StreamAdapter` that wraps and takes ownership of the `*http.Response.Body`, implementing `io.Closer` and guaranteeing `Body.Close()` is called exactly once—either on clean completion or on error return. It buffers and frames SSE chunks, delegating the extracted data to the underlying `genai.Observer`. The `genai.Observer` itself enforces the hard global memory limit (`maxResponseBodySize`) for content concatenation (preventing OOMs), calculates Time-To-First-Token (TTFT), and finalizes the OpenTelemetry span, ensuring correct terminal state ordering even on premature stream aborts.

### 2. SDKs as JSON Extractors
HTTP SDK instrumentations are reduced to lightweight JSON mappers. They provide a callback to the `StreamAdapter`:

```go
type ToolCall struct {
    ID       string
    Function string
    Args     string
}

type ExtractedData struct {
    ID                 string
    Model              string
    PromptTokens       int64
    CompletionTokens   int64
    TotalTokens        int64
    FinishReasons      []string
    ContentDelta       string
    ToolCalls          []ToolCall
    ProviderAttributes []attribute.KeyValue
}

type ChunkExtractor func(rawEvent []byte) (ExtractedData, error)
```

The adapter splits the SSE stream by event boundaries (`\n\n`) and invokes `ChunkExtractor`. The SDK parses the event block and returns `ExtractedData`.
*   **Error Propagation:** If the `ChunkExtractor` encounters malformed proprietary JSON, returning an error ensures the `genai.Observer` correctly records `error.type` on the span instead of silently dropping the chunk.
*   **Protocol Framing:** The central adapter MUST NOT strip `data: ` prefixes natively. OpenAI uses `data: [DONE]`, but Anthropic uses `event: message_stop`. The `ChunkExtractor` is responsible for handling its provider's proprietary framing and JSON schema.
*   **Semantic Integrity:** The `ID` and `Model` fields ensure `gen_ai.response.id` and `gen_ai.response.model` conventions are met dynamically as chunks arrive.
*   **State Aggregation:** Because tokens and tool calls are streamed incrementally, the `genai.Observer` tracks running maximums for tokens (preventing double-counting if a provider sends cumulative usage per-chunk) and concatenates streamed `ToolCall.Args` by ID. The `ChunkExtractor` remains strictly stateless.
*   **Extensibility:** The `ProviderAttributes` array allows Anthropic to pass through unique metrics (e.g., `gen_ai.anthropic.usage.cache_read_input_tokens`) without polluting the generic struct.
*   **OOM Protection:** The `ContentDelta` is bounded and concatenated by the base `genai.Observer`, ensuring safe `gen_ai.content.completion` emission across all transport types.

### 3. Agent Frameworks (LangChain)
For non-HTTP frameworks like LangChain, the SDK instrumentation will bypass the `StreamAdapter` (as there is no SSE or `io.TeeReader`) and map their Go structs directly into the base `genai.Observer`, inheriting the same centralized memory bounds and semantic output across all paradigms.

### 4. Out of Scope: MCP
Model Context Protocol (MCP) relies on multiplexed JSON-RPC (often over WebSockets) where multiple requests share a stream and backends can fan-out. This requires tracking JSON-RPC IDs and child spans. `genai.Observer` is explicitly scoped to linear Request/Response lifecycles. MCP instrumentation must use a distinct `mcp.Observer` pattern.

## Migration Path

1. Merge `pkg/genai.Observer` and `genai.StreamAdapter`.
2. Update the `openai-go` weaver instrumentation rules to inject `genai.StreamAdapter` in place of the existing HTTP response body interception hooks.
3. Refactor `openai-go` middleware to wrap responses with `genai.StreamAdapter`.
4. Pass `parseChatResponse` and `parseCompletionResponse` to the adapter as `ChunkExtractor` callbacks.
5. Delete `instrumentation/github.com/openai/openai-go/streaming.go` after the weaver rule update is validated by integration tests.

## Backward Compatibility

This refactor is entirely internal to the HTTP middleware. It requires no API changes for `otelc` users. Emitted telemetry will remain exactly compliant with the current `gen_ai.*` semantic conventions.

## Alternatives Considered

1. **Status Quo (Duplicating State Machines):** Write a new `streaming.go` for Gemini. Rejected. Duplicating `io.TeeReader` logic multiplies the surface area for memory leaks and OOM vulnerabilities when capturing content.
2. **Stateless Helper Functions:** Export stateless parsing helpers. Rejected. Aggregating usage tokens, calculating TTFT, and concatenating string deltas requires state persistence across the HTTP request lifecycle, mandating a stateful observer.
3. **Inline `span.SetAttributes()` per Provider:** Each provider middleware calls `span.SetAttributes()` directly without a shared observer. Rejected. This pushes semantic convention enforcement into every individual provider instrumentation. When the GenAI semantic conventions update (e.g., a new required attribute), every provider implementation must be updated independently—the exact drift problem this ADR is designed to prevent.

## Consequences

* **Positive:** Mitigates OOM vectors across all SDKs by centralizing content capture limits.
* **Positive:** Reduces code footprint for new SDKs (Gemini integration becomes a trivial JSON unmarshaling callback).
* **Positive:** Extensible to Agent Frameworks without HTTP middleware dependencies.
* **Negative:** Minor allocation overhead introduced by the callback interface and `ExtractedData` mapping structs.
* **Negative:** A `ChunkExtractor` error introduces a policy decision: abort the stream and record `error.type`, or skip the malformed chunk and continue with partial telemetry. The correct policy (abort-on-error) must be explicitly enforced by `genai.Observer`; skipping malformed chunks silently would produce incomplete span data and defeat the purpose of centralized correctness.
