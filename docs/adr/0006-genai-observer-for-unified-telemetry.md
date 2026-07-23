# 0006. GenAI Observer for Unified Telemetry Abstraction

Date: 2026-07-23

## Status

Proposed

## Context

OpenTelemetry semantic conventions for GenAI (`gen_ai.*`) require capturing detailed request parameters, token usage metrics, finish reasons, and occasionally full prompt/completion content. 

Currently, our `openai-go` and `anthropic-sdk-go` instrumentations implement this logic independently. Each SDK instrumentation contains redundant, complex state machines for parsing Server-Sent Events (SSE), mapping idiosyncratic provider JSON to OpenTelemetry attributes, and bounding memory allocation for content capture.

As the LFX Term 3 roadmap targets expansion to Gemini, LangChain, and MCP, continuing to duplicate this telemetry logic inside each SDK introduces several critical risks:
1. **Maintenance Burden:** Updates to OpenTelemetry Semantic Conventions must be manually propagated across every SDK implementation.
2. **Behavioral Drift:** Discrepancies in how metrics (e.g., Anthropic's cache tokens vs OpenAI's prompt tokens) are aggregated.
3. **Memory Safety (OOM):** Content capture (`OTEL_INSTRUMENTATION_GENAI_CAPTURE_MESSAGE_CONTENT`) requires accumulating streaming chunks. Implementing memory bounds ad-hoc inside each SDK risks memory leaks or OOM panics if implemented incorrectly.
4. **Integration Coupling:** Frameworks like LangChain and MCP are not inherently HTTP-based, making standard `net/http` middleware injection patterns inapplicable.

## Decision

We will introduce a central `genai.Observer` interface within the `pkg/genai` namespace to serve as the unified telemetry engine for all GenAI instrumentations.

SDK-specific instrumentations (e.g., `openai-go`, `langchain`) will act purely as lightweight **Extractors**. Their only responsibility will be parsing their native request/response structures into generic `genai.Request` and `genai.Response` representations. 

The `genai.Observer` will handle:
- Span creation, lifecycle management, and Semantic Convention mapping.
- Memory-bounded accumulation of streaming SSE chunks for content capture.
- Standardized error handling and attribute assignment.

Because the `Observer` interface accepts a standard `context.Context` rather than a `net/http.Request`, it natively supports both HTTP middleware injection and standard `HookContext` before/after execution patterns required by non-HTTP frameworks like LangChain.

## Consequences

* **Positive:** Semantic Convention mapping is completely DRY (Don't Repeat Yourself).
* **Positive:** Strict, centrally-managed memory bounds on content capture protect all instrumented applications from OOM risks universally.
* **Positive:** Expanding support to new AI providers becomes significantly faster, requiring only a lightweight mapping struct.
* **Negative:** SDK extractors must allocate intermediate `genai.Request` and `genai.Response` structs to pass data to the Observer, introducing a very minor runtime allocation overhead.
