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

## Still to do

Not yet covered — URLs the app builds itself and hands to the **browser** via DOM attributes, which
no transport shim can intercept:

- `features/book/composables/useCoverVersions.ts` — `coverUrl()`, the main library cover builder
- `features/book-dock/composables/useBookDockDetail.ts` and `features/book-dock/lib/file-display.ts`
- `features/book/lib/metadata-fetch.ts` — `COVER_PROXY_PATH`
- `features/reader/audiobook/AudiobookReaderView.vue` — `mediaSession` artwork and one `<img :src>`
- `features/reader/cbz/CbzReaderView.vue` — `pageUrl()`

Each is a small `withBase(...)` wrap. They are listed here rather than done blind because the set
was arrived at by grepping, and the authoritative check is a click-through with the network panel
open: anything still requesting the Home Assistant origin root shows up there immediately.

## Upstreaming

All of this is useful to anyone reverse-proxying BookOrbit at a subpath, not just Home Assistant
users, and belongs upstream rather than here. Track it against
<https://github.com/bookorbit/bookorbit> — and per their contributing guide, propose the approach on
an issue and wait for maintainer assignment before opening a PR.
