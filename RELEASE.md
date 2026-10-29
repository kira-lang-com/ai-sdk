# 26.10.4

The first source release of the Kira model SDK includes:

- Canonical model, request, response, and backend interfaces, with routing and local request validation in the gateway.
- OpenAI, OpenAI-compatible, OpenRouter, and Anthropic backends, with protocol codecs separated from transport.
- Streaming through Kira's Network library, using HTTP/3 over QUIC or HTTP/1.1 over TLS.
