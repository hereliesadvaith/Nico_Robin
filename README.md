# Nico Robin

An example of setting up an AI assistant that has data and workflows behind it.
Nothing here is assistant-specific code: the stack runs n8n and Directus, both
of which expose an MCP server, and a client connected to the two of them gets
whatever data and whatever workflows you have put in them.

Directus is the data side — collections, files, anything you model in it — and
n8n is the action side. Add a collection and the assistant can read it; build a
workflow and the assistant can run it. What the assistant can do is exactly
what those two hold, which is the point of the example.

## What's here

| File | Purpose |
| --- | --- |
| `compose.yml` | n8n and Directus, each with its own Postgres. |
| `.env.example` | Template for the settings `compose.yml` requires. |

Both apps also serve their own API to the other over the compose network, so a
workflow can read and write Directus content at `http://directus:8055` without
going out through the proxy.

The two apps get a Postgres each rather than sharing one, so neither can reach
the other's data and either can be moved or rebuilt on its own.

n8n publishes 5678 and Directus 8055, both on `127.0.0.1` only; neither Postgres
publishes anything. TLS is expected to terminate in a reverse proxy you run on
the server.

---

## Server setup

### 1. What you need

- A Linux server (Ubuntu 22.04/24.04) with **4 GB RAM** for the full stack —
  two Node apps and two Postgres instances.
- `your-domain.com` and `cms.your-domain.com` pointing at the server's public
  IP. Confirm before requesting a certificate:

  ```bash
  dig +short your-domain.com
  curl -s https://api.ipify.org; echo
  ```

  If the server has a dynamic IP, install your DNS provider's updater (DuckDNS,
  No-IP and similar ship a cron script) so the record follows it.
- Ports **80** and **443** open. Port 80 is needed for Let's Encrypt's HTTP-01
  challenge.

### 2. Install Docker

```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER   # log out and back in
```

### 3. Firewall

Open SSH and the web ports only. **5678 and 8055 stay closed** — nginx reaches
both apps over loopback.

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

Generate the secrets and put them in `.env`:

```bash
openssl rand -hex 32   # -> N8N_ENCRYPTION_KEY
openssl rand -hex 24   # -> N8N_DB_PASSWORD
openssl rand -hex 16   # -> DIRECTUS_KEY
openssl rand -hex 32   # -> DIRECTUS_SECRET
openssl rand -hex 24   # -> DIRECTUS_DB_PASSWORD
chmod 600 .env
```

Set the domains and the Directus admin email and password too. Compose refuses
to start if anything is missing, rather than booting with a broken config.

> **Back up `.env` off the server.** `N8N_ENCRYPTION_KEY` decrypts every stored
> credential. A database backup restored without the matching key gives you
> workflows whose credentials cannot be read. Never change it once workflows
> exist.

### 5. Start

```bash
docker compose up -d
docker compose ps
docker compose logs -f n8n directus
```

n8n is on `127.0.0.1:5678` and Directus on `127.0.0.1:8055`, neither yet
reachable from outside. Directus runs schema migrations on first boot, so it
takes a minute longer than n8n to come up.

### 6. Reverse proxy

Install nginx and certbot:

```bash
sudo apt install -y nginx certbot python3-certbot-nginx
```

Create `/etc/nginx/sites-available/n8n`:

