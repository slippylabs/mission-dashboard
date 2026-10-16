# Deploying mission-dashboard

Upcoming launches and active crewed missions from Launch Library 2, over a
baked SVG world map with a live day/night terminator. Live at
`mission.slippylabs.com`.

| What | Where |
|---|---|
| Application | `/opt/mission-dashboard` (runs as `missiondash`) |
| Served static files | `/var/www/mission.slippylabs.com` |
| API | gunicorn on `127.0.0.1:8024`, proxied at `/api/` |
| Service | `mission-dashboard.service` |
| Cache | `cache/` — API responses and proxied imagery, not in git |

**No credentials.** Launch Library 2 is a public API and needs no key, so
there is no `.env` and nothing secret to move.

The files the original (pre-slippylabs) deployment used — `deploy.sh`, an
nginx vhost for another domain, `ssl-params.conf` and `mission-backend.service`
— were removed on 2026-10-16; they targeted a different host, port and user
and must never be copied onto this box.

## Files in `deploy/slippylabs/`

| Repo file | Live path |
|---|---|
| `mission-dashboard.service` | `/etc/systemd/system/mission-dashboard.service` |
| `10-memory.conf`, `10-narrow-writes.conf`, `cpu-priority.conf` | `/etc/systemd/system/mission-dashboard.service.d/` |
| `mission.slippylabs.com.conf` | `/etc/nginx/sites-available/` (symlinked into `sites-enabled/`) |

## Updating

Code and venv are **root-owned** (since 2026-10-16); the service user
`missiondash` owns only `cache/` and `.gunicorn/`, and the app dir itself is
`root:missiondash 0750`, so a bug in the service cannot rewrite its own code.
Install as root, never as missiondash. Dependencies: `requirements.txt` (pins
urllib3 >= 2.8.0); upgrade with `sudo -H venv/bin/pip install -r requirements.txt`.

```bash
sudo install -o root -g root -m 0644 app.py ll2.py /opt/mission-dashboard/
sudo systemctl restart mission-dashboard
# static front end
sudo cp -r www/. /var/www/mission.slippylabs.com/
```

## The rate limit shapes everything

Launch Library 2 allows very few requests per hour for anonymous callers, and
`cache/` exists entirely to stay inside it. Two consequences:

- **A fresh box starts cold.** The first page load is slow and the pre-warmer
  needs a few minutes to fill in. This is expected, not a fault.
- **Don't clear `cache/` casually.** Doing so mid-day can exhaust the hour's
  budget and leave the dashboard empty until it resets.

The image proxy exists for the same reason plus CSP: mission imagery is
fetched server-side, cached, and re-served same-origin.

## The world map is a build artifact

`www/assets/world.js` is generated, not hand-written:

```bash
./venv/bin/python tools/build_world.py
```

It's committed because regenerating it needs the source geometry and it
changes almost never. If the map ever renders blank, check that file exists
before looking anywhere else.

## Data quality filters worth keeping

LL2's feed contains entries that are technically launches but nonsense on a
mission dashboard — Starman, mass simulators, and similar. `ll2.py` filters
them deliberately. If something odd starts appearing, that filter is where to
look, and the right fix is usually to widen it rather than special-case the
display.

## Backups

There is nothing to back up. No database, no credentials, no user state — the
cache is disposable and the rest is in git. Losing this box costs the time to
re-clone and re-warm the cache.
