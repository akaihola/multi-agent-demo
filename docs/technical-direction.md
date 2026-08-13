# Multi-Agent Demo: Current Technical Direction

## Purpose

This repository is a deliberately small proof of concept for a **static, browser-only application that can run a prompt against language models regardless of where they run**.

The initial demonstration is intentionally smaller than a chat application:

> Open a static page, choose a model, enter one prompt, run it, and receive one response — with the model either running locally in the browser or provided by a remote LLM API using credentials supplied by the user.

The application has no backend of its own. It should remain easy to inspect, copy, self-host, modify, and understand.

Despite the repository's working name, **multi-agent behavior is explicitly out of scope for the initial demo**. It may be explored in a later version.


## Core constraints

Preserve these constraints aggressively:

- **No application backend.** No proxy, serverless function, edge worker, token broker, or application server.
- **Static deployment.** GitHub Pages or any ordinary static web server should be sufficient.
- **Browser execution.** Application logic executes in the user's browser.
- **Browser-native implementation.** Prefer ordinary HTML, CSS, JavaScript, Web APIs, and ES modules.
- **No mandatory build step unless a concrete dependency makes it worthwhile.**
- **Multiple model sources.** Demonstrate more than one hosted provider and at least one browser-local model.
- **Bring your own credentials.** Remote-provider credentials belong to the user, never to the deployed application.
- **Minimal UI and minimal glue.**
- **One prompt / one response first.** Do not begin with conversation state.
- **Completed responses only in the staged plan.** Incremental token/chunk display is a later-version capability, outside the current implementation sequence.
- **No multi-agent orchestration in the initial demo.**
- **Extension-first architecture.** Features that naturally fit the extension model should use the same public extension API as third-party extensions.

The last point supersedes the earlier "no premature plugin abstraction" rule. The extension mechanism is now itself one of the central experiments, but it must remain extremely small.

## Guiding architecture: a tiny host plus extensions

The application should be divided conceptually into a **small trusted host** and **extensions**.

The host owns only capabilities that must exist in order for extensions to exist:

```text
Static application host
  |
  +-- extension discovery / loading / lifecycle
  +-- extension registry
  +-- tiny public host API
  +-- activation, rollback, and deterministic cleanup
  +-- model and UI registries
  +-- minimal shell / contribution points
          |
          +-- UI extensions
          +-- model/provider extensions
          +-- local-model extensions
          +-- tool extensions
          +-- diagnostics extensions
          +-- future user/LLM-generated extensions
```

The host should not become an internal application framework. Its job is to provide stable primitives and registration points.

A useful design test is:

> Could a built-in feature be removed from the repository and loaded as an ordinary extension without changing its implementation substantially?

If yes, it probably belongs behind the extension API.

## What should be extensions from the beginning?

### Model/provider integrations

Hosted providers are natural extensions. An extension can contribute one or more model definitions and the function used to invoke them.

This means OpenAI, Anthropic, Google, OpenRouter, or a provider abstraction library can be integrated without teaching the core about each vendor.

The provider bake-off remains useful: ClientAgentJS, Vercel AI SDK, BundleLLM, `@livx.cc/ask`, official SDKs, and plain `fetch()` can be evaluated as implementation strategies *inside* model extensions rather than as architectures for the entire application.

For v0 there is no separate provider registry. A model descriptor carries simple provider metadata. Add a provider abstraction only if the second hosted integration demonstrates a concrete shared lifecycle or credential requirement.

### Browser-local model runtimes

WebLLM and/or Transformers.js should appear through the same model contribution surface where practical. Local versus hosted should be metadata/capability of a model, not a separate application mode.

A local model extension may additionally expose loading progress, compatibility information, cache state, and model download size. Before Stage 3 begins, record the exact model, runtime version, compressed download size, supported browsers, and measured first-load time.

### UI contributions

The host should expose a very small number of deliberate UI contribution points rather than allowing arbitrary core DOM mutation as the primary API.

Examples might eventually include:

- main content/view;
- settings section;
- toolbar/action;
- status/diagnostics panel.

The initial prompt/result UI **is a built-in extension**. It registers the main view through the same public UI contribution API available to other trusted extensions.

### Tools