```nginx
server {
    listen 80;
    server_name your-domain.com;

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

And `/etc/nginx/sites-available/directus`:

```nginx
server {
    listen 80;
    server_name cms.your-domain.com;

    location / {
        proxy_pass http://127.0.0.1:8055;

        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Directus pushes realtime updates over a websocket.
        proxy_http_version 1.1;
        proxy_set_header Upgrade    $http_upgrade;
        proxy_set_header Connection "upgrade";

        # File uploads into the asset library.
        client_max_body_size 100m;
    }
}
```

Enable them and add TLS:

```bash
sudo ln -s /etc/nginx/sites-available/n8n /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/directus /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
sudo certbot --nginx -d your-domain.com -d cms.your-domain.com
```

Certbot edits the server blocks to listen on 443 and installs a renewal timer.

### 7. Verify

```bash
curl -I https://your-domain.com                     # 200, no TLS warning
curl -I http://your-domain.com                      # 301 to https
curl -s https://cms.your-domain.com/server/ping     # pong
ss -ltn | grep -E '5678|8055'                       # 127.0.0.1 only
```

Then open <https://your-domain.com> and create the n8n owner account. Do this
immediately: until you do, the setup page is open to anyone who finds the
domain, and a hostname is not a secret.

Log into Directus at <https://cms.your-domain.com> with `DIRECTUS_ADMIN_EMAIL`
and `DIRECTUS_ADMIN_PASSWORD`, change that password, then clear the pair from
`.env`. For n8n's own calls, create a static access token in Directus rather
than reusing the admin login.

### Without a reverse proxy

If you skip nginx there is no TLS: the login password and every credential you
enter cross the network in the clear. Only reasonable for a short test.

Change the port publishing in `compose.yml` from `127.0.0.1:5678:5678` to
`5678:5678`, set `N8N_PUBLIC_URL=http://your-domain.com:5678` in `.env`, and
open the port with `sudo ufw allow 5678/tcp`. Move to the proxy setup before
connecting any real account.

## Backups

Two things are needed to restore, and both must be kept:

1. **Both databases** — n8n's workflows, credentials and execution history, and
   Directus's schema and content.
2. **`.env`** — without `N8N_ENCRYPTION_KEY` the restored n8n credentials are
   undecryptable.
3. **`directus_uploads`** — files in the asset library live on disk, not in the
   database.

```bash
docker compose exec -T n8n-postgres pg_dump -U n8n n8n \
  | gzip > n8n-db-$(date +%F).sql.gz
docker compose exec -T directus-postgres pg_dump -U directus directus \
  | gzip > directus-db-$(date +%F).sql.gz

# uploads (confirm the volume name with `docker volume ls`;
# the prefix is the project directory name)
docker run --rm -v nico-robin_directus_uploads:/data:ro -v "$PWD":/backup \
  alpine tar czf /backup/directus-uploads-$(date +%F).tar.gz -C /data .
```

Restore:

```bash
docker compose up -d n8n-postgres directus-postgres
gunzip -c n8n-db-2026-01-01.sql.gz \
  | docker compose exec -T n8n-postgres psql -U n8n -d n8n
gunzip -c directus-db-2026-01-01.sql.gz \
  | docker compose exec -T directus-postgres psql -U directus -d directus
docker compose up -d
```

Copy the dumps and `.env` off the server, and run them from cron.

## Updates

Take the backups above first, then:

```bash
docker compose pull
docker compose up -d
docker image prune -f
```

Images are pinned (`n8n:2.36.9`, `directus:12.3.1`, `postgres:16-alpine`), so a
`pull` cannot move you across a major version unexpectedly. Bump the tags in
`compose.yml` deliberately, and take the dumps first — both apps run
irreversible schema migrations on startup, so rolling back means restoring the
backup.

## Operational notes

- **`N8N_PROXY_HOPS=1`** and **`IP_TRUST_PROXY=1`** tell each app to trust
  `X-Forwarded-*` from exactly one proxy. Raise them if you later add Cloudflare
  or a load balancer in front of nginx.
- **Execution history is pruned** at 14 days (`EXECUTIONS_DATA_*`). Without this
  the database grows until the disk fills.
- **Container logs are capped** at 10 MB × 3 files per service.
- **Health checks gate startup order**: each app waits for its Postgres to
  accept connections, so a reboot brings the stack up in the right order.
- **n8n reaches Directus at `http://directus:8055`** on the compose network,
  which never leaves the host.
- **Postgres, not SQLite**, which risks corruption on unclean shutdown.

## Status

This is the primary setup — the stack, nothing on top of it. From here you
create your own data in Directus (an `expenses` collection, notes, whatever you
want to track) and your own workflows in n8n, and connect a client to both MCP
servers. That is what turns it into your assistant.
