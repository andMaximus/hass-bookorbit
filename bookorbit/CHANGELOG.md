# Changelog

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
