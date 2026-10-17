# oxlify documentation site

Static documentation for the `oxlify` node daemon, built with TypeScript and Vite.

## Requirements

- Node.js 20 or newer
- npm 10 or newer

## Commands

```bash
npm install
npm run dev
npm run build
npm run preview
```

`npm run dev` starts a local server with hot reload. `npm run build` type-checks with
`tsc --noEmit`, then writes the production site to `dist/`. `npm run preview` serves
the built `dist/` folder locally.

## Layout

```
index.html              landing page
docs/index.html         introduction
docs/<slug>/index.html  one directory per documentation page
public/favicon.svg      static assets copied into the build
src/main.ts             entry point, wires every module
src/lib/                TypeScript modules (chrome, search, code, OS switch, reveal)
src/styles/             design tokens and component styles
vite.config.ts          multi-page build inputs and clean URL redirects
```

Pages use clean URLs — `docs/installation/index.html` is served as `/docs/installation/`.
Visiting `/docs/installation` without the trailing slash issues a 301 redirect, which the
Vite plugin in `vite.config.ts` handles for both `npm run dev` and `npm run preview`.
A host with its own directory-index handling (nginx `try_files $uri $uri/`, Netlify,
Vercel, GitHub Pages) will do the same without configuration.

## Editing

Page content lives in the HTML files under `docs/`. Shared navigation is defined once in
`src/lib/nav.ts` and drives the sidebar, footer, search index and previous/next links.

OS-specific instructions use `data-os-btn` buttons and `data-os` panels; the selected
distribution is stored in `localStorage` under `oxlify-os` and follows the reader across
every page. Theme preference is stored under `oxlify-theme`.
