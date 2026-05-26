# Changelog

All notable changes to `wpbase` are documented here.

## Versioning policy

This repo follows [Semantic Versioning](https://semver.org/):
`MAJOR.MINOR.PATCH`. Each release is a git tag (`vX.Y.Z`) on `main`.

- **MAJOR** — breaking change to the CLI surface: a command or flag is
  renamed/removed, an exit code changes meaning, or the on-disk layout
  `wpbase` manages (`base/`, `.base-version`, `.checksum`) changes shape.
- **MINOR** — additive, backward-compatible: a new command or flag, a new
  optional config file, new output that doesn't break existing usage.
- **PATCH** — bug fix or internal cleanup with no surface change.

Unreleased work goes under `## [Unreleased]`; move it to a versioned
section when cutting the release.

## [Unreleased]

_Nothing yet._

## [1.0.0] - 2026-05-26

Initial release.

### Commands

- `list` — registered projects with their pinned version and drift status.
- `versions` — available base releases from GitHub.
- `install <ver> <path>` — bootstrap a project's `base/` and scaffold the
  runnable project-root templates (`docker-compose.yml`, `.env`,
  `.env.wordpress`) when absent.
- `update [<path>]` / `update-all` — roll a project (or all of them) to a base
  version. `update-all` is dry-run unless `--yes`/`--build`.
- `diff <path>` — checksum drift plus a `diff -rq` against the fetched tarball.
- `verify` — read-only, offline drift check across all projects (exit 3 on drift).
- `uninstall <path>` — remove the managed `base/` and untrack the project,
  keeping user-owned files.
- `self-update` — replace the installed script with the latest from its own repo.

### Base handling

- Tarball download (no git on host) with a `v`-prefix fallback, cached under
  `/var/cache/wpbase/<version>/`.
- Atomic `base/` swap (stage → swap → cleanup).
- Versioned snapshot copied into `base/`: `configs/`, `scripts/`,
  `overrides/.gitkeep`, `Dockerfile.base`, `compose.base.yml`, `VERSION`.
- `.checksum`-based drift detection (warn, never block); tolerant of older
  tarballs that lack `compose.base.yml`.
- Docker Compose ≥ v2.24 preflight on `--build` paths.

### Other

- Logging of every `info`/`warn`/`die` to `/var/log/wpbase.log`
  (override `WPBASE_LOG_FILE`).
- Config under `/etc/wpbase/`: `repo`, `projects.list`, optional `self-repo`.
- Flags: `--version`, `--yes`/`-y`, `--build`, `--dry-run`, `--force`.
- Exit codes: `0` OK, `1` user error, `2` network/IO, `3` drift/partial failure.
