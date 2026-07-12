# `trustless-build` action

GitHub Action that wires a runner to a Trustless Nix binary cache and runs a
build command. No long-lived secrets: the action mints an ephemeral SSH key
by exchanging the job's OIDC token, writes the AWS `credential_process`
profile and Nix substituter config, then runs your command.

## Usage

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      id-token: write   # required — used to mint the exchange token
      contents: read
    steps:
      - uses: actions/checkout@v4
      - uses: DeterminateSystems/nix-installer-action@main
      - uses: trustless-build/build-action@v1
        # Default command is `nix flake check`. Override as needed:
        # with:
        #   command: nix build .#my-package
```

The repo running this action must be mapped to a cache in your Trustless org
(via direct assignment or a cache access rule). The cache must have a
signing key (i.e. have been built against at least once) — the action will
fail with a clear message otherwise.

## Inputs

| Name | Default | Description |
|------|---------|-------------|
| `command` | `nix flake check` | Command to run after cache setup. |
| `working-directory` | `.` | Working directory for `command`. |
| `api-endpoint` | `https://www.trustless.build/api` | Trustless API for the OIDC exchange. |
| `ssh-host` | `trustless.build` | Host used for `credential_process` and `known_hosts`. |
| `audience` | `trustless` | OIDC audience claim. |
| `aws-config-path` | `/root/.aws/config` | Where the `[profile trustless-…]` block is written. Read by the nix daemon (root) to mint substituter creds. |
| `nix-conf-path` | `/etc/nix/nix.conf` | Where substituter + trusted-public-keys lines are appended (system-wide). |
| `github-token` | `${{ github.token }}` | Token used to download the release asset. Only override when this action's repo is private and consumed from a different repo — see "Private action repo" below. |

## Private action repo

If you're consuming this action from a different repo than the one hosting it
— even within the same organization — the default `GITHUB_TOKEN` can't read
the action repo's releases, because it's scoped to the caller's repo only.
The symptom is `release not found` on the download step.

Fix: create a PAT (or fine-grained token, or GitHub App install token) with
`Contents:read` on the action's repo, add it as a secret to the consumer
repo, and pass it in:

```yaml
- uses: owner/build-action@v1
  with:
    github-token: ${{ secrets.TRUSTLESS_BUILD_PAT }}
```

You don't need this if the action's repo is public, or if it lives in the
same repo as the caller (in which case `uses: ./path/to/action` skips the
download entirely).

## Prerequisites

- **Nix must be installed** by a prior step. This action doesn't install it —
  use `DeterminateSystems/nix-installer-action` or `cachix/install-nix-action`.
- **`id-token: write`** permission on the job, so GitHub will issue the OIDC
  token used for the exchange.

## How it works

1. Downloads the `trustless` CLI pinned to the same ref the
   action is used with (`@v1`, `@v1.2.3`, or a commit SHA).
2. Runs `trustless setup`, which:
   - Exchanges the GitHub OIDC token for an ephemeral SSH key tied to this
     job (1h lifetime by default) and writes it to `/root/.ssh/id_ed25519`.
   - Adds the server-pinned build-host key to `/root/.ssh/known_hosts`.
   - Writes a keepalive `Host` block (`ServerAliveInterval 15`,
     `ServerAliveCountMax 3`) to `/root/.ssh/config`, so every SSH
     connection to the build host detects a dead server and fails within
     ~45s instead of hanging the job until the workflow timeout.
   - Writes an AWS `[profile trustless-<cache>]` block with a
     `credential_process = ssh <build-host> creds <cache>` line.
   - Appends `extra-substituters` and `extra-trusted-public-keys` entries
     for your cache to the Nix config.
   - Exercises the `credential_process` once, so a broken credential chain
     fails setup with the real ssh error instead of surfacing later as an
     S3 403.
3. Copies the SSH key, host-key line, and keepalive block to the runner
   user's `~/.ssh` (the build and version-push steps run unprivileged).
4. Reloads `nix-daemon` so the new config is picked up.
5. Runs your `command` via `trustless run`, which injects
   `store = ssh-ng://<build-host>` + `eval-store = auto` through NIX_CONFIG:
   evaluation happens on the runner, builds are realised remotely. A literal
   `nix flake check` command is special-cased — against a remote store it
   silently skips building drvs the store hasn't seen, so `run` enumerates
   the flake's checks and builds them explicitly.

All config writes are idempotent (BEGIN/END markers) — re-running the action
replaces its blocks in place without duplicating or clobbering unrelated
config.

## Troubleshooting

- **"this repo has no cache assigned"** — your repo isn't mapped to a cache
  yet. Configure a cache access rule in your Trustless org. Access rules are
  scoped per SCM connection, so the rule must be created under the connection
  matching your CI provider (e.g. your GitHub connection for GitHub Actions).
- **"the assigned cache has no signing key yet"** — the cache exists but has
  never been built against. Kick off a first build (manually via `ssh
  trustless.build` or via a Workflow run that skips the substituter).
- **`nix build` doesn't use the substituter** — check `nix show-config` for
  `extra-substituters` + `trusted-public-keys` lines. If they're missing,
  the `trusted-users` setting might be too restrictive; Nix silently drops
  substituters from untrusted users.
