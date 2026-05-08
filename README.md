# Tailscale GitOps Action

A GitHub Action to manage your [Tailscale ACL policy](https://tailscale.com/kb/1018/acls) using a GitOps workflow. This is a drop-in replacement for [`tailscale/gitops-acl-action`](https://github.com/tailscale/gitops-acl-action) that caches a pre-compiled `gitops-pusher` binary instead of compiling it from source on every run, reducing execution time from 60–90 seconds to **2–3 seconds**.

## Usage

```yaml
name: Sync Tailscale ACLs

on:
  push:
    branches: ["main"]
  pull_request:
    branches: ["main"]

jobs:
  acls:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write

    steps:
      - uses: actions/checkout@v4

      - name: Deploy ACL
        if: github.event_name == 'push'
        uses: matchory/tailscale-gitops-action@v1
        with:
          oauth-client-id: ${{ secrets.TS_OAUTH_ID }}
          audience: ${{ secrets.TS_AUDIENCE }}
          tailnet: ${{ secrets.TS_TAILNET }}
          policy-file: ./policy.hujson
          action: apply

      - name: Test ACL
        if: github.event_name == 'pull_request'
        uses: matchory/tailscale-gitops-action@v1
        with:
          oauth-client-id: ${{ secrets.TS_OAUTH_ID }}
          audience: ${{ secrets.TS_AUDIENCE }}
          tailnet: ${{ secrets.TS_TAILNET }}
          policy-file: ./policy.hujson
          action: test
```

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `tailnet` | Yes | — | Your tailnet name (e.g. `example.com`, `your-org.github`) |
| `api-key` | No | — | Tailscale API key (expires every 90 days) |
| `oauth-client-id` | No | — | OAuth or OIDC Federated Identity Client ID |
| `oauth-secret` | No | — | OAuth Client Secret |
| `audience` | No | — | OIDC Federated Identity Audience |
| `policy-file` | Yes | `./policy.hujson` | Path to your policy file in the repository |
| `action` | Yes | — | `test` (validate only) or `apply` (validate and push) |
| `tailscale-release` | No | `b4d39e...` | Tailscale commit hash to build `gitops-pusher` from |

### Authentication

Exactly **one** of the following authentication methods must be provided:

1. **API Key** — set `api-key` (note: keys expire after 90 days)
2. **OAuth Client** — set `oauth-client-id` and `oauth-secret`
3. **OIDC Federated Identity** — set `oauth-client-id` and `audience` (requires `id-token: write` permission)

## How It Works

The action compiles Tailscale's [`gitops-pusher`](https://pkg.go.dev/tailscale.com/cmd/gitops-pusher) as a static binary on the first run and caches it using [`actions/cache`](https://github.com/actions/cache). The cache key includes the runner OS, architecture, and the pinned Tailscale commit hash, so subsequent runs skip compilation entirely and restore the binary from cache.

| Run | What happens | Time |
|-----|-------------|------|
| First (cold cache) | Installs Go, compiles binary, caches it | ~25s |
| Subsequent (warm cache) | Restores binary from cache, executes it | ~2–3s |

When using OIDC federated identity, the action fetches an ID token from GitHub's OIDC provider before invoking the binary.

### Updating the Tailscale version

The `tailscale-release` input controls which commit of `gitops-pusher` is compiled. Changing this value (either in this action's default or as an input in your workflow) automatically invalidates the cache and triggers a one-time recompilation.

## License

MIT
