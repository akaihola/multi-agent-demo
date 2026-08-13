# Extension Architecture Research

## Why this deserves its own design step

The current direction is to make the application a **tiny browser host in which as many features as practical are extensions**, including built-in features.

That makes the extension mechanism foundational. It should therefore be researched and deliberately minimized before the rest of the application grows around it.

The objective is not to invent a novel plugin framework. The objective is to identify the smallest proven set of ideas that gives us:

- a stable host/extension boundary;
- easy-to-understand lifecycle;
- explicit registration of capabilities;
- deterministic cleanup;
- multiple kinds of contributions (models, tools, UI, diagnostics);
- future import/export of extensions;
- a plausible path to LLM-generated extensions;
- no build-system or framework requirement;
- implementation in ordinary browser JavaScript and ES modules.

## Research references

### VS Code: activation + contribution points + host API

VS Code extensions have a manifest, an entry point, activation, contribution points, and a host-provided API. Its production system is vastly larger than this project needs, but three ideas transfer well:

1. **Extensions declare/contribute capabilities instead of mutating internals.**
2. **Activation has an explicit lifecycle.**
3. **The host owns a stable API rather than exposing its internal implementation.**

For this project, avoid VS Code's marketplace/package/dependency/compatibility machinery initially. We want the architectural kernel, not the ecosystem infrastructure.

References:

- https://code.visualstudio.com/api/references/extension-manifest
- https://code.visualstudio.com/api/references/activation-events
- https://code.visualstudio.com/api/get-started/extension-anatomy
- https://code.visualstudio.com/api/advanced-topics/extension-host

### Obsidian: registration ownership and cleanup

Obsidian plugins expose a simple lifecycle and provide registration helpers for commands, events, views, intervals, and other contributions. A particularly useful property is that registrations belong to the plugin lifecycle and can be cleaned up when it unloads.

That suggests a strong rule for this project:

> Anything an extension registers should be represented by a disposable owned by that extension, and unloading the extension should dispose all of them.

References:

- https://docs.obsidian.md/Plugins/Getting+started/Build+a+plugin
- https://docs.obsidian.md/Plugins/Events
- https://docs.obsidian.md/plugins/guides/load-time

### pluggy / pytest: narrow contracts beat monkey-patching

Python's `pluggy` is the crystallized plugin mechanism behind pytest. It separates a host that specifies hooks from plugins that implement them, with a registry mediating calls.

The most relevant lesson is not its Python decorators. It is that a plugin architecture can be small when the host carefully chooses the objects and hooks it exposes. This produces loose coupling without exposing arbitrary internal state.

`pluggy` also demonstrates useful concepts to keep in mind later — multiple implementations, first-result hooks, call ordering, historic hooks, validation and tracing — but we should **not implement these until an actual use case requires them**.

Reference:

- https://pluggy.readthedocs.io/en/stable/

### The browser already provides most runtime primitives

We should resist installing a JavaScript plugin framework until we have demonstrated that browser primitives are insufficient.

Useful native pieces include:

- ES modules and dynamic `import()` for trusted code loading;
- `EventTarget` / `CustomEvent` for simple notifications;
- `AbortController` / `AbortSignal` for lifecycle cancellation;
- `Blob` URLs where runtime-created module resources are useful;
- Web Workers for isolated non-DOM execution;
- sandboxed iframes for stronger browser-origin isolation patterns;
- `postMessage()` for capability-oriented communication across isolation boundaries;
- IndexedDB for persistent extension packages;
- DOM elements and `<template>` for UI contributions.

The extension API should compose these primitives rather than replacing them with an elaborate framework.

## Relationship to the v0 implementation contract

This document explains the design rationale and longer-term possibilities. For Stage 0, the normative source is `docs/implementation-contract-v0.md`. If an example here differs from the contract, follow the contract and repair this document.

## Recommended conceptual model

Use three concepts:

```text
HOST
  owns lifecycle, registry, permissions, storage, contribution points

EXTENSION
  activates against a small host API and registers contributions

CONTRIBUTION
  a provider/model/tool/action/view/etc. registered with the host
```

The v0 lifecycle is frozen in `docs/implementation-contract-v0.md`:

```js
export const manifest = {
  id: 'example.extension',
  name: 'Example extension',
  version: '0.1.0',
  apiVersion: 1
};

export async function activate(api) {
  api.models.register(/* contribution */);

  return () => {
    // optional cleanup for resources not registered through the host
  };
}
```

`activate()` may be synchronous or asynchronous. The host owns every registration and disposes them in reverse registration order on unload. If activation throws, the host rolls back all partial registrations before reporting the failure. Registration disposal is idempotent. Duplicate extension IDs and duplicate model IDs are rejected rather than replaced.

## Registration API rather than hook soup

For the initial application, a **registry/contribution API** looks simpler than a generalized hook system.

For example:

```js
api.models.register(model)
api.models.list()
api.models.get(id)
api.models.subscribe(listener)
api.models.invoke(id, input)
api.ui.registerView(slot, view)
```

