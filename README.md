# Soil Vocabulary Explorer

A responsive Vue 3 + Vite application for browsing a SKOS concept hierarchy, inspecting concept descriptions and navigating broader/narrower relations.

## Run locally

Requirements: Node.js 18+.

```sh
npm install
npm run dev
```

Open the local URL printed by Vite. To make a production build:

```sh
npm run build
npm run preview
```

## Features

- Collapsible concept tree in the sidebar; clicking a concept opens its details.
- Deep links such as `/#soil` (or `../vocab#soil` when hosted at that path). IDs are resolved against full URIs and their final fragment/path segment.
- Selecting a concept from search or broader/narrower relations expands its ancestors in the tree.
- Search covers preferred labels, alternate labels, IDs, and available definitions/descriptions.
- Separate visual treatment for procedures. Procedure detection uses type/category metadata and a conservative label/ID heuristic; adapt `procedure` in `src/App.vue` if the source JSON uses a dedicated field.
- Responsive layout for mobile and desktop.

## Vocabulary source

By default, the app fetches `https://soilwise-he.github.io/soil-vocabs/assets/soilvoc_data.json` at runtime. You can override the source by setting `VITE_VOCABULARY_URL` in your build environment.

For a same-origin deployment, place the full JSON file at `public/soilvoc_data.json` and set `VITE_VOCABULARY_URL=./soilvoc_data.json` when building (or change `DATA_URL` in `src/App.vue` to `` `${import.meta.env.BASE_URL}soilvoc_data.json` ``). This base-aware path also works when the app is deployed under a subfolder.

The normalizer supports this vocabulary's `vocabulary.top_concepts` structure, recursively traverses `narrower`, imports nested `procedures` as child entries, reads `uri`, `label`, `altLabel`, `definition` and `definitions[].text`, and recognizes the explicit `isProcedure` flag. It also retains support for common SKOS JSON / JSON-LD forms (`@graph`, arrays, `concepts`, `items`, `data`, and `skos:broader` / `skos:narrower` properties). Procedures are displayed as leaf entries.

## URL behavior

The app uses the concept's last URI fragment/path segment as the hash key, e.g. `#soil`. If two concepts share that final segment, change `conceptUrl()` and `resolveId()` to use an encoded full URI or another unique identifier.

## Deploy to GitHub Pages

This repository includes a GitHub Actions workflow at `.github/workflows/deploy.yml`. It builds the Vite app and publishes `dist` to GitHub Pages whenever you push to the `main` branch; you can also run it manually from the **Actions** tab using **Deploy to GitHub Pages** → **Run workflow**.

1. Push this project to a GitHub repository, with the app files (including `package.json`) at the repository root.
2. In GitHub, open **Settings → Pages** and set the build/deployment source to **GitHub Actions**.
3. Push to `main` (or manually run the workflow from **Actions**).
4. Once the workflow succeeds, open the URL shown in the `github-pages` deployment environment.

The Vite config uses `base: './'`, so built assets use relative paths and work for both user/organization pages and project pages hosted under a repository subpath. The vocabulary JSON is fetched from the configured runtime URL by default; make sure that URL is publicly accessible, or configure `VITE_VOCABULARY_URL` in the workflow build step if you want to use a different source.
