# Multi-Agent Demo

> The name is temporary. The first version is intentionally a single-model interaction, not a multi-agent system.

Multi-Agent Demo is an experimental browser application for trying language models without installing an application backend. The goal is one small page where you can choose a model, enter a prompt, and receive one completed response.

## Status

The project is currently at the design stage. There is no runnable application yet.

## Planned experience

The first useful version will let you:

- choose from supported hosted models;
- supply your own provider credential when required;
- choose a small model that runs locally in a compatible browser;
- enter one prompt and see one response.

Hosted models receive the prompt directly from your browser. A local model runs in your browser after its model files have been downloaded.

## Privacy and credentials

The application will not have its own server or accounts. Provider credentials are intended to remain in the current browser page and are not intended to be stored.

A hosted provider still receives any prompt sent to its model and applies its own terms and privacy policy. Only enter a provider credential into a copy of the application whose source and hosting location you trust.

## Browser requirements

Hosted models should work in modern browsers when the provider permits direct browser requests. Running a model locally additionally requires compatible WebGPU hardware and browser support, and the first run downloads the model files.
