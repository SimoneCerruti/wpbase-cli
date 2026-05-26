# wpbase

A tiny pure-bash CLI to manage [`containerized-wordpress`](https://github.com/simonecerruti/containerized-wordpress) releases across multiple WordPress projects on a single host.

It downloads versioned base tarballs from GitHub, applies them to per-project directories, and optionally rebuilds containers — all without needing `git` installed on the host.

## What it solves

You have several WordPress sites running on the same VPS, each in its own Docker Compose project. They all share the same hardened base image (WordPress + Nginx + PHP-FPM + cron + supervisor) and customize it through a strict override mechanism.

When you ship a new version of the base — fixing a bug, bumping PHP, hardening nginx — you want to roll it out to every site you maintain. Doing this by hand means `cd`-ing into each project, `wget`-ing a tarball, extracting it carefully into `base/`, bumping a version file, rebuilding, restarting. That's a script.

This is that script.

## Requirements

- Linux host (tested on Debian)
- `bash`, `curl`, `tar`, `sha256sum` — all standard
- `docker compose` **v2.24+** if you want `--build` (needed for the `include:` in `compose.base.yml`)
- Public internet access to `api.github.com` and `github.com`

No git on the host. No Python. No jq.

## Install

Pick a tagged release:

```bash
VERSION=1.0.0
curl -fsSL "https://raw.githubusercontent.com/simonecerruti/wpbase-cli/v${VERSION}/wpbase" \
  | sudo tee /usr/local/bin/wpbase > /dev/null
sudo chmod +x /usr/local/bin/wpbase
```

Or clone and copy:

```bash
git clone https://github.com/simonecerruti/wpbase-cli.git
sudo install -m 755 wpbase-cli/wpbase /usr/local/bin/wpbase
```

## Configure

`wpbase` reads from `/etc/wpbase/`:

```bash
sudo mkdir -p /etc/wpbase

# Point at the containerized-wordpress repo
echo "https://github.com/simonecerruti/containerized-wordpress" | sudo tee /etc/wpbase/repo

# List your projects (one absolute path per line)
sudo tee /etc/wpbase/projects.list <<'EOF'
/srv/projects/site-one
/srv/projects/site-two
/srv/projects/site-three
EOF
```

Lines starting with `#` are comments. Empty lines are ignored.

Cache lives in `/var/cache/wpbase/` (auto-created). Override paths via `WPBASE_CONF_DIR` and `WPBASE_CACHE_DIR` env vars.

## Usage

### List registered projects and their pinned versions

```bash
wpbase list
```

Output:

```
PROJECT                                            VERSION    STATUS
/srv/projects/site-one                             1.0.0      clean
/srv/projects/site-two                             1.0.0      modified
/srv/projects/site-three                           -          not-initialized
```

Statuses: `clean`, `modified` (manual drift detected), `missing-base`, `not-initialized`.

### See which versions are available

```bash
wpbase versions
```

Lists tags from `containerized-wordpress` GitHub releases, newest first.

### Bootstrap a new project

```bash
wpbase install 1.0.0 /srv/projects/new-site
```

Downloads `v1.0.0` and writes `base/`, `.base-version`, and a checksum into the project.

On a fresh project it also scaffolds the runnable project-root files from the release templates — `docker-compose.yml` (from `compose.example.yml`), `.env` (from `.env.example`), and `.env.wordpress` (from `.env.wordpress.example`) — **only if they don't already exist**. Existing files are never overwritten, and `wpbase update` leaves these project-root files alone entirely.

Then fill in the scaffolded files and keep the secrets out of git:

```bash
cd /srv/projects/new-site
$EDITOR .env .env.wordpress          # set project name, domain, DB creds, etc.
printf '.env\n.env.wordpress\n' >> .gitignore
docker compose up -d --build
```

Then add the path to `/etc/wpbase/projects.list`.

### Update a single project

```bash
# Dry-run-ish: confirm prompt, no rebuild
wpbase update /srv/projects/site-one --version 1.0.0

# Non-interactive, rebuild and restart
wpbase update /srv/projects/site-one --version 1.0.0 --yes --build
```

If you omit `--version`, the latest GitHub release is used.

### Update every project

```bash
# Default: dry-run, shows what would change
wpbase update-all

# Apply with per-project confirmation, no rebuild
wpbase update-all --version 1.0.0

# Apply everywhere, rebuild and restart everything
wpbase update-all --version 1.0.0 --yes --build
```

`update-all` is dry-run by default. You need to explicitly pass `--yes` or `--build` for it to do anything.

### Detect manual edits to `base/`

```bash
wpbase diff /srv/projects/site-one
```

Shows checksum drift and (if drifted) a `diff -rq` against the freshly fetched tarball.

## Flags

| Flag             | Meaning                                                  |
| ---------------- | -------------------------------------------------------- |
| `--version X.Y.Z`| Pin to a specific version (default: latest release)      |
| `--yes` / `-y`   | Skip confirmation prompts                                |
| `--build`        | Run `docker compose build && up -d` after update (needs Compose v2.24+) |
| `--dry-run`      | Show what would happen, do nothing                       |
| `--force`        | Reinstall even if already at the target version          |
| `--help` / `-h`  | Show help and exit                                       |

## How it works

For each project, `wpbase`:

1. Downloads `https://github.com/<repo>/archive/refs/tags/v<version>.tar.gz` into `/var/cache/wpbase/<version>/`
2. Extracts a subset (`configs/`, `scripts/`, `overrides/`, `Dockerfile.base`, `compose.base.yml`, `VERSION`) into `<project>/.base-staging-XXXXXX/`
3. Atomically swaps the staging dir into `<project>/base/`
4. Writes `<project>/.base-version` and `<project>/base/.checksum`
5. Optionally runs `docker compose build --no-cache wordpress && docker compose up -d`

Drift detection compares the stored checksum against a fresh hash of `base/` on every operation. Drift is warned about, never blocking.

## Exit codes

| Code | Meaning                                              |
| ---- | ---------------------------------------------------- |
| `0`  | Success                                              |
| `1`  | User error (bad args, missing config, etc.)          |
| `2`  | Network or IO error                                  |
| `3`  | Partial failure during `update-all` (some succeeded) |

## Limitations

- GitHub anonymous API rate limit (60 req/hour). Heavy use can hit it.
- No locking. Don't run two `wpbase` instances simultaneously on the same host.
- No automatic rollback. If `--build` fails after a base update, you fix forward.
- Linux only. Won't work on macOS without GNU coreutils.

## Related repositories

- [`containerized-wordpress`](https://github.com/simonecerruti/containerized-wordpress) — the actual Docker base this CLI manages
- Per-project repos — one per WordPress site, each pinned to a specific base version

See `CLAUDE.md` in this repo for design rationale and contribution guidelines.

## License

MIT