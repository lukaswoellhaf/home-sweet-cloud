# home-sweet-cloud 🏠

Infrastructure as Code repository for creating and managing a personal Kubernetes cluster.

## Tech Stack

- **[ThinkPad E580](https://psref.lenovo.com/Product/ThinkPad/ThinkPad_E580)** — DIY home server (Intel Core i5-8250U, 8 GB DDR4, 256 GB NVMe SSD) running Ubuntu Server 24.04 LTS
- **k3s** — Lightweight single-node Kubernetes distribution
- **Helm** - Kubernetes package manager
- **cert-manager** - Automated SSL/TLS certificate management
- **Traefik** - Ingress controller
- **Prometheus** - Metrics collection
- **Grafana** - Metrics visualization and alerting
- **Loki & Promtail** - Log aggregation

## Project Structure

```
infrastructure/     Base cluster setup (cert-manager, Traefik, RBAC, Headlamp)
applications/       Application deployments
  monitoring/       Prometheus, Grafana, Loki
  bytestash/        Code snippet manager
  portfolio-website/ Personal website
  chat/             Open WebUI
```

## Infrastructure

The infrastructure chart provides:

- Let's Encrypt SSL certificates via cert-manager
- Traefik ingress controller (HTTPS enforced at Cloudflare edge)
- Headlamp dashboard for cluster management
- RBAC configuration with deployment service account

## Applications

[**ByteStash**](https://github.com/jordan-dalby/ByteStash) - Self-hosted code snippet manager with authentication and OIDC support. Accessible at [code.lukaswoellhaf.com](https://code.lukaswoellhaf.com)

[**Portfolio Website**](https://github.com/lukaswoellhaf/lukaswoellhafcom) - Personal portfolio website. Accesible at [lukaswoellhaf.com](https://lukaswoellhaf.com)

[**Chat**](https://github.com/lukaswoellhaf/home-sweet-cloud) — Open WebUI — accessible at [chat.lukaswoellhaf.com](https://chat.lukaswoellhaf.com)

## DNS

**Registrar:** Netcup &nbsp;|&nbsp; **DNS management:** Cloudflare &nbsp;|&nbsp; **Public exposure:** Cloudflare Tunnel

| Record | Type | Value |
|---|---|---|
| `lukaswoellhaf.com` | CNAME | `d1fd43b6-….cfargotunnel.com` (Tunnel) |
| `code.lukaswoellhaf.com` | CNAME | `d1fd43b6-….cfargotunnel.com` (Tunnel) |
| `grafana.lukaswoellhaf.com` | CNAME | `d1fd43b6-….cfargotunnel.com` (Tunnel) |
| `chat.lukaswoellhaf.com` | CNAME | `d1fd43b6-….cfargotunnel.com` (Tunnel) |
| `www.lukaswoellhaf.com` | CNAME | `lukaswoellhaf.com` |
| `lukaswoellhaf.com` | MX | `mx1.improvmx.com` (10), `mx2.improvmx.com` (20) |
| `lukaswoellhaf.com` | TXT | SPF (`v=spf1 include:spf.improvmx.com ~all`), Google Site Verification |

Cloudflare edge enforces HTTPS. TLS certificates are provisioned by **cert-manager** via Let's Encrypt on the cluster.

## Monitoring

Metrics and logs are retained for **7 days**. Grafana is accessible at [grafana.lukaswoellhaf.com](https://grafana.lukaswoellhaf.com).

Grafana's native alerting sends `warning` and `critical` alerts to Discord via a provisioned contact point. The webhook URL is injected at deploy time via the `DISCORD_WEBHOOK_URL` GitHub secret. Custom alert rules cover:
- Node high CPU/memory usage (>90%) and low disk space (<15%)
- Pod crash looping or not ready for >10 minutes
- PersistentVolume filling up (<15% free)

## Cluster Provisioning

All cluster configurations and deployments are automated via GitHub Actions pipelines:

0. **Set up the server** — Follow [server-setup.md`](docs/server-setup.md) to configure the ThinkPad as a headless Ubuntu home server (BIOS, SSH hardening, firewall, lid-close behavior).
1. **Install k3s**: Run `install-k3s` workflow to bootstrap the cluster using k3sup
2. **Deploy Cloudflare Tunnel Config**: Re-run `deploy-cloudflare-tunnel` whenever `infrastructure/cloudflare-tunnel/config.yml` changes
3. **Deploy Infrastructure**: Run `deploy-base-infra` workflow to install cert-manager and base infrastructure
4. **Deploy Applications**: Run `deploy-apps` workflow to deploy individual applications

Required GitHub secrets: `SSH_PRIVATE_KEY`, `PAT_SECRET_MANAGER` (for automatic secret management), `GRAFANA_ADMIN_USERNAME`, `GRAFANA_ADMIN_PASSWORD`, `DISCORD_WEBHOOK_URL`, and application-specific secrets.

## Local Development

The QA pipeline runs automatically on pull requests and pushes to main:

- Helm chart linting and validation
- Security scanning with Trivy

Pre-commit hook scans for leaked secrets using Gitleaks:

```bash
# Install the git hook scripts
pre-commit install

# Test the hook
pre-commit run --all-files
```

## Headlamp Access

Access the Headlamp dashboard via SSH tunnel and port-forward:

```bash
# SSH to server via Tailscale
ssh lkswadmin@100.96.66.77 -L 8080:localhost:8080

# In another terminal, port-forward to Headlamp service
kubectl port-forward -n default svc/infrastructure-headlamp 8080:80
```

Open http://localhost:8080 in your browser.
