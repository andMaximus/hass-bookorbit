# Home Assistant Add-on: BookOrbit

[BookOrbit](https://bookorbit.app) is a self-hosted library and reading platform for ebooks, PDFs,
audiobooks and comics, with a web reader, Kobo Sync, a KOReader plugin and OPDS delivery.

This add-on packages upstream's official image together with the PostgreSQL 18 + pgvector database
it requires, so there is nothing else to install.

## Installation

1. Add this repository to the add-on store:
   `https://github.com/andMaximus/hass-bookorbit`
2. Install **BookOrbit**.
3. Set **External URL** in the Configuration tab (see below) — this one matters, and changing it
   later means re-pairing devices.
4. Start the add-on and open the **Log** tab.
5. On first start the log prints a **setup token**. Open the web interface and complete the setup
   wizard with it.

The first start takes longer than later ones: it creates the database cluster and runs the schema
migrations.

### The setup token

The wizard asks for a one-time token. The add-on generates it for you and prints it in the log:

```
================================================================
 BookOrbit first run

 Complete the setup wizard at:
   http://homeassistant.local:3000

 Setup token (the wizard asks for this):
   1f2c1c0db76bfa130c4ecb20df760b10

 Stored in /data/secrets.env if you need it again.
================================================================
```

It is stored inside the add-on and reused across restarts, so you can always retrieve it later from
the log of the first run — or by running
`docker exec addon_<slug>_bookorbit cat /data/secrets.env` from the host, if you have SSH access.

## Configuration

### Option: `app_url` (required)

The URL you and your devices use to reach BookOrbit.

**Get this right before you pair any devices.** Kobo Sync, the KOReader plugin, OPDS clients and
e-mail links all embed this value. Changing it afterwards means re-pairing.

- Local network only: `http://homeassistant.local:3000` or `http://192.168.1.x:3000`
- Behind a reverse proxy or Cloudflare Tunnel: the public HTTPS URL, e.g. `https://books.example.com`

> **The "Open Web UI" button ignores this setting.** Home Assistant builds that link from the
> hostname you are currently viewing Home Assistant on plus the add-on's mapped port — the
> placeholder syntax supports nothing else, so it cannot be pointed at `app_url`. On a local network
> it lands in the right place. Behind a reverse proxy or a tunnel it will not, because your proxy
> hostname does not serve port 3000. Use your `app_url` directly there.

### Option: `library_browse_root`

Where the in-app library folder picker starts. Default `/media`.

- Network storage added with the **media** usage → `/media`
- Network storage added with the **share** usage → `/share`
- To see both → `/`

If the picker cannot see your books, this is almost always why.

### Option: `book_dock_path`

Optional drop folder for hands-free importing, e.g. `/share/bookdock`. Leave empty to use
BookOrbit's default inside the add-on's own data directory.

**Keep this on Home Assistant's local storage, not on a NAS share.** Instant pickup relies on
inotify, which does not fire for files written on the far side of an SMB or NFS mount. See
[Network shares](#network-shares-nas).

### Option: `puid` / `pgid`

The user and group BookOrbit runs as. Default `0` / `0` (root), which is what Home Assistant's own
`/media` and `/share` require.

Change these only if your library lives on a share whose files are owned by a specific user — see
[Network shares](#network-shares-nas).

### Option: `log_level`

`trace`, `debug`, `info` (default), `notice`, `warning`, `error` or `fatal`.

### Option: `node_max_old_space_size`

JavaScript heap limit in MB. `auto` (default) sizes it from the add-on's memory limit. Raise it for
very large libraries (250k+ books).

### Option: `oidc_allow_local_issuers`

Set to `true` to allow OIDC discovery against private/LAN addresses. **Required** if your Authentik,
Keycloak or Authelia runs on your own network — without it, discovery is rejected.

### Option: `disable_local_auth`

Rejects username/password sign-in. Only takes effect once at least one active administrator has
linked an enabled OIDC provider; BookOrbit refuses to start if enabling it would leave nobody able
to log in.

**Recovery:** if your identity provider is unavailable, set this back to `false` and restart.

### Option: `client_url`

Only needed when the web interface is served from a different origin than `app_url` (CORS). Leave
empty otherwise.

### Option: `trust_proxy`

Leave empty unless you know you need it. The default already trusts Home Assistant's own add-on
network, so a reverse proxy or `cloudflared` add-on works without changing this.

### Option: `timezone`

Overrides the timezone inherited from Home Assistant. Leave empty to follow Home Assistant.

### Options: `use_external_database` / `database_url`

Set `use_external_database: true` and provide a `database_url` such as
`postgres://user:password@host:5432/bookorbit` to use your own PostgreSQL server instead of the
bundled one. The bundled database will not start.

Your server must be **PostgreSQL 16 or newer** with these extensions available:
`uuid-ossp`, `pg_trgm`, `unaccent`, `vector` (pgvector).

## Network shares (NAS)

Most libraries live on a NAS. Use **Home Assistant's own network storage** rather than trying to
mount anything inside the add-on:

**Settings → System → Storage → Add network storage**, choose SMB or NFS, and pick usage
**media** or **share**.

The mount then appears to the add-on as `/media/<name>` or `/share/<name>`, and Home Assistant
handles credentials and reconnection after a NAS reboot. Set `library_browse_root` to match.

### NFS `root_squash` will break writes while reads look fine

Most NAS NFS exports default to `root_squash`, which maps root to `nobody`. BookOrbit will then
scan your library perfectly and fail the moment it writes — metadata write-back, Book Dock moves,
cover extraction — usually with a permission error buried in the log.

Fix it either way:

- export the share with `no_root_squash`, **or**
- set `puid` and `pgid` to the UID/GID that owns the files on the NAS.

SMB/CIFS mounts are not affected: Home Assistant mounts them so files appear root-owned, which
matches the default `puid: 0`.

### Book Dock does not auto-detect files on a share

`inotify` — which BookOrbit's Book Dock watcher uses — never fires for changes made on the server
side of an SMB or NFS mount. A file your NAS drops into the folder stays invisible until something
rescans.

Options, best first:

1. Put the Book Dock folder on Home Assistant's local storage instead of the NAS.
2. Trigger a rescan on a schedule from Home Assistant:

   ```yaml
   # configuration.yaml
   rest_command:
     bookorbit_dock_rescan:
       url: "http://homeassistant.local:3000/api/v1/book-dock/rescan"
       method: POST
       headers:
         Authorization: "Bearer YOUR_API_TOKEN"
   ```

   Then call `rest_command.bookorbit_dock_rescan` from a time-pattern automation.
3. Use the manual rescan button in the Book Dock UI.

**Libraries themselves are unaffected** — they use scheduled scans (configured per library), which
do not depend on inotify.

### Other things worth knowing

- **First scans over SMB are slow.** Cover extraction reads whole files. A large library's first
  scan can take hours. The database always stays on fast local storage.
- **Home Assistant backups do not include your books.** The add-on backup covers its own data
  (database, covers, book bucket) only. Your library stays the NAS's responsibility.
- **A NAS that goes offline** shows up as scan errors, not a crash. The add-on and its database keep
  running.

## Remote access

### Do I need HTTPS?

**No.** BookOrbit runs fine over plain HTTP, and so does Kobo Sync — a Kobo pointed at an
`http://` endpoint syncs without complaint.

What TLS protects is the per-device sync token travelling across an untrusted network. If you reach
BookOrbit over a VPN such as Tailscale or WireGuard, that traffic is already encrypted and plain
HTTP is fine. If you expose it to the internet, put it behind something that terminates TLS.

### Reverse proxy add-ons

Point your proxy (NGINX Proxy Manager, Traefik, Caddy) at `http://bookorbit:3000` on Home
Assistant's internal add-on network, and set `app_url` to the public HTTPS URL.

### Cloudflare Tunnel

Add a public hostname to your existing tunnel pointing at the add-on, and set `app_url` to that
hostname. No other option needs changing — the default trusted-proxy setting already covers Home
Assistant's add-on network, so BookOrbit sees the request as HTTPS and generates correct links.

Four things to watch:

1. **Cloudflare Access must bypass the device endpoints.** A Kobo, the KOReader plugin and OPDS
   clients cannot complete an interactive Access login — they receive the HTML login page instead of
   JSON and sync fails silently. Add a **Bypass** policy for those paths, or keep devices on a VPN
   and use the tunnel only for the browser interface.
2. **100 MB upload limit** on Cloudflare's free plan. Large audiobook or comic uploads through the
   tunnel will fail. Upload over the LAN/VPN, or use the Book Dock folder. Downloads are unaffected.
3. **100 s origin timeout (Error 524).** Reachable when a large book is converted on the fly.
4. **Pick your hostname before pairing devices** — see `app_url`.

## Kobo Sync

BookOrbit generates the endpoint URL for you in **Settings → Kobo**. The device configuration points
at your `app_url`, so it must be reachable from wherever the Kobo is.

Plain HTTP works. Over Tailscale or another VPN that is a perfectly reasonable setup.

## Home Assistant Ingress

This add-on does **not** use Ingress (the sidebar panel), and access is through its port instead.

BookOrbit's web client is built with absolute asset paths, so under an Ingress path prefix the
browser requests its JavaScript from Home Assistant's own URL space and never reaches the add-on.
Fixing that requires changes to BookOrbit's client build, not a proxy trick — it is planned for a
future release of this add-on.

Note that Ingress could never be the only way in regardless: Kobo, KOReader and OPDS devices have no
Ingress session and always need the add-on's real URL.

## Backups

The add-on uses **cold** backups: Home Assistant stops it briefly while the snapshot is taken. This
is deliberate — copying a running PostgreSQL data directory can produce an archive that will not
restore.

Backups include the database, covers and generated secrets. They do **not** include your book files.

## Upgrading PostgreSQL

The bundled database is PostgreSQL 18. If a future version of this add-on ships a newer major
version, the add-on **refuses to start** rather than risk your data, and the log says so:

```
PostgreSQL major version mismatch - refusing to start.
   Data directory: PostgreSQL 18
   This add-on:    PostgreSQL 19
```

Your data is intact. To migrate:

1. Roll back to the previous add-on version.
2. In BookOrbit, export anything you need, or take a full Home Assistant backup.
3. Stop the add-on and, with host SSH access, dump the database:
   `pg_dump --format=custom` from inside the add-on container.
4. Update the add-on, delete `/data/postgres`, start it (this creates a fresh cluster), then restore
   the dump with `pg_restore`.

Release notes will say when this applies. It will not happen silently.

## Troubleshooting

**`Permission denied` on `/init` or `/run/s6/basedir/bin/init` at startup.** Fixed in 2.9.0.3,
which drops the add-on's custom AppArmor profile in favour of the Supervisor's default one. Update
the add-on.

**The add-on stops right after starting.** Check the Log tab. The most common causes are an invalid
`app_url` (it must be a full URL including the scheme) and a PostgreSQL major-version mismatch.

**The folder picker cannot see my books.** Set `library_browse_root` to `/media`, `/share` or `/`
depending on where your storage is mounted.

**Scanning works but nothing can be written.** Almost always NFS `root_squash` — see
[Network shares](#network-shares-nas).

**I lost the setup token.** It is in the first-run log, and in `/data/secrets.env` inside the add-on.

**Kobo sync does nothing.** Check that `app_url` is reachable from the device, and that
Cloudflare Access (if you use it) is not intercepting the request.

## Support

Issues with **this add-on** (packaging, database, Home Assistant integration):
<https://github.com/andMaximus/hass-bookorbit/issues>

Issues with **BookOrbit itself** (the application, its features, its UI):
<https://github.com/bookorbit/bookorbit/issues>

This is an unofficial community add-on and is not affiliated with the BookOrbit project.
