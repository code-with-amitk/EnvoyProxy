# Trace: `server.go`

## Problem

Assignment asks you to trace a request through **`internal/extproc/server.go`** line-by-line. You need to know how Envoy's gRPC messages become prompt parsing and API key injection.

**Read first:**

```text
Documentation/Checklist/Trace.ai_gateway_server.go.md   full function + line tree
server.go L128                    Server.Process — gRPC entry
processor_impl.go L213 / L325     business logic (parse, inject key)
Documentation/Source_Code_Navigation/7.AIGateway.md     overview
```

---

## Answer

Envoy sends gRPC messages to `Process()`. **Stream 1** (listener) parses the prompt. **Stream 2** (upstream) injects the real OpenAI key. AI Gateway does **not** call OpenAI — Envoy sends HTTPS after getting mutated headers back.

---

## How

```text
Agent → HTTPS :8080
        │
        ▼
┌──────────────────────────── Envoy (proxy) ────────────────────────────────────┐
│  ext_proc.cc sendRequest L708   gRPC client — sends headers/body to gateway │
│  ✗ no business logic here — transport + pause filter chain until reply       │
└──────────────────────────────┬──────────────────────────────────────────────┘
                               │ gRPC :1063
                               ▼
┌──────────────────────── AI Gateway — server.go ──────────────────────────────┐
│  Process() L128              ★ entry — dispatch to processor by stream type ★│
│    Stream 1 routerProcessor  ★ L213 parse prompt (listener path) ★           │
│    Stream 2 upstreamProcessor ★ L325 inject API key (upstream path) ★        │
└──────────────────────────────┬──────────────────────────────────────────────┘
                               │ mutated headers back to Envoy
                               ▼
                    Envoy TLS → api.openai.com  (AI Gateway never calls OpenAI)
```

---

## Changes

*Read-only trace — no design changes.* Detail: `Documentation/Checklist/Trace.ai_gateway_server.go.md`
