# Hosting OrcaWeb on the Raspberry Pi

This is the HolyBitsLLC hosting plan for serving this fork of
[OrcaWeb](https://github.com/Hiosdra/OrcaWeb) (OrcaSlicer compiled to
WebAssembly) from a Raspberry Pi, behind nginx, in front of the
`snapmaker-server` print platform.

**Status:** plan only — nothing here has been deployed.

## Architecture

OrcaWeb does **all slicing client-side in the browser**. The WebAssembly
engine (OrcaSlicer v2.4.2) runs in a Web Worker in the visitor's browser; the
Pi only serves static files. No server-side slicing engine runs on the Pi.

```
Browser (user)
  │  loads static SPA from the Pi over HTTPS
  │  downloads slicer.wasm / slicer-mt.wasm (~38 MB) same-origin
  │  slices STL/3MF/OBJ/STEP → raw G-code  (100% client-side)
  ▼
nginx on Raspberry Pi
  ├─ /        → OrcaWeb static build (dist/)
  ├─ headers  → COOP: same-origin + COEP: require-corp  ⇒ enables MT engine
  └─ /api/*   → proxied to snapmaker-server (FastAPI :8000) + serial bridge :8001
         ▼
snapmaker-server  → USB/serial (115200) → Snapmaker 2.0 / Artisan / J1
```

## Build

The WASM engine binaries are **not committed** — they are downloaded from the
repo's GitHub Releases (or built from `orca-wasm/` in CI). Build as follows:

```bash
git clone https://github.com/HolyBitsLLC/SnapmakerOrcaWeb.git
cd SnapmakerOrcaWeb

npm ci

# Download the prebuilt engine into public/wasm/:
#   ST (single-threaded)  — npm run setup downloads slicer.js + slicer.wasm
#   MT (multithreaded)    — additionally fetch the "-multithreaded" release
#                           assets slicer-mt.js + slicer-mt.wasm into
#                           public/wasm/ (see .github/workflows/deploy.yml)
npm run setup

# Build the static bundle → dist/
#   VITE_BASE defaults to / (root); set it to a subpath if serving under one.
npm run build
```

`npm run build` runs `tsc -b && vite build` and emits everything to `dist/`.
`public/` (including `public/wasm/`) is copied verbatim into `dist/`.

## nginx

Serve `dist/` with cross-origin isolation **and** the correct WASM MIME type.
The multithreaded engine requires `SharedArrayBuffer`, which is only available
when `self.crossOriginIsolated` is true — that needs both headers below on every
document/worker response. Without them the app silently falls back to the
single-threaded engine.

```nginx
server {
    listen 443 ssl http2;
    server_name orcaweb.example.com;

    root /srv/orcaweb/dist;
    index index.html;

    # Cross-origin isolation → enables the MT (multithreaded) WASM engine.
    add_header Cross-Origin-Opener-Policy "same-origin" always;
    add_header Cross-Origin-Embedder-Policy "require-corp" always;

    # SPA fallback.
    location / {
        try_files $uri $uri/ /index.html;
    }

    # WebAssembly binaries: correct Content-Type + long-lived caching.
    location ~* \.wasm$ {
        types { application/wasm wasm; }
        default_type application/wasm;
        add_header Cache-Control "public, max-age=2592000, immutable";
    }

    # Long-cache other hashed static assets (Vite emits content-hashed names).
    location ~* \.(js|css|svg|png|ico|woff2?)$ {
        expires 30d;
        add_header Cache-Control "public, immutable";
    }
}
```

### COEP caveats (important)

`Cross-Origin-Embedder-Policy: require-corp` means **every** subresource —
scripts, styles, images, fonts, workers, and the `.wasm` itself — must be
either same-origin or explicitly labelled with `Cross-Origin-Resource-Policy`
(or served with CORS `Access-Control-Allow-Origin`). A single misconfigured
third-party asset silently downgrades the whole page to the ST engine, because
the `slicer.worker.ts` `canUseThreads` probe fails and the app falls back.

- Keep all assets same-origin (no CDN fonts/images) or CORP-label them.
- The ~38 MB `slicer.wasm` / `slicer-mt.wasm` is the dominant first-load cost;
  it is cached by the PWA service worker (30-day `CacheFirst` runtime cache)
  and by nginx, so it is paid once per client, not per visit.
- Prefer serving the `.wasm` same-origin from the Pi. The upstream Cloudflare
  mirror loads it cross-origin from GitHub Pages because Cloudflare caps single
  assets at 25 MiB; nginx on the Pi has no such limit.

## G-code handoff to the Snapmaker server

OrcaWeb's WASM bridge returns **raw G-code text** (it has no printer-upload
path — its product boundary is local slicing + G-code download). To print on a
Snapmaker machine, hand the G-code to `snapmaker-server`, which speaks the
Snapmaker plain-text host channel over USB serial (115200):

- **OctoPrint (preferred today):** POST the G-code to OctoPrint's REST API
  (`POST /api/files/local` upload, then `POST /api/files/local/<file>` to
  select, then start) — OctoPrint already speaks the Snapmaker serial protocol
  through the existing raw TCP↔serial bridge.
- **Future endpoint:** add `POST /api/job/upload` to `snapmaker-server`
  (FastAPI, :8000) that accepts the G-code and streams it over the serial
  link (or the raw bridge on :8001).
- **Raw bridge:** feed the G-code through the existing `:8001` raw TCP↔serial
  bridge via `socat`/OctoPrint as an interim path.

The exact mechanism is an open question (see below); document the flow now,
wire the endpoint later.

## Auth / SSO

OrcaWeb has no auth. Put it behind the same oauth2-proxy (or Cloudflare
Access) gate as `snapmaker-server`, so slicing and printing share one
authenticated boundary. The "send to printer" bridge must be behind the same
gate to avoid an unauthenticated print trigger.

## Snapmaker note

This fork adds Snapmaker machine presets (Artisan, J1, A250, A350, U1) sourced
from upstream `OrcaSlicer/OrcaSlicer` v2.4.2. The built-in presets carry only
bed geometry, height, nozzle diameter, and model name; the per-machine
`gcode_flavor` (Klipper for the U1, Marlin otherwise) and start/end G-code are
not bundled (see `mkdocs-docs/profiles.md`). For production slicing on these
machines, import the full machine profile JSON so the engine gets the correct
G-code dialect.

## Open questions

- Which handoff path to build first: an OctoPrint upload, a new
  `snapmaker-server` upload endpoint, or the raw `:8001` bridge.
- Whether the U1's Klipper/Moonraker toolchanger is in scope for slicing here
  (the platform currently targets 2.0/Artisan/J1 plain-text channel).
- Whether the U1 multicolor engine features (wipe tower / toolchange) are
  present in upstream v2.4.2 or remain Snapmaker-fork-only (see the scoping
  plan's Q1).
