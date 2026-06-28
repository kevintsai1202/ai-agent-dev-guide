# SSE streaming and progressive rendering

> §1–§3 (event protocol, frontend fetch/ReadableStream, lenient/sanitize progressive rendering) are **backend-framework-agnostic**. §4–§5 are the empirical **Embabel + Spring AI** backend implementation (marked "▼ Embabel example"); for the general contract see the "Backend contract" section in [`../SKILL.md`](../SKILL.md).

## 1. SSE event protocol

The backend pushes via `SseEmitter`, the frontend parses. Recommended events:

| Event | data | When |
| ---- | ---- | ---- |
| `status` | `{"phase":"analyzing"}` / `{"phase":"generating","agent":"..."}` / `{"phase":"fusing"}` | Phase transition |
| `intent` | `{"status":"running"}` → `{"status":"done","compound":bool,"subCount":n}` | Intent classification (1st LLM call) |
| `compound` | `{"subQueries":[...],"total":n}` | Compound query decomposition |
| `subagent` | `{"index":i,"total":n,"query":"..."}` | Start of the i-th sub-analysis |
| `plan` | `{"steps":[...],"goal":"...","total":n}` | GOAP has planned a path |
| `step` | `{"index":i,"name":"...","status":"running"}` / `{"...":"done","durationMs":ms}` | Step start/completion |
| `chunk` | text fragment of the spec JSON | Render streaming (for progressive rendering) |
| `complete` | `{"success":true,"cost":..,"totalTokens":..,"llmCalls":..}` | Completion + cost |
| `error` | `{"code":"LLM_TIMEOUT","message":"..."}` | Error (with classification code) |

Error classification codes are mapped centrally on the backend (`LLM_TIMEOUT` / `LLM_QUOTA_EXCEEDED` / `INVALID_SPEC` / `NO_COMPONENTS` / `INTERNAL_ERROR`), and the frontend then maps them to user-facing messages.

## 2. Frontend uses fetch + ReadableStream (not EventSource)

`EventSource` doesn't support a POST body, so it can't carry the query → always parse SSE yourself:

```ts
const res = await fetch("/api/dashboard/generate", {
  method: "POST", headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ content: query }), signal: controller.signal,
});
const reader = res.body!.getReader();
const decoder = new TextDecoder();
let buffer = "";
// Read chunk by chunk, split events on \n\n; keep the remainder in buffer
buffer += decoder.decode(value, { stream: true });
const events = buffer.replace(/\r\n/g, "\n").split("\n\n");
buffer = events.pop() || "";
for (const ev of events) {
  // Parse event: and data: (join multiple data lines with \n; strip a single leading space after "data:")
}
```

Key points:

- Normalize `\r\n` to `\n` before splitting (cross-platform).
- After `data:`, strip **only one** leading space (per the SSE spec); join multiple `data:` lines with `\n`.
- The last segment without a trailing `\n\n` stays in the buffer for the next round.
- Use `AbortController` to cancel on component unmount / re-send.

## 3. Progressive rendering: lenientParse → sanitize → setSpec

As the LLM emits JSON, the frontend attempts to render whatever is currently available on each chunk received. Two functions are the core:

### lenientParseSpec — complete unclosed delimiters to parse half-finished JSON

```ts
function lenientParseSpec(s: string): any | null {
  try { return JSON.parse(s); } catch {}
  // Scan to track unclosed { [ and string state, append the missing closers;
  // then try candidates like "drop the trailing incomplete key / key:value" and JSON.parse each
}
```

Key idea: track `inStr`/`esc` and the `{[` stack, produce a few "closing candidate strings", and try parsing each one; if all fail return `null` (keeping the last successful spec).

### sanitizeSpec — keep half-finished elements from crashing the Renderer

```ts
function sanitizeSpec(spec: any): Spec | null {
  if (!spec?.elements || typeof spec.elements !== "object") return null;
  const elements: Record<string, any> = {};
  for (const [k, el] of Object.entries<any>(spec.elements)) {
    if (!el || typeof el !== "object") continue;
    elements[k] = {
      type: el.type,
      props: el.props && typeof el.props === "object" ? el.props : {}, // backfill props
      children: Array.isArray(el.children) ? el.children : [],          // backfill children
    };
  }
  // Filter out child references pointing to "elements not yet emitted" (common in half-finished streams)
  for (const el of Object.values<any>(elements))
    el.children = el.children.filter((c: any) => typeof c === "string" && elements[c]);
  const root = spec.root && elements[spec.root] ? spec.root : null;
  return root ? ({ root, elements } as Spec) : null; // root not emitted yet → don't render yet
}
```

**Why sanitize is mandatory**: mid-stream, some element's `props` is still `undefined`, and json-render's internal `resolveBindings` throws `Cannot convert undefined or null to object`. Backfilling `props:{}`/`children:[]` + filtering dangling child refs stabilizes it.

On receiving a `chunk`: `jsonAccumulator += data; const partial = sanitizeSpec(lenientParseSpec(jsonAccumulator)); if (partial) setSpec(partial);`

## 4. True streaming vs schema enforcement (backend trade-off)

General trade-off: true streaming (emit tokens as they come, frontend can render progressively) and schema enforcement (type-safe but buffers into a final snapshot) are usually mutually exclusive — any LLM backend hits this. ▼ Embabel example: `StreamingPromptRunner`'s two modes are mutually exclusive:

| Method | Behavior | Trade-off |
| ---- | ---- | ---- |
| `streaming().withPrompt(p).generateStream()` → `Flux<String>` | **Truly progressive** token emission | No schema enforcement → must use a strengthened prompt to force the format + validate after parsing |
| `...createObjectStream(Class)` → `Flux<T>` | Schema-safe | But **buffers into a single final snapshot**, no progressive effect |

The POC's `streamingRender()` strategy: when chunk streaming is on and the model supports it → use `generateStream` to emit and `forwardChunk(token)` to push to the frontend as it goes; after accumulation, parse with `objectMapper.readValue` and validate via `hasValidShape` (root + non-empty elements); **any streaming failure falls back to a normal `createObject`**. Combined with the frontend's lenient/sanitize, this forms robust two-sided fault tolerance.

Speedup tip: a "large JSON assembly" step like render can use a faster small model (e.g. nano), while analysis steps still use the main model to maintain quality — `ai.withLlm(RENDER_MODEL)`.

## 5. Controller streaming orchestration skeleton (▼ Embabel example)

```text
1. send status:analyzing
2. send intent:running → run IntentSplitAgent → send intent:done(compound,subCount)
3. setActive(progressListener)  // point the global broker at this connection
   if single: setChunkStreaming(true); chooseAndRunAgent → DashboardSpec; close
   if compound: for each sub-query chooseAndRunAgent (send subagent event, listener.reset())
            → collect multiple specs → FusionAgent fuses (enable chunkStreaming only for this fusion segment)
4. finally: setChunkStreaming(false); clear(listener)
5. if forwardedCount==0: streamSpec(spec)   // render didn't stream live → push the full spec in chunks
6. send complete(cost,tokens,llmCalls); emitter.complete()
```

Key: `forwardedCount` determines "whether the render step already streamed tokens to the frontend live", avoiding "streamed once + pushed again" duplication.