Tools should use the extension mechanism from their first appearance. A tool extension contributes a name, description/schema, and implementation callable by a model or by the UI.

This is deliberately postponed until after hosted providers and the first browser-local model work.

### Diagnostics and developer tooling

Logging, request/event inspection, model metadata display, and an extension inspector are excellent extension candidates. They should not require special architecture in the core beyond the same event and contribution APIs available to other extensions.

### Persistence-backed user extensions

A later experimental feature may allow JavaScript extensions to be created at runtime, persisted in IndexedDB or localStorage, exported as text, and imported into another instance of the application.

This creates the path to the more unusual experiment: allowing an LLM to inspect the application's documented extension API and generate a new extension for the user to review and activate.

This is **not** the same as allowing the model to rewrite the trusted core. The preferred model is self-extension through a stable API, not unconstrained self-modification.

## What should stay in the host?

Keep the host intentionally boring.

For v0, host responsibilities are limited to:

- explicitly loading and unloading built-in extension modules;
- validating the frozen v0 manifest and contribution shapes;
- maintaining model and UI registries;
- handing each extension a small host API;
- rolling back partial registrations when activation fails;
- cleaning up registrations deterministically when an extension unloads;
- providing a minimal DOM shell and the named `main` contribution point.

Persistence, import/export, user-extension enablement, permission prompts, and isolated runtimes are possible **later host responsibilities**, not v0 work.

Provider-specific behavior, tools, model catalogs, application commands, optional panels, and experiments should not accumulate in the host.

## Extension API research direction

We should **adopt established extension-system ideas rather than inventing an elaborate framework**.

Useful reference architectures include:

- **VS Code:** manifest + contribution points + activation + a stable host API. The useful idea is declarative contributions and a small activation lifecycle, not VS Code's enormous API or packaging machinery.
- **Obsidian:** a straightforward plugin lifecycle where a plugin loads, registers capabilities/listeners, and unloads with cleanup. The useful idea is registration ownership and automatic cleanup.
- **pluggy / pytest:** a tiny host/plugin distinction with explicit hook specifications and registered implementations. The useful idea is a deliberately narrow contract between host and plugins.
- **Web platform primitives:** native ES modules, dynamic `import()`, `EventTarget`/`CustomEvent`, `AbortController`, Web Workers, sandboxed iframes, `postMessage`, IndexedDB, and the DOM should be preferred over creating equivalents.

The likely shape is intentionally tiny, conceptually similar to:

```js
export const manifest = {
  id: 'example.extension',
  version: '0.1.0'
};

export function activate(api) {
  const disposable = api.models.register({
    id: 'example:model',
    label: 'Example model',
    generate: async ({ prompt }) => ({ text: '...' })
  });

  return () => disposable.dispose();
}
```

This is a sketch, not yet a frozen API. The key ideas are:

1. one obvious entry point;
2. a tiny capability-oriented `api` object;
3. explicit registration rather than monkey-patching;
4. registrations return disposables/cleanup handles;
5. unload is deterministic;
6. the API can grow by adding namespaces/capabilities without exposing core internals.

See `docs/extension-architecture.md` for the focused research and design criteria.

## Security and trust boundary

The extension idea changes the threat model substantially.

### Built-in extensions

Built-ins shipped with the static application are part of the trusted application distribution even though they use the public extension API.

### Imported or LLM-generated extensions

Arbitrary JavaScript running in the page's main realm can read DOM state, credentials, and other extension data. Therefore imported or generated extensions must **not** be described as sandboxed merely because they use an extension API.

A later extension runtime should distinguish at least:

- trusted built-in code;
- explicitly user-trusted imported code;
- untrusted/generated code that should ideally execute in a more constrained realm.

Web Workers or sandboxed iframes plus message passing are likely candidates for genuinely untrusted extensions, but they restrict direct DOM access and complicate the API. That trade-off should be researched before runtime-generated JavaScript is enabled.

Do not use `eval()` as the architecture. Dynamic execution may be an implementation detail for a trusted experimental mode, but the architecture should be based on extension packages, capabilities, and lifecycle.

### Credentials

Remote API keys should initially remain in memory. Never expose credentials through the generic extension API. The prompt UI passes a credential only when invoking the selected model; the host routes it to that model's implementation and never places it in registry descriptors, events, storage, URLs, or logs. The exact v0 invocation contract is defined in `docs/implementation-contract-v0.md`.

