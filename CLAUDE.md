# CLAUDE.md - wpbase

This file gives Claude Code the context to work on the `wpbase` host-side CLI.

## Repository purpose

`wpbase` is a single pure-bash script installed on each Hetzner VPS at `/usr/local/bin/wpbase`. It manages the lifecycle of one or more WordPress site projects on a host by downloading versioned base releases from GitHub and applying them to project directories.

This script is the third leg of a three-part system. The other two are:

1. `containerized-wordpress` -- the reusable Docker base (separate repo). Its `CLAUDE.md` is the authoritative reference for the overall architecture.
2. Per-project repos -- one per WordPress site, each containing a pinned snapshot of the base.
3. **This repo** -- the CLI that bridges the two.

If something here contradicts `containerized-wordpress/CLAUDE.md`, the containerized-wordpress one wins.

## Why a separate repo

- `containerized-wordpress` is consumed via GitHub release tarballs from the host. The host never clones it.
- `wpbase` itself, instead, lives somewhere the user must explicitly install from (curl + chmod, or a release). Keeping it separate avoids version coupling: the CLI evolves on a different cadence than the base.
- Per-project repos are private and per-site. They have nothing to do with `wpbase` source.

## Repository layout

```
wpbase-cli/
  wpbase                       -- the script (single file, no submodules)
  CLAUDE.md                    -- this file
  README.md                    -- user-facing install + usage
  CHANGELOG.md
  .github/workflows/release.yml  -- optional: release on tag
```

The intent is: this repo stays tiny. Just the script + docs + release plumbing. No tests-as-files (smoke tests live as a bats suite or in CI, see Roadmap).

## What `wpbase` does

Given a host with several WordPress projects, each pinned to some version of `containerized-wordpress`, `wpbase` lets you:

- List projects and their current base version + drift status
- Bootstrap a new project's `base/` directory
- Update a project (or all of them) to a newer base version
- See what version of the base is currently installed
- Diff a project's `base/` against the pinned version to detect manual tampering

## Configuration

`wpbase` reads from `/etc/wpbase/`:

- `repo` -- single line, GitHub URL of the base repo. Example: `https://github.com/simonecerruti/containerized-wordpress`
- `projects.list` -- one absolute path per line. Lines starting with `#` are comments. Empty lines ignored.

Cache: `/var/cache/wpbase/<version>/` -- extracted tarballs are kept here for reuse.

Override via env: `WPBASE_CONF_DIR`, `WPBASE_CACHE_DIR`.

## Commands (summary)

```
wpbase list                       -- show projects and pinned versions
wpbase versions                   -- list available releases from GitHub
wpbase install <ver> <path>       -- bootstrap base/ in a new project
wpbase update [<path>]            -- update one project (or use --version X)
wpbase update-all                 -- update every registered project
wpbase diff <path>                -- detect manual edits to base/
```

Flags: `--version X.Y.Z`, `--yes`, `--build`, `--dry-run`, `--force`.

Special default: `update-all` is `--dry-run` unless explicitly overridden with `--yes` or `--build`. This protects against fat-fingering a mass update.

## Key design decisions and constraints

### 1. Pure bash + curl + tar. No other dependencies.

This is non-negotiable. The script must run on a fresh Debian VPS with nothing extra installed. No Python, no jq, no yq, no git on host. If a feature requires more, propose adding it explicitly first.

Anonymous GitHub API calls (60 req/h) are used for resolving "latest". Token support (`/etc/wpbase/github-token` -> `Authorization: token ...` header) is on the roadmap; the parsing is simple regex on the JSON response.

### 2. Tarball download, not git clone.

The host never has git. `wpbase` downloads `https://github.com/<slug>/archive/refs/tags/v<ver>.tar.gz` (with a fallback to the tag without `v` prefix). Extracted with `tar --strip-components=1`. This is the auto-generated artifact GitHub creates for every tag/release.

### 3. Atomic base/ swap, not in-place edit.

