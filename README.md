# sabo-cinematic-ui

Self-contained **cinematic 3D landing UI** package for **SABO M&T** (SABO Media & Technology). Static HTML + Three.js experiences you can open locally or host as static files—no Next.js build required.

## Preview locally

From this repo root:

```bash
npx serve .
```

or:

```bash
python -m http.server 8080
```

Then open:

- **Windland (offline, source of truth):** [http://localhost:8080/cinematic-windland/index.offline.html](http://localhost:8080/cinematic-windland/index.offline.html)  
  (adjust port if you used `serve` default)

Or use the npm script (zero-install via `npx`):

```bash
npm run preview
```

Opens on port **4173** — visit `/cinematic-windland/index.offline.html`.

## Entry points

| Path | Role |
|------|------|
| `cinematic-windland/index.offline.html` | **Source of truth** — bundled/offline Windland cinematic (GSAP, Lenis, Three.js, local `_assets`) |
| `index.html` | Gallery / hub linking cinematic demos |
| `cinematic-gemini/`, `cinematic-sougen/` | Additional cinematic variants (when present) |
| `_assets/` | Shared JS, fonts, Three.js post-processing |
| `AGENT-GUIDE.md`, `HUONG-DAN.txt` | Agent and operator notes |

**Default locale:** Vietnamese — append `?lang=vi` (e.g. `index.offline.html?lang=vi`).

## Sync back to sabo-mt-website

This package is extracted from `sabo-mt-website/product/website/`. After editing here:

1. Copy this folder’s contents into `sabo-mt-website/product/website/` (overwrite matching paths).
2. In the **sabo-mt-website** repo root, run:

   ```bash
   node scripts/sync-showreel.mjs
   ```

That publishes assets under `public/showreel/` for the Next.js site.

## Tooling note

**Not for Lovable** — these pages are raw HTML + Three.js, tuned for **Cursor** and design iteration. Integrate into the marketing site via `sync-showreel.mjs`, not by importing as a React component from this repo alone.

## License / brand

SABO M&T internal showcase assets. Do not commit secrets or `.env` files.
