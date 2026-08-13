# Trusted same-realm extensions

- **Date:** 2026-08-13
- **Status:** Accepted
- **Scope:** Extension/plugin runtime for the demo

## Context

The demo aims to remain a small, static, browser-native application whose built-in features and later user-provided features use one understandable extension API.

Possible extension runtimes included:

- trusted ES modules in the application’s main realm;
- Web Workers with a message-based capability API;
- sandboxed iframes with a proxied UI API;
- multiple trust tiers combining those approaches.

Worker and iframe designs provide stronger realm boundaries, but they would require message protocols, serialization rules, remote lifecycle handling, and a declarative or proxied approach for UI extensions. Those mechanisms would become a large part of the demo before its core model and extension ideas had been proven.

## Decision

All extensions in this demo execute as **trusted same-realm ES modules**.

This applies to:

- built-in extensions;
- developer-authored extensions;
- later imported extensions;
- later LLM-generated extensions after user review and activation.

The host provides `manifest + activate(api) + explicit registrations + disposables`. That API defines supported integration and cleanup; it is not a security boundary.

Extensions are loaded as ES modules. The plugin system must not be based on `eval()` or `new Function()`. A later portable-source loader may use dynamic `import()` with a deliberately created module URL, but the resulting module still has full same-realm authority.

## Trust and activation rules

- The demo has no untrusted, sandboxed, or permission-restricted extension tier.
- An active extension may inspect or modify the DOM and JavaScript state, access in-page credentials, make network requests, monkey-patch globals, and interact with other same-realm code.
- Imported extension code must be presented as executable code comparable to installing software.
- Imported or generated extensions require explicit user review and activation.
- The application must never automatically activate LLM-generated code.
- Source provenance and dependency provenance should be displayed where available.
- API-level credential routing still matters because it reduces accidental exposure and coupling, even though it cannot prevent a malicious active extension from finding a credential in page state.

## Consequences

### Benefits

- minimal browser-native implementation;
- direct DOM access for UI extensions;
- no message transport or serialization layer;
- the same extension shape works for providers, local-model adapters, tools, UI, and diagnostics;
- extension source remains easy for humans and language models to read and produce;
- built-ins can dogfood the exact public extension API.

### Costs and risks

- extension failure isolation is lifecycle/error containment, not realm isolation;
- a malicious or compromised extension has the page’s effective authority;
- a compromised extension dependency creates the same risk;
- fine-grained permissions cannot be enforced by the host API;
- users must understand that trust is granted to code, not merely to a declared manifest.

These risks are acceptable for this experimental demo because simplicity, inspectability, and extension-authoring flexibility are primary goals.

## Rejected alternatives

### Web Worker extension runtime

Rejected for the demo because it prevents direct DOM access and requires a message-based API, serialization, asynchronous proxies, and a separate solution for UI extensions. A model runtime may still use Workers internally for computation.

### Sandboxed iframe extension runtime

Rejected for the demo because it adds document lifecycle, layout, communication, origin, and content-security-policy complexity and makes a uniform UI contribution API substantially harder.

### Multiple trust tiers

Rejected because it would require maintaining more than one runtime and explaining inconsistent capabilities. The demo deliberately chooses one simple, explicit trust model.
