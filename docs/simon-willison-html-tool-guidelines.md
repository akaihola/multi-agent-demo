# Simon Willison-style browser application guidelines

## Purpose

This document defines implementation guidelines for `multi-agent-demo` based on recurring patterns in Simon Willison's static HTML/JavaScript applications, especially `simonw/tools`, and cross-checks those patterns against standalone repositories such as `simonw/llm-prices`, `simonw/viewport-preview`, and `simonw/gistpreview.github.io`.

The goal is not to imitate incidental visual details. It is to adopt the engineering style that makes these applications unusually easy to understand, modify, self-host, and hand to a coding agent.

For this project, the most important consequence is:

> Prefer a small, browser-native application that can be understood by opening its HTML file over a conventional frontend application with a framework, package graph, build pipeline, and backend.

This aligns closely with the existing technical direction of `multi-agent-demo`: static deployment, no application backend, browser execution, BYOK credentials, minimal glue code, and no premature abstraction.

## Sources examined

The main reference is Simon's `simonw/tools` repository. It contains more than 200 browser tools and even includes a repository guide documenting its common structures and patterns. Representative code examined includes `openai-webrtc.html`, which is particularly relevant because it calls an LLM API from a browser and handles a user-supplied API token.

Standalone repositories were used to distinguish general Simon Willison patterns from conventions that exist only because `tools` contains hundreds of utilities:

- `simonw/tools` — the main corpus of single-file HTML tools.
- `simonw/llm-prices` — a larger static browser application with a substantial `index.html` and data files.
- `simonw/viewport-preview` — an extremely small standalone application whose entire implementation is one `index.html`.
- `simonw/gistpreview.github.io` — an example where JavaScript is split into `main.js`, showing that single-file HTML is a strong preference rather than an absolute rule.
- Simon's December 2025 article, **Useful patterns for building HTML tools**, which explicitly explains the reasoning behind many of these choices.

## 1. Start with the browser platform, not a frontend framework

Use ordinary HTML, CSS, and JavaScript by default.

Do not introduce React, Vue, Svelte, Next.js, a component framework, or an application bundler merely because this is an interactive application. Simon explicitly values the absence of a build step because it improves hacking speed, portability, inspectability, and self-hosting.

For `multi-agent-demo`, the default implementation should therefore be something that a developer can serve with a trivial static HTTP server and inspect directly in the repository.

A framework should only be introduced if a concrete requirement demonstrates that browser-native code has become materially harder to maintain.

## 2. Prefer a single HTML entry point

The characteristic Simon-style tool combines markup, CSS, and JavaScript in one HTML file. `simonw/tools` does this extensively; `viewport-preview` is a particularly clear standalone example.

For the first POC, prefer:

```text
index.html
```

with CSS in `<style>` and application JavaScript in `<script>` or `<script type="module">`.

This has useful properties:

- the entire application can be read in one pass;
- there is no module graph to reconstruct before understanding it;
- copying or self-hosting the application is trivial;
- an LLM coding agent can inspect and edit the complete UI and behavior with minimal context gathering;
- deployment is simply static file hosting.

This is a preference, not a purity test. `gistpreview.github.io` demonstrates that Simon will split JavaScript into a separate file when useful. If `multi-agent-demo` grows enough that a single file actively harms clarity, split along obvious boundaries rather than adopting a framework.

## 3. No build step unless it earns its existence

The browser should consume the source files directly whenever practical.

Avoid requiring:

- npm installation just to run the app;
- Vite/Webpack/Rollup for ordinary application code;
- transpilation;
- generated JavaScript bundles;
- framework-specific development servers.

A developer should ideally be able to run:

```bash
python -m http.server
```

and open the application.

This constraint should influence library selection. A library that is technically excellent but requires a substantial Node/bundling toolchain is less aligned with this project than a slightly smaller browser-native alternative.

## 4. Load dependencies directly from versioned CDN URLs

Simon commonly loads third-party JavaScript from cdnjs or jsDelivr, including ES-module imports from `+esm` endpoints. His tools use this for libraries ranging from Marked and PDF.js to Pyodide, SQLite WASM, and MicroPython.

For `multi-agent-demo`, prefer dependencies that can be used like:

```html
<script type="module">
  import something from "https://cdn.jsdelivr.net/npm/package@VERSION/+esm";
</script>
```

or with a versioned ordinary `<script src="...">`.

Guidelines:

- pin versions rather than silently following `latest`;
- prefer well-established CDNs;
- prefer libraries that publish browser-ready ESM;
- keep the number of dependencies small;
- do not introduce npm solely to retrieve a dependency that the browser can load itself.

This is especially important in the provider-abstraction bake-off: browser-first libraries deserve extra weight if they can be imported directly without a build pipeline.

