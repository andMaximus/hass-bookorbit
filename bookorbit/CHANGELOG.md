# Changelog

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