The registry needs both producer- and consumer-facing operations. The prompt UI must be able to list models, retrieve public descriptors, observe registration/removal, and invoke a selected model without receiving its private implementation function. v0 implements only `models` and the `main` UI slot; tools, events, storage, commands, and additional UI slots are later additions.

Each registration should return a tiny disposable:

```js
const registration = api.models.register(model);
registration.dispose();
```

The host tracks these registrations under the active extension and automatically disposes them when the extension unloads.

This is easier to reason about than arbitrary hooks because most current requirements are additive: "this extension contributes a model/tool/action/view" rather than "this extension intercepts and rewrites application behavior".

If genuine interception/composition requirements appear later, add a hook/middleware concept then.

## Keep the host API capability-oriented

Do not hand extensions an `app` object containing all internal state.

Prefer small namespaces such as:

```js
api.models
api.tools
api.ui
api.events
api.storage
api.commands
```

An extension should only see the capabilities it needs. This helps comprehension immediately and leaves room for a future permission/sandbox system.

The API object can even be constructed per extension so capabilities can later be withheld or proxied.

## Candidate contribution types

Do not implement all of these immediately. They are a map of what naturally fits the architecture.

### Models

A model contribution describes an invokable language model.

The canonical v0 shape is:

```js
{
  id: 'example:model',
  label: 'Example model',
  providerId: 'example',
  providerLabel: 'Example',
  credential: null,
  generate: async ({ prompt, credential, signal }) => ({ text: '...' })
}
```

The completed result is always `Promise<{ text: string }>`; returning a bare string is invalid. Public descriptors returned by `list()` and `get()` omit `generate`. The prompt UI invokes through `api.models.invoke()`, and the host routes the supplied credential only to the selected model implementation.

The first contract is optimized for **one prompt / one completed response**. Tools are added later. Incremental token/chunk display is a later-version capability and does not complicate v0.

### Providers

v0 has **no provider registry**. Extensions register models, and each model carries simple `providerId`, `providerLabel`, and credential-field metadata.

The second hosted integration should reveal whether providers have a real shared lifecycle or credential concern. Introduce a separate provider contribution type only when that need is demonstrated; do not create two abstractions in anticipation.

### Tools

Later:

```js
{
  name: 'example_tool',
  description: '...',
  inputSchema: {...},
  execute: async (input, context) => result
}
```

Tools should be ordinary contributions, not a special agent framework.

Tools can precede retained conversation state. A single visible run may keep temporary invocation state for a model → tool → model loop and then discard it after rendering one final response. Persistent messages, a transcript, and multi-turn state remain a separate later feature.

### Commands/actions

A command registry can decouple functionality from presentation:

```js
api.commands.register({
  id: 'extensions.export',
  title: 'Export extensions',
  run: async () => {...}
});
```

A UI extension can then render buttons/menu entries for commands without owning the underlying behavior.

This is a useful VS Code idea at a much smaller scale.

### UI

UI extensibility is where plugin APIs tend to become complicated. Start with named slots/contribution points rather than a virtual component framework.

v0 implements one contribution point:

- `main` — the primary application view, filled by the built-in prompt UI extension.

Possible later contribution points include `toolbar`, `settings`, and `status`.

A contribution might provide a function that receives a container element and returns cleanup:

```js
api.ui.registerView('main', {
  id: 'prompt',
  mount(container) {
    // ordinary DOM code
    return () => { /* cleanup */ };
  }
});
```

Do not expose arbitrary selectors into the host DOM as the official API.

### Events

Use events for observation, not as the only API for everything.

Examples:

- extension activated/deactivated;
- model registered/unregistered;
- generation started/completed/failed;
- tool registered/invoked.

A browser-native `EventTarget` wrapper may be sufficient. If every operation turns into an event protocol, the architecture has gone too far.

### Storage

Built-in extensions may need preferences. Future user extensions need persistent data.

Expose namespaced storage rather than raw shared IndexedDB/localStorage keys:

```js
api.storage.get('key')
api.storage.set('key', value)
api.storage.delete('key')
```

The host can prefix/isolate data by extension ID and later change the underlying storage mechanism without changing extensions.

Credentials should **not** go through generic extension storage in the initial design.

## Manifest: start tiny

A runtime extension package will eventually need metadata, but avoid designing an npm-like package format now.

The frozen v0 minimum is:

```js
{
  id: 'author.extension-name',
  name: 'Extension name',
  version: '0.1.0',
  apiVersion: 1
}
```

Possible future fields:

- description;
- requested permissions;
- entry point;
- integrity/hash/signature;
- dependencies;
- homepage/source/provenance;
- generated-by metadata.

Only add fields when the runtime actually uses them.

## Built-ins should dogfood the public API

This is the most important architectural discipline.

If built-in providers, tools, commands, and optional UI features bypass the public extension API, that API will remain theoretical and third-party extensions will always be second-class.

Therefore:

- core host code must not know individual model providers;
- built-in provider integrations register through the public model/provider API;
- built-in tools register through the public tool API;
- diagnostics register through the same UI/event APIs;
- tests should load tiny fake extensions through the same loader.

The v0 host provides only the DOM shell and error fallback. The prompt UI is definitively a built-in extension registered in the `main` slot.

