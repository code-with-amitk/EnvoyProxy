- [Agent → AI Gateway `server.go` — exact call flow](#agent--ai-gateway-servergo--exact-call-flow)
  - [How control reaches `server.go`](#how-control-reaches-servergo)
  - [End-to-end diagram (lab chat completion)](#end-to-end-diagram-lab-chat-completion)
  - [Stream 1 — listener ext_proc (router filter)](#stream-1--listener-ext_proc-router-filter)
  - [Envoy continues (not in AI Gateway)](#envoy-continues-not-in-ai-gateway)
  - [Stream 2 — upstream ext_proc → LLM credential handoff](#stream-2--upstream-ext_proc--llm-credential-handoff)
  - [Response path (back through Stream 1)](#response-path-back-through-stream-1)
  - [What `server.go` does vs `processor_impl.go`](#what-servergo-does-vs-processor_implgo)
  - [Quick `less` commands](#quick-less-commands)


# Agent → AI Gateway `server.go` — exact call flow

File: `ai-gateway/internal/extproc/server.go`  
Repo root: `/home/amit/ProtocolNine/ai-gateway/`  
gRPC listen: `:1063` · Admin/metrics: `:1064`

Lab: agent POST `http://127.0.0.1:8080/v1/chat/completions` → Envoy ext_proc → AI Gateway → Envoy router → upstream ext_proc → AI Gateway → **TLS to `api.openai.com`**

---

## How control reaches `server.go`

```text
scripts/extproc.sh starts out/extproc-linux-amd64
  │
  ├─ mainlib/main.go          extproc.NewServer()                    L302
  ├─ mainlib/main.go          server.Register("/v1/chat/completions") L306
  ├─ mainlib/main.go          extprocv3.RegisterExternalProcessorServer(s, server) L377
  └─ mainlib/main.go          grpc.Server.Serve(extProcLis)          L422  ← listens :1063

Agent/curl → Envoy :8080
  │
  └─ ext_proc.cc              Filter::sendRequest()                  L708
       └─ gRPC stream         ExternalProcessor.Process              ← server.go L128
```

Proto: `envoy/service/ext_proc/v3/external_processor.proto` — `Process(stream ProcessingRequest) returns (stream ProcessingResponse)`.

---

## End-to-end diagram (lab chat completion)

```text
Agno/curl POST :8080/v1/chat/completions  (Authorization: Bearer gateway-injected)
  │
  ├─ [Envoy listener ext_proc — gRPC stream 1]
  │    ext_proc.cc            Filter::sendRequest()                  L708
  │    server.go              Server.Process()                       L128
  │         └─ stream.Recv()                                         L163
  │         └─ processorForPath()                                    L100  → /v1/chat/completions
  │         └─ NewFactory → newRouterProcessor()                     processor_impl.go L67
  │         └─ routerProcessorsPerReqID[internalReqID] = p           L227
  │         └─ processMsg() RequestHeaders                           L249
  │              └─ passThroughProcessor.ProcessRequestHeaders()     processor.go L46
  │         └─ stream.Send() → Envoy continues (headers)             L239
  │         └─ stream.Recv() RequestBody                             L163
  │         └─ processMsg() RequestBody                              L306
  │              └─ routerProcessor.ProcessRequestBody()             processor_impl.go L213
  │                   └─ ParseBody (OpenAI JSON, model header)       L226
  │         └─ stream.Send() (+ x-ai-eg-internal-req-id header)      L264–289, L239
  │
  ├─ [Envoy — not AI Gateway]
  │    router → openai_cluster → upstream ext_proc opens stream 2
  │
  ├─ [Envoy upstream ext_proc — gRPC stream 2]
  │    server.go              Server.Process()                       L128  (new stream)
  │         └─ isUpstreamFilter = true (req.GetAttributes())         L180
  │         └─ processorForPath() → newUpstreamProcessor()           processor_impl.go L69
  │         └─ setBackend()                                          L373
  │              └─ resolveBackendName("openai")                     L404
  │              └─ upstreamProcessor.SetBackend()                   processor_impl.go L632
  │         └─ processMsg() RequestHeaders                           L249
  │              └─ upstreamProcessor.ProcessRequestHeaders()       processor_impl.go L325
  │                   └─ apiKeyHandler.Do()                          api_key.go L29
  │                        └─ Authorization: Bearer <real OpenAI key>
  │         └─ stream.Send() CONTINUE_AND_REPLACE                    L239
  │
  ├─ [Envoy → LLM — AI Gateway done with request inject]
  │    TLS → api.openai.com:443
  │
  └─ [Response — Stream 1 encode path from Envoy]
       server.go              stream.Recv() ResponseBody               L163
            └─ processMsg() ResponseBody                               L339
                 └─ routerProcessor.ProcessResponseBody()             processor_impl.go L175
                      └─ upstreamProcessor.ProcessResponseBody()      processor_impl.go L495
                           └─ token usage, metrics, dynamic metadata L571–599
            └─ stream.Send() → Envoy → agent                         L239
```

Two gRPC streams per LLM call; linked by **`x-ai-eg-internal-req-id`** (router injects **L264–289**, upstream reads **L184**).

---

## Stream 1 — listener ext_proc (router filter)

```text
server.go                   Server.Process()                           L128
  │
  ├─ stream.Recv()                                                     L163
  ├─ headersToMap(:path, x-request-id)                                 L177
  ├─ internalReqID = x-request-id + UUID                               L191
  ├─ processorForPath()                                                L202
  │    └─ processorFactories["/v1/chat/completions"]                   L112
  │    └─ NewFactory → newRouterProcessor()                            processor_impl.go L67
  ├─ store routerProcessor in map                                      L227
  │
  ├─ Message: RequestHeaders
  │    processMsg()                                                    L234
  │    └─ p.ProcessRequestHeaders()                                    L259
  │         └─ passThroughProcessor (no-op continue)                   processor.go L46
  │    stream.Send()                                                   L239
  │
  ├─ Message: RequestBody (BUFFERED JSON prompt)
  │    processMsg()                                                    L234
  │    └─ routerProcessor.ProcessRequestBody()                         processor_impl.go L213
  │         └─ eh.ParseBody()                                          L226
  │         └─ set x-ai-eg-model header                                L263–271
  │         └─ ClearRouteCache: true                                   L308
  │    inject internal-req-id into response headers                    L264–289
  │    stream.Send()                                                   L239
  │
  └─ (later) ResponseHeaders / ResponseBody — see below
```

---

## Envoy continues (not in AI Gateway)

```text
Envoy ext_proc unblocks → router filter
  └─ router.cc              Router::Filter::decodeHeaders()            L478
       └─ openai_cluster (or openai_economy_cluster)
            └─ upstream ext_proc (headers only, request_attributes with backend name)
                 └─ opens gRPC stream 2 → server.go Process() again
```

---

## Stream 2 — upstream ext_proc → LLM credential handoff

```text
server.go                   Server.Process()                           L128
  │
  ├─ isUpstreamFilter = (req.GetAttributes() != nil)                   L180
  ├─ internalReqID from x-ai-eg-internal-req-id                        L184
  ├─ processorForPath() → newUpstreamProcessor()                       L202, processor_impl.go L69
  ├─ setBackend()                                                      L221
  │    └─ attributes["envoy.filters.http.ext_proc"]                    L374
  │    └─ resolveBackendName() → "openai"                              L379, L404
  │    └─ s.config.Backends["openai"]                                  L385
  │    └─ lookup routerProcessor from cache                            L392
  │    └─ upstreamProcessor.SetBackend()                               L398, processor_impl.go L632
  │         └─ u.handler = backend.Handler (apiKeyHandler)               L647
  │
  ├─ Message: RequestHeaders only (body mode NONE on upstream)
  │    processMsg()                                                    L234
  │    └─ upstreamProcessor.ProcessRequestHeaders()                    processor_impl.go L325
  │         └─ translator.RequestBody (from router's stored body)      L345
  │         └─ apiKeyHandler.Do()                                      api_key.go L29
  │              └─ Authorization: Bearer <OPENAI_API_KEY from extproc.yaml>
  │         └─ CONTINUE_AND_REPLACE                                    L441
  │    stream.Send()                                                   L239
  │
  └─ AI Gateway does NOT call OpenAI — returns mutated headers to Envoy
       Envoy TLS → api.openai.com
```

---

## Response path (back through Stream 1)

```text
OpenAI JSON response → Envoy encode ext_proc → gRPC stream 1
  │
  ├─ server.go              stream.Recv() ResponseHeaders               L163
  │    └─ processMsg() ResponseHeaders                                 L326
  │         └─ routerProcessor.ProcessResponseHeaders()                processor_impl.go L165
  │
  └─ server.go              stream.Recv() ResponseBody (BUFFERED)
       └─ processMsg() ResponseBody                                    L339
            └─ routerProcessor.ProcessResponseBody()                  processor_impl.go L175
                 └─ upstreamProcessor.ProcessResponseBody()           processor_impl.go L495
                      └─ translator.ResponseBody → token usage         L550
                      └─ metrics.RecordTokenUsage()                    L587
                      └─ buildDynamicMetadata (llm_* keys)             L591
       └─ stream.Send() → Envoy access log / agent                     L239
```

---

## What `server.go` does vs `processor_impl.go`

```text
server.go
  ├─ gRPC stream loop (Recv / Send)                                    L128–244
  ├─ Pick processor by :path                                           L100
  ├─ Router vs upstream detection                                      L180
  ├─ Link streams (internal-req-id, setBackend)                        L264–289, L373
  └─ Dispatch phase → processMsg()                                       L246

processor_impl.go  (business logic — parse, inject key, tokens)
  ├─ routerProcessor.ProcessRequestBody()                              L213
  ├─ upstreamProcessor.SetBackend()                                    L632
  ├─ upstreamProcessor.ProcessRequestHeaders()                         L325
  └─ upstreamProcessor.ProcessResponseBody()                           L495
```

---

## Quick `less` commands

```bash
cd ~/ProtocolNine/ai-gateway

less +128 internal/extproc/server.go              # Server.Process
less +163 internal/extproc/server.go              # stream.Recv
less +100 internal/extproc/server.go              # processorForPath
less +246 internal/extproc/server.go              # processMsg
less +373 internal/extproc/server.go              # setBackend

less +302 cmd/extproc/mainlib/main.go             # NewServer + Register
less +377 cmd/extproc/mainlib/main.go             # RegisterExternalProcessorServer

less +213 internal/extproc/processor_impl.go      # router ProcessRequestBody
less +325 internal/extproc/processor_impl.go      # upstream ProcessRequestHeaders
less +632 internal/extproc/processor_impl.go      # SetBackend
less +495 internal/extproc/processor_impl.go      # ProcessResponseBody (tokens)

less +29 internal/backendauth/api_key.go          # Authorization inject
```

**Envoy caller side:** `Documentation/Checklist/Trace._envoy_proxy_ext_proc.cc.md`
