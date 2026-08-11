# EnvoyProxy

- This repository is about the journey of:
    - Building envoy proxy
    - Building AI gateway
    - Setup AI to treat Envoy Proxy as its LLM endpoint
    - Envoy handles routing that call to the real upstream (OpenAI, Anthropic, or another provider)

## Documentation
- [0. Environment and Planning](./Documentation/0.Environment_and_Planning.md)
- [1. EnvoyProxy Build & Run](./Documentation/1.EnvoyProxy_Build_Run.md)
- [2. EnvoyAIGateway Build &Integrate](./Documentation/2.EnvoyAIGateway_Build_Integrate.md)
- [3. Agno Agent MCP Setup](./Documentation/3.Agent_MCP_Setup.md)
- [4. Observability using Prometheus and Graffana (Dashboards)](./Documentation/4.Observability.md)
- [Design Discussion](./Documentation/Design_Discussion/5.3.Design_Discussion.md)
  - [Agent to Agent Gateway](./Documentation/Design_Discussion/5.3.4.Agent_to_Agent_Gateway.md)
  - [Chatbot Parsers Default Parser](./Documentation/Design_Discussion/5.3.5.Chatbot_Parsers_Default_Parser.md)
  - [Code Detection Risk Scoring](./Documentation/Design_Discussion/5.3.3.Code_Detection_Risk_Scoring.md)
  - [Content Safety](./Documentation/Design_Discussion/5.3.2.Content_Safety.md)
  - [Data Loss Prevention](./Documentation/Design_Discussion/5.3.1.Data_Loss_Prevention.md)
- Checklist
  - [Envoy Proxy source/extensions/filters/http/ext_proc/ext_proc.cc](./Documentation/Checklist/Trace._envoy_proxy_ext_proc.cc.md)
  - [ai gateway internal/extproc/server.go](./Documentation/Checklist/Trace.ai_gateway_server.go.md)
