# ai-sdk

A model SDK for Kira, built on one idea: **what a model *is* is separate from
where it is served.** A model names one capability surface; the many endpoints
it can be reached at are data. Code selects a model by what it needs the model to
do, and routing decides — later, elsewhere — which backend answers.

```
ai-sdk/
  core/        the canonical layer and the gateway; no wire format lives here
  openai/      the chat-completions protocol, written once as reusable functions
  openrouter/  the OpenAI protocol plus provider routing, built on openai/
  anthropic/   the Messages protocol: a different wire, the same canonical types
  tests/       each codec against byte vectors, and generations over HTTP/3
  examples/demo             one real streamed answer from whichever key is set
  examples/openrouter-live  one real completion with a tool call
```

## Transport

HTTP comes from Kira's `Network` library
(`Network`, bundled in Kira's std and imported by name): HTTP/3 over QUIC or HTTP/1.1 over TLS 1.3, `text/event-stream`
decoding, and one value the program owns and moves with `wait` or `poll`.

A backend never touches a socket. It is a codec: it encodes a canonical
request into an HTTP request, and decodes a whole body or one server-sent event
back into canonical values. The `Gateway` sends what a backend encodes through
the program's `Network`. So every backend streams, multiplexes and speaks
HTTP/3 the same way, and each is tested with byte vectors alone.

## The shape

```kira
construct Backend {
    @Required let service: Service
    @Required let protocol: Protocol
    let region: Region? = .None

    @Required function encode(modelID: borrow String, request: borrow Request, stream: Bool)
        -> Result<Net.Request, AIError>
    @Required function decode(status: Number, body: borrow [UN8], model: borrow Model)
        -> Result<Response, AIError>
    @Required function decodeEvent(event: borrow Net.ServerEvent, assembly: borrow mut StreamAssembly)
        -> Result<[StreamEvent], AIError>
}
```

`OpenAIBackend`, `OpenAICompatibleBackend`, `AnthropicBackend` and
`OpenRouterBackend` each extend `Backend`. A `Router` holds them as
`Any Backend` and resolves a model to the first backend that agrees on
service, protocol and region. A `Gateway` validates a request against its
model locally, before any network I/O, then routes, encodes and sends it.

## Streaming

```kira
var network = try Net.openNetwork(Net.NetworkOptions {}) otherwise return
let flow = gateway([AnthropicBackend(apiKey: key)])
var generation = try flow.start(network, request) otherwise return
while !generation.finished {
    for event in network.wait(.None) {
        for produced in generation.take(event) {
            match produced {
                TextDelta(part) -> show(part.text)
                Completed(response) -> finish(response)
                Failed(problem) -> report(problem)
                else -> {}
            }
        }
    }
}
```

`take` ignores events for other requests, so any number of generations and any
other traffic share one `Network`. Text, reasoning and tool calls arrive as
`Start`/`Delta`/`End` brackets, and the stream ends with exactly one
`Completed(Response)` or `Failed(AIError)`. A service that answers a
streaming request with one whole document still produces the same events.
`flow.generate(network, request, until)` waits for a whole answer instead.

`generation.exchange` keeps the wire record of the call before any decoding
(URL, model id, request fields with the key shown by name only, request body,
status, response fields, raw body), for diagnosing a failed or empty turn.

`Response.servedModel` is the model the service says answered, which differs
from `Response.model` when a provider falls back. Reasoning keeps its
signature, so a tool-use turn replays to Anthropic intact.

## Same model, different routes

`AI.Providers.OpenAI.GPT.5.6.Sol` is an OpenAI model. One of its endpoints is
an Anthropic-shaped proxy; another is a Bedrock region. The model's provider
and the service it is served through are different facts. Point the same model
at a different backend and it goes a different way, unchanged:

```kira
let flow = gateway([OpenAIBackend(apiKey: key)])                       // OpenAI direct
let flow = gateway([OpenRouterBackend(apiKey: key)])                   // OpenRouter
let flow = gateway([OpenAICompatibleBackend(baseURL: url)])            // any compatible server
```

## Run it

```
kira test tests
ANTHROPIC_API_KEY=... kira run examples/demo -- "Why is the sky blue?"
```

`OPENROUTER_API_KEY` or `OPENAI_API_KEY` work in place of the Anthropic key.

## Not yet

- **Region narrowing is a runtime check.** Kira has no enum inheritance, so
  `regionBroad` names each zone's bucket and routing checks the relation when
  it resolves.
- **Structured output on Anthropic.** The Messages encoder refuses a request
  with an `outputSchema` rather than answering free text.
- **Redacted thinking** blocks from Anthropic are dropped, so a turn that had
  one cannot be replayed exactly.
- **TLS 1.2-only servers** cannot be reached: `Network` speaks TLS 1.3 only.
- **`core/models/`** holds generated model tables in an older dialect. They are
  not part of any package until they are brought up to date.
