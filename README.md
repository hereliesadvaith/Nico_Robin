# Nico Robin

An AI personal assistant built on [n8n](https://n8n.io/), with the personality of
Nico Robin from *One Piece*.

The assistant connects to n8n workflows, so anything that can be wired up as a
workflow — reminders, searches, messages, home automation, and so on — becomes
something it can do.

Deployment target: **https://advaith.duckdns.org**

## What's here

| File | Purpose |
| --- | --- |
| `compose.yml` | n8n + Postgres. |
| `.env.example` | Template for the settings `compose.yml` requires. |
| `files/` | Mounted at `/files` in the container, for workflows that read or write files. |

n8n publishes port 5678 on `127.0.0.1` only, and Postgres publishes nothing.
TLS is expected to terminate in a reverse proxy you run on the server.

---

## Server setup

### 1. What you need

- A Linux server (Ubuntu 22.04/24.04) with **2 GB RAM minimum**. Postgres plus
  an AI Agent workflow will OOM-kill Node on 1 GB.
- `advaith.duckdns.org` pointing at the server's public IP. Confirm before
  requesting a certificate:

  ```bash
  dig +short advaith.duckdns.org
  curl -s https://api.ipify.org; echo
  ```

  DuckDNS records are updated by pinging their API — if the server has a dynamic
  IP, install their updater cron so the record follows it.
- Ports **80** and **443** open. Port 80 is needed for Let's Encrypt's HTTP-01
  challenge.

### 2. Install Docker

```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER   # log out and back in
```

### 3. Firewall

Open SSH and the web ports only. **5678 stays closed** — nginx reaches n8n over
loopback.

```bash
sudo ufw allow OpenSSH
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
```

### 4. Configure

```bash
git clone <this-repo> nico-robin && cd nico-robin
cp .env.example .env
```

Generate the two secrets and put them in `.env`:

```bash
openssl rand -hex 32   # -> N8N_ENCRYPTION_KEY
openssl rand -hex 24   # -> POSTGRES_PASSWORD
chmod 600 .env
```

The domain values are already set for `advaith.duckdns.org`. Compose refuses to
start if either secret is missing, rather than booting with a broken config.

> **Back up `.env` off the server.** `N8N_ENCRYPTION_KEY` decrypts every stored
> credential. A database backup restored without the matching key gives you
> workflows whose credentials cannot be read. Never change it once workflows
> exist.

### 5. Start

```bash
docker compose up -d
docker compose ps          # both services should reach (healthy)
docker compose logs -f n8n
```

n8n is now on `127.0.0.1:5678`, not yet reachable from outside.

### 6. Reverse proxy

Install nginx and certbot:

```bash
sudo apt install -y nginx certbot python3-certbot-nginx
```

Create `/etc/nginx/sites-available/n8n`:

```nginx
server {
    listen 80;
    server_name advaith.duckdns.org;

    location / {
        proxy_pass http://127.0.0.1:5678;

        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # n8n pushes execution updates over a websocket.
        proxy_http_version 1.1;
        proxy_set_header Upgrade    $http_upgrade;
        proxy_set_header Connection "upgrade";

        # Long-running workflows must not be cut off mid-execution.
        proxy_read_timeout 3600s;
        proxy_send_timeout 3600s;

        # Allow reasonable file uploads into workflows.
        client_max_body_size 50m;
    }
}
```

Enable it and add TLS:

```bash
sudo ln -s /etc/nginx/sites-available/n8n /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
sudo certbot --nginx -d advaith.duckdns.org
```

Certbot edits the server block to listen on 443 and installs a renewal timer.

### 7. Verify

```bash
curl -I https://advaith.duckdns.org        # 200, no TLS warning
curl -I http://advaith.duckdns.org         # 301 to https
docker compose ps                          # both (healthy)
ss -ltn | grep 5678                        # should show 127.0.0.1:5678 only
```

Then open <https://advaith.duckdns.org> and create the owner account. Do this
immediately: until you do, the setup page is open to anyone who finds the
domain — and DuckDNS subdomains are guessable.

### Without a reverse proxy

If you skip nginx, n8n has to be exposed directly and there is no TLS: the login
password and every credential you enter cross the network in the clear. Only
reasonable for a short test. In `.env`:

```ini
N8N_BIND=0.0.0.0
N8N_PROTOCOL=http
N8N_PUBLIC_URL=http://advaith.duckdns.org:5678
N8N_PROXY_HOPS=0
N8N_SECURE_COOKIE=false
```

and open the port with `sudo ufw allow 5678/tcp`. Move to the proxy setup before
connecting any real account.

## Backups

Two things are needed to restore, and both must be kept:

1. **The Postgres database** — workflows, credentials, execution history.
2. **`.env`** — without `N8N_ENCRYPTION_KEY` the restored credentials are
   undecryptable.

The `n8n_data` volume also holds binary execution data, worth including if your
workflows handle files.

```bash
# database
docker compose exec -T postgres pg_dump -U n8n n8n | gzip > n8n-db-$(date +%F).sql.gz

# n8n data volume (confirm the name with `docker volume ls`;
# the prefix is the project directory name)
docker run --rm -v nico-robin_n8n_data:/data:ro -v "$PWD":/backup \
  alpine tar czf /backup/n8n-data-$(date +%F).tar.gz -C /data .
```

Restore:

```bash
docker compose up -d postgres
gunzip -c n8n-db-2026-01-01.sql.gz | docker compose exec -T postgres psql -U n8n -d n8n
docker compose up -d
```

Copy the dumps and `.env` off the server, and run the dump from cron.

## Updates

```bash
docker compose exec -T postgres pg_dump -U n8n n8n | gzip > pre-upgrade.sql.gz
docker compose pull
docker compose up -d
docker image prune -f
```

Images are pinned (`n8n:2.36.9`, `postgres:16-alpine`), so a `pull` cannot move
you across a major version unexpectedly. Bump the tag in `compose.yml`
deliberately, and take the dump first — n8n runs irreversible schema migrations
on startup, so rolling back means restoring the backup.

## Operational notes

- **`N8N_PROXY_HOPS=1`** tells n8n to trust `X-Forwarded-*` from exactly one
  proxy. Raise it if you later add Cloudflare or a load balancer in front of
  nginx; set it to 0 if nothing proxies n8n.
- **Execution history is pruned** at 14 days / 10,000 records
  (`EXECUTIONS_DATA_*`). Without this the database grows until the disk fills.
- **Container logs are capped** at 10 MB × 3 files per service.
- **Health checks gate startup order**: n8n waits for Postgres to accept
  connections, so a reboot brings the stack up in the right order.
- **Postgres, not SQLite**, which risks corruption on unclean shutdown. To go
  back, drop the `postgres` service and the `DB_*` variables; n8n falls back to
  SQLite inside the `n8n_data` volume.

## Status

Early work in progress — the stack is defined, but no assistant workflows exist
yet.
