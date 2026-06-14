# home-sweet-cloud: Master Plan

This document is the single source of context for the entire project. Hand it to any new session to continue work without losing context.

---

## Project Goal

Build a personal, self-hosted, production-grade Kubernetes cluster running on a physical ThinkPad E580 at home, exposed to the internet securely via Cloudflare Tunnel and Tailscale. All infrastructure is managed as code (Helm + GitHub Actions). The cluster hosts personal applications including a portfolio website, a code snippet manager, and a full monitoring stack.

---

## Infrastructure Overview

| Layer | Technology | Purpose |
|---|---|---|
| Physical host | ThinkPad E580 | Bare metal server, always-on |
| OS | Ubuntu Server 24.04 LTS | Headless, no GUI |
| Networking | LAN (enp3s0) via home router | Primary connectivity |
| Remote access | Tailscale | Secure private mesh network |
| Public exposure | Cloudflare Tunnel | No open inbound ports needed |
| Kubernetes | k3s | Lightweight single-node cluster |
| Package management | Helm | All workloads deployed as charts |
| Ingress | Traefik | With automatic HTTPS redirect |
| TLS | cert-manager + Let's Encrypt | Wildcard and per-service certs |
| DNS | Cloudflare | Manages lukaswoellhaf.com and subdomains |
| CI/CD | GitHub Actions | k3s install, infra deploy, app deploy |
| Secret scanning | Gitleaks (pre-commit) | Prevent credential leaks |
| Monitoring | Prometheus + Grafana + Loki | Metrics, dashboards, alerts |
| Alerting | Grafana → Discord webhook | Warning and critical alerts |

### Repository structure

```
infrastructure/         Base cluster setup (cert-manager, Traefik, RBAC, Headlamp)
applications/
  monitoring/           Prometheus, Grafana, Loki, Promtail
  bytestash/            Code snippet manager (code.lukaswoellhaf.com)
  portfolio-website/    Personal website (lukaswoellhaf.com)
  chat/                 (in progress)
```

### Key identifiers

| Item | Value |
|---|---|
| Server hostname | homesweetcloud-node1 |
| Admin user | lkswadmin |
| LAN interface | enp3s0 |
| Disk type | NVMe (/dev/nvme0n1) |
| LAN IP (DHCP) | 192.168.0.172 (Phase 1 only) |
| Tailscale IP | 100.96.66.77 (Phase 2 onwards; primary SSH access) |
| Primary domain | lukaswoellhaf.com |
| DNS provider | Cloudflare |

---

## Phases

### Phase 1: Base Server Setup — COMPLETE

**Goal:** Stable, secure, headless Ubuntu Server baseline before anything Kubernetes-related.

**Exit criteria — all met:**
1. ThinkPad auto-boots after power loss (BIOS AC restore configured).
2. SSH works with key-based auth only (password auth fully disabled).
3. Closing the lid does not suspend the system (logind configured).
4. UFW firewall active, only port 22 allowed inbound.
5. Unattended security updates enabled.
6. Reboot and reconnect tests passed.

**Key decisions made:**
- No full-disk encryption (required for unattended reboot).
- LAN-first routing over Wi-Fi (route-metric: 10 on enp3s0).
- Dedicated SSH key per server recommended over reusing existing key.

**Critical gotchas discovered (documented in thinkpad-phase1-server-setup.md):**
1. Ubuntu Server 24.04 only configures Wi-Fi in netplan by default. LAN interface `enp3s0` is `unmanaged` and needs a separate netplan file.
2. `dhclient` and `nmcli` are not installed. Use `networkctl` and `netplan` only.
3. `cloud-init` installs `/etc/ssh/sshd_config.d/50-cloud-init.conf` with `PasswordAuthentication yes` which silently overrides the main config. Must create `99-hardening.conf` drop-in with `AuthenticationMethods publickey` to fully lock it down.
4. `ChallengeResponseAuthentication` is removed in OpenSSH 9.x. Use `KbdInteractiveAuthentication no` instead.
5. Verify SSH effective config with `sudo sshd -T`, not by reading config files.
6. On Ubuntu with systemd socket activation, restart both `ssh.socket` and `ssh` after SSH config changes.

**Runbook:** `thinkpad-phase1-server-setup.md`

---

### Phase 2: Network — COMPLETE

**Goal:** Establish secure remote access via Tailscale and public exposure via Cloudflare Tunnel. No open inbound ports on the home router. All infrastructure is code-managed.

**Track A: Tailscale Setup**

1. Create Tailscale account at tailscale.com (free personal plan is sufficient)
2. Install Tailscale on Mac → sign in → Mac enrolled in tailnet
3. SSH to server via LAN IP one last time: `ssh lkswadmin@192.168.0.172`
4. On server: `curl -fsSL https://tailscale.com/install.sh | sh`
5. On server: `sudo tailscale up` → authenticate via the printed URL
6. Note Tailscale IP: `tailscale ip -4` (will be something like `100.x.y.z`)
7. Confirm UFW allows Tailscale interface: `sudo ufw allow in on tailscale0`
8. Open a new SSH session to the Tailscale IP from Mac — verify it works, then close LAN session