## 5. Let the application remain genuinely static

Static hosting is not merely a deployment optimization; it is part of the architecture.

The normal application should require no server-side runtime. GitHub Pages or any ordinary static web host should be sufficient.

Simon does use tiny server-side components when browser security rules make them genuinely unavoidable — OAuth client secrets are one example — but treats those as explicit exceptions rather than a default architecture.

For `multi-agent-demo`, preserve the existing stronger rule for the POC: **no application backend at all**. If a future capability requires one, isolate it and document exactly why browser-only implementation is impossible.

## 6. Use direct browser APIs enthusiastically

A recurring characteristic of Simon's tools is willingness to use what modern browsers already provide instead of reaching for libraries or servers.

Relevant APIs and techniques include:

- `fetch()` for CORS-enabled HTTP APIs;
- `navigator.clipboard` for copy/paste workflows;
- `<input type="file">`, `File`, `Blob`, and object URLs for local files;
- URL query parameters and fragments for state;
- `localStorage` for larger persistent state;
- WebRTC where appropriate;
- WebAssembly and WebGPU for capabilities that historically required native/server software;
- generated `Blob`s and download links for browser-created files.

For this project specifically, direct browser `fetch()`, WebGPU/WebLLM, browser storage, and URL state should be considered first-class architecture tools.

## 7. Treat CORS as an architectural capability boundary

Simon repeatedly exploits APIs that deliberately support cross-origin browser access. He has even built tools specifically for investigating CORS behavior.

For a backend-free LLM application, CORS support is not a minor implementation detail: it determines whether a provider belongs in the direct-browser architecture at all.

Therefore:

- test real browser requests early;
- do not infer browser support merely because a JavaScript SDK exists;
- distinguish SDK browser compatibility from API CORS compatibility;
- keep provider failures visible and understandable;
- prefer providers and abstractions that work directly under normal browser security rules.

A tiny diagnostic implementation using plain `fetch()` is acceptable when determining whether a provider works, even if the final application uses a higher-level abstraction.

## 8. BYOK fits the pattern, but make the security boundary explicit

`openai-webrtc.html` is a useful precedent: it accepts an OpenAI API token directly in a password field and uses it from browser JavaScript.

For `multi-agent-demo`, BYOK should similarly be simple and transparent:

- use `<input type="password">` for credentials;
- make it clear that the key is used by code running in the browser;
- never embed an application-owner secret in static source;
- initially keep keys in memory unless persistence is an intentional feature;
- if keys are persisted later, make that visible to the user and provide a clear way to remove them.

Simon notes that `localStorage` can be useful for secrets or larger state in personal HTML tools. Our POC should be more conservative initially because it is explicitly demonstrating third-party LLM credentials: memory-only is a better default, with opt-in persistence as a possible later convenience.

## 9. Make useful state linkable when it is safe to do so

Simon frequently stores tool state in query parameters or URL fragments. This makes a static application surprisingly powerful: a URL can reproduce a useful configuration without a database or account system.

For `multi-agent-demo`, good candidates include:

- selected provider;
- selected model;
- non-sensitive UI options;
- possibly prompts or conversation examples when explicitly requested by the user.

Never put API keys or other secrets in URLs. URLs leak through browser history, logs, screenshots, referrers, and copy/paste.

Prefer fragments for client-only shareable state where that provides a useful privacy or implementation property, and query parameters when ordinary URL semantics are preferable.

## 10. Keep the UI conventional and functional

The visual style across Simon's tools varies, but the structural pattern is consistent: straightforward forms, semantic labels, ordinary buttons, obvious status areas, centered content, and responsive layouts.

Common details include:

- `<meta name="viewport">`;
- global `box-sizing: border-box`;
- a centered `max-width` container;
- 16px-ish form controls that remain usable on mobile;
- labels connected to inputs;
- simple responsive grid/flex layouts;
- media queries where needed;
- disabled buttons during unavailable/loading states;
- explicit success/error/status messages;
- visible focus states and basic accessibility semantics.

Do not spend early POC effort building a design system. A small amount of local CSS is preferable to adding a UI framework.

## 11. Design mobile behavior deliberately

Mobile support is not an afterthought in these tools. Layouts generally collapse naturally to one column and controls are kept large and simple.

For the chat POC:

- make the default layout work at phone width before adding desktop enhancements;
- keep provider/model selection and credential entry usable without horizontal scrolling;
- make the transcript readable on a narrow screen;
- keep the composer easy to reach and operate;
- ensure loading/download states for browser models remain understandable on mobile;
- test long model names, errors, and generated text for overflow.

A desktop-only multi-column layout should not be the starting point.

