# n8n

Workflow automation for the deal pipeline, on **VPS2**. The UI and webhooks
are at `https://n8n.webmanage.net`: Cloudflare, then VPS1's Caddy (`hosting`
repo, `N8N_UPSTREAM=http://10.20.0.2:5678`), then over the WireGuard tunnel
to this container, which listens only on VPS2's tunnel address `10.20.0.2`.

Moved here from the `reviewgate` stack on VPS1 on 2026-09-30 (more memory on
VPS2; VPS1 is the tighter host) and upgraded from 1.95.3 to 2.41.4 in the same
step. No workflows existed at the time; [WORKFLOW_MAP.md](WORKFLOW_MAP.md) is
the plan for them. reviewgate can queue reruns through an n8n webhook when its
`N8N_REVIEW_RERUN_WEBHOOK_URL` is set; that URL is unaffected by the move.

## Files

| File | Purpose |
|---|---|
| `docker-compose.yml` | The `n8n` service, data volume `n8n_data`, port `10.20.0.2:5678` |
| `.env.example` | Settings; copy to `.env` (`chmod 600`), never commit it |
| `workflows/` | Exported workflow JSON, mounted read-only at `/workflows` |

`N8N_ENCRYPTION_KEY` decrypts saved credentials. It must stay the same for as
long as the data volume exists; a different key makes every saved credential
unreadable.

## First-time setup on VPS2

Docker must start after WireGuard (`deal-pipeline`
`scripts/install_docker_after_wireguard.sh`), or n8n can't bind `10.20.0.2`
at boot.

```bash
git clone https://github.com/SamTaneja8/n8n.git ~/n8n && cd ~/n8n
cp .env.example .env && chmod 600 .env   # set N8N_ENCRYPTION_KEY (see "Moving from VPS1")
docker compose up -d
curl -s -o /dev/null -w "%{http_code}\n" http://10.20.0.2:5678/healthz   # 200
```

### Moving from VPS1 (done once, 2026-09-30)

1. VPS1: back up the old volume with n8n stopped:
   `docker stop reviewgate-n8n && docker run --rm -v reviewgate_n8n_data:/v:ro -v ~/:/out alpine tar czf /out/n8n_data-$(date +%F).tar.gz -C /v . ; docker start reviewgate-n8n`
2. Copy the archive to VPS2 (`scp`), and copy `N8N_ENCRYPTION_KEY` from
   `~/reviewgate/.env` into `~/n8n/.env` by hand (never through chat).
   Compare fingerprints on both hosts:
   `grep -oP '^N8N_ENCRYPTION_KEY=\K.*' <file> | sha256sum | cut -c1-12`
3. VPS2: restore into the new volume, then start:
   `docker volume create n8n_data && docker run --rm -v n8n_data:/v -v ~/:/in alpine sh -c 'tar xzf /in/n8n_data-*.tar.gz -C /v && chown -R 1000:1000 /v' && cd ~/n8n && docker compose up -d`
4. VPS1: point Caddy at VPS2 (`N8N_UPSTREAM=http://10.20.0.2:5678` in
   `~/hosting/.env`, then `~/hosting/scripts/up.sh`) and log in at
   `https://n8n.webmanage.net`.
5. VPS1: remove the old container once VPS2 works
   (`cd ~/reviewgate && docker compose up -d --remove-orphans`). Keep the
   old volume and archive for a week, then `docker volume rm reviewgate_n8n_data`.

## Upgrading

Pin a version; never follow `latest`/`stable`. Find the version the `stable`
tag points to, read n8n's breaking changes for the versions in between, back
up, then change `N8N_IMAGE` in `.env`:

```bash
docker pull -q docker.n8n.io/n8nio/n8n:stable && docker run --rm --entrypoint n8n docker.n8n.io/n8nio/n8n:stable --version
docker stop n8n && docker run --rm -v n8n_data:/v:ro -v ~/:/out alpine tar czf /out/n8n_data-$(date +%F).tar.gz -C /v . ; docker start n8n
sed -i 's#^N8N_IMAGE=.*#N8N_IMAGE=docker.n8n.io/n8nio/n8n:<version>#' .env && docker compose up -d
docker logs --since 2m n8n 2>&1 | grep -iE "error|migrat" | head; curl -s -o /dev/null -w "%{http_code}\n" http://10.20.0.2:5678/healthz
```

Rollback: put the old version back in `N8N_IMAGE`, restore the archive into
`n8n_data` (as in "Moving from VPS1", step 3, after `docker compose down`),
`docker compose up -d`. A newer n8n migrates its database on start, so an
older version needs the pre-upgrade archive, not just the old image.

## Backups

The data (SQLite database, settings, encrypted credentials) lives in the
`n8n_data` volume, which the nightly database backups don't cover. Take an
archive (the `docker stop … tar … docker start` line above) before every
upgrade, and export workflows to `workflows/` and commit them when you change
them: `docker exec n8n n8n export:workflow --all --separate --output=/tmp/wf && docker cp n8n:/tmp/wf/. ./workflows/`.
