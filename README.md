# coolify-custom-scripts

Custom Docker Compose files and service templates for [Coolify](https://coolify.io), plus a Claude Code skill to generate and submit them upstream.

---

## What's in this repo

| Path | Purpose |
|---|---|
| `SKILL.md` | Claude Code skill (`/coolify-template`) that automates template creation and both PRs |
| `openreplay/` | OpenReplay self-hosted compose (Iran mirror variant) |
| `posthog-iran-server/` | PostHog self-hosted compose (Iran mirror variant) |

---

## Submitting a new one-click service to coollabsio/coolify

The `/coolify-template` skill handles the full pipeline — from reading upstream docs to opening PRs in both the main repo and the docs repo. The steps below explain what happens and what you need to do manually.

### Prerequisites

- [Claude Code](https://claude.ai/code) installed and authenticated
- [GitHub CLI](https://cli.github.com/) (`gh`) installed and authenticated (`gh auth login`)
- A local clone of the official repo: `git clone https://github.com/coollabsio/coolify`
- PHP available in that repo's environment (for `php artisan generate:services`)
- [Bun](https://bun.sh/) available (for `bun run generate:services` in the docs repo)
- Your GitHub account must be **>= 30 days old** (enforced by the PR quality bot)

---

### Step 1 — Run the skill

Open Claude Code inside the **coollabsio/coolify** clone, then:

```
/coolify-template <service-name>
```

or pass a GitHub URL directly:

```
/coolify-template https://github.com/owner/your-service
```

The skill will:

1. Fetch the upstream `docker-compose.yml` and all referenced `.env` files
2. Identify the public entry point and map all variables to Coolify conventions (`SERVICE_PASSWORD_*`, `SERVICE_URL_*`, etc.)
3. Write `templates/compose/<service>.yaml`
4. Find and save the SVG logo to `public/svgs/<service>.svg`
5. Run `php artisan generate:services` to validate the YAML and regenerate JSON indexes
6. Clone `coollabsio/coolify-docs`, create the service `.mdx` page and regenerate the docs listings
7. Open both PRs automatically

---

### Step 2 — Review the generated template

Check the output file before committing:

- Every `${VAR}` is resolved — no undefined variables
- Traefik labels handle routing (no nginx proxy container)
- External config files are embedded in `entrypoint:` heredocs (Coolify cannot mount local files)
- Migration/init containers use `restart: on-failure` and `depends_on: condition: service_healthy`
- Schema SQL URLs are pinned to a version tag, not `main`

Verify the JSON was generated correctly:

```bash
python3 -c "
import json
d = json.load(open('templates/service-templates-latest.json'))
e = d['<service>']
print('OK:', list(e.keys()))
"
```

---

### Step 3 — Fork and create a branch

```bash
# First time only
gh repo fork coollabsio/coolify --clone=false
git remote add fork https://github.com/<your-username>/coolify.git

# Every PR
git fetch origin next
git checkout -b feat/<service>-template origin/next
```

Branch name must **not** be `main`, `master`, or `v4.x`.

---

### Step 4 — Stage and commit (carefully)

Stage only the two new files — do **not** stage the JSON indexes:

```bash
git add templates/compose/<service>.yaml public/svgs/<service>.svg
git commit -m "feat(services): add <Service> one-click service template"
```

Commit message rules enforced by the PR quality bot:

- Must follow `type(scope): description` (conventional commits)
- Must **not** contain `Co-Authored-By: Claude` or `Generated with Claude Code`

---

### Step 5 — Push and open the coolify PR

```bash
git push fork feat/<service>-template

gh pr create \
  --repo coollabsio/coolify \
  --head <your-username>:feat/<service>-template \
  --base next \
  --title "feat(services): add <Service> one-click service template" \
  --body "$(cat <<'EOF'
## Changes

Added a one-click service template for <Service>. <One or two sentences describing what the service is and why it's useful for self-hosters.>

## Issues

- Fixes N/A

## Category

- [ ] Bug fix
- [ ] Improvement
- [ ] New feature
- [x] Adding new one click service
- [ ] Fixing or updating existing one click service

## Preview

<screenshot or "live testing pending">

## AI Assistance

- [ ] AI was NOT used to create this PR
- [x] AI was used (please describe below)

**If AI was used:**

- Tools used: Claude (claude.ai/code)
- How extensively: Generated the compose YAML and SVG placement; reviewed and tested manually.

## Testing

- YAML validated via `php artisan generate:services`
- Schema migration URLs verified at pinned version tag

## Contributor Agreement

> [!IMPORTANT]
>
> - [x] I have read and understood the contributor guidelines.
> - [x] I have searched existing issues and PRs to ensure this isn't a duplicate.
> - [ ] I have tested all the changes thoroughly with a local development instance of Coolify.
EOF
)"
```

---

### Step 6 — Open the docs PR to coollabsio/coolify-docs

After the coolify PR is open, a GitHub Actions bot will comment asking for a matching docs PR. The skill handles this automatically, but if you're doing it manually:

```bash
cd /tmp && git clone https://github.com/coollabsio/coolify-docs.git && cd coolify-docs
bun install   # do NOT use --frozen-lockfile
```

Create `content/docs/services/<service>.mdx`:

```mdx
---
title: "ServiceName"
description: "Short description used on the listing card."
category: "Monitoring"
icon: "/docs/images/services/<service>-logo.svg"
---

# ServiceName

## What is ServiceName?

[2-3 paragraphs]

## Links

- [Official website](https://example.com?utm_source=coolify.io)
- [GitHub](https://github.com/org/repo?utm_source=coolify.io)
```

Valid categories (case-sensitive): `Administration`, `AI`, `Analytics`, `Automation`, `Backup`, `Bookmarks`, `Browser`, `Business`, `CMS`, `Communication`, `Database`, `Development`, `Documentation`, `Email`, `File Management`, `Finance`, `Forum`, `Gaming`, `Media`, `Monitoring`, `Networking`, `Productivity`, `Project Management`, `Security`, `Social Media`, `Storage`, `Utilities`, and more — see SKILL.md for the full list.

All external links must include `?utm_source=coolify.io`.

Copy the logo and regenerate listings:

```bash
cp /path/to/coolify/public/svgs/<service>.svg public/images/services/<service>-logo.svg
bun run generate:services
```

Commit and push — exactly 4 files:

```bash
git checkout -b feat/<service>-docs
git add content/docs/services/<service>.mdx \
        public/images/services/<service>-logo.svg \
        src/generated/services.json \
        content/docs/services/all.mdx
git commit -m "feat(services): add <Service> documentation"
git push origin feat/<service>-docs

gh pr create \
  --repo coollabsio/coolify-docs \
  --base next \
  --title "feat(services): add <Service> documentation" \
  --body "Adds documentation for <Service> to accompany coollabsio/coolify#<PR-number>."
```

Then reply to the bot comment on the main coolify PR:

```
Hi! I've opened a matching docs PR: coollabsio/coolify-docs#<docs-pr-number>
```

---

### PR quality rules (enforced automatically)

The repo uses [`peakoss/anti-slop`](https://github.com/peakoss/anti-slop) to auto-close low-quality PRs. The full list:

| Rule | Requirement |
|---|---|
| Target branch | Must be `next` |
| Source branch | Must not be `main`, `master`, or `v4.x` |
| Description length | <= 2500 characters |
| Blocked files | Do not include `service-templates-latest.json` or `service-templates.json` |
| Commit footer | No `Co-Authored-By: Claude` in commit messages |
| Blocked terms | No `STRAWBERRY` in the PR body (anti-bot trap in the PR template — do not copy it) |
| PR template | Must include the Contributor Agreement section |
| Conventional title | Must follow `type(scope): description` |
| Account age | GitHub account must be >= 30 days old |

If the quality bot closes the PR, read the actual workflow before guessing at the cause:

```bash
cat .github/workflows/pr-quality.yaml
```

---

### Diagnosing a rejected PR

Common causes:

- **Wrong base branch** — PR must target `next`, not `v4.x` or `main`
- **JSON files committed** — `service-templates*.json` are blocked; remove them with `git restore --staged templates/service-templates*.json`
- **`Co-Authored-By: Claude` in commit** — amend or squash the commit to remove it
- **Description too long** — keep it under 2500 characters
- **`STRAWBERRY` in body** — the PR template contains this word as a bot trap; never copy it verbatim

---

## SKILL.md reference

`SKILL.md` is a [Claude Code skill](https://docs.anthropic.com/claude-code/skills) that lives in this repo. It is automatically loaded when you run `/coolify-template` inside Claude Code.

The skill covers:

- Phase 1: Research (upstream compose + env files + logo)
- Phase 2: Template file (variable conventions, Traefik routing, embedded configs, migrations)
- Phase 3: SVG logo placement
- Phase 4: `php artisan generate:services` + JSON verification
- Phase 5: Fork setup, branch, commit, and PR to `coollabsio/coolify`
- Phase 6: Docs page, logo, `bun run generate:services`, and PR to `coollabsio/coolify-docs`

To use the skill in a different project, copy `SKILL.md` into that project's root or `.claude/` directory.
