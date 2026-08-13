# Simon Willison-style browser application guidelines

## Purpose

These guidelines capture recurring engineering patterns from Simon Willison's static HTML/JavaScript applications and apply them to this repository.

### Source projects implementors should inspect

Before making substantial architectural or UI decisions, browse the source of these Simon Willison projects. They are concrete examples of the style this document refers to:

- **Tools** — https://github.com/simonw/tools — the main reference corpus: many small static browser tools, mostly plain HTML/CSS/JavaScript. In particular inspect `TOOLS_GUIDE.md`, representative `.html` tools, the Playwright tests, and GitHub Pages workflows.
- **Tools guide** — https://github.com/simonw/tools/blob/main/TOOLS_GUIDE.md — Simon's repository-level guide to recurring structure and implementation patterns.
- **Viewport Preview** — https://github.com/simonw/viewport-preview — an especially small and legible example of a complete browser application in one HTML file.
- **LLM Prices** — https://github.com/simonw/llm-prices — a larger static application showing how substantial interactive behavior can remain browser-native and framework-free.
- **Gist Preview** — https://github.com/simonw/gistpreview.github.io — useful counterexample to one-file purity: a tiny static application that separates JavaScript into `main.js` while retaining a very simple architecture.
- **OpenAI WebRTC tool source** — https://github.com/simonw/tools/blob/main/openai-webrtc.html — relevant mainly as an example of a comparatively substantial browser application, direct API credential entry, explicit state/status UI, and plain JavaScript. **Do not copy its voice/WebRTC functionality; voice is out of scope for this project.**

Treat these as source-reading references, not templates to copy mechanically. Look for recurring decisions: how little machinery is required, when code stays inline versus splits into modules, how browser APIs are used directly, how errors/loading are represented, and how the deployed source remains inspectable.

The goal is not to imitate Simon's visual styling. It is to copy the properties that make these applications easy to inspect, hack on, self-host, test, and hand to a coding agent.

The project-specific architecture has evolved since this document was first written. In particular:

- voice/audio is no longer in scope;
- the first interaction is **one prompt / one response**, not chat;
- streaming is deliberately postponed until after multi-provider and browser-local model support;
- multi-agent behavior is postponed beyond the initial demo;
- the application is now intentionally **extension-first**: built-in features should dogfood a very small public extension API wherever practical.

Those decisions are documented in `technical-direction.md` and `extension-architecture.md` and take precedence over older assumptions about the POC.

## 1. Start with the browser platform

Use ordinary HTML, CSS, JavaScript, and browser APIs by default.

Do not introduce React, Vue, Svelte, Next.js, a component framework, or a bundler simply because the application is interactive. A framework should earn its existence by solving a demonstrated problem.

The ideal local development loop remains something as simple as:

```bash
python -m http.server
```

## 2. Single HTML file is a preference, not a rule

Many Simon tools are complete single HTML files. `viewport-preview` is a particularly clear example. But `gistpreview.github.io` splits JavaScript into `main.js`, and larger static applications use multiple assets.

For this project, do not force everything into `index.html` once natural architectural boundaries exist.

A small set of native ES modules is fully consistent with the Simon style:

```text
index.html
app.js
host/extensions.js
extensions/prompt-ui.js
extensions/provider-example.js
```

The important constraint is **directly understandable source with no unnecessary build machinery**, not literal one-file purity.

## 3. No build step unless it earns its existence

Prefer source files the browser consumes directly.

Avoid requiring npm installation, Vite/Webpack/Rollup, transpilation, or generated bundles merely to run ordinary application code.

A build step is acceptable only if a chosen dependency makes it materially simpler and the deployed result remains ordinary static assets.

## 4. Prefer browser-ready, version-pinned dependencies

Simon commonly uses versioned CDN dependencies, including browser ESM imports.

Prefer dependencies that can be loaded directly, for example:

```js
import something from 'https://cdn.jsdelivr.net/npm/package@VERSION/+esm';
```

Guidelines:

