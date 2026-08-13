# Implementation Contract v0

## Status and authority

This document is the normative implementation contract for **Stage 0: the fake-model walking skeleton**.

“Frozen v0 contract” means the team has agreed to treat these interfaces and behaviors as a stable baseline while Stage 0 is implemented. It prevents each implementor from making a different reasonable choice. It does **not** mean the design can never change: a necessary change must be deliberate, explained in a decision note, reflected here, and reconciled with the other documents.

For Stage 0, this document takes precedence over examples in the broader design documents.

## Stage 0 outcome

A static page loads two explicitly imported built-in extensions:

1. a fake-model extension registers one deterministic model;
2. a prompt UI extension registers the main view, discovers that model through the public registry, submits one prompt through the public invocation API, and renders one completed response.

Stage 0 proves the extension boundary before any real provider or local runtime is introduced.

## Scope

Stage 0 includes:

- a tiny extension host;
- explicit loading of trusted built-in ES modules;
- manifest validation;
- activation, failure rollback, unload, and deterministic cleanup;
- one model registry with producer and consumer operations;
- one `main` UI contribution point;
- a built-in fake model;
- a built-in prompt UI;
- loading, success, validation-error, and invocation-error states;
- Playwright tests in Chromium, including a phone-sized viewport.

Stage 0 excludes:

- real hosted providers and credentials;
- browser-local model runtimes;
- tools and tool-call loops;
- retained conversation state or a transcript;
- incremental token/chunk response display;
- persistence and URL state;
- runtime-imported or generated extensions;
- permission systems, sandboxed extension runtimes, marketplaces, dependencies between extensions, and extension packaging;
- multi-agent behavior;
- a production build system or frontend framework.

## Required repository shape

Stage 0 should use this small structure:

```text
/
├── AGENTS.md
├── README.md
├── index.html
├── app.js
├── host/
│   └── extension-host.js
├── extensions/
│   ├── fake-model.js
│   └── prompt-ui.js
├── tests/
│   ├── conftest.py
│   └── test_app.py
├── requirements-dev.txt
└── docs/
    ├── implementation-contract-v0.md
    ├── technical-direction.md
    ├── extension-architecture.md
    └── simon-willison-html-tool-guidelines.md
```

Add another source file only when it creates a clear, present boundary. Do not create empty directories or placeholders for later stages.

All application URLs and imports must be relative so the same files work at the root of a local server and under a GitHub Pages project subpath.

## Runtime and browser baseline

- The application is directly served static source. There is no production build.
- Use ordinary HTML, CSS, JavaScript, native ES modules, and Web APIs.
- Stage 0 automated support is the Chromium version installed by Playwright.
- The same tests must cover a normal desktop viewport and a phone-sized viewport.
- The page must fail visibly in its main shell if bootstrapping an extension fails; operational errors must not exist only in the console.
- JavaScript source should remain usable under a strict Content Security Policy: no `eval()`, `new Function()`, inline event handlers, or `javascript:` URLs.

## Extension runtime decision

The extension/plugin system is based on **trusted same-realm ES modules**. Stage 0 built-ins and any later imported or generated extensions use this same runtime model.

The public API limits supported coupling; it does not limit what extension code can technically access. There is no sandboxed or permission-restricted extension tier in this demo. Later user-supplied source must be reviewed and explicitly trusted before activation. See `docs/decisions/2026-08-13-trusted-same-realm-extensions.md`.

## Built-in loading

Stage 0 does not perform filesystem discovery. A static browser cannot enumerate the `extensions/` directory.

`app.js` explicitly imports the two built-ins and activates them:

```js
import * as fakeModel from './extensions/fake-model.js';
import * as promptUi from './extensions/prompt-ui.js';
```

Activate the fake model before the prompt UI for a simple deterministic bootstrap. The prompt UI must nevertheless subscribe to registry changes so later registration and removal are reflected without reloading the page.

No dynamic import, remote module loading, persisted module source, or manifest scanning is part of v0.

## Extension module contract

An extension module exports exactly the concepts below:

```js
export const manifest = {
  id: 'example.extension',
  name: 'Example extension',
  version: '0.1.0',
  apiVersion: 1
};

export async function activate(api) {
  // Register contributions through api.
  // May return optional extension-owned cleanup.
}
```