This is API-level discipline, not a same-realm security boundary. Trusted built-in modules and loaded third-party dependencies execute in the same page and can technically inspect DOM and JavaScript state. Real isolation for imported code is later research.

## Browser-local models

WebLLM remains the strongest first candidate for local inference because it is specifically designed for browser LLM execution with WebGPU, model loading/caching, and an OpenAI-like interface.

Transformers.js remains a useful alternative, particularly if the project later explores tasks beyond chat/text generation.

A community Browser AI provider for Vercel AI SDK may allow local inference to sit behind the same abstraction as hosted models and remains worth a spike.

For the POC, use a genuinely small quantized model. The goal is architectural proof, not frontier-model quality. On an ordinary laptop and roughly 100 Mbit/s connection, a first-load target comfortably below a minute strongly favors a download well below the theoretical ~750 MB/minute maximum.

## Hosted providers and BYOK

A backend-less application can call a hosted provider only where browser-originated requests and authentication are compatible with the browser security model.

The initial approach is BYOK:

1. choose provider/model;
2. enter the user's credential if required;
3. keep it in memory;
4. send the request directly from browser to provider;
5. display the response.

A static application cannot keep an application-owned secret. Browser compatibility must be tested with real requests; an SDK claiming browser compatibility does not automatically mean the provider's API supports the necessary CORS behavior.

OAuth/delegated authorization may be explored later where a provider genuinely supports an appropriate public-client flow.

## Provider abstraction bake-off

Do not prematurely choose the largest framework. Implement the same one-prompt/one-response provider extension using a few candidate approaches and compare the resulting code.

Primary candidates:

- **ClientAgentJS** — browser-first and direct-BYOK oriented;
- **Vercel AI SDK** — mature common model APIs and potentially a route to hosted and browser-local models;
- **BundleLLM** — useful browser-native BYOK/OAuth comparison;
- **`@livx.cc/ask`** — broad advertised provider coverage, pending real browser/CORS verification;
- official provider SDKs — fallback;
- plain `fetch()` — diagnostic baseline and fallback.