- pin versions;
- use established CDNs;
- prefer browser-ready ESM;
- keep the dependency count small;
- do not introduce npm solely to retrieve code the browser can load directly.

This should be a meaningful criterion in the provider-library bake-off.

## 5. Keep the deployed application genuinely static

GitHub Pages or an ordinary static server should be sufficient.

There is deliberately no application backend in the initial demo. If a future feature truly requires server-side behavior, isolate it and document why the browser alone cannot provide it.

## 6. Use browser APIs directly

Relevant browser-native capabilities include:

- `fetch()`;
- native ES modules and dynamic `import()`;
- `EventTarget` / `CustomEvent`;
- `AbortController`;
- Clipboard APIs;
- File/Blob APIs;
- URL query parameters/fragments;
- localStorage and IndexedDB;
- Web Workers and sandboxed iframes;
- `postMessage()`;
- WebAssembly and WebGPU.

For the extension architecture, prefer composing these primitives rather than building replacements for them.

## 7. Treat CORS as an architecture boundary

A JavaScript SDK being browser-compatible does not prove that a provider API can be called directly from a static page.

Test real browser requests early. Keep CORS failures understandable. Plain `fetch()` is a useful diagnostic baseline even if the final provider extension uses a higher-level library.

## 8. BYOK should be explicit and conservative

`openai-webrtc.html` provides a relevant precedent for accepting a user-supplied OpenAI token directly in browser UI, though this project no longer intends to implement its audio/WebRTC behavior.

For this project:

- credentials belong to the user;
- use password controls;
- never ship an application-owned secret;
- initially keep credentials in memory;
- never log credentials;
- do not expose credentials through a generic extension API;
- if persistence is later added, make it opt-in and easy to clear.

The extension system makes dependency and imported-code trust especially important because same-page JavaScript can potentially inspect credentials.

## 9. Make safe state linkable

URL state can make a static application surprisingly useful.

Good candidates later include selected model/provider and non-sensitive UI options. Never place API keys or other secrets in URLs.

Do not add URL-state machinery until it provides a real benefit to the experiment.

## 10. Keep the UI conventional and functional

Common Simon-style structural patterns include:

- `<meta name="viewport">`;
- `box-sizing: border-box`;
- centered, bounded content;
- ordinary labels, inputs, selects, textareas, and buttons;
- readable 16px-ish controls on mobile;
- simple responsive grid/flex layouts;
- explicit loading/error/status areas;
- disabled controls during unavailable states;
- visible focus states;
- semantic HTML and lightweight ARIA where necessary.

Do not build a design system for the POC.

## 11. Design mobile behavior deliberately

Start with a layout that works at phone width.

For the initial prompt/result UI:

- provider/model selection must not overflow;
- credential input must remain usable;
- prompt and result should be readable at narrow widths;
- local-model loading status should remain understandable;
- long model names and errors must wrap sensibly.

## 12. Keep interactions direct

The current initial interaction is deliberately tiny:

```text
select model
  -> show any required credential/loading controls
enter prompt
  -> Run
  -> disable appropriate controls
  -> invoke selected model
  -> render one result or error
  -> restore controls
```

No transcript, conversation reducer, chat state, or streaming machinery is required initially.

The extension host may use a small event mechanism because extension lifecycle and observation are now explicit architectural requirements. This is different from introducing a generalized application-wide event-bus framework. Prefer native `EventTarget`/`CustomEvent` and explicit registries; use events for observation rather than turning every operation into an event protocol.

## 13. Make loading and errors ordinary UI

Represent asynchronous states plainly:

- model loading/download;
- WebGPU initialization;
- authentication failure;
- CORS/network errors;
- rate limits;
- provider errors;
- later, streaming interruption.

Do not hide operationally important information in the developer console.

## 14. Use semantic HTML and lightweight accessibility

Use real buttons for actions, labels for controls, keyboard navigation, visible focus, and ARIA only where native semantics are insufficient.

The first one-prompt/one-response UI should be accessible without needing a UI framework.

## 15. Treat model and extension content as untrusted

Prefer `textContent` for model/user text. Do not insert model output directly with `innerHTML`. If Markdown arrives later, render and sanitize it deliberately.

