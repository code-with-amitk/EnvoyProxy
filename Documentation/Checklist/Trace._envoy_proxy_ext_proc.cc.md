- [Agent → Envoy → `ext_proc.cc` — exact call flow](#agent--envoy--ext_procc--exact-call-flow)
  - [End-to-end diagram (lab listener ext_proc)](#end-to-end-diagram-lab-listener-ext_proc)
  - [Phase 0 — Agent to HCM (before ext_proc)](#phase-0--agent-to-hcm-before-ext_proc)
  - [Phase 1 — Request headers (decode)](#phase-1--request-headers-decode)
  - [Phase 2 — Request body BUFFERED (decode)](#phase-2--request-body-buffered-decode)
  - [Phase 3 — gRPC send helpers (shared)](#phase-3--grpc-send-helpers-shared)
  - [Phase 4 — AI Gateway responds (decode unblocks)](#phase-4--ai-gateway-responds-decode-unblocks)
  - [Phase 5 — Router + upstream (after ext_proc)](#phase-5--router--upstream-after-ext_proc)
  - [Phase 6 — Response path (encode)](#phase-6--response-path-encode)
  - [Phase 7 — Upstream cluster ext_proc (2nd stream)](#phase-7--upstream-cluster-ext_proc-2nd-stream)
  - [Failure path](#failure-path)
  - [Lab YAML → functions](#lab-yaml--functions)
  - [Quick `less` commands](#quick-less-commands)


# Agent → Envoy → `ext_proc.cc` — exact call flow

File: `envoy/source/extensions/filters/http/ext_proc/ext_proc.cc`  
Repo root: `/home/amit/ProtocolNine/envoy/`

Lab filter order on `:8080`: **Lua → local_ratelimit → ext_proc → router**  
Listener ext_proc: `request_body_mode: BUFFERED`, `response_body_mode: BUFFERED`  
gRPC target: `127.0.0.1:1063` (AI Gateway)

---

## End-to-end diagram (lab listener ext_proc)

```text
Agno/curl POST :8080/v1/chat/completions
  │
  ├─ conn_manager_impl.cc   ActiveStream::decodeHeaders()          L1358
  │    └─ filter_manager_.decodeHeaders()                          L1604
  │         └─ Lua → local_ratelimit → ext_proc.cc (below)
  │
  ├─ ext_proc.cc            Filter::decodeHeaders()                L909
  │    └─ Filter::onHeaders()                                      L884
  │         └─ Filter::openStream()                                L762
  │         └─ Filter::buildHeaderRequest()                        L1162
  │         └─ Filter::sendRequest()                               L708  → gRPC → AI Gateway
  │         ← StopIteration (pause chain)
  │
  ├─ ext_proc.cc            Filter::decodeData()                   L1253
  │    └─ Filter::onData()                                         L1070
  │         └─ Filter::handleDataBufferedMode()                    L942  (buffer…)
  │         └─ Filter::handleDataBufferedMode()                    L942  (end_stream)
  │              └─ Filter::setupBodyChunk()                       L1417
  │              └─ Filter::sendBodyChunk() → sendRequest()         L1431 / L708
  │
  ├─ AI Gateway responds
  │    └─ Filter::onComplete()                                     L723
  │         └─ Filter::onReceiveMessage()                          L1810
  │              └─ decoding_state_.handleHeadersResponse()        L1867
  │              └─ decoding_state_.handleBodyResponse()           L1876
  │         → chain continues → router → openai_cluster
  │
  └─ Response encode path
       └─ Filter::encodeHeaders()                                  L1357
       └─ Filter::encodeData()                                     L1401
            └─ Filter::onData(encoding_state_)                     L1070
            └─ onReceiveMessage → handleBodyResponse (response)   L1881
```

---

## Phase 0 — Agent to HCM (before ext_proc)

```text
Agno/curl → TCP :8080
  │
  ├─ conn_manager_impl.cc   ActiveStream::decodeHeaders()          L1358
  │    └─ filter_manager_.decodeHeaders()                          L1604
  │         └─ filter_manager.cc   per-filter decode dispatch      L592
  │              └─ Lua → local_ratelimit → ext_proc (next phase)
  │
  └─ conn_manager_impl.cc   ActiveStream::decodeData()             L1638
       └─ filter_manager_.decodeData()                            L1645
            └─ same filter chain → ext_proc Filter::decodeData()  L1253
```

---

## Phase 1 — Request headers (decode)

```text
ext_proc.cc                 Filter::decodeHeaders()                L909
  │
  ├─ Filter::mergePerRouteConfig()                                 L911  (def L2132)
  ├─ Filter::onHeaders(decoding_state_, headers)                   L929 → L884
  │    ├─ Filter::openStream()                                     L886 → L762
  │    ├─ Filter::buildHeaderRequest()                             L899 → L1162
  │    │    ├─ Filter::addAttributes()                             L1169 → L1626  (upstream only)
  │    │    ├─ Filter::addDynamicMetadata()                        L1170 → L1555
  │    │    └─ Filter::encodeProtocolConfig()                        L1175 → L1141
  │    └─ Filter::sendRequest()                                    L903 → L708  → gRPC
  │
  └─ return FilterHeadersStatus::StopIteration                     L906  (pause chain)
```

---

## Phase 2 — Request body BUFFERED (decode)

```text
ext_proc.cc                 Filter::decodeData()                   L1253
  └─ Filter::onData(decoding_state_, data, end_stream)             L1256 → L1070
       │
       ├─ (if header response still pending)                        L1091–L1114
       │    └─ StopIterationAndBuffer                              L1112
       │
       └─ switch BUFFERED                                         L1119–L1121
            └─ Filter::handleDataBufferedMode()                   L942
                 ├─ not end_stream → StopIterationAndBuffer       L966
                 └─ end_stream:
                      ├─ Filter::openStream()                     L945 → L762
                      ├─ Filter::setupBodyChunk()                  L957 → L1417
                      ├─ Filter::sendBodyChunk()                   L958 → L1431
                      │    └─ Filter::sendRequest()               L1435 → L708
                      └─ StopIterationNoBuffer                    L962
```

---

## Phase 3 — gRPC send helpers (shared)

```text
Filter::openStream()                                               L762   lazy bidi stream; streams_started stat
Filter::buildHeaderRequest()                                       L1162  header ProcessingRequest
Filter::setupBodyChunk()                                           L1417  body ProcessingRequest
Filter::sendBodyChunk()                                            L1431  body send + timeout callback
Filter::sendRequest()                                              L708   client_->sendRequest(...) — all outbound msgs
Filter::closeStream()                                              L804
Filter::closeStreamMaybeGraceful()                                 L843
```

---

## Phase 4 — AI Gateway responds (decode unblocks)

```text
gRPC callback
  │
  ├─ Filter::onComplete()                                            L723
  │    └─ Filter::onReceiveMessage()                                 L726 → L1810
  │         ├─ Filter::setDecoderDynamicMetadata()                   L1866 → L1677
  │         ├─ decoding_state_.handleHeadersResponse()               L1867  → resume decode
  │         ├─ decoding_state_.handleBodyResponse()                  L1876  → resume decode
  │         ├─ Filter::onProcessHeadersResponse()                    L2339  (optional hook)
  │         └─ Filter::onProcessBodyResponse()                       L2368  (optional hook)
  │
  └─ immediate block (403/404 from AI Gateway):
       Filter::onReceiveMessage() case immediate_response            L1895–L1916
       Filter::sendImmediateResponse()                               L1915 → L2099
       → local reply to agent; no upstream

After header + body responses → filter Continue → router runs
```

---

## Phase 5 — Router + upstream (after ext_proc)

```text
ext_proc unblocks → router.cc       Router::Filter::decodeHeaders()  L478
  └─ openai_cluster (or openai_economy_cluster)
       └─ upstream ext_proc (2nd filter instance, request_body_mode: NONE)
            └─ same ext_proc.cc functions — headers only (Phase 7)
```

---

## Phase 6 — Response path (encode)

```text
OpenAI response → Envoy encode filter chain
  │
  ├─ Filter::encodeHeaders()                                         L1357
  │    └─ Filter::onHeaders(encoding_state_, ...)                    L1373 → L884
  │
  ├─ Filter::encodeData()                                            L1401
  │    └─ Filter::onData(encoding_state_, ...)                      L1404 → L1070
  │         └─ Filter::handleDataBufferedMode()                      L1121 → L942
  │
  ├─ Filter::onReceiveMessage()                                      L1810
  │    ├─ encoding_state_.handleHeadersResponse()                    L1871
  │    ├─ encoding_state_.handleBodyResponse()                       L1881  (token metadata)
  │    └─ Filter::setEncoderDynamicMetadata()                        L1870 → L1674
  │
  └─ Filter::encodeTrailers()                                        L1409  (if trailers)
```

---

## Phase 7 — Upstream cluster ext_proc (2nd stream)

```text
openai_cluster upstream ext_proc  (request_body_mode: NONE)
  │
  ├─ Filter::decodeHeaders()                                         L909
  ├─ Filter::onHeaders()                                             L884
  ├─ Filter::buildHeaderRequest()                                    L1162
  ├─ Filter::addAttributes()                                         L1626  (per_route_rule_backend_name: openai)
  ├─ Filter::sendRequest()                                           L708   → AI Gateway
  └─ Filter::onReceiveMessage() → handleHeadersResponse()            L1810 / L1867
       → CONTINUE_AND_REPLACE (Authorization injected); no decodeData body phase

Linked to listener stream via AI Gateway x-ai-eg-internal-req-id (server.go)
```

---

## Failure path

```text
gRPC / HTTP error
  └─ Filter::onError()                                               L734
       ├─ failure_mode_allow: true
       │    ├─ Filter::logFailOpen()                                L729
       │    └─ continue without ext_proc                             L744–L749
       └─ failure_mode_allow: false
            └─ Filter::sendImmediateResponse()                       L751–L758 → L2099

gRPC status error     Filter::onGrpcError()                           L1994
Stream closed         Filter::onGrpcClose()                           L2022 → onGrpcCloseWithStatus L2024
Fail-open check       Filter::failureModeAllow()                      L1155
Message timeout       Filter::onMessageTimeout()                      L2048
```

---

## Lab YAML → functions

```text
config/envoy.yaml
  │
  ├─ processing_mode.request_header_mode: SEND
  │    └─ Filter::decodeHeaders → Filter::onHeaders                 L909 / L884
  │
  ├─ processing_mode.request_body_mode: BUFFERED
  │    └─ Filter::onData → Filter::handleDataBufferedMode           L1070 / L942
  │
  ├─ processing_mode.response_body_mode: BUFFERED
  │    └─ Filter::encodeData → Filter::onData(encoding_state_)       L1401 / L1070
  │
  ├─ grpc_service: extproc_cluster
  │    └─ Filter::openStream                                        L762
  │
  ├─ request_attributes: [...]
  │    └─ Filter::addAttributes                                     L1626
  │
  ├─ metadataOptions.receivingNamespaces
  │    └─ Filter::setDecoderDynamicMetadata                         L1677
  │
  └─ failure_mode_allow
       └─ Filter::failureModeAllow / Filter::onError                 L1155 / L744
```

---

## Quick `less` commands

```bash
cd ~/ProtocolNine/envoy

# HCM entry
less +1358 source/common/http/conn_manager_impl.cc   # ActiveStream::decodeHeaders
less +1638 source/common/http/conn_manager_impl.cc   # ActiveStream::decodeData

# ext_proc decode (request)
less +909  source/extensions/filters/http/ext_proc/ext_proc.cc   # Filter::decodeHeaders
less +884  source/extensions/filters/http/ext_proc/ext_proc.cc   # Filter::onHeaders
less +762  source/extensions/filters/http/ext_proc/ext_proc.cc   # Filter::openStream
less +708  source/extensions/filters/http/ext_proc/ext_proc.cc   # Filter::sendRequest
less +1253 source/extensions/filters/http/ext_proc/ext_proc.cc   # Filter::decodeData
less +942  source/extensions/filters/http/ext_proc/ext_proc.cc   # Filter::handleDataBufferedMode

# ext_proc receive (AI Gateway reply)
less +1810 source/extensions/filters/http/ext_proc/ext_proc.cc   # Filter::onReceiveMessage

# ext_proc encode (response)
less +1357 source/extensions/filters/http/ext_proc/ext_proc.cc   # Filter::encodeHeaders
less +1401 source/extensions/filters/http/ext_proc/ext_proc.cc   # Filter::encodeData

# failure
less +734  source/extensions/filters/http/ext_proc/ext_proc.cc   # Filter::onError
```

**AI Gateway receiver:** `Documentation/Checklist/Trace.ai_gateway_server.go.md`  
**Full Envoy core path:** `Documentation/Source_Code_Navigation/6.Envoy_Core.md`
