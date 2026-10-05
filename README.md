# ai-sdk

A model SDK for Kira, built on one idea: **what a model *is* is separate from
where it is served.** A model names one capability surface; the many endpoints
it can be reached at are data. Code selects a model by what it needs the model to
do, and routing decides — later, elsewhere — which backend answers.

```
ai-sdk/
  core/        the canonical layer — no wire format, no socket lives here
  openai/      the OpenAI protocol, written once as reusable functions
  openrouter/  the OpenAI protocol plus provider routing, built on openai/
  anthropic/   the Messages protocol — a different wire, the same canonical types
  examples/demo/   a runnable tour: routing, regions, validation, a real call
```

## Transport

The SDK writes no FFI of its own. HTTP comes from Foundation — `httpSend`,
`HttpHeader`, `HttpAnswer` — which every package already reaches through
`import Foundation`. Foundation owns the `kira_network` binding (autobound from
its header, not hand-written) and ships the archive with the toolchain, so there
is nothing for the SDK to declare and no `NativeLibs` to carry. A backend calls
`httpSend`; nothing above it, and nothing in this repo, knows a socket.

## The shape

Everything conforms to one construct family:

```kira
construct Backend {
    @Required let service: Service
    @Required let protocol: Protocol
    let region: Option<Region> = .None

    @Required function generate(modelID: borrow String, request: borrow Request)
        -> Result<Response, AIError>
    @Required function streamEvents(modelID: borrow String, request: borrow Request)
        -> Result<[StreamEvent], AIError>
}
```

`OpenAIBackend`, `AnthropicBackend`, and `OpenRouterBackend` each `extends
Backend`. A `Router` holds them as `Any Backend`, and resolves a model to the
first backend that agrees on service, protocol, and region. A `Gateway`
validates a request against its model — locally, before any network I/O — then
routes and dispatches it.

## Same model, different routes

`gpt56Sol()` is an OpenAI model. One of its endpoints is an Anthropic-shaped
proxy; another is a Bedrock region. The model's provider and the service it is
served through are different facts, and the second never edits the first. Point
the same model at a different backend and it goes a different way, unchanged:

```kira
let flow = gateway([OpenAIBackend(apiKey: key)])        // OpenAI direct
let flow = gateway([openRouter(key)])                   // OpenRouter, OpenAI protocol
let flow = gateway([OpenAICompatibleBackend(baseURL: url)])  // any compatible endpoint
```

## OpenRouter is OpenAI, plus

`openrouter/` imports `openai/`'s encoder, send, and decoder — it does not copy
them. All it adds is a `provider` block on the request body and a title header.
When the OpenAI encoder gains a field, OpenRouter gets it for free, because there
is only one encoder.

## Run the tour

```

These commands want **`kk`**, the native frontend of the
[Kira Language Framework](https://github.com/kira-lang-com/klf-kira), which
`klf build .` produces in that repository. `kk`'s binary is also called `kira`,
so the two are told apart by which one is on your `PATH`, not by the name you
type. The oracle compiler from
[kira-lang-com/kira](https://github.com/kira-lang-com/kira) aborts partway
through semantic analysis on this codebase.

kira run examples/demo            # from the kira toolchain, against this repo
```

It checks region narrowing, route selection, and request validation with no
network, then sends one real completion over TLS to the loopback service the
toolchain ships — so the encode/send/decode path runs for real with no key and
no internet.

## Not yet

- **Region narrowing is a runtime check.** The design wants enum inheritance
  (`enum AWSUSRegion extends Region.US`), which Kira does not have yet; until it
  lands, `regionNarrows` does at run time what the subtype relation would do at
  compile time. When it lands, `core/app/Region.kira` collapses into the enum
  and nothing above it changes.
- **Streaming is materialized.** `streamEvents` returns the real events of a real
  completion, delivered at once — the shipped network FFI has no SSE primitive
  yet. A consumer written against `StreamEvent` stays correct when incremental
  transport lands beneath it.