Imported or generated extension code is a much stronger trust boundary: arbitrary same-realm JavaScript is executable code, not content. See `extension-architecture.md` for the isolation discussion.

## 16. Keep code locally understandable

Simon-style code favors ordinary functions and explicit DOM work over elaborate class hierarchies.

The new extension architecture should preserve that property. A tiny host API is acceptable; an internal framework with dependency injection, global stores, decorators, generated bindings, or deep inheritance is not.

An extension should ideally be understandable from a short example such as:

```js
export function activate(api) {
  return api.models.register({
    id: 'example:model',
    label: 'Example',
    generate: async ({ prompt }) => ({ text: prompt.toUpperCase() })
  }).dispose;
}
```

(The exact API is still to be designed.)

## 17. Progressive complexity: current sequence

The current sequence is:

1. tiny extension-host walking skeleton;
2. one hosted provider, one prompt, one response;
3. second hosted provider / abstraction bake-off;
4. one browser-local WebGPU model;
5. streaming;
6. simple tools as extensions;
7. conversation/multi-turn state only if useful;
8. later, portable/importable/LLM-generated extensions;
9. multi-agent orchestration only in a future version.

Each step should leave the application runnable and understandable.

This sequence deliberately postpones streaming even if a selected library offers it immediately. The point is to introduce one architectural complication at a time.

## 18. Built-ins should dogfood the extension API

This is the main project-specific addition to the Simon-style approach.

Providers, local model runtimes, tools, diagnostics, commands, and optional UI features should use the same small public registration API wherever practical.

That keeps the core small and gives the extension API constant real-world exercise.

Do not take this as permission to build a large plugin framework. The desired extension kernel should be smaller and more obvious than the features built on it.

## 19. Use a real browser for tests

The `simonw/tools` repository uses Playwright with pytest and a tiny local HTTP server. That is a good fit here.

Early Playwright coverage should include:

- host loads;
- built-in extensions activate;
- extension registrations clean up correctly;
- provider/model selection;
- credential controls;
- one-prompt/one-response behavior;
- loading and error states;
- mocked provider calls;
- mobile viewport behavior;
- local-model compatibility/loading where practical.

Most tests should use fake providers/extensions. A tiny optional real-API suite can exist later if useful.

There is no need for voice/audio testing because voice is out of scope.

## 20. Keep deployment boring

Prefer GitHub Pages. Do not add a container or cloud-specific application framework just to serve static files.

Repository source and production assets should remain identical or nearly identical.

## 21. Preserve provenance and failed experiments

This repository is a POC. Record what was tested, particularly:

- provider CORS behavior;
- provider-library bake-off results;
- WebGPU/local-model compatibility;
- extension API decisions;
- sandbox/isolation experiments;
- approaches rejected for unnecessary complexity.

Failed spikes are useful project knowledge.

## What to copy from Simon's approach

- browser-native HTML/CSS/JS;
- static hosting;
- no build step by default;
- direct use of browser APIs;
- version-pinned dependencies;
- CORS-aware architecture;
- straightforward responsive UI;
- explicit loading/error states;
- real-browser testing;
- source that humans and coding agents can inspect quickly;
- willingness to split into a few plain JavaScript modules when that improves clarity.

## What not to copy mechanically

- exact visual styling;
- single-file structure after it harms clarity;
- `simonw/tools` index/colophon machinery;
- localStorage credential persistence merely because some personal tools use it;
- every browser capability just because it exists;
- audio/WebRTC functionality from `openai-webrtc.html`;
- repository machinery unrelated to this experiment.

## The project-specific synthesis

The intended style is now:

> **Simon-style static browser software on the outside; an exceptionally small extension host on the inside.**

Native ES modules, direct DOM code, Web APIs, static hosting, and Playwright keep the implementation concrete. A tiny `activate(api)`-style extension contract keeps providers, tools, UI experiments, and future generated extensions decoupled from the host.

If the extension architecture starts to look like a frontend framework, package manager, or miniature VS Code, it has missed the point.