When installing or updating a project's `base/`, `wpbase`:

1. Stages files into `<project>/.base-staging-XXXXXX/`
2. Moves the old `<project>/base/` to `<project>/base.old/`
3. Moves staging to `<project>/base/`
4. Removes `<project>/base.old/`

This minimizes the time window in which `base/` is missing or partial. Containers building during this window may fail (acceptable: it's seconds, not minutes), but the on-disk state is never half-applied.

### 4. Warn on drift, do not block.

On every update, `wpbase` checks `<project>/base/.checksum` against a fresh hash of the directory. If they differ, it warns and proceeds. Rationale: drift detection helps catch accidents, but we trust the operator to know what they're doing. Blocking would make emergencies harder to fix.

`wpbase diff <path>` provides a richer drift view using `diff -rq` against the freshly fetched tarball.

### 5. Subset extraction.

When installing, only these paths are copied from the release tarball into `<project>/base/`:

- `configs/`
- `scripts/`
- `overrides/` (the base's empty placeholder `.gitkeep`)
- `Dockerfile.base`
- `VERSION`

Everything else (release notes, README, CLAUDE.md, .github/) is ignored. This keeps the project repo clean.

### 6. Version pin file.

`<project>/.base-version` is a single-line file containing the version string (no `v` prefix). It's the source of truth for "what version is this project on". Updated atomically after a successful install.

### 7. update-all defaults to dry-run.

Unless `--yes` or `--build` is passed, `update-all` shows what it would do without touching anything. This is a strong safety property: running `wpbase update-all` by accident never changes anything.

### 8. confirm() per project for interactive mode.

When `--yes` is not passed, the script prompts per project. This is deliberate: you might want to update most projects but skip one that's in maintenance.

## Coding conventions

- `set -uo pipefail` at top. NOT `-e` -- the script uses explicit exit code handling and per-iteration error counting in `update-all`. Don't add `-e` without rewriting those loops.
- `die "msg" [exit_code]`, `info "msg"`, `warn "msg"` helpers. Always use them, never `echo` ad-hoc.
- Exit codes: `0` OK, `1` user error (bad args, missing config), `2` network/IO error, `3` partial failure on `update-all` (some projects failed, some succeeded).
- Function names: `snake_case`. Variable names: `UPPER_SNAKE` for globals/env, `snake_case` for locals. Use `local` inside functions.
- English everywhere. Italian only in error messages that an Italian admin would read directly. There aren't any currently in `wpbase` -- this is sysadmin-facing tooling.

## Testing approach

Currently there are no automated tests. For now, manual smoke tests on a throwaway VPS:

```bash
# Setup
sudo install -m 755 wpbase /usr/local/bin/wpbase
sudo mkdir -p /etc/wpbase
echo "https://github.com/simonecerruti/containerized-wordpress" | sudo tee /etc/wpbase/repo
sudo touch /etc/wpbase/projects.list

# Smoke tests
wpbase --help                                  # arg parser sanity
wpbase versions                                # network + JSON parsing
mkdir -p /tmp/test-project
wpbase install 1.0.0 /tmp/test-project         # bootstrap
ls /tmp/test-project/base                      # configs/, scripts/, etc.
cat /tmp/test-project/.base-version            # "1.0.0"
echo /tmp/test-project | sudo tee -a /etc/wpbase/projects.list
wpbase list                                    # should show test-project as clean
echo "BREAK" > /tmp/test-project/base/VERSION
wpbase list                                    # should show "modified"
wpbase diff /tmp/test-project                  # should show the diff
wpbase update /tmp/test-project                # should warn about drift, then update
wpbase update-all                              # should be dry-run
wpbase update-all --yes                        # should actually do it
```

A proper `bats` suite is on the roadmap. Anyone touching `wpbase` should at minimum run these manually.

## Common modifications and where to make them

| Want to change                          | Where                                             |
| --------------------------------------- | ------------------------------------------------- |
| A new command                           | Add `cmd_<name>()`, register in the case at end   |
| A new flag                              | Add to the `while [[ $# -gt 0 ]]` parser          |
| Tarball URL format                      | `fetch_tarball()`                                 |
| What gets extracted into base/          | `install_base_into_project()` -- the for/cp block |
| What counts for drift detection         | `project_base_checksum()`                         |
| Adding GitHub token support             | `fetch_latest_version()`, `fetch_all_versions()`  |
| Changing confirmation UX                | `confirm()` and call sites                        |
| Adding hooks (pre/post install/update)  | `install_base_into_project()` and `cmd_update()`  |

## Known limitations / gotchas

1. **GitHub anonymous rate limit (60 req/h).** Heavy use of `wpbase versions` or many `update-all` runs in short succession can hit it. Symptom: `versions` returns empty / install fails. Fix: wait an hour, or add token support (see Roadmap).

2. **Tarball URL ambiguity.** GitHub serves both `archive/refs/tags/v1.0.0.tar.gz` AND `archive/refs/tags/1.0.0.tar.gz`. `fetch_tarball()` tries with `v` prefix first, falls back to without. If the repo uses an unusual tagging convention, this might fail.

3. **Race conditions on concurrent `update-all`.** No locking. Don't run two instances simultaneously on the same host. (Easy fix if needed: `flock /var/lock/wpbase.lock`.)

4. **`docker compose build` requires the project to be a valid compose project.** If the user has a broken `docker-compose.yml`, `--build` will fail. The script reports failure per-project but does not roll back the base update. Rationale: a half-rolled update is still a valid state to debug from.

5. **No `wpbase uninstall`.** Removing a project from `projects.list` is manual. There's no command to wipe `base/` from a project. This is intentional -- destructive ops should be explicit.

6. **`.checksum` includes everything except itself.** The `find ... ! -name '.checksum'` filter excludes the file from its own hash. Don't add other auto-generated files to `base/` without similar exclusion, or every operation will look like drift.

7. **`base.old/` cleanup is best-effort.** If `wpbase` crashes between the rename and the rm, `base.old/` will linger. Detecting and cleaning this on next run is on the Roadmap.

## Release workflow

Match the containerized-wordpress convention. Versions are SemVer-ish (`MAJOR.MINOR.PATCH`):

- MAJOR: breaking change to CLI surface (renamed/removed commands or flags, changed exit codes)
- MINOR: new commands/flags, backward compatible
- PATCH: bug fixes

Cut a release:

```bash
git tag v1.1.0
git push origin v1.1.0
```

If `.github/workflows/release.yml` is configured, that creates a GitHub Release with auto-generated notes. Users install with:

```bash
curl -fsSL https://raw.githubusercontent.com/<owner>/<repo>/v1.1.0/wpbase \
  | sudo tee /usr/local/bin/wpbase > /dev/null
sudo chmod +x /usr/local/bin/wpbase
```

(Or pinning to `main` for the brave.)

## Roadmap

- [ ] `bats` test suite covering all commands against a local mock GitHub
- [ ] `flock` to prevent concurrent runs
- [ ] GitHub token support via `/etc/wpbase/github-token`
- [ ] `wpbase uninstall <path>` -- removes base/, untracks from projects.list
- [ ] `wpbase verify` -- verify all projects match their pinned version (read-only mass diff)
- [ ] Resume-on-crash: detect lingering `base.old/` and offer to roll back
- [ ] Shell completion (bash + zsh)
- [ ] Logging to `/var/log/wpbase.log` (currently only stdout/stderr)
- [ ] Self-update command: `wpbase self-update` -- download latest wpbase from its own repo

## How to verify this CLAUDE.md is still accurate

If wpbase is significantly modified, re-read this file and update:

1. The "Commands" section if commands/flags changed
2. The "Common modifications" table if internal structure changed
3. The "Known limitations" if a limitation was fixed
4. The "Roadmap" if a roadmap item shipped

This file is the contract between the codebase and its future contributors. Keep it honest.