# Trace: `ext_proc.cc`

## Problem

Assignment asks how **Envoy calls the ext_proc gRPC service**. You need to see where the filter chain pauses and what gets sent to AI Gateway.

**Read first:**

```text
Documentation/Checklist/Trace._envoy_proxy_ext_proc.cc.md   full function + line tree
ext_proc.cc L909 / L1253          decodeHeaders / decodeData
ext_proc.cc L1810                 onReceiveMessage — AI Gateway reply
Documentation/Source_Code_Navigation/6.Envoy_Core.md      Envoy filter chain
```

---

## Answer

On each request, ext_proc **opens a gRPC stream**, sends headers then body (BUFFERED in lab), and **pauses** the filter chain until AI Gateway replies. Same on the response path. Upstream cluster runs ext_proc again (headers only) for key injection. If AI Gateway is down, `failure_mode_allow` decides pass-through vs error.

---

## How

```text
Agent request
        │
        ▼
┌──────────────────────────── Envoy :8080 (proxy) ─ ext_proc filter ───────────┐
│  HCM L1358           enter HTTP filter chain                                  │
│  ★ Lua L? ★          Envoy-only string policy (before ext_proc)               │
│  ★ rate limit ★      Envoy-only RPM / token budget                            │
│  decodeHeaders L909  ★ pause chain · open gRPC stream ★                       │
│  decodeData L1253    ★ send BUFFERED body to AI Gateway ★                     │
│  onReceiveMessage L1810  resume chain with AI Gateway reply                    │
│  router              ★ Envoy-only cluster pick ★                              │
│  upstream ext_proc   second gRPC stream (headers only — key inject)         │
│  TLS cluster         → api.openai.com                                         │
└──────────────────────────────┬──────────────────────────────────────────────┘
                               │ gRPC :1063
                               ▼
┌──────────────────────── AI Gateway ──────────────────────────────────────────┐
│  ★ parse · policy · inject key · token metrics ★  (see Trace.server.go.md)  │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Changes

*Read-only trace — no design changes.* Detail: `Documentation/Checklist/Trace._envoy_proxy_ext_proc.cc.md`