## Runtime-generated extensions

The longer-term experiment is unusually interesting:

1. the application can expose its extension API documentation to a language model;
2. the model proposes JavaScript implementing an extension;
3. the user can inspect the code;
4. the extension can be stored locally;
5. the user explicitly activates it;
6. the extension can be exported as text and imported into another instance.

This turns extension source code itself into a portable artifact.

The important architectural choice is that generated code extends the app **through the same public API**. The model should not need to patch core source code or depend on private DOM structure.

## Do not make `eval()` the plugin system

Executing arbitrary JavaScript with `eval()` or `new Function()` in the page's main realm gives that code essentially the same authority as the application itself. It can access the DOM, monkey-patch globals, inspect in-page secrets, and bypass the intended API.

For trusted local experiments this may be acceptable as an explicit "unsafe" mode, but it is not an isolation boundary.

For imported or generated code, research these options:

### Web Worker

Advantages:

- separate global realm;
- no direct DOM access;
- natural message-passing boundary;
- good for providers, tools, transformations, and computation.

Disadvantages:

- UI extensions need a declarative/proxied UI API or a different execution mode;
- not a complete security sandbox if powerful network/storage capabilities are handed through carelessly.

### Sandboxed iframe

Advantages:

- browser-enforced document isolation options;
- can host UI;
- communicates through `postMessage()`.

Disadvantages:

- more lifecycle/layout complexity;
- permissions and CSP/origin details need careful design.

### Trusted same-realm ES module

Advantages:

- by far the simplest;
- native modules and DOM access;
- ideal for built-ins and explicitly trusted developer extensions.

Disadvantages:

- no meaningful isolation from the host.

A sensible long-term design may support more than one trust tier rather than forcing all extensions through the same runtime.

## Import/export format

The simplest portable format may initially be a single JavaScript module containing its manifest and `activate()` export.

That has attractive properties:

- human-readable;
- easy to copy/paste;
- easy for an LLM to generate;
- no archive/package tooling;
- native module semantics;
- source itself is the artifact.

If metadata must be inspected before code execution, either use a tiny separate JSON envelope or a constrained metadata header. Do not parse arbitrary JavaScript just to discover permissions.

This decision can wait until runtime extensions are implemented.

## API versioning

Because generated/shared extensions may persist for a long time, the API needs a simple compatibility story. Stage 0 accepts only integer `apiVersion: 1`.

Start with an integer `apiVersion`. Prefer additive API evolution. New namespaces/functions should not break old extensions.

Do not build semver range negotiation or compatibility shims until there is a real second API version.

## Failure isolation

Even trusted extensions will throw errors.

The host should:

- catch activation errors;
- identify the extension responsible;
- continue loading unrelated extensions where possible;
- dispose partial registrations after failed activation;
- catch/report errors at extension invocation boundaries;
- provide enough diagnostics to disable a broken user extension.

The extension manager itself should be one of the most heavily tested pieces of the application.

## Testing implications

The extension architecture makes excellent deterministic tests possible.

Create tiny test extensions that:

- register a model;
- register two contributions;
- throw during activation;
- register then unload;
- attempt duplicate IDs;
- listen to an event;
- use namespaced storage;
- mount/unmount a UI contribution.

Playwright can verify the host and UI while small JavaScript-level tests can verify registry semantics if useful. Avoid a large test framework unless it earns its place.

## What not to build yet

Do not implement these in the first extension-host milestone:

- marketplace;
- dependency resolver;
- remote extension catalog;
- signing infrastructure;
- package manager;
- generalized hook ordering;
- middleware chains;
- extension-to-extension imports;
- semantic-version dependency resolution;
- background auto-update;
- LLM-generated code execution;
- sophisticated permission prompts;
- full sandboxed UI framework.

The first milestone needs only enough architecture to prove that built-in features can genuinely dogfood a tiny extension API.

## Proposed first spike

Before implementing any LLM provider, build the exact Stage 0 walking skeleton from `docs/implementation-contract-v0.md`:

```text
index.html
  -> loads app.js
  -> app explicitly imports two built-in ES-module extensions

fake-model extension
  -> registers one deterministic model

prompt-ui extension
  -> registers the main view
  -> discovers the fake model through the registry
  -> sends one prompt through models.invoke()
  -> renders one completed response
```

Then ask:

- Is the entire extension host still understandable in one sitting?
- Can an extension be explained with a ~20-line example?
- Does unloading leave no residue?
- Can a second contribution type be added without redesign?
- Are host internals hidden enough that extensions cannot accidentally depend on them?
- Did we invent anything the browser already provides?

If the answers are good, only then add real provider extensions.

## Current recommendation

Do **not** adopt a large existing JavaScript plugin framework yet. Adopt the proven *shape* seen across VS Code, Obsidian, and pluggy, implemented with browser primitives:

> **manifest + `activate(api)` + explicit registries/contribution points + disposables + deterministic unload**

That is small enough to understand, flexible enough for the known roadmap, and naturally compatible with future portable or LLM-generated extensions.

The burden of proof should be on every additional abstraction.