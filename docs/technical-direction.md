# Multi-Agent Demo: Initial Technical Direction

## Purpose

This repository starts as a deliberately minimal proof of concept for a **static, browser-only chat application that can talk to different language models regardless of where they run**.

The central experiment is simple:

> Can a user open a static web application, choose a model, and chat with it — where the model may either run locally in the browser or be provided by a remote LLM API using credentials supplied by the user — without the application having any backend of its own?

The proof of concept should answer that question with as little application-specific code and infrastructure as practical.

This is not initially an attempt to build a polished multi-provider AI application, an agent framework, or another OpenRouter. The interesting property is that the application itself can remain a set of static assets that anyone can download and serve themselves.

## Core constraints

The first prototype should preserve these constraints aggressively:

- **No application backend.** No proxy, serverless function, edge worker, token broker, or application server.
- **Static deployment.** It should be possible to host the application on an ordinary static web server.
- **Browser execution.** All application logic executes in the user's browser.
- **Multiple model sources.** At least one browser-local model and more than one remote provider should ultimately be demonstrable.
- **Bring your own credentials.** For remote providers, credentials belong to the user rather than to the deployed application.
- **Minimal glue code.** Prefer established JavaScript libraries over writing and maintaining provider adapters ourselves.
- **Minimal UI.** The POC only needs enough interface to choose a model/provider, enter credentials when necessary, submit a prompt, and see the response.
- **No premature abstraction.** The POC should prove the concept before designing a comprehensive provider/plugin architecture.

## User experience to prove

A minimal interaction could be:

1. Open the static application.
2. Choose a model from one selector.
3. If it is a browser model, allow the application to download/load it.
4. If it is a hosted model, ask for whatever credential or authorization that provider requires.
5. Enter a message.
6. Receive the model's response in the same chat interface.
7. Switch to another model and repeat.

The important conceptual point is that **local versus remote is a property of the selected model, not a different application mode**.

## Architecture

At its simplest:

```text
                    Static HTML application
                            |
                     common chat UI
                            |
                    model/provider layer
                      /             \
                     /               \
          browser-local model       hosted model API
             WebGPU/WASM              HTTPS/CORS
                 |                        |
          downloaded weights          user's API key
```

There is deliberately no middle tier.

## Browser-local models

Modern browsers can run useful small language models locally, most interestingly through WebGPU. Model weights are downloaded to the user's machine and inference then happens locally.

Candidate libraries include:

### WebLLM

