# Changelog

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
