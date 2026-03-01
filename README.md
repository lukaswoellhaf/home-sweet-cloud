# home-sweet-cloud 🏠

Infrastructure as Code repository for creating and managing a personal Kubernetes cluster.

## Tech Stack

- **Hetzner Cloud** - VPS CPX22 (2 VCPU, 4 GB RAM, 80 GB STORAGE)
- **k3s** - Lightweight Kubernetes distribution
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
```

## Infrastructure

The infrastructure chart provides:

- Let's Encrypt SSL certificates via cert-manager
- Traefik ingress controller with automatic HTTPS redirect
- Headlamp dashboard for cluster management
- RBAC configuration with deployment service account

## Applications

[**ByteStash**](https://github.com/jordan-dalby/ByteStash) - Self-hosted code snippet manager with authentication and OIDC support. Accessible at [code.lukaswoellhaf.com](https://code.lukaswoellhaf.com)

[**Portfolio Website**](https://github.com/lukaswoellhaf/lukaswoellhafcom) - Personal portfolio website. Accesible at [lukaswoellhaf.com](https://lukaswoellhaf.com)

## DNS Configuration

Domains managed via Netcup.
- `lukaswoellhaf.com` (domain)
- `code.lukaswoellhaf.com` (subdomain)
- `grafana.lukaswoellhaf.com` (subdomain)

Required DNS records:

```
@         A       <server-ip>           # Root domain IPv4
@         AAAA    <server-ipv6>         # Root domain IPv6
code      A       <server-ip>           # Subdomain IPv4
code      AAAA    <server-ipv6>         # Subdomain IPv6
grafana   A       <server-ip>           # Subdomain IPv4
grafana   AAAA    <server-ipv6>         # Subdomain IPv6
www       CNAME   @                     # Redirect www to root domain
@         MX      10 <mx-server-1>      # Primary mail server
@         MX      20 <mx-server-2>      # Secondary mail server
@         TXT     <spf-record>          # SPF record for email authentication
```

## Monitoring

Metrics and logs are retained for **7 days**. Grafana is accessible at [grafana.lukaswoellhaf.com](https://grafana.lukaswoellhaf.com).

Grafana's native alerting sends `warning` and `critical` alerts to Discord via a provisioned contact point. The webhook URL is injected at deploy time via the `DISCORD_WEBHOOK_URL` GitHub secret. Custom alert rules cover:
- Node high CPU/memory usage (>90%) and low disk space (<15%)
- Pod crash looping or not ready for >10 minutes
- PersistentVolume filling up (<15% free)

## Cluster Provisioning

All cluster configurations and deployments are automated via GitHub Actions pipelines:

1. **Install k3s**: Run `install-k3s` workflow to bootstrap the cluster using k3sup
2. **Deploy Infrastructure**: Run `deploy-base-infra` workflow to install cert-manager and base infrastructure
3. **Deploy Applications**: Run `deploy-apps` workflow to deploy individual applications

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
# Create SSH tunnel to the server
ssh -L 8080:localhost:8080 <user>@<server-ip>

# In another terminal, port-forward to Headlamp service
kubectl port-forward -n default svc/infrastructure-headlamp 8080:80
```

Open http://localhost:8080 in your browser.