## 12. Favor immediate, direct interactions

Many Simon tools react directly to `input`, `change`, paste, drop, or button events. State management is normally just JavaScript variables plus the DOM rather than a separate state framework.

For `multi-agent-demo`, keep the interaction model equally explicit:

```text
user selects model
  -> update credential/loading controls
user submits prompt
  -> disable/send loading state
  -> call selected model
  -> stream/update transcript
  -> restore controls or show error
```

Do not add a global store, reducer architecture, event bus, or dependency injection system until a real problem requires one.

## 13. Make loading and errors part of the ordinary UI

Simon tools commonly disable controls during asynchronous work, change button labels to indicate progress, and show errors in dedicated visible areas.

This matters even more for LLMs because operations may involve:

- loading a large browser model;
- waiting for WebGPU initialization;
- API authentication failures;
- rate limits;
- CORS failures;
- streamed generation;
- network interruption.

Represent these states plainly. Avoid hiding operational information behind developer-console-only errors.

For local models, expose model download/loading progress if the underlying library provides it.

## 14. Use semantic HTML and lightweight accessibility

`llm-prices` demonstrates useful accessibility details such as `aria-label`, `aria-sort`, keyboard-focus styling, semantic buttons for sortable controls, and visually hidden explanatory text.

Guidelines:

- use actual `<button>` elements for actions;
- use `<label for>` for form controls;
- preserve keyboard navigation;
- provide visible `:focus` states;
- use ARIA when native HTML alone does not communicate state;
- do not make color the only indicator of status;
- ensure streaming transcript updates remain readable rather than visually chaotic.

This should be achieved with normal HTML rather than an accessibility abstraction library.

## 15. Escape or sanitize content at trust boundaries

`viewport-preview` includes a small explicit HTML escaping helper before interpolating user-controlled values into generated markup. This is representative of the right level of security discipline for tiny browser tools: simple code does not mean ignoring trust boundaries.

For this project:

- prefer `textContent` when inserting model/user text into the DOM;
- do not place LLM output directly into `innerHTML`;
- if Markdown rendering is added, use a well-understood renderer plus sanitization appropriate to untrusted model output;
- never construct executable HTML from provider responses;
- validate/escape URL-derived state before rendering it.

LLM output must be treated as untrusted content.

## 16. Keep functions small and close to the DOM they serve

The representative tools generally use ordinary named functions and direct DOM references rather than elaborate class hierarchies.

Good POC structure might look like:

```js
const modelSelect = document.querySelector('#model');
const apiKeyInput = document.querySelector('#api-key');
const transcript = document.querySelector('#transcript');

function updateModelControls() { ... }
function appendMessage(role, text) { ... }
function showError(message) { ... }
async function sendMessage() { ... }
```

If provider abstraction is supplied by a dependency, application code should stay at roughly this level. Do not recreate a second internal framework around the library.

## 17. Prefer progressive complexity

A strong pattern in Simon's work is that small tools can become surprisingly capable without changing their fundamental architecture. Browser files, CORS APIs, WASM, Pyodide, WebRTC, and URL persistence can all be layered onto static HTML.

Apply the same discipline here:

1. one hosted provider, one prompt, one response;
2. second hosted provider through the chosen abstraction;
3. browser-local model;
4. streaming if easy;
5. model loading/progress;
6. only then convenience features such as persistent preferences or shareable state.

Each step should leave the application runnable and understandable.

## 18. Testing: use a real browser for behavior that matters

The `tools` repository uses Playwright with pytest and serves files from a tiny local HTTP server. Simon also explicitly values coding agents being able to test browser applications themselves.

For `multi-agent-demo`, browser-level tests are more valuable than unit-test infrastructure for most early risks. Test things such as:

- initial page renders;
- provider selection changes credential controls;
- submit/disabled/loading behavior;
- transcript rendering;
- error rendering;
- URL state restoration if implemented;
- mobile viewport layout;
- provider calls through mocked/stubbed network responses where practical.

Keep test infrastructure separate from runtime infrastructure. It is fine for tests to require Python/Playwright even though the deployed application has no build or server runtime.

## 19. Keep deployment boring

A Simon-style application should be deployable as static files to GitHub Pages or equivalent hosting.

For this repository:

- prefer GitHub Pages for the public demo;
- do not add a container just to serve static files;
- do not require a cloud-specific application framework;
- keep local development compatible with any trivial HTTP server;
- keep production artifacts identical, or nearly identical, to repository source.

The absence of a deployment architecture is a feature.

## 20. Preserve provenance and reasoning

Simon frequently records prompts/transcripts and links them from commit history. His `tools` repository also builds a colophon showing tool histories.

