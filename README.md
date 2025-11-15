# tailscale-authenticate-action

Issue a short-lived Tailscale API token for a GitHub Actions workflow using its OIDC identity. Use it whenever a workflow needs to call the API without storing long‑lived API keys or OAuth client secrets.

## Prerequisites

1. Head to the [Trust credentials page](https://login.tailscale.com/admin/settings/trust-credentials) in the Tailscale admin console and create a new credential for GitHub Actions.
   - Select **OpenID Connect**, choose **GitHub** as the Issuer, and enter the Subject that is allowed to use this credential (for example `repo:octo-org/octo-repo:*`). See [GitHub's subject claim examples](https://docs.github.com/en/actions/reference/security/oidc#example-subject-claims) for more patterns.
2. Save the client ID and the audience as GitHub Action  variables or secrets (for example in `TS_OIDC_CLIENT_ID` and `TS_OIDC_AUDIENCE`).
3. Grant the job `id-token: write` permission so GitHub can mint an ID token at runtime.

## Usage

### Call Tailscale API

```yaml
name: Call Tailscale API

on:
  push:
    branches: [main]

jobs:
  fetch-tailnet-devices:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write  # required: enables GitHub's OIDC provider

    steps:
      - uses: actions/checkout@v5

      - name: Authenticate to Tailscale
        id: tailscale-auth
        uses: ciffelia/tailscale-authenticate-action@v1
        with:
          client-id: ${{ vars.TS_OIDC_CLIENT_ID }}
          audience: ${{ vars.TS_OIDC_AUDIENCE }}

      - name: List devices
        run: |
          curl "https://api.tailscale.com/api/v2/tailnet/-/devices" \
            -H "Authorization: Bearer $API_TOKEN"
        env:
          API_TOKEN: ${{ steps.tailscale-auth.outputs.access-token }}
```

The `access-token` output is a short-lived OAuth bearer token and should be used immediately in subsequent steps. Do not persist it.

### Sync tailnet policy file using [tailscale/gitops-acl-action](https://github.com/tailscale/gitops-acl-action)

```yaml
name: Sync Tailscale ACLs

on:
  push:
    branches: [main]

jobs:
  sync-acls:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write  # required: enables GitHub's OIDC provider

    steps:
      - uses: actions/checkout@v5

      - name: Authenticate to Tailscale
        id: tailscale-auth
        uses: ciffelia/tailscale-authenticate-action@v1
        with:
          client-id: ${{ vars.TS_OIDC_CLIENT_ID }}
          audience: ${{ vars.TS_OIDC_AUDIENCE }}

      - name: Deploy ACLs
        uses: tailscale/gitops-acl-action@v1
        with:
          tailnet: "-"  # default tailnet of the access token
          api-key: ${{ steps.tailscale-auth.outputs.access-token }}
          action: apply
```

## Inputs

Both client ID and audience are generated when creating the trust credential in the Tailscale admin console.

| Name | Required | Description |
| --- | --- | --- |
| `client-id` | ✅ | OAuth client ID of the trust credential. |
| `audience` | ✅ | Audience string of the trust credential. |

## Outputs

| Name | Description |
| --- | --- |
| `access-token` | Short-lived Tailscale API bearer token derived from the workflow's OIDC identity. |

## Further reading

- [Workload identity federation: better infra and CI/CD auth with Tailscale](https://tailscale.com/blog/workload-identity-beta)
- [Workload identity federation · Tailscale Docs](https://tailscale.com/kb/1581/workload-identity-federation)
