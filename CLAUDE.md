# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

Unofficial Docker packaging of ProtonMail Bridge (IMAP/SMTP). There is no application code and no test suite; the repo is Dockerfiles, entrypoint scripts, and CI that tracks upstream releases.

## Commands

- Build the source image: `docker build --build-arg version=$(cat VERSION) ./build` (the `version` build-arg is required; it is the upstream git tag, e.g. `v3.27.1`)
- Build the deb image: `docker build ./deb`
- Compose (uses `./build`): `docker compose build`, `docker compose run protonmail-bridge init` (one-time interactive login), `docker compose up -d`
- Run the version check locally: `python3 update-check.py false`. It rewrites `VERSION` and `deb/PACKAGE` and **commits and pushes** via git. Pass `true` as the argument to skip the push. It also sets a local git user config.

## Architecture

Two parallel image variants, each with its own `Dockerfile`, `entrypoint.sh` and `gpgparams`:

- `build/` is the primary variant. It is a multi-stage build that clones `ProtonMail/proton-bridge` at `${version}` and runs `make build-nogui vault-editor` on `debian:sid-slim`, which is used because it is the only base with riscv64. It is multi-arch (amd64, arm64, arm/v7, riscv64). The entrypoint is `ENTRYPOINT`, so extra `docker run` args are passed through to `proton-bridge --cli`.
- `deb/` is the legacy variant, amd64 only. `install.sh` downloads the official .deb from the URL in `deb/PACKAGE` and rewrites its `Depends:` line to cut dependencies. The entrypoint is `CMD`.

Both entrypoints behave the same:
- `init`: generates a GPG key from `gpgparams`, runs `pass init pass-key` (the bridge keeps credentials in `pass`), then starts the bridge CLI for interactive login. The `build` version also `pkill`s a running bridge first, so `init` can be run inside a live container (k8s).
- Otherwise: starts two `socat` forwarders (25→1025, 143→1143), because the bridge binds only to 127.0.0.1, and feeds the bridge CLI a named pipe (`faketty`) so it does not exit on stdin EOF.
- State lives in `/root` (the `protonmail` volume).

Keep `build/` and `deb/` entrypoints in sync when changing shared behaviour. They are duplicated, not shared.

## CI / release flow (`.github/workflows/`)

- `update-check.yaml` runs daily. `update-check.py` queries the GitHub API for the latest upstream release and updates `VERSION` and `deb/PACKAGE`. It commits `Bump version to <tag>` and pushes with `PERSONAL_TOKEN`, which is what triggers the other workflows.
- `build.yaml` triggers on changes to `build/*` or `VERSION`. On non-master refs it builds once and runs a Trivy scan. On master it builds per platform by digest (a QEMU matrix), then a `merge` job creates multi-arch manifests tagged `build` and `<version>-build` on Docker Hub and GHCR.
- `deb.yaml` is the equivalent for the deb image (tags `latest` and `<version>`). `mirror.yaml` is also present.
- `docker-compose.yml` has an `x-arcane` metadata block (icon/urls) used by the Arcane UI; it is not a Compose feature.
