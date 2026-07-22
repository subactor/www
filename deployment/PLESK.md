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
export PLESK_SYNC_ALLOWED_SOURCES=/home/tom/github/subactor
PYTHONPATH=. python3 -c '
from urirun_connector_plesk.core import site_sync
import json, os
assert not os.environ.get("PLESK_SYNC_APPLY"), "refuse: PLESK_SYNC_APPLY set"
r = site_sync(
  source_dir="/home/tom/github/subactor/subactor-com",
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
  sourceDir: "/home/tom/github/subactor/subactor-com",
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
    "source_dir": "/home/tom/github/subactor/subactor-com",
    "remote_path": "/httpdocs",
    "host": "prototypowanie.pl",
    "domain": "subactor.com",
    "apply": false
  }
}
```

Source must be under `PLESK_SYNC_ALLOWED_SOURCES` (colon-separated prefixes).
The canonical source is `/home/tom/github/subactor/subactor-com`; the logical
resource alias remains `workspace:www` for backward-compatible founder commands.

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
3. Open one bounded mutation window: either set both kill switches on the
   urirun-node host:

```bash
export AUTONOMY_MUTATIONS_ENABLED=1
export PLESK_SYNC_APPLY=1
```

   or use the Founder CLI's short-lived mutate lease. Do not leave either
   mechanism enabled permanently.

4. Issue a signed, single-use apply grant bound to the unchanged dry-run
   `plan_hash`, exact target, actor and intent pack. The grant is an ephemeral
   credential: pass it directly to the runtime and never persist it in a ticket,
   manifest, log or repository.

5. Re-run the same URI with `"apply": true`, the exact `plan_hash` and the
   grant. The supported autonomous route is:

```bash
cd /home/tom/github/subactor/platform
node packages/founder-cli/bin/subactor.mjs ask "opublikuj www na subactor.com" --apply --yes
```

The Bridge request timeout must be at least the configured Plesk transport
budget plus verification overhead. The current runtime derives it from
`PLESK_TRANSPORT_TOTAL_BUDGET` and adds 15 seconds.

OpenRouter / `subactor ask` is only needed for NL → plan routing. FTP/SFTP
ensure + sync are deterministic connector calls — platform orchestrates them
without an LLM once the URI/plan is known.

### Manual fallback

If urirun is unavailable, upload the repository root contents to `httpdocs/`
with your usual SFTP/FTP client, preserving `.htaccess` and `.well-known/`.
