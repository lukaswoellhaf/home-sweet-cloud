# Headlamp Access

Use an SSH tunnel through Tailscale:

## Connect

```bash
# Clean up any stale connections
pkill -f "ssh.*-L 4466" 2>/dev/null
ssh <server-user>@<server-ip> 'pkill -f "port-forward.*4466" 2>/dev/null'

# Terminal 1 — SSH tunnel (keep running)
ssh -L 4466:localhost:4466 <server-user>@<server-ip>

# Terminal 2 — start port-forward on the server (sudo prompts for password)
ssh -t <server-user>@<server-ip> \
  'POD=$(sudo kubectl get pods -n default -l app.kubernetes.io/name=headlamp -o jsonpath="{.items[0].metadata.name}") && sudo kubectl port-forward -n default $POD 4466:4466'

# Terminal 3 — generate a short-lived auth token (sudo prompts for password)
ssh -t <server-user>@<server-ip> \
  'sudo kubectl create token dashboard-admin -n default --duration 4h' 2>/dev/null
```

Open http://localhost:4466, paste the token, and click **Authenticate**.

## Disconnect

```bash
# Kill local SSH tunnel
pkill -f "ssh.*-L 4466"

# Kill port-forward on the server
ssh <server-user>@<server-ip> 'pkill -f "port-forward.*4466"'
```
