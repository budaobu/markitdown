# MarkItDown - Web Version (Pyodide Edition)

A browser-based file to Markdown converter using **Vite + Vue 3** and **Pyodide** to run the official [Microsoft MarkItDown](https://github.com/microsoft/markitdown) Python library directly in your browser.

## How it works
- **Pyodide**: Loads a complete Python environment in the browser via WebAssembly.
- **MarkItDown Global**: use `micropip` to install `markitdown` from PyPI.
- **Client-side**: Files are processed locally in your browser's memory. No uploads to any server.

## Features
- **Format Support**: Supports formats handled by `markitdown` (PDF, Word, Excel, PowerPoint, mostly via Python-only dependencies).
- **Theme Support**: Dark/Light mode.
- **Offline Capable**: Once the Pyodide environment works and caches, it runs efficiently.

## Development

```bash
npm install
npm run dev
```

## Deployment to Cloudflare Pages

1. Connect your repository to Cloudflare Pages.
2. Settings:
   - **Framework**: Vite
   - **Build Command**: `npm run build`
   - **Output Directory**: `dist`
