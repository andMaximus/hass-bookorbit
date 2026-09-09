# Changelog

## 2.9.0.7

Fixes "Failed to load book" when opening a book in the Ingress panel.

The epub reader loads foliate with a hand-written `<script src="/assets/foliate/view.js">`.
That file lives in `public/`, so it never passes through Vite's asset pipeline
and the relative build base did not touch it - under Ingress it resolved to the
Home Assistant origin root, 404'd, and the reader reported the book as unloadable.

The metadata provider icons had the same shape (`/assets/provider-icons/*.svg`)
and would have rendered broken. Both now go through the prefix helper.

Anything under `public/` is copied verbatim and referenced by hand, which makes it
precisely the category a build-time base cannot fix; these were the only two.

## 2.9.0.6

Fixes being bounced back to the sign-in page on the first navigation inside the
Ingress panel.

BookOrbit sets its refresh cookie with `Path=/api/v1/auth`. Under Ingress the
browser is at `/api/hassio_ingress/<token>/`, so requests go to
`/api/hassio_ingress/<token>/api/v1/auth/refresh` - a path that cookie is never
sent to. Signing in worked because the access token comes back in the response
body and is held in memory; the first navigation that refreshed the token got no
cookie, took a 401, and the route guard sent the user back to sign in.

nginx now re-scopes every cookie onto the session prefix with `proxy_cookie_path`.

Reproduced and verified behind a stand-in for the Supervisor, with a cookie jar so
the browser's own path-matching rules applied:

  2.9.0.5   login 200, cookie Path=/api,                       refresh 401
  2.9.0.6   login 200, cookie Path=/api/hassio_ingress/.../api, refresh 200

## 2.9.0.5

Adds a Home Assistant sidebar panel via Ingress.

Home Assistant serves add-ons under `/api/hassio_ingress/<session-token>/`, and
BookOrbit's client is built assuming it lives at the origin root, so under that
prefix the browser asks Home Assistant for `/assets/...`, Home Assistant answers,
and the add-on is never contacted. No proxy inside the add-on can intercept a
request the browser sends somewhere else, so the client is rebuilt from a pinned
upstream checkout with a small patch set (see `patches/README.md`) and served by
nginx with the session prefix injected as a `<base href>`.

The panel is a separate path rather than a mode switch: nginx serves the patched
client from its own root and proxies only the API and websockets, so the mapped
port keeps serving upstream's untouched build. Kobo, KOReader, OPDS clients and
reverse proxies never pass through nginx, and devices continue to use `app_url`
because they hold no Ingress session.

Verified with the panel and the port side by side: the panel gets a relative-asset
build with `<base href>` injected, deep links and the API work through it, and the
port still serves upstream's build with 188 origin-absolute asset references and
no `<base>` tag. Confirmed with zero AppArmor denials under an enforcing kernel.

## 2.9.0.4

Restores a scoped AppArmor profile, this time developed against a real enforcing
kernel instead of by inspection.

2.9.0.3 dropped the profile because it had been shipped twice without ever being
tested under enforcement. It is back because there is now a rig that reproduces
the Supervisor's behaviour: AppArmor enabled in the WSL2 kernel, a native dockerd
in a Debian distro, and a gate that aborts the test unless the container really
reports `bookorbit (enforce)`.

That rig reproduced the original bug exactly - `/bin/sh: can't open '/init'` with
`apparmor="DENIED" ... name="/init" requested_mask="r"` - and then found five more
that inspection had missed:

- `/run/` itself, which s6-linux-init chmods and chowns before staging into it
- `/etc/s6-overlay/**` needs write: s6 compiles its service database there at boot
- `/usr/lib/bashio/**` needs exec, since /usr/bin/bashio is a symlink into it
- `/dev/shm/**` for PostgreSQL's shared buffers, and `/app/** rm` for the native
  Node addons it mmaps with PROT_EXEC
- `network inet dgram`, without which Node cannot resolve DNS

Verified with zero denials across a full boot: initdb, the Drizzle migrations, the
setup wizard, login, and authenticated API calls. Not yet exercised under
confinement: a library scan, cover extraction, a kepubify conversion, Kobo sync
and OIDC login.

## 2.9.0.3

Removes the custom AppArmor profile so the add-on starts.

The profile was shipped without ever being tested under enforcement - Docker
Desktop has no AppArmor, so neither local runs nor CI exercised it, and only the
Supervisor applies it. It broke startup twice: first by granting execute but not
read on `/init` (a `#!/bin/sh` script, so the interpreter must read it), then by
granting `rwlk` but not `ix` on `/run`, where s6-overlay stages and executes its
runtime from `/run/s6/basedir/bin/init`.

The Supervisor's default add-on profile now applies, which is what most add-ons
ship with. Nothing else about the add-on's confinement changes: it still
requests no elevated capabilities, no host namespaces and no privileged access.

A scoped profile is worth having for the +1 security rating and for real
confinement, but it has to be developed the way it should have been the first
time: loaded in complain mode on an actual Supervisor, exercised through a first
run, a library scan, a cover extraction, a kepubify conversion and a device
sync, and built from the denials that produces.

## 2.9.0.2

Corrects the version number: 2.9.0-2 is a semver prerelease and sorts below
2.9.0, so the Supervisor showed no update available. Add-on-only releases now
append a fourth component instead.

Fixes the add-on failing to start under Home Assistant with
`/bin/sh: can't open '/init': Permission denied`.

The AppArmor profile granted `ix` (execute) on `/init` but not `r` (read).
s6-overlay's `/init` is a `#!/bin/sh` script, so the kernel execs `/bin/sh`
with it as an argument and the interpreter has to read the file. Plain
`docker run` does not apply the profile, so this only appeared once the
Supervisor enforced it.

Read is now granted broadly across the image, which is where the profile was
wrong rather than merely strict. Execution stays scoped to the real binary
directories, writes stay scoped to /data, /media, /share, /config, /tmp and
/run, and the capability set is unchanged - still nothing elevated.

## 2.9.0

Initial release, packaging [BookOrbit v2.9.0](https://github.com/bookorbit/bookorbit/releases/tag/v2.9.0).

- Bundles PostgreSQL 18 with pgvector, uuid-ossp, pg_trgm and unaccent, so no separate database
  add-on is needed.
- Generates and persists all six application secrets on first run, and prints the setup token to
  the add-on log.
- Reuses upstream's own entrypoint verbatim, so Drizzle migrations, permission handling and the
  Node heap autosizing behave exactly as they do in the official Docker image.
- Refuses to start on a PostgreSQL major-version mismatch instead of crash-looping.
- Cold backups, so the database is never snapshotted while running.
- Ships a scoped AppArmor profile; no elevated capabilities are requested.
- Supports an external PostgreSQL server as an alternative to the bundled one.
