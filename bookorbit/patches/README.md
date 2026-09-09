# Client patches for Ingress

Home Assistant Ingress serves an add-on under `/api/hassio_ingress/<session-token>/` on the Home
Assistant origin. BookOrbit's client is built with an absolute base, so under that prefix the
browser asks the Home Assistant origin root for `/assets/…` and `/api/…`. Home Assistant answers
those itself and the add-on is never contacted — which is why no proxy or runtime shim inside the
add-on can fix it, and why the client has to be rebuilt.

These patches are applied to a pinned upstream checkout at image build time. The rebuilt client is
shipped as `/app/public-ingress` **alongside** the untouched upstream assets in `/app/public-stock`,
and `/app/public` is a symlink pointed at one of them by the `ingress` option. Port-mode users
therefore run byte-identical upstream code.

## Applying

```sh
git apply --verbose bookorbit/patches/*.patch
BOOKORBIT_PREFIXED_BUILD=1 pnpm --filter client run build-only
```

`git apply` fails loudly rather than mis-patching, so an upstream release that moves this code turns
the upstream-sync PR red instead of shipping a broken UI.

## What 0001 does

| Concern | Approach |
| --- | --- |
| Assets and lazy chunks | `base: './'` in `vite.config.ts`, gated on `BOOKORBIT_PREFIXED_BUILD`. Makes `index.html` and the chunk loader resolve against the document rather than the origin root. |
| Where the prefix comes from | A `<base href>` tag the add-on's nginx injects into `index.html`. `client/src/lib/ingress-base.ts` reads it; with no tag `basePath` is `/` and every helper is an identity, so unprefixed deployments are unaffected. |
| API calls | `window.fetch` and `XMLHttpRequest.prototype.open` are wrapped once at startup. Most calls go through `lib/api.ts`, but the auth flows and preference-sync composables call `fetch` directly and the three upload paths use XHR — patching the transports covers all of them, including ones added later. |
| Router | `createWebHistory(basePath)`. Upstream passes `import.meta.env.BASE_URL`, which is the literal `'./'` in a relative-base build. |
| WebSockets | `path` on `createAuthenticatedSocket`, which every namespace goes through. |
| Downloads | `downloadFromUrl` sets `anchor.href` and clicks it — browser navigation, so neither transport shim sees it. Rewritten at that one chokepoint. |
| Service worker | Disabled in a prefixed build: it would register against a scope it cannot reach and cache URLs that die when the session token rotates. |

## Verified

Built from a pinned v2.9.0 checkout with the patch applied, then served behind a simulated prefix
(nginx at `/pfx/`, injecting `<base href="/pfx/">` the way the add-on's nginx will from
`X-Ingress-Path`):

| Check | Result |
| --- | --- |
| Client builds with the patch | clean, 33s |
| `index.html` asset references | all `./assets/...`, zero origin-absolute |
| Origin-absolute `"/assets/"` literals left in the JS | 0 |
| Chunk loader | resolves via `import.meta.url` |
| Entry script under the prefix | 200 — and 404 at the origin root |
| A genuinely lazy chunk (absent from `index.html`) | 200 under the prefix, 404 at root |
| SPA deep link `/pfx/books/123` | 200, with `<base>` injected |
| Service worker | not emitted |

The 404s at the origin root are the point: under real Ingress those requests reach Home Assistant's
own frontend instead of the add-on, which is what made every proxy-side workaround impossible.

Not yet verified, because it needs a browser against a real Ingress session rather than curl: the
`fetch`/XHR rewrites, the router base, socket.io, and the DOM-attribute URLs below.

## Covered by 0001

Every URL the app hands to the browser through a DOM attribute, which no transport shim can
intercept, is wrapped with `withBase(...)`:

- `features/book/composables/useCoverVersions.ts` — `coverUrl()`, the main library cover builder
- `features/book-dock/composables/useBookDockDetail.ts`, `features/book-dock/lib/file-display.ts`
- `features/book/lib/metadata-fetch.ts` — the cover proxy, including the pathname comparison that
  also has to accept the prefixed form
- `features/reader/audiobook/AudiobookReaderView.vue` — the `<img>` and the `mediaSession` artwork
- `features/reader/cbz/composables/useCbz.ts` — `pageUrl()`, the comic page images
- `features/reader/epub/composables/useFoliate.ts` — the epub reader loads `foliate/view.js` with a
  hand-written `<script src>`, outside Vite's asset pipeline, so a relative build base never
  touched it. This one is why "Failed to load book" appeared in the panel.
- `features/book/lib/provider-icons.ts` — same shape: `/assets/provider-icons/*.svg` from `public/`

Anything under `public/` is copied verbatim and referenced by hand, so it is exactly the category a
build-time base cannot fix. `index.html`'s own references (including `theme-init.js`) are rewritten
by Vite and need nothing.

## Upstreaming

All of this is useful to anyone reverse-proxying BookOrbit at a subpath, not just Home Assistant
users, and belongs upstream rather than here. Track it against
<https://github.com/bookorbit/bookorbit> — and per their contributing guide, propose the approach on
an issue and wait for maintainer assignment before opening a PR.
