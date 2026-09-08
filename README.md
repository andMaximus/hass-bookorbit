# Home Assistant Add-on: BookOrbit

[![Add repository to Home Assistant][repo-badge]][repo-link]

A Home Assistant add-on for [BookOrbit][bookorbit] — a self-hosted library and reading platform for
ebooks, PDFs, audiobooks and comics, with a web reader, Kobo Sync, a KOReader plugin, OPDS delivery
and OIDC single sign-on.

![BookOrbit dashboard](https://raw.githubusercontent.com/bookorbit/bookorbit/main/docs/images/dashboard-overview.png)

## Why this add-on exists

BookOrbit needs PostgreSQL 16+ with the `pgvector`, `uuid-ossp`, `pg_trgm` and `unaccent`
extensions. A Home Assistant add-on is a single container and the Supervisor offers no PostgreSQL
service, so this add-on bundles the database alongside the application.

It does that without forking anything: the image is built **from upstream's official release image,
pinned by digest**, and BookOrbit's own entrypoint runs unmodified — so schema migrations,
permission handling and heap sizing behave exactly as they do in the official Docker deployment.
The add-on adds PostgreSQL 18 (the same major upstream bundles), the supervision tree, and the
Home Assistant integration around it.

## Installation

1. Add this repository to your Home Assistant add-on store:

   [![Add repository to Home Assistant][repo-badge]][repo-link]

   Or manually: **Settings → Add-ons → Add-on Store → ⋮ → Repositories**, and add
   `https://github.com/andMaximus/hass-bookorbit`.

2. Install the **BookOrbit** add-on.
3. Set **External URL** in the Configuration tab — get this right before pairing any devices.
4. Start it, then open the **Log** tab for your one-time setup token.

Full documentation, including network shares, remote access and Kobo setup, is in
[DOCS.md](bookorbit/DOCS.md) and on the add-on's Documentation tab.

## What's inside

| | |
| --- | --- |
| Application | [BookOrbit][bookorbit] 2.9.0, official image pinned by digest |
| Database | PostgreSQL 18 + pgvector 0.8.1, bundled and loopback-only |
| Architectures | `amd64`, `aarch64` |
| Supervision | s6-overlay v3, ordered start and clean shutdown |
| Confinement | No elevated capabilities, no host namespaces, Supervisor's default AppArmor profile |
| Backups | Cold, so the database is never copied while running |

Secrets — the JWT signing key, the setup token, the database password and the three encryption keys
for stored SMTP, migration and indexer credentials — are generated on first run, persisted, and
never rotated automatically. Rotating them would log every user out and make stored credentials
undecryptable.

## Known limitations

- **No Ingress panel.** BookOrbit's web client is built with absolute asset paths, so under an
  Ingress path prefix the browser requests its JavaScript from Home Assistant's own URL space and
  never reaches the add-on. This needs a change to BookOrbit's client build rather than a proxy
  trick; it is planned. See [DOCS.md](bookorbit/DOCS.md#home-assistant-ingress).
- **Book Dock auto-detection does not work on network shares**, because inotify does not fire for
  changes made on the far side of an SMB or NFS mount. Libraries use scheduled scans and are
  unaffected. Workarounds in [DOCS.md](bookorbit/DOCS.md#network-shares-nas).
- **`armv7` and `armhf` are not supported** — upstream publishes `amd64` and `arm64` only.

## Support

| Where | What |
| --- | --- |
| [This repository][issues] | Packaging, database, Home Assistant integration |
| [bookorbit/bookorbit][upstream-issues] | The application itself, its features and UI |

## Licence

AGPL-3.0-only, matching [BookOrbit][bookorbit] upstream. See [LICENSE](LICENSE).

This is an **unofficial community add-on** and is not affiliated with or endorsed by the BookOrbit
project. The BookOrbit name and artwork belong to its authors.

[bookorbit]: https://github.com/bookorbit/bookorbit
[issues]: https://github.com/andMaximus/hass-bookorbit/issues
[upstream-issues]: https://github.com/bookorbit/bookorbit/issues
[repo-badge]: https://my.home-assistant.io/badges/supervisor_add_addon_repository.svg
[repo-link]: https://my.home-assistant.io/redirect/supervisor_add_addon_repository/?repository_url=https%3A%2F%2Fgithub.com%2FandMaximus%2Fhass-bookorbit