Before a candidate enters the repository, verify its canonical source and browser support, record the source URL and tested version in a decision note, and pin the deployed version. The current verified local-runtime reference is [WebLLM](https://github.com/mlc-ai/web-llm). Do not copy an unverified package name into production code merely because it appears in this research list.

Choose primarily on:

- static-browser compatibility;
- no-build / browser-ESM friendliness;
- amount of glue code;
- hosted provider coverage;
- fit with WebLLM/local inference;
- error normalization;
- whether its non-streaming API stays small and does not prevent later incremental output;
- package/dependency weight;
- ease of wrapping as an extension without leaking library concepts into the host.

## Staged implementation plan

`docs/implementation-contract-v0.md` is normative for Stage 0. `AGENTS.md` is the entry point for implementation work.

Each stage should leave a small, runnable application. Introduce one major kind of complexity at a time.

### Stage 0 — Fake-model walking skeleton

Implement the frozen contract in `docs/implementation-contract-v0.md`:

- the host explicitly imports two built-in ES-module extensions;
- a fake-model extension registers one deterministic model;
- the prompt UI extension discovers it through the registry;
- one prompt produces one completed response;
- activation rollback, duplicate rejection, and unload cleanup work;
- Playwright verifies the flow in Chromium and a phone-sized viewport.

Do **not** solve runtime-generated extensions, permissions, marketplaces, dependency graphs, sophisticated manifests, or real provider access here.

### Stage 1 — One hosted model, one prompt, one response

Add one provider/model as an extension.

UI:

- provider/model selection only as needed;
- API key input;
- prompt textarea;
- Run button;
- result area;
- loading and error state.

No transcript, retained conversation state, tool loop, or incremental response display.

### Stage 2 — Multi-provider

Add a second hosted provider through the same extension surface and run the provider-abstraction bake-off.

This stage should prove that provider interchangeability is real rather than an interface designed around the first provider.

### Stage 3 — Browser-local model

Add one WebGPU/local model through the same model contribution API.

Prove:

- compatibility detection;
- model download/load progress;
- local inference;
- local and hosted models can be selected and invoked through the same user flow.

### Stage 4 — Simple tools

Introduce a very small tool contribution API and one or two browser-native tools.

Good early tools are deterministic, understandable, and easy to test. Tools should be ordinary extensions and should use the same lifecycle/registration mechanism as providers and UI contributions.

A tool-enabled run may use a transient model → tool → model loop to produce one final response. That temporary invocation state is not retained conversation history, so tools do not require a chat UI or a conversation store.

This stage is where interactions can become more interesting without introducing multi-agent orchestration.

### Stage 5 — Conversation, only if useful

If the experiments benefit from it, add conversation history and multi-turn state after the underlying model/tool architecture is proven.

Conversation is not needed to prove the initial architectural idea.

### Later-version capabilities

These are deliberately outside the current staged plan:

- incremental response display, where generated text appears token-by-token or in small chunks;
- multi-agent behavior and orchestration.

They should be designed only after the simpler application has demonstrated value. Their future implementation must not complicate the v0 model contract now.

### Later experiment — Portable and LLM-generated extensions

Only after the extension API is stable enough to be pleasant for humans:

- store user extensions in IndexedDB/localStorage;
- import/export extensions as text/packages;
- allow copying an extension between app instances;
- expose extension API documentation/source context to an LLM;
- let the model propose extension code;
- require explicit review/trust/activation;
- investigate sandboxed execution for untrusted extensions.

The fun experiment is **an application that can extend itself through the same small public API used by its built-in features**, not a model with unrestricted access to rewrite the host.

## Testing strategy

Use Playwright against the real static application.

Early tests should focus on:

- host loads;
- built-in extensions activate;
- registrations appear and disappear correctly;
- one-prompt/one-response UI behavior;
- provider selection and credential controls;
- mocked provider responses and failures;
- local-model capability fallback where practical;
- mobile viewport behavior;
- extension import/export later;
- cleanup when extensions are disabled/reloaded.

Most tests should use fake providers/extensions so they are deterministic and free. Keep only a very small optional suite for real provider APIs if it proves valuable.

## Repository structure

Do not interpret "Simon-style" as "everything must stay in one HTML file". Start with the smallest structure, but this project has natural module boundaries and may move to multiple native ES modules early without introducing a bundler.

A likely shape is:

```text
multi-agent-demo/
├── index.html
├── app.js
├── host/
│   ├── extensions.js
│   └── ...                 # only when genuinely needed
├── extensions/
│   ├── prompt-ui.js
│   ├── provider-*.js
│   └── local-*.js
├── docs/
│   ├── technical-direction.md
│   ├── simon-willison-html-tool-guidelines.md
│   └── extension-architecture.md
└── tests/
```

Keep files coarse-grained. Splitting into ES modules is for clarity, not an invitation to recreate a framework directory tree.

## Explicitly out of scope for the initial demo

- multiple agents;
- agent orchestration/delegation;
- backend services;
- accounts/database;
- RAG;
- MCP unless a later tool experiment specifically justifies it;
- sophisticated persistent chat history;
- production-grade secret management;
- extension marketplace/discovery service;
- extension dependency resolution;
- automatic execution of arbitrary LLM-generated code;
- polished design system;
- packaging this as a generalized npm framework.

## Questions the prototype should answer

1. Can a tiny static host support useful features almost entirely through one understandable extension API?
2. Which responsibilities truly must remain in the host?
3. Can hosted and browser-local models share one model contribution contract?
4. Which browser-capable provider library minimizes glue without infecting the host API with its own abstractions?
5. Can tools and their transient invocation loops be added as ordinary extensions without requiring retained conversation state?
6. Is the extension API small enough that a human can understand it in minutes and an LLM can generate correct extensions from a short specification?
7. What isolation model is practical for imported/generated JavaScript extensions?
8. How small can the total application-specific code remain?

## Guiding principle

The project should optimize for two surprising demonstrations, in order:

> **One tiny static web application, no backend — choose an LLM wherever it happens to live.**

Then, if the extension experiment succeeds:

> **Almost everything in that application is an extension using the same tiny public API — including extensions the user can eventually import, export, or ask a model to create.**

If either demonstration requires a complicated framework before it works, simplify the architecture.