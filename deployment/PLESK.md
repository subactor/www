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

Confirm apply gate is off first (`PLESK_SYNC_APPLY` must be empty / unset).

**Exact local dry-run (connector package, no remote write):**

```bash
cd /home/tom/github/urirun-connectors/urirun-connector-plesk
PYTHONPATH=. python3 -c '
from urirun_connector_plesk.core import site_sync
import json, os
assert not os.environ.get("PLESK_SYNC_APPLY"), "refuse: PLESK_SYNC_APPLY set"
r = site_sync(
  source_dir="/home/tom/github/subactor/www",
  remote_path="/httpdocs",
  host="prototypowanie.pl",
  domain="subactor.com",
  apply=False,
)
print(json.dumps({k: r[k] for k in ("ok","dry_run","files_planned","domain","host","remote_path","preserve_remote") if k in r}, indent=2))
'
```

**Bridge OQL planner (same allowlist / hashes, also dry-run):**

```bash
cd /home/tom/github/subactor/connectors/services/bridge
node --input-type=module -e '
import { planHttpdocsSync } from "./src/plesk-httpdocs-sync.mjs";
const r = await planHttpdocsSync({
  sourceDir: "/home/tom/github/subactor/www",
  remotePath: "/httpdocs",
  host: "prototypowanie.pl",
  domain: "subactor.com",
  apply: false,
});
console.log(JSON.stringify({ok:r.ok, dry_run:r.dry_run, files_planned:r.files_planned, uri_process:r.uri_process}, null, 2));
'
```

```json
{
  "uri": "plesk://host/site/command/sync",
  "payload": {
    "source_dir": "/home/tom/github/subactor/www",
    "remote_path": "/httpdocs",
    "host": "prototypowanie.pl",
    "domain": "subactor.com",
    "apply": false
  }
}
```

Source must be a directory named `www`, or under
`PLESK_SYNC_ALLOWED_SOURCES` (colon-separated prefixes).

Reusable Planfile recipe: `www-httpdocs-sync.urirun.json`. Ticket import:
`www-httpdocs-sync.planfile-ticket.yaml` (or umbrella
`.planfile/imports/www-httpdocs-sync.yaml`). Step-catalog modules:
`sync_www_httpdocs_dry_run` / `create_www_httpdocs_sync_ticket`.

**NL (subactor-local, not desktop nlp2uri):** phrases in
`agents/nlp-uri-phrases.yaml` → `plesk://host/site/command/sync` (apply=false).
LLM intent model: `www-httpdocs-sync.pl.aql`.

### Apply (explicit opt-in only)

1. Dry-run must succeed and look correct.
2. Vault entries `plesk-sftp` (preferred) and/or `plesk-ftp` must exist
   (create via `plesk://host/ftpuser/command/ensure` with `kind=system`).
3. Export confirmation env on the urirun-node host:

```bash
export PLESK_SYNC_APPLY=1
```

4. Re-run the same URI with `"apply": true`. Founder/admin autonomy contracts
   should leave apply steps with `human_approval: false` when the env gate is
   the safety boundary. Without `PLESK_SYNC_APPLY=1` the connector returns
   `plesk_sync_apply_required` and does not upload.

OpenRouter / `subactor ask` is only needed for NL → plan routing. FTP/SFTP
ensure + sync are deterministic connector calls — platform orchestrates them
without an LLM once the URI/plan is known.

### Manual fallback

If urirun is unavailable, upload the repository root contents to `httpdocs/`
with your usual SFTP/FTP client, preserving `.htaccess` and `.well-known/`.