### Manifest rules

The host validates:

- `id`, `name`, and `version` are non-empty strings;
- `apiVersion` is exactly the integer `1`;
- no active or activating extension already has the same `id`;
- `activate` is a function.

No other manifest field has Stage 0 behavior. Unknown fields may be ignored.

### Activation and cleanup rules

- `activate(api)` may return directly or return a promise.
- The host creates a distinct API object and registration owner for each extension.
- Every successful registration is automatically owned by the activating extension.
- Each registration returns `{ dispose() }`.
- Calling `dispose()` more than once has no additional effect.
- If activation throws or rejects, the host disposes all partial registrations in reverse registration order and marks the extension failed.
- A failed extension ID does not remain reserved and may be activated again.
- On normal unload, the host first calls the optional cleanup function returned by `activate()`, then disposes owned registrations in reverse registration order.
- Cleanup continues after an individual cleanup error; the host reports the errors after attempting all cleanup.
- Unloading an inactive extension is a no-op.
- One extension failing must not corrupt the registries or prevent an unrelated extension from being activated.

The return value of `activate()` is either `undefined` or a cleanup function. Returning a disposable from a single registration is unnecessary because the host already owns it.

## v0 host API

The API exposed to each built-in is:

```js
{
  models: {
    register(definition),
    list(),
    get(id),
    subscribe(listener),
    invoke(id, input)
  },
  ui: {
    registerView(slot, view)
  }
}
```

Do not expose a general `app` object, DOM selectors, host registries, extension-manager internals, storage, commands, tools, or a general event bus.

## Model contribution contract

A model extension registers this private definition:

```js
{
  id: 'fake:echo',
  label: 'Fake Echo',
  providerId: 'fake',
  providerLabel: 'Built-in fake',
  credential: null,
  generate: async ({ prompt, credential, signal }) => ({
    text: `Fake response: ${prompt}`
  })
}
```

### Validation

- `id`, `label`, `providerId`, and `providerLabel` are non-empty strings.
- Model IDs are globally unique. Registering a duplicate rejects the second registration; it never replaces the first.
- `generate` is a function.
- `credential` is either `null` or:

```js
{
  kind: 'api-key',
  label: 'API key',
  required: true,
  placeholder: 'Optional display hint'
}
```

Stage 0 uses `credential: null`, but the metadata shape is frozen now so the first hosted model does not require provider-aware UI.

### Public descriptors

`models.list()` returns a new array of public, immutable model descriptors in registration order. `models.get(id)` returns the matching public descriptor or `undefined`.

A public descriptor contains `id`, `label`, `providerId`, `providerLabel`, and `credential`. It never exposes `generate`.

### Subscription

`models.subscribe(listener)` returns a disposable. The listener receives synchronous notifications after a successful mutation:

```js
{ type: 'registered', model: publicDescriptor }
{ type: 'unregistered', model: publicDescriptor }
```

Subscription is not an initial-state delivery mechanism; consumers call `list()` first and then subscribe. A listener error is reported but does not undo the registry mutation or prevent other listeners from running.

### Invocation

`models.invoke(id, input)`:

- rejects if the model ID is unknown;
- accepts `{ prompt, credential, signal }`;
- requires a non-empty prompt after trimming for validation, but passes the original prompt text to `generate`;
- passes the credential only to the selected model implementation;
- accepts an optional `AbortSignal`;
- awaits `generate()`;
- validates that the result is an object with a string `text`;
- returns exactly `{ text }`.

A bare string result is invalid.

The registry must not place the prompt, credential, or result in generic lifecycle notifications.

## Credential boundary

The fake model requires no credential, but the v0 API anticipates the first hosted model.

The prompt UI derives its credential control from the selected model’s public metadata. A credential value:

- stays in the prompt UI’s current-page state until changed or the page is reloaded;
- is supplied only in the selected invocation call;
- is not written to storage, URLs, logs, error messages, DOM attributes, or registry notifications;
- is never exposed in public model descriptors.