**Track B: GitHub Actions Integration**

9. In Tailscale admin console → Settings → Keys → Generate auth key (check: **ephemeral**, **reusable**, **pre-authorized**)
10. Add `TAILSCALE_AUTHKEY` as GitHub Actions repository secret
11. Update GitHub Actions **variable** `vars.SERVER_IP` to the Tailscale IP (100.x.y.z)
12. The `.github/workflows/` files have been pre-updated with `tailscale/github-action@v4` steps (completed)
13. Verify workflows work by manually triggering a test run

**Track C: Cloudflare Setup**

14. Create Cloudflare account at cloudflare.com
15. Add site `lukaswoellhaf.com` → Cloudflare auto-imports existing DNS records from Netcup
16. Note Cloudflare's 2 nameserver hostnames (e.g., `*.ns.cloudflare.com`)
17. Log in to Netcup → update nameservers to Cloudflare's NS values → propagation takes 1–24h
18. Wait for Cloudflare dashboard to show zone as **Active**
19. Confirm existing services on Hetzner still resolve correctly

**Track D: Cloudflare Tunnel Setup (bootstrap once + automated deployment)**

20. One-time manual bootstrap on server as `root`: install `cloudflared`, run `cloudflared tunnel login`, then `cloudflared tunnel create homesweetcloud`
21. Store bootstrap outputs as GitHub secrets: `CLOUDFLARED_TUNNEL_UUID` and `CLOUDFLARED_TUNNEL_CREDENTIALS_JSON`
22. Keep repo config at `infrastructure/cloudflare-tunnel/config.yml` as source of truth for ingress
23. Run `deploy-cloudflare-tunnel` GitHub Actions workflow to deploy service config and credentials on server
24. Workflow deploys rendered config to `/etc/cloudflared/config.yml` and enables/restarts `cloudflared`
25. Optional workflow input `testDomainRoute` configures a test DNS route (for example `test.lukaswoellhaf.com`) for validation
26. Verify service and tunnel health; remove optional test route when done

**Track E: Verification**

27. Confirm `ssh lkswadmin@<tailscale-ip>` works from outside LAN if possible
28. Confirm Cloudflare dashboard shows tunnel `homesweetcloud` as **Healthy**
29. Update MASTER-PLAN.md Phase status to "COMPLETE"

**Exit criteria:**
1. SSH reachable via Tailscale IP from outside LAN
2. At least one test service verified reachable via Cloudflare Tunnel over HTTPS
3. No ports open on home router
4. cloudflared running as systemd service (persistent)

**Runbook:** See steps above; `infrastructure/cloudflare-tunnel/config.yml` is the Track D config source

**Key decisions:**
- Auth key (not OAuth) for GitHub Actions Tailscale — simpler for personal home lab
- cloudflared configured locally (config file on server), deployed via GitHub Actions workflow
- Credentials JSON stays on server only; config.yml in repo (IaC)
- NS migration Netcup → Cloudflare happens now, but A records unchanged until Phase 5/6

---

### Phase 3: Kubernetes (k3s) — COMPLETE

**Goal:** Install k3s and validate a working single-node cluster.

**Planned steps:**
1. Install k3s using k3sup (matches existing GitHub Actions workflow).
2. Copy kubeconfig to local machine.
3. Validate cluster: `kubectl get nodes`, `kubectl get pods -A`.
4. Install Helm on server or use from local machine.
5. Run the `install-k3s` GitHub Actions workflow as the canonical install path.

**Exit criteria:**
1. Single-node cluster in `Ready` state.
2. System pods all running.
3. kubectl works from local Mac via Tailscale.

**Key config decisions:**
- k3s with Traefik as ingress (built-in).
- cert-manager for TLS.
- No external load balancer; Traefik handles ingress.

---

### Phase 4: Base Infrastructure (Helm) — IN PROGRESS

**Goal:** Deploy the `infrastructure/` Helm chart which sets up cert-manager, Traefik config, RBAC, and Headlamp dashboard.

**Planned steps:**
1. Re-run `deploy-cloudflare-tunnel` so the updated tunnel config is applied.
2. Run `deploy-base-infra` GitHub Actions workflow.
3. Verify cert-manager is issuing a Let's Encrypt certificate.
4. Verify HTTPS behavior for tunnel-routed hosts (Cloudflare edge enforces HTTP→HTTPS).
5. Verify Headlamp dashboard is accessible via SSH tunnel + `kubectl port-forward`.

**Required GitHub secrets:**
- `SSH_PRIVATE_KEY` — for k3sup and deployment access
- `PAT_SECRET_MANAGER` — for automatic secret management

**Exit criteria:**
1. cert-manager running and issuing certs.
2. Traefik ingress working with TLS.
3. HTTPS redirect active at Cloudflare edge for tunnel-routed hosts.
4. `DEPLOYMENT_SA_TOKEN` and `CLUSTER_CA_CERT` published for Phase 5 workflows.

