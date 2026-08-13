# AGENTS.md

This is the entry point for implementation agents working in this repository.

## Current milestone

Implement **Stage 0: the fake-model walking skeleton** and nothing beyond it.

The normative specification is [docs/implementation-contract-v0.md](docs/implementation-contract-v0.md). Read it completely before changing application code.

## Progressive disclosure

Read only the material needed for the task:

1. **Always read first:** [the v0 implementation contract](docs/implementation-contract-v0.md) — exact Stage 0 behavior, interfaces, files, tests, and operating rules.
2. **For scope, sequence, or product-level architecture:** [technical direction](docs/technical-direction.md).
3. **For rationale or future extension-system design:** [extension architecture research](docs/extension-architecture.md).
4. **For browser-native implementation and UI style:** [Simon Willison-style guidelines](docs/simon-willison-html-tool-guidelines.md).
5. **For dependency, CORS, runtime, or model choices:** read the relevant decision note under `docs/decisions/` if one exists; create one when making a consequential choice.

For Stage 0 conflicts, the implementation contract wins. Repair conflicting secondary documentation in the same change.

## Non-negotiable constraints

- Static browser application; no application backend.
- Ordinary HTML, CSS, JavaScript, Web APIs, and native ES modules.
- No application build step unless a later documented dependency demonstrates the need.
- Use relative URLs and imports so deployment under a GitHub Pages project subpath works.
- One prompt produces one completed response.
- The prompt UI and fake model are built-in extensions using the public v0 API.
- No real provider, local model, persistent conversation, tools, imported extensions, or multi-agent behavior in Stage 0.
- Do not expand the v0 host API speculatively.
- Treat user/model text as untrusted and render it with `textContent`.
- Keep `README.md` strictly end-user facing; implementation guidance belongs here or in `docs/`.

## Working method

1. Inspect the current tree and working state before editing.
2. Implement the smallest contract-compliant slice.
3. Keep source browser-readable and files coarse-grained.
4. Add or update deterministic Playwright tests with every behavior change.
5. Run the full test suite and inspect the page at desktop and phone-sized viewports.
6. Report what changed, what was verified, and any remaining contract deviation.

Do not silently decide an unresolved architectural question. If a change to the v0 contract is necessary, update the contract deliberately and record the reason in a short decision note.

## Project operations

The application has no production build. Serve it locally with:

```bash
python -m http.server 8000
```

Development-only Python packages are pinned in `requirements-dev.txt`. After that file exists, the standard verification flow is:

```bash
python -m pip install -r requirements-dev.txt
python -m playwright install chromium
python -m pytest
```

Automated browser support begins with Playwright Chromium. Tests must also exercise a phone-sized viewport. Broader hosted-provider and WebGPU compatibility is addressed in later stages.

## Dependency and research records

Before adding a runtime dependency:

- verify its canonical project URL and browser support;
- record the exact tested version;
- pin the deployed version;
- document why plain browser APIs are insufficient;
- record CORS or compatibility evidence when relevant.

Use `docs/decisions/YYYY-MM-DD-short-title.md` for these notes. A decision note should state the context, decision, evidence, consequences, and rejected alternatives. Failed spikes are useful evidence and should be recorded.