This is API-level least privilege, not code isolation. All active extensions and runtime dependencies execute in the page’s realm and can technically inspect page state, including credentials. Later imported or generated extensions use the same full-authority model and therefore require source review and explicit trust before activation.

## UI contribution contract

v0 supports only:

```js
api.ui.registerView('main', {
  id: 'prompt',
  mount(container) {
    // Render ordinary DOM.
    return () => {
      // Remove listeners and mounted content.
    };
  }
});
```

Rules:

- `slot` must be `'main'`;
- `view.id` is a non-empty string;
- `view.mount` is a function;
- only one view may occupy `main`; a duplicate is rejected, not replaced;
- the host calls `mount(container)` when registration succeeds;
- `mount` may return `undefined` or a cleanup function;
- disposing the registration runs mount cleanup and clears only content owned by that view;
- the prompt UI is a built-in extension; the host shell must not implement the prompt form itself.

## Prompt UI behavior

The main view contains:

- one model selector;
- a credential field shown only when selected-model metadata requires it;
- a labeled prompt textarea;
- a Run button;
- a visible status/error region;
- a result region.

Behavior:

1. Read current models with `models.list()`, render them, then subscribe to changes.
2. Preserve the selected model when possible as the list changes.
3. Disable Run when there is no model, while an invocation is running, or when a required credential is empty.
4. On Run, validate a non-empty prompt and show an inline error without invoking the model when invalid.
5. During invocation, show a plain loading state and prevent duplicate submission.
6. On success, render the completed `text` result.
7. On failure, show a useful sanitized error and restore usable controls.
8. Render prompt-derived, model-derived, and error text with `textContent`, never `innerHTML`.
9. The form is keyboard usable, labels are programmatically associated with controls, focus remains visible, and long text wraps at phone width.

There is no transcript. Each successful run replaces the previous result.

## Fake model behavior

The built-in fake model:

- has ID `fake:echo`;
- requires no credential;
- returns a deterministic completed result derived from the prompt;
- performs no network access;
- can be configured in tests to reject so the UI error path is deterministic.

Its purpose is architectural testing, not realistic language generation.

## Testing contract

Use `pytest` with Playwright as development-only tooling. Pin exact versions in `requirements-dev.txt`.

Tests serve the repository over loopback HTTP; they do not load application files through `file://` and do not require a separately started server.

The Stage 0 suite verifies at minimum:

- the page boots without uncaught errors;
- both built-ins activate;
- the fake model appears in the selector;
- one prompt produces the deterministic completed response;
- empty prompt validation does not invoke the model;
- loading disables duplicate submission;
- invocation failure is visible and controls recover;
- model registration and removal update the selector;
- duplicate extension, model, and main-view IDs are rejected without replacing existing contributions;
- an activation failure rolls back partial registrations;
- unload removes all owned registrations and mounted UI;
- disposal is idempotent;
- user/model/error text is not interpreted as HTML;
- the primary flow works at a phone-sized viewport without horizontal overflow.

Browser-level registry tests may dynamically import the local host module from the served origin. Do not add a production global debugging API solely for tests.

## Project operations

Local manual server:

```bash
python -m http.server 8000
```

Standard verification after `requirements-dev.txt` exists:

```bash
python -m pip install -r requirements-dev.txt
python -m playwright install chromium
python -m pytest
```

Development dependencies do not create an application build step and are not shipped to users.

Before adding any runtime dependency, record its canonical URL, exact tested and pinned version, browser-loading method, size, and reason in `docs/decisions/YYYY-MM-DD-short-title.md`. Provider experiments additionally record real-browser CORS results. The Stage 3 decision note must identify the exact local model and runtime, compressed download size, compatibility results, and measured first-load time; target a model download no larger than 500 MB and a first load under one minute on roughly 100 Mbit/s where hardware permits.

## Acceptance gate

Stage 0 is complete when:

- every required file and behavior above exists;
- the Playwright suite passes in a clean environment;
- the desktop and phone-sized flows have been visually inspected;
- no real provider/runtime or later-stage abstraction has entered the host;
- the entire extension host remains understandable in one sitting;
- a built-in extension can be explained using the manifest, `activate(api)`, explicit registration, and automatic cleanup;
- documentation contains no conflicting v0 interface example.

Only then begin Stage 1.
