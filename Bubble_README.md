## Authentification and Gitlab Server Setup

How the bubble-robotics instance is deployed, start to finish. It is a slightly
awkward shape — Aldine shares a box with GitLab, so it cannot own ports 80/443
— and every step below exists because of that or because sign-in is Google-only.

- `https://aldine.bubble-robotics.com`, reachable from anywhere
- co-hosted with GitLab, proxied by GitLab Omnibus's bundled nginx
- accounts are Google Workspace only; there is no password login at all
- the checkout lives in `~/aldine`; system steps need `sudo`, compose calls
  assume your user is in the `docker` group (otherwise prefix those too)
- do not run the clone under `sudo -i`: `~` becomes `/root` and every path
  below stops matching. And `sudo cmd > /file` redirects as *your* user and
  fails — pipe through `sudo tee` instead

### 1. DNS

One record on `bubble-robotics.com`: CNAME `aldine` → `public.bubble-robotics.com`.
The name must resolve publicly before step 4 — Let's Encrypt validates from
outside the network.

```bash
dig +short aldine.bubble-robotics.com     # must end at the office public IP
```

### 2. Google OAuth client

In the Cloud project that already holds the GitLab client
(https://console.cloud.google.com):

1. **OAuth consent screen** — confirm **User type: Internal**. This is what
   limits sign-in to `bubble-robotics.com` accounts, and it is the *only* thing
   that does: Aldine creates an account for any verified email a configured
   provider returns.
2. **Credentials → Create credentials → OAuth client ID**, type **Web
   application**, authorised redirect URI exactly:
   `https://aldine.bubble-robotics.com/api/auth/oauth/google/callback`

A second client rather than reusing GitLab's: the two would share one secret,
so rotating it for Aldine would sign everyone out of GitLab. No JavaScript
origins are needed — the code exchange is server-side.

### 3. Code and config

```bash
git clone https://github.com/trahloff/Aldine.git ~/aldine
```

`~/aldine/.env`, mode 600, never committed:

```dotenv
ALDINE_PUBLIC_URL=https://aldine.bubble-robotics.com
# loopback only; nginx is the only thing that talks to the app
ALDINE_APP_BIND=127.0.0.1
# 8080 is GitLab's Puma; the vhost expects 8090
ALDINE_PORT=8090
# saves repeating -f flags on every compose call
COMPOSE_FILE=docker-compose.full.yml:deploy/docker-compose.prod.yml:images.yml
AUTH_ENABLED=1
# no password endpoints at all, so no SMTP and no reset flow to secure
ALDINE_SSO_ONLY=1
GOOGLE_OAUTH_CLIENT_ID=
GOOGLE_OAUTH_CLIENT_SECRET=
```

`images.yml` runs the published images instead of building; on a box already
serving GitLab, building the compiler means compiling TeX Live locally.
Validate the merge with `docker compose config >/dev/null`.

### 4. First certificate

The real vhost cannot be installed yet — nginx refuses to start when
`ssl_certificate` points at a file that does not exist — and bolting an ACME
location onto GitLab's own server block does not reliably work: with
`redirect_http_to_https` on, the port-80 block is a separate 301-only server,
and an unmatched `Host` lands on whichever block is nginx's default. The
challenge then reaches GitLab itself, which answers **404**.

So serve the challenge from a block that must match, via `deploy/nginx.aldine-bootstrap.conf`
(port 80, ACME only, no certificate references):

```ruby
# /etc/gitlab/gitlab.rb — reconfigure rewrites its own config on every run,
# which is why the vhost lives outside /var/opt/gitlab and is pulled in
nginx['custom_nginx_config'] = "include /etc/nginx/conf.d/*.conf;"
```

```bash
sudo apt install -y certbot
sudo mkdir -p /etc/nginx/conf.d /var/www/acme/.well-known/acme-challenge
sudo cp ~/aldine/deploy/nginx.aldine-bootstrap.conf /etc/nginx/conf.d/aldine.conf
sudo gitlab-ctl reconfigure
```

Prove the path works before spending a certbot attempt — Let's Encrypt allows
five failed validations per hostname per hour. Test with an explicit `Host`
header against loopback: from the box, the public name resolves to the office
public IP, which the router does not loop back, and `curl -s` hides the
resulting connection error so you get a confusing blank line.

```bash
echo probe | sudo tee /var/www/acme/.well-known/acme-challenge/probe
sudo chmod -R a+rX /var/www/acme
curl -i --max-time 5 -H 'Host: aldine.bubble-robotics.com' \
  http://127.0.0.1/.well-known/acme-challenge/probe        # want 200 + "probe"
```

403 means webroot permissions (nginx runs as `gitlab-www`); 404 means the
request is being handled by another server block. Then:

```bash
sudo certbot certonly --dry-run --webroot -w /var/www/acme -d aldine.bubble-robotics.com
sudo certbot certonly --webroot -w /var/www/acme -d aldine.bubble-robotics.com \
  --deploy-hook 'gitlab-ctl hup nginx' -m paul@bubble-robotics.com --agree-tos
```

`--webroot`, never `--nginx`: certbot cannot parse Omnibus's generated config.

### 5. Real vhost

```bash
sudo cp ~/aldine/deploy/nginx.aldine.conf /etc/nginx/conf.d/aldine.conf
sudo gitlab-ctl reconfigure          # bad config fails here; nothing is applied
sudo rm /var/www/acme/.well-known/acme-challenge/probe
curl -sI https://aldine.bubble-robotics.com | head -1      # 502 until step 6
```

That 502 is the success signal: TLS terminated, vhost matched, nothing upstream
yet. Check GitLab still answers too. The vhost keeps its own ACME location for
renewals, caps bodies at 32m (ZIP import), rate-limits `/api/auth/`, and names
its WebSocket map `$aldine_connection_upgrade` — GitLab defines
`$connection_upgrade` in the same http block, and duplicates are a hard error.

### 6. Start Aldine

```bash
cd ~/aldine && docker compose pull && docker compose up -d
docker compose ps               # app + compiler → healthy
```

### 7. Verify

- sign in with a work account; a personal `@gmail.com` must be refused by Google
- new project → ⌘S → PDF renders (compiler container + shared volume)
- same project in two browsers → live cursors (`/collab` WebSocket headers)
- `sudo certbot renew --dry-run`

### 8. Backups and upgrades

```bash
sudo cp ~/aldine/deploy/aldine-backup.{service,timer} /etc/systemd/system/
# systemd does not expand `~`; the unit needs the absolute path
sudo sed -i "s|^WorkingDirectory=.*|WorkingDirectory=$HOME/aldine|" \
  /etc/systemd/system/aldine-backup.service
sudo systemctl daemon-reload && sudo systemctl enable --now aldine-backup.timer
```

Daily 03:30 snapshot of both volumes to `/var/backups/aldine`, 14 kept; restore
with `deploy/restore.sh <tarball>` after `docker compose down`. Upgrades are
`git pull && docker compose pull && docker compose up -d`.

### When it breaks

| Symptom | Cause |
| --- | --- |
| `redirect_uri_mismatch` at Google | `ALDINE_PUBLIC_URL` differs from the console URI, character for character |
| 502 through nginx | app down, or `ALDINE_PORT` no longer matches the vhost |
| Login page with no Google button | `GOOGLE_OAUTH_*` blank — and `ALDINE_SSO_ONLY=1` leaves no other way in |
| ACME challenge 404 | a server block other than Aldine's answered; see step 4 |
| `duplicate map name` on reconfigure | the WebSocket map was renamed back to `$connection_upgrade` |
| Editor loads, no live cursors | the `/collab` block lost its `Upgrade` headers |

### Known gap

Nothing in Aldine restricts sign-in by email domain, so the Internal consent
screen is the whole boundary for an internet-facing instance: the day that
client is switched to External, or a GitHub login client is configured here,
this becomes open registration. Worth closing with a domain allowlist plus
Google's `hd` hint.