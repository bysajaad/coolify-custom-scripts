---
name: coolify-template
description: "Create a one-click deployable Docker Compose service template for Coolify (coollabsio/coolify). Use when the user wants to add a new service to Coolify's template catalog."
trigger: /coolify-template
---

# /coolify-template

Turn an open-source project's Docker Compose setup into a Coolify one-click service template, including the YAML file, SVG logo, and a PR to the official repo.

## Usage

```
/coolify-template <service-name>
/coolify-template openreplay
/coolify-template <github-url>
```

---

## Phase 1 — Research

Before writing anything, fetch the upstream project's official Docker Compose:

1. Find `docker-compose.yml` (or `.yaml`) in the project's GitHub repo — check `docker-compose/`, `deploy/`, `scripts/`, and the repo root.
2. Fetch ALL referenced env files (`.env`, `docker-envs/*.env`, `common.env`). Every `${VAR}` in the compose file must be resolved.
3. Note the **entry point**: which container/port is the public-facing one.
4. Note any **external config files** mounted into containers — these must be embedded (Coolify can't mount external files).
5. Check whether there are **migration steps** that must run before the app is healthy.
6. Find the SVG logo: check `frontend/app/svg/`, `static/`, `public/`, or the project's brand assets.

---

## Phase 2 — Template File

Create `templates/compose/<service>.yaml`. The repo root is `/Users/sajad/GitHub/bysajaad/coolify`.

### Comment Header (required)

```yaml
# documentation: https://docs.example.com/deploy/
# slogan: One-sentence description of what the service does.
# category: <one of: analytics, automation, backend, cms, database, dev, monitoring, storage, ...>
# tags: tag1, tag2, tag3
# logo: svgs/<service>.svg
# amd_only: true        # include only if the images have no ARM builds
```

Do NOT include `# port:` when using Traefik labels for routing — it conflicts with multi-service setups.

### Variable Conventions

| Pattern | Purpose | Example |
|---|---|---|
| `$SERVICE_PASSWORD_NAME` | Random 24-char password | `$SERVICE_PASSWORD_POSTGRES` |
| `$SERVICE_PASSWORD_64_NAME` | Random 64-char secret (for JWTs, S3 keys) | `$SERVICE_PASSWORD_64_JWT` |
| `$SERVICE_USER_NAME` | Random username | `$SERVICE_USER_REDIS` |
| `SERVICE_URL_SERVICENAME_PORT` | Assigns a Coolify domain to this service+port | `SERVICE_URL_FRONTEND_8080` |
| `${SERVICE_URL_SERVICENAME}` | Reference the full URL (https://...) | `AWS_ENDPOINT=${SERVICE_URL_FRONTEND}` |
| `${SERVICE_FQDN_SERVICENAME_PORT}` | Reference just the domain (no protocol) | Used in Traefik `Host()` rules |

**One service gets `SERVICE_URL_*`** — this becomes the public entry URL. For multi-service apps, give it to the frontend/SPA service.

### Routing: Traefik Labels (not nginx)

Coolify's Traefik handles all external routing. For multi-service apps where different URL paths go to different backends, use Traefik labels directly — not an nginx proxy container.

The service that declares `SERVICE_URL_FRONTEND_8080` gets Coolify's auto-generated catch-all route. Other services get explicit PathPrefix labels using `${SERVICE_FQDN_FRONTEND_8080}` as the Host:

```yaml
  chalice:
    labels:
      - traefik.enable=true
      - "traefik.http.routers.chalice-myapp.rule=Host(`${SERVICE_FQDN_FRONTEND_8080}`) && PathPrefix(`/api`)"
      - traefik.http.routers.chalice-myapp.priority=20
      - traefik.http.middlewares.chalice-strip.stripprefix.prefixes=/api
      - traefik.http.routers.chalice-myapp.middlewares=chalice-strip
      - traefik.http.services.chalice-myapp.loadbalancer.server.port=8000
```

Router name suffix (`-myapp`) must be globally unique across Traefik — always include the service name.

**Priority**: Traefik selects the most specific matching rule. Set `priority=20` (or higher) on path-specific routers to beat the Coolify auto-generated catch-all. Use higher numbers for longer/more specific prefixes (`/v2/api` → priority 30, `/api` → priority 20).

**Minio/S3 paths**: Do NOT strip the prefix — the full path (e.g., `/mobs/recording.bin`) must reach the storage backend so it resolves to `bucket=mobs, key=recording.bin`.

**WebSockets**: Traefik handles WebSocket upgrades natively; no special labels needed.

### Embedded Configs

Coolify cannot mount external config files. If a service needs a config file (nginx.conf, kong.yml, etc.), embed it in the container entrypoint:

```yaml
  nginx:
    image: nginx:1-alpine
    entrypoint:
      - /bin/sh
      - -c
      - |
        cat > /etc/nginx/conf.d/default.conf << 'EOF'
        server {
          listen 80;
          ...nginx config here ($ signs are safe inside single-quoted heredoc)...
        }
        EOF
        exec nginx -g 'daemon off;'
```

### Migrations / Init Containers

Use `restart: on-failure` + `depends_on: condition: service_healthy` for one-shot migration jobs:

```yaml
  db-migrate:
    image: postgres:17-alpine
    restart: on-failure
    depends_on:
      postgresql:
        condition: service_healthy
    entrypoint:
      - /bin/sh
      - -c
      - |
        apk add --no-cache wget
        wget -q -O /tmp/schema.sql https://raw.githubusercontent.com/owner/repo/v1.2.3/path/to/schema.sql
        psql -f /tmp/schema.sql 2>&1 || echo "May already exist"
    environment:
      - PGHOST=...
      - PGPASSWORD=$SERVICE_PASSWORD_POSTGRES
```

Always pin the schema URL to a specific version tag (not `main`) so re-deploys are deterministic. Verify the URL returns HTTP 200 before committing:
```bash
curl -s -o /dev/null -w "%{http_code}" "https://raw.githubusercontent.com/..."
```

### Internal DNS with Network Aliases

Use Docker network aliases so internal service-to-service calls use predictable hostnames:

```yaml
  postgresql:
    networks:
      myapp-net:
        aliases:
          - db.internal.myapp
```

Define a named network at the bottom and list it on every service that needs internal communication:

```yaml
networks:
  myapp-net:
```

### `AWS_ENDPOINT` for S3/MinIO

When services generate presigned URLs that browsers must access, set `AWS_ENDPOINT` to the **public URL** (`${SERVICE_URL_FRONTEND}`), not the internal minio address. This is because presigned URLs embed the endpoint — they must be reachable from the user's browser.

Only use the internal URL (`http://minio:9000`) for services that only write/read objects directly and never expose presigned URLs to users.

---

## Phase 3 — SVG Logo

Save at `public/svgs/<service>.svg`. Source priority:
1. Official brand assets in the project's GitHub repo (`static/`, `frontend/app/svg/`, `public/`)
2. Official website favicon/logo
3. Simple SVG mark (not the full wordmark)

Strip unnecessary metadata and `<title>` elements.

---

## Phase 4 — Regenerate JSON

```bash
php artisan generate:services
```

This regenerates `templates/service-templates-latest.json` and `templates/service-templates.json`.

**Do NOT commit these JSON files** — they are in the blocked-paths list of the PR quality check and are auto-regenerated by CI.

Verify the entry was created:
```bash
python3 -c "import json; d=json.load(open('templates/service-templates-latest.json')); e=d['<service>']; print('OK:', list(e.keys()))"
```

---

## Phase 5 — PR to coollabsio/coolify

### Setup (first time)

```bash
gh auth login
gh repo fork coollabsio/coolify --clone=false
git remote add fork https://github.com/<username>/coolify.git
```

### Branch from `next`

```bash
git fetch origin next
git checkout -b feat/<service>-template origin/next
git add templates/compose/<service>.yaml public/svgs/<service>.svg
git commit -m "feat(services): add <Service> one-click service template"
git push fork feat/<service>-template
```

Branch name must NOT be `main`, `master`, or `v4.x`.

### Diagnosing a rejected PR

If the quality bot closes the PR, inspect the workflow directly — the bot comment is generic:

```bash
cat .github/workflows/pr-quality.yaml   # read the actual rules before guessing
```

Common rejection causes in this repo:
- Wrong target branch (`v4.x` not allowed — must be `next`)
- Committed `service-templates-latest.json` or `service-templates.json` (blocked paths)
- Description over 2500 characters
- Commit footer containing `Co-Authored-By: Claude` or `Generated with Claude Code`

### PR Quality Rules (enforced by `peakoss/anti-slop`)

| Rule | Requirement |
|---|---|
| Target branch | Must be `next` |
| Source branch | Must NOT be `main`, `master`, or `v4.x` |
| Description length | ≤ 2500 characters |
| Blocked terms | No `STRAWBERRY`, no `Generated with Claude Code` |
| Blocked files | Do not include `service-templates-latest.json` or `service-templates.json` |
| Commit footer | No `Co-Authored-By: Claude` in commit messages |
| PR template | Must include the Contributor Agreement section |
| Conventional title | Must follow `type(scope): description` format |
| Account age | GitHub account must be ≥ 30 days old |

The word `STRAWBERRY` is a hidden anti-bot trap in the PR template — it's designed to auto-close AI-generated PRs that blindly follow instructions. Do not include it.

### PR Description Template

```markdown
## Changes

<human-written description of what was added and why>

## Issues

- Fixes N/A

## Category

- [ ] Bug fix
- [ ] Improvement
- [ ] New feature
- [x] Adding new one click service
- [ ] Fixing or updating existing one click service

## Preview

<screenshot or note that live testing is pending>

## AI Assistance

- [ ] AI was NOT used to create this PR
- [x] AI was used (please describe below)

**If AI was used:**

- Tools used: Claude (claude.ai/code)
- How extensively: <brief honest description>

## Testing

- YAML validated via `php artisan generate:services`
- Schema migration URLs verified at pinned version tag
- <any other tests run>

## Contributor Agreement

> [!IMPORTANT]
>
> - [x] I have read and understood the contributor guidelines.
> - [x] I have searched existing issues and PRs to ensure this isn't a duplicate.
> - [ ] I have tested all the changes thoroughly with a local development instance of Coolify.
```

### Submit

```bash
gh pr create \
  --repo coollabsio/coolify \
  --head <username>:feat/<service>-template \
  --base next \
  --title "feat(services): add <Service> one-click service template" \
  --body "..."
```

---

## Checklist

- [ ] Upstream Docker Compose and all env files read
- [ ] Entry point service identified, `SERVICE_URL_*_PORT` assigned
- [ ] All `${COMMON_*}` / `${APP_*}` variables mapped to `SERVICE_PASSWORD_*` equivalents
- [ ] Traefik labels added for every path that needs public routing
- [ ] External config files embedded (no volume mounts of local files)
- [ ] Migration/init containers added if schema setup is required
- [ ] Schema URLs pinned to a version tag and verified HTTP 200
- [ ] SVG logo saved to `public/svgs/<service>.svg`
- [ ] `php artisan generate:services` ran without errors
- [ ] JSON files NOT staged
- [ ] PR targets `next` branch
- [ ] PR description ≤ 2500 chars, no blocked terms
