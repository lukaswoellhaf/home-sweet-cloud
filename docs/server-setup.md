# Server Setup: ThinkPad E580 → Home Server

How to turn a ThinkPad E580 into a Ubuntu home server.

---

## 1. BIOS

- Set **AC Power Recovery** → **Always On**.
- Keep **Intel VT-x** enabled.

---

## 2. Install Ubuntu Server 24.04 LTS

- Minimal profile, OpenSSH server enabled.
- After install: `sudo apt update && sudo apt full-upgrade -y && sudo reboot`

---

## 3. LAN Interface

> Run `ip -br link` first to confirm your Ethernet interface name.

Create a netplan file for the Ethernet port:

```bash
sudo tee /etc/netplan/10-lan-enp3s0.yaml <<'EOF'
network:
  version: 2
  renderer: networkd
  ethernets:
    enp3s0:
      dhcp4: true
      dhcp4-overrides:
        route-metric: 10
      optional: true
EOF
sudo chmod 600 /etc/netplan/10-lan-enp3s0.yaml
sudo netplan generate && sudo netplan apply
```

Verify: `ip -4 addr show enp3s0` should show a LAN IP.

---

## 4. SSH Hardening

### 4.1 Copy your public key
```bash
ssh-copy-id <server-user>@<server-ip>
```

### 4.2 Harden the main config

Ensure `/etc/ssh/sshd_config` contains:
```
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
KbdInteractiveAuthentication no
UsePAM yes
```

### 4.3 Override cloud-init
Ubuntu's `50-cloud-init.conf` silently re-enables passwords. Create a drop-in that takes precedence:

```bash
sudo tee /etc/ssh/sshd_config.d/99-hardening.conf >/dev/null <<'EOF'
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
PermitRootLogin no
AuthenticationMethods publickey
EOF
sudo chmod 600 /etc/ssh/sshd_config.d/99-hardening.conf
```

### 4.4 Apply
```bash
sudo sshd -t
sudo systemctl restart ssh.socket && sudo systemctl restart ssh
sudo sshd -T | grep -iE "passwordauth|kbdinteractive|pubkeyauth|permitroot|authmethods"
```

Test: a new SSH session with `-o PreferredAuthentications=password` must fail.

---

## 5. Firewall

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow OpenSSH
sudo ufw enable
```

---

## 6. Lid-Closed Operation

```bash
sudo sed -i 's/^#HandleLidSwitch=.*/HandleLidSwitch=ignore/' /etc/systemd/logind.conf
sudo sed -i 's/^#HandleLidSwitchExternalPower=.*/HandleLidSwitchExternalPower=ignore/' /etc/systemd/logind.conf
sudo systemctl restart systemd-logind
```

Close the lid for 2 minutes, then SSH in — host must still be up.

---

## 7. Automatic Security Updates

```bash
sudo apt install -y unattended-upgrades
sudo dpkg-reconfigure --priority=low unattended-upgrades
sudo systemctl enable --now unattended-upgrades
```

---

## 8. Final Validation

```bash
sudo reboot
```

After reboot, verify:
- SSH key login works
- `sudo ufw status` shows active
- Lid-close behavior holds
- `unattended-upgrades` service is running