We do not need to copy that machinery immediately, but the underlying habit is valuable:

- make commits describe the experiment or decision, not just the changed files;
- link relevant research/discussion where useful;
- keep architecture notes in `docs/`;
- for AI-assisted implementation spikes, record what was tested and what failed, especially around browser compatibility and CORS.

This repository is a POC, so failed experiments are useful project knowledge.

## What to copy versus what not to copy

### Copy these principles

- browser-native HTML/CSS/JS;
- static hosting;
- no build step by default;
- one-file application while it remains clear;
- versioned CDN dependencies;
- direct use of browser APIs;
- CORS-aware architecture;
- simple responsive forms;
- URL state for safe shareable configuration;
- local storage only when persistence is useful and intentional;
- real-browser testing;
- explicit loading/error states;
- simple code that coding agents and humans can inspect quickly.

### Do not copy mechanically

- exact colors, fonts, shadows, or visual styling from individual tools;
- the `tools` repository's generated index/colophon machinery unless this repository grows into a collection of applications;
- localStorage credential persistence just because some personal tools use it;
- every feature that browsers make possible;
- single-file structure after it has genuinely become harder to understand than a few well-named files.

The objective is Simon's **simplicity and leverage**, not stylistic cosplay.

## Recommended repository shape for the first POC

```text
multi-agent-demo/
├── index.html
├── docs/
│   ├── technical-direction.md
│   └── simon-willison-html-tool-guidelines.md
├── tests/                 # add when browser behavior warrants it
│   └── ...
└── README.md              # add a short run/deploy description
```

If a library absolutely requires local files that cannot sensibly be loaded from a CDN, add only those files required. If JavaScript becomes unwieldy, the next step should be something like `app.js` or a few native ES modules — not an automatic migration to a framework.

## Concrete rules for `multi-agent-demo`

Use these as implementation defaults:

1. **The POC is a static web page.** No backend and no runtime server beyond static hosting.
2. **Start with `index.html` containing its own CSS and JavaScript.** Split only when clarity improves.
3. **No React and no frontend framework.** Reconsider only after a demonstrated maintenance problem.
4. **No mandatory build step.** Source should run directly in a modern browser via a static HTTP server.
5. **Prefer browser-ready, version-pinned CDN imports.** Library choice should account for this.
6. **Use native DOM APIs and browser APIs directly.** Avoid abstractions that merely wrap simple browser functionality.
7. **Treat CORS testing as part of provider evaluation.** A provider is not browser-compatible until a real browser request proves it.
8. **Keep BYOK keys memory-only initially.** Never put secrets in source or URLs.
9. **Render user and model text safely.** Default to `textContent`; sanitize any future rich rendering.
10. **Build mobile-first enough that the POC is pleasant on a phone.** Use ordinary responsive CSS, not a CSS framework.
11. **Show progress, loading, and errors explicitly.** Especially model downloads, initialization, API failures, and streaming state.
12. **Keep state simple.** DOM + a few JavaScript variables first; URL/localStorage only for state that benefits from persistence or sharing.
13. **Use Playwright for important end-to-end browser behavior.** Do not introduce heavy unit-test architecture for trivial DOM functions.
14. **Deploy the same static source that is in Git.** GitHub Pages is the natural first target.
15. **Optimize for inspectability.** A human or coding agent should be able to open the repository and understand the complete application quickly.

## Implication for the provider-library bake-off

The existing `technical-direction.md` proposes comparing ClientAgentJS, Vercel AI SDK, BundleLLM, and possibly `@livx.cc/ask`. Simon's patterns provide an additional selection criterion beyond API elegance.

For each candidate, record:

| Criterion | Question |
| --- | --- |
| Direct browser use | Can it run in a static page with no server runtime? |
| No-build use | Can it be imported from a versioned CDN/ESM URL? |
| CORS reality | Do target providers actually work from a browser? |
| Code footprint | How much application glue does one chat request require? |
| Streaming | Does streaming work without framework-specific UI machinery? |
| Credential handling | Can the app supply an in-memory user key explicitly? |
| Local-model fit | Can WebLLM/browser models fit the same interface cleanly? |
| Debuggability | When something fails, is the browser/network behavior understandable? |
| Portability | Can the resulting app be copied to arbitrary static hosting unchanged? |

A library that saves twenty lines of provider normalization but forces a Node build pipeline may be a worse fit than a slightly more explicit browser-native option.

## Final design heuristic

When choosing between two implementations, prefer the one for which this statement is more true:

> I can open `index.html`, understand what the application does, serve it from any static host, inspect every network request in browser developer tools, and modify it without first reconstructing a frontend toolchain.

That is the most transferable pattern from Simon Willison's HTML tools to this repository.