[WebLLM](https://github.com/mlc-ai/web-llm) is the strongest starting candidate for the local-model side of the POC.

Advantages:

- specifically designed for running LLMs in the browser;
- WebGPU acceleration;
- model loading/caching machinery already implemented;
- streaming generation;
- OpenAI-style chat-completions API, which may reduce glue code;
- established project with a selection of preconfigured models.

Disadvantages / questions:

- model downloads can still be hundreds of MB or several GB;
- WebGPU/browser/hardware compatibility needs graceful handling;
- first-load latency is fundamentally different from calling an API.

For the POC, choose a genuinely small model. The goal is not to demonstrate frontier-model quality; it is to demonstrate that the same static UI can seamlessly address a model which exists entirely inside the browser.

### Transformers.js

[Transformers.js](https://github.com/huggingface/transformers.js) is another important option. It is broader than WebLLM and can run many transformer architectures in-browser.

Advantages:

- broad Hugging Face ecosystem;
- supports many tasks beyond chat/LLMs;
- WebGPU support;
- useful if the project later grows beyond text generation.

For this particular POC, that generality may make WebLLM the simpler direct choice if all we need is chat.

### Browser AI provider for Vercel AI SDK

A particularly interesting option is the community [Browser AI provider](https://ai-sdk.dev/providers/community-providers/browser-ai) for the Vercel AI SDK. It aims to expose browser inference engines such as WebLLM and Transformers.js through the same AI SDK model interface used for hosted providers.

If this works cleanly in a purely static browser application, it could eliminate a substantial part of the glue layer we originally expected to need. This deserves an early spike.

## Hosted providers directly from the browser

A backend-less application can only call a hosted model provider if the provider permits browser-originated requests and has an authentication mechanism compatible with public client code.

There are two broad approaches.

### User-supplied API key (BYOK)

The simplest POC experience is:

1. user chooses a provider;
2. application asks for an API key;
3. key exists only in the browser;
4. browser sends requests directly to the provider.

This is intentionally different from embedding an application owner's API key in JavaScript. A static application **cannot keep a secret**. Any credential shipped with it must be considered public.

Even with BYOK, the UI should clearly explain that the key is being made available to JavaScript running on the page. For an initial POC, keeping the key only in memory is preferable to persisting it in `localStorage`.

Some official SDKs deliberately make browser use opt-in because exposing an API key to browser code is dangerous in the usual application architecture. For example, the OpenAI JavaScript SDK has `dangerouslyAllowBrowser`, and Anthropic has an analogous browser opt-in. Those flags are warnings rather than security mechanisms.

For this project the distinction matters: **the user's own temporary BYOK key in a locally/self-hosted static application is a different threat model from a developer accidentally shipping their private production key to every visitor.** It still needs explicit warnings and careful handling.

### OAuth / delegated authorization

OAuth or another delegated authorization flow would be preferable where a provider genuinely supports a browser/public-client flow for model API access. It avoids asking the user to paste a long-lived secret into the application and could potentially integrate with existing user entitlements.

However, support varies considerably between providers and products. Consumer chat subscriptions generally should not be assumed to grant API access. Provider-by-provider research is needed before making OAuth part of the POC.

For the first implementation, BYOK is likely the shortest route to proving remote-provider access.

## Provider abstraction: avoid writing it ourselves if possible

The initial discussion started from the observation that all of the technical building blocks exist, so application code ought to be very small. The ideal dependency would provide roughly:

```js
const model = selectModel(...)
const result = await model.generate(messages)
```

regardless of whether `model` ultimately means a browser-local WebGPU model or a hosted API.

Writing `fetch()` adapters for every provider would be straightforward, but it is not the interesting part of this project and would immediately create maintenance work. The POC should therefore first test existing libraries.

## Vercel AI SDK

[Vercel AI SDK](https://ai-sdk.dev/) is currently the most interesting general abstraction to investigate first.

Why it is attractive:

- common APIs for multiple model providers;
- official/community provider packages;
- streaming and message abstractions already solved;
- supports major hosted providers;
- Browser AI provider may bring WebLLM/Transformers.js under the same abstraction;
- potentially allows the actual application to remain extremely small.

Questions to answer with a spike:

- Can the relevant AI SDK pieces run cleanly in a static browser build without any server-side runtime assumptions?
- Which provider packages themselves permit browser execution?
- How much bundling/build tooling is required?
- Does Browser AI behave similarly enough to hosted providers for the common chat path to remain genuinely simple?
- Does the resulting bundle remain reasonable for a deliberately tiny demo?

The AI SDK is often demonstrated in full-stack applications, so we should verify browser-only behavior rather than infer it from server-oriented examples.

## Direct official SDKs

A fallback is to use official provider SDKs directly in the browser where supported, for example OpenAI or Anthropic with their explicit browser opt-ins.

Advantages:

- little uncertainty about provider-specific features;
- less abstraction to debug;
- easy to understand during a proof of concept.

Disadvantages:

- application owns the normalization layer;
- adding providers creates more glue code;
- streaming/error handling/model naming differ;
- browser-local models become a separate integration path.

This is a good fallback but not the preferred first architecture if an existing common abstraction works.

## Plain `fetch()`

Calling provider REST APIs directly is the lowest-level fallback.

It has essentially zero dependency cost and makes CORS behavior obvious, but it maximizes provider-specific application code. It therefore works against the goal of demonstrating how little glue is necessary.

Use it for diagnosis or for a provider whose SDK causes browser problems, not as the default architecture.

## Candidate first implementation

The first technical spike should try this stack:

```text
Static app
  |
  +-- Vercel AI SDK common API
        |
        +-- hosted provider package (BYOK)
        |
        +-- second hosted provider package (BYOK)
        |
        +-- Browser AI provider
                |
                +-- WebLLM
```

If the Browser AI provider introduces friction, simplify to:

```text
Static app
  |
  +-- Vercel AI SDK --> hosted providers
  |
  +-- WebLLM --------> browser model
```

Only if the AI SDK itself proves awkward in a pure browser environment should we fall back to official provider SDKs or tiny handwritten adapters.

## Suggested POC scope

The first working version should resist feature creep. A reasonable target is one page containing:

- model/provider selector;
- API-key field shown only when required;
- tiny transcript area;
- prompt input;
- Send button;
- loading/model-download status;
- streamed output if it comes essentially for free from the selected library.

A good initial model matrix would be:

| Source | Purpose |
| --- | --- |
| One small WebLLM model | Prove fully local browser inference |
| Anthropic | Prove direct hosted-provider BYOK |
| OpenAI or another CORS-capable provider | Prove that hosted providers are interchangeable |

Exact providers and model names should be selected during implementation based on current browser/CORS support rather than hard-coded into the architecture document.

## What the POC intentionally does not need

Do not initially add:

- backend services;
- accounts;
- database or conversation persistence;
- complex settings;
- provider plugin framework;
- agent/tool execution;
- MCP;
- RAG;
- authentication owned by this application;
- automatic provider/model discovery;
- sophisticated key management;
- production security guarantees;
- elaborate styling;
- a generalized framework intended for npm publication.

Those may become interesting later, but they obscure the experiment now.

## Static HTML does not necessarily mean zero build step

There are two separate goals which should not be conflated:

1. **The deployed application is static and needs no backend.**
2. **The source consists of one hand-written HTML file with CDN script tags and no build tooling.**

The first is a core requirement. The second is optional.

If Vite or another tiny build setup makes npm dependencies, modules, workers, and WebGPU libraries dramatically easier to use, that is a reasonable trade. The output can still be ordinary static HTML/JS/CSS deployable anywhere.

For the earliest experiment, prefer the smallest setup that the chosen libraries support reliably.

## Security model

A browser-only BYOK application makes several security properties unusually visible.

### There are no application secrets

Anything included in the static deployment is public. Never include an application-owned provider key.

### User keys are exposed to the page

A pasted API key can be read by JavaScript executing in that page. This makes dependency integrity and XSS important even for a small demo.

The initial version should therefore:

- keep keys in memory where practical;
- avoid analytics and unrelated third-party scripts;
- avoid logging credentials;
- avoid transmitting keys anywhere except the selected provider;
- clearly tell the user what will happen;
- make source code small enough to inspect easily.

A self-hosted/downloadable application has a useful property here: technically inclined users can inspect exactly what they are running.

### CORS is not a security mechanism for the key

CORS determines whether browser JavaScript can read a provider's response. It does not make a key embedded in a static app secret.

## Model download target

One motivating constraint discussed for local inference was an ordinary Windows laptop and roughly a 100 Mbit/s connection, with a desirable initial model download of under about one minute.

At 100 Mbit/s the theoretical maximum transferred in one minute is about 750 MB; real-world throughput and model initialization reduce the practical budget. This points toward a small quantized model rather than the multi-billion-parameter models often used in WebLLM demos.

For the POC, responsiveness and accessibility matter more than model quality. We can later expose larger local models as optional choices.

## Questions the prototype should answer

The prototype is successful if it gives concrete answers to these questions:

1. Can one static application genuinely support both browser-local and remote hosted LLMs?
2. Which mainstream providers can currently be called directly from browsers?
3. How safe and understandable can BYOK be made without a backend?
4. Can an existing JS abstraction hide most provider differences?
5. Can that same abstraction include WebGPU inference?
6. How small can the application-specific code actually become?
7. What is the smallest local model that still makes the demo feel useful on an average laptop?
8. Are browser compatibility and CORS limitations small enough that this is useful beyond a technical demo?

## First milestone

The first milestone should be intentionally narrow:

> **A static page where a user can select either one local WebGPU model or one hosted model, type a message, and receive a response through the same UI.**

Implementation order:

1. Establish the smallest static project/build setup.
2. Spike Vercel AI SDK + Browser AI/WebLLM.
3. Make one small local model answer a prompt.
4. Add one hosted provider with a user-entered API key.
5. Put both behind one model selector and one chat path.
6. Add a second hosted provider to verify that the abstraction is genuinely reusable.
7. Measure the amount of application-specific code and simplify it.

Only after this works should we decide whether the experiment deserves a reusable architecture or a real product name.

## Naming

`multi-agent-demo` is intentionally a working repository name, not a product decision.

Names explored during brainstorming included concepts around:

- `omni`, `any`, `one`, and `poly`;
- models, minds, assistants, and agents;
- plugging, routing, gates, bridges, docks, and multiplexing;
- `OmniMux` and `AI Mux`;
- science-fiction-inspired terminology.

None felt compelling enough to justify choosing a permanent name before the technical idea is proven. Keeping the working name boring is therefore a feature rather than a problem.

## Longer-term possibilities

If the POC works unusually well, several directions become interesting:

- a tiny reusable browser library exposing local and remote models uniformly;
- more providers and local runtimes;
- OAuth/delegated provider authentication where genuinely available;
- model capability metadata and filtering;
- installable/PWA operation;
- fully offline mode for local models;
- local conversation storage;
- user-defined providers compatible with OpenAI-style APIs;
- agent/tool support;
- a polished self-hostable universal LLM chat client.

Those are deliberately possibilities, not current requirements.

## Guiding principle

The project should optimize for one surprising demonstration:

> **One tiny static web application, no backend, one chat interface — choose an LLM wherever it happens to live.**

If the implementation becomes complicated before proving that statement, simplify it.