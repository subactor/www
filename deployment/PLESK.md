# Plesk deployment (subactor.com httpdocs)

Static site in this repository root (`index.html`, `assets/`, `pl/`,
`privacy.html`, `security.html`, `robots.txt`, `sitemap.xml`,
`site.webmanifest`, …) is published to the subscription `httpdocs/`.

Keep remote `.htaccess` and `.well-known/` (sync does not delete them).
Enable TLS, route `hello@subactor.com` and `security@subactor.com`, then
verify desktop, mobile, OpenGraph, CSP and `security.txt`.

## Automated path (preferred)

Architecture:

```text
NL / agent intent
  → Planfile ticket (uri_processes) or OQL plesk.site.sync
    → process.run / urirun-node
      → plesk://host/site/command/sync
        → SFTP (preferred) or FTP tree upload to /httpdocs
```

### URI contract

| URI | Role |
| --- | --- |
| `plesk://host/site/query/methods` | probe SFTP/FTP authorization |
| `plesk://host/site/command/sync` | dry-run plan (default) or apply upload |
| `plesk://host/site/command/publish` | alias of sync |

OQL equivalents (bridge): `plesk.site.sync` / `plesk.httpdocs.sync` (dry-run planner;
apply dispatches the URI when `TASK_RUNTIME_ENABLED` + urirun-node are up).

### Dry-run (safe default)

```bash
# Via panel / bridge process.run, or urirun CLI against the node registry:
# payload apply=false — never writes remote files
```

```json
{
  "uri": "plesk://host/site/command/sync",
  "payload": {
    "source_dir": "/absolute/path/to/www",
    "remote_path": "/httpdocs",
    "host": "YOUR_PLESK_SSH_HOST",
    "domain": "subactor.com",
    "apply": false
  }
}
```

Source must be a directory named `www`, or under
`PLESK_SYNC_ALLOWED_SOURCES` (colon-separated prefixes).

Reusable Planfile recipe: see `www-httpdocs-sync.urirun.json` and step-catalog
modules `sync_www_httpdocs_dry_run` / `create_www_httpdocs_sync_ticket`.

### Apply (explicit opt-in only)

1. Dry-run must succeed and look correct.
2. Vault entries `plesk-sftp` (and/or `plesk-ftp`) must exist.
3. Export confirmation env on the urirun-node host:

```bash
export PLESK_SYNC_APPLY=1
```

4. Re-run the same URI with `"apply": true` (human_approval on the ticket apply
   step). Without `PLESK_SYNC_APPLY=1` the connector returns
   `plesk_sync_apply_required` and does not upload.

### Manual fallback

If urirun is unavailable, upload the repository root contents to `httpdocs/`
with your usual SFTP/FTP client, preserving `.htaccess` and `.well-known/`.
