# Tailscale CI Setup

## 1. Tailnet policy

Get the server's Tailscale IP:

```bash
tailscale ip -4
```

Set the GitHub Actions variable `SERVER_IP` to this IP or the server's MagicDNS name.

In [Access controls](https://console.tailscale.com/admin/acls), merge and save this fragment. Replace `<SERVER_TAILSCALE_IP>` with the IP above; preserve existing policy entries.

```json
{
  "tagOwners": { "tag:ci": ["autogroup:admin"] },
  "hosts": { "home-server": "<SERVER_TAILSCALE_IP>" },
  "grants": [
    {
      "src": ["tag:ci"],
      "dst": ["home-server"],
      "ip": ["tcp:22", "tcp:6443"]
    }
  ]
}
```

## 2. OAuth client

In [Trust credentials](https://console.tailscale.com/admin/settings/trust-credentials):

1. Select **Credential → OAuth**.
2. Expand **Keys** and enable **Auth keys → Write** only.
3. Allow **`tag:ci`**, then generate the credential. Copy the ID and secret; the secret is shown once.

OAuth is incompatible with Tailnet Lock.

## 3. GitHub secrets

Under **Settings → Secrets and variables → Actions**, add:

| Secret | Value |
| --- | --- |
| `TS_OAUTH_CLIENT_ID` | Client ID |
| `TS_OAUTH_SECRET` | Client secret |

Push the workflow changes, then start a **new** [Deploy Applications](../.github/workflows/deploy-apps.yml) run on that revision.

