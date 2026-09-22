# github-vault-action

Composite GitHub Action that fetches secrets from a [`github-vault`](https://github.com/rsxdalv/github-vault)
instance using GitHub's OIDC token. **No long-lived credentials in CI.**

The action exposes every fetched secret as both a step output and an
environment variable (`${{ env.NAME }}`). That makes it the natural place to
**bootstrap CI secrets** for the rest of the workflow — see the "Bootstrapping
a workflow" example below.

## Inputs

| Name           | Required | Default        | Description                                                    |
|----------------|----------|----------------|----------------------------------------------------------------|
| `host`         | yes      |                | Base URL of the vault (e.g. `https://vault.example.com`)         |
| `names`        | yes      |                | Comma-separated list of secret names to fetch                  |
| `audience`     | no       | `github-vault` | JWT audience claim for the OIDC token                          |
| `fail-on-deny` | no       | `true`         | Fail the step if any name was denied / not found               |

## Outputs

| Name     | Description                                            |
|----------|--------------------------------------------------------|
| `json`   | Raw JSON response from the vault                       |
| `denied` | Newline-separated list of names that were denied        |
| `<NAME>` | Per-secret output for each requested name               |

## Required permissions

```yaml
permissions:
  id-token: write   # required to mint the OIDC token
```

## Basic usage

```yaml
- uses: rsxdalv/github-vault-action@v1
  with:
    host: ${{ secrets.VAULT_HOST }}
    names: DEPLOY_KEY,NPM_TOKEN
```

## Bootstrapping a workflow (Pattern A)

The canonical use case: **fetch CI secrets from the vault at the top of every
workflow** so downstream steps can reference them as regular env vars — no
per-repo secret storage needed.

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
    steps:
      - uses: rsxdalv/github-vault-action@v1
        with:
          host: ${{ secrets.VAULT_HOST }}
          names: POSITORY_URL,DEPLOY_WEBHOOK_URL

      # The values are now available as ${{ env.POSITORY_URL }} etc.
      # Downstream actions can read them straight from `with:`:
      - uses: rsxdalv/pository-deploy-action@main
        with:
          host: ${{ env.POSITORY_URL }}
          file: dist/*.deb

      - uses: rsxdalv/ansible-deploy-action@v1
        with:
          webhook-url: ${{ env.DEPLOY_WEBHOOK_URL }}
          service: my-app
```

The only secret you still need in the repo itself is `VAULT_HOST` — everything
else flows through the vault.

## How it works

1. Mints a GitHub OIDC token bound to this repo via the runner's
   `ACTIONS_ID_TOKEN_REQUEST_TOKEN` + `ACTIONS_ID_TOKEN_REQUEST_URL`.
2. `POST {host}/api/v1/secrets` with `{ "names": [...] }` and the token.
3. Parses the response, sets per-secret outputs + env vars, and exports them.
4. Surfaces denied names as `::warning` annotations and (by default) fails
   the step. Set `fail-on-deny: "false"` to tolerate partial fetches.