**Routing decision for tunnel-routed hosts:**
- Cloudflare Tunnel remains the public entrypoint.
- Cloudflare edge enforces HTTP→HTTPS for public traffic.
- Do not attach Traefik HTTP→HTTPS redirect middleware to tunnel-routed ingresses to avoid redirect loops.

---

### Phase 5: Applications — TODO

**Goal:** Deploy all applications via the `deploy-apps` GitHub Actions workflow.

**Applications:**

| App | Chart path | URL | Status |
|---|---|---|---|
| Portfolio website | applications/portfolio-website | lukaswoellhaf.com | deployed on Hetzner, migrate to ThinkPad |
| ByteStash | applications/bytestash | code.lukaswoellhaf.com | deployed on Hetzner, migrate to ThinkPad |
| Monitoring (Prometheus + Grafana + Loki) | applications/monitoring | grafana.lukaswoellhaf.com | deployed on Hetzner, migrate to ThinkPad |
| Chat | applications/chat | TBD | in progress |

**Required GitHub secrets (app-specific):**
- `GRAFANA_ADMIN_USERNAME`
- `GRAFANA_ADMIN_PASSWORD`
- `DISCORD_WEBHOOK_URL`

**Exit criteria:**
1. All apps reachable at their domains over HTTPS.
2. Grafana alerts firing correctly to Discord.
3. Monitoring retaining 7 days of metrics and logs.

---

### Phase 6: Migration from Hetzner — TODO

**Goal:** Move all workloads from the existing Hetzner Cloud VPS (CPX22) to the ThinkPad cluster. Decommission Hetzner server.

**Planned steps:**
1. Confirm all apps running correctly on ThinkPad cluster.
2. Update DNS records at Netcup to point to Cloudflare Tunnel / new IPs.
3. Validate all domains resolve and HTTPS works.
4. Monitor for 48 hours.
5. Cancel Hetzner VPS.

**DNS records to update in Cloudflare:**
```
@         A       <new-ip>
@         AAAA    <new-ipv6>
code      A       <new-ip>
grafana   A       <new-ip>
```

**Exit criteria:**
1. All services available at their domains from the ThinkPad cluster.
2. No traffic going to Hetzner.
3. Hetzner VPS decommissioned.

---

## GitHub Actions Workflows

| Workflow | Trigger | Purpose |
|---|---|---|
| `install-k3s` | Manual | Bootstrap k3s cluster using k3sup |
| `deploy-base-infra` | Manual | Deploy infrastructure/ Helm chart |
| `deploy-cloudflare-tunnel` | Manual | Deploy cloudflare-tunnel config/credentials and manage cloudflared service |
| `deploy-apps` | Manual | Deploy individual application charts |
| QA pipeline | PR + push to main | Helm lint, Trivy security scan |

---

## Local Development Setup (Mac)

Pre-commit hook for secret scanning:

    pre-commit install
    pre-commit run --all-files

SSH access to server:

    ssh lkswadmin@192.168.0.172       # LAN (Phase 1)
    ssh lkswadmin@<tailscale-ip>       # After Phase 2

kubectl access (after Phase 3):

    export KUBECONFIG=~/.kube/homesweetcloud-node1
    kubectl get nodes

---

## Current State

| Phase | Status |
|---|---|
| Phase 1: Base Server Setup | Complete |
| Phase 2: Tailscale + Cloudflare Tunnel | Complete |
| Phase 3: k3s install | Complete |
| Phase 4: Base infrastructure Helm | In progress |
| Phase 5: Applications | Not started (running on Hetzner) |
| Phase 6: Hetzner migration | Not started |

**Phase 2 completed state:**
- ✅ `infrastructure/cloudflare-tunnel/config.yml` created (template with UUID placeholder)
- ✅ `.github/workflows/deploy-cloudflare-tunnel.yml` added for Track D automation
- ✅ `.github/workflows/install-k3s.yml` updated with Tailscale step
- ✅ `.github/workflows/deploy-base-infra.yml` updated with Tailscale step
- ✅ `.github/workflows/deploy-apps.yml` updated with Tailscale steps (all 4 jobs)
- ✅ GitHub secrets/vars configured for Tailscale and tunnel deployment
- ✅ One-time `cloudflared` bootstrap completed as `root`
- ✅ `deploy-cloudflare-tunnel` workflow executed successfully
- ✅ Test DNS route verified through Cloudflare Tunnel with expected HTTP 404 catch-all response

**Phase 4 implementation prepared state:**
- ✅ `.github/workflows/deploy-base-infra.yml` now installs `kubectl`, lints the infrastructure chart, and waits for the deployment service account secret to populate before publishing credentials
- ✅ `infrastructure/cloudflare-tunnel/config.yml` now includes the routing changes needed for Phase 4 validation
- ✅ Temporary Phase 4 validation resources were added to the infrastructure chart for rollout verification

**Next action:** Re-run `deploy-cloudflare-tunnel`, then run `deploy-base-infra` as the canonical Phase 4 rollout path.

