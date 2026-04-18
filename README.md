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
| `aws-config-path` | `/root/.aws/config` | Where to write the AWS profile block. |
| `nix-conf-path` | `/etc/nix/nix.conf` | Where to append substituter + trusted-key lines. |

## Prerequisites

- **Nix must be installed** by a prior step. This action doesn't install it —
  use `DeterminateSystems/nix-installer-action` or `cachix/install-nix-action`.
- **`id-token: write`** permission on the job, so GitHub will issue the OIDC
  token used for the exchange.

## How it works

1. Downloads the `trustless-provisioner` binary pinned to the same ref the
   action is used with (`@v1`, `@v1.2.3`, or a commit SHA).
2. Runs `trustless-provisioner auth`, which:
   - Exchanges the GitHub OIDC token for an ephemeral SSH key tied to this
     job (1h lifetime by default) and writes it to `/root/.ssh/id_ed25519`.
   - Adds the `trustless.build` host key to `/root/.ssh/known_hosts`.
   - Writes an AWS `[profile trustless-<cache>]` block with a
     `credential_process = ssh trustless.build creds <cache>` line.
   - Appends `extra-substituters` and `extra-trusted-public-keys` entries
     for your cache to the Nix config.
3. Reloads `nix-daemon` so the new config is picked up.
4. Runs your `command`.

All config writes are idempotent (BEGIN/END markers) — re-running the action
replaces its blocks in place without duplicating or clobbering unrelated
config.

## Troubleshooting

- **"this repo has no cache assigned"** — your repo isn't mapped to a cache
  yet. Configure a cache access rule in your Trustless org.
- **"the assigned cache has no signing key yet"** — the cache exists but has
  never been built against. Kick off a first build (manually via `ssh
  trustless.build` or via a Workflow run that skips the substituter).
- **`nix build` doesn't use the substituter** — check `nix show-config` for
  `extra-substituters` + `trusted-public-keys` lines. If they're missing,
  the `trusted-users` setting might be too restrictive; Nix silently drops
  substituters from untrusted users.
