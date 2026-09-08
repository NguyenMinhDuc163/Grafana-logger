# Beszel Hub

Beszel is the infrastructure and container metrics UI. It is intentionally
separate from Grafana, which is used for logs only.

Open `http://OBSERVABILITY_HOST:BESZEL_PORT`, create the first administrator,
then use Beszel's **Add System** flow to obtain an agent install command for
each monitored machine (currently Proxmox and `dev-server`). Install agents
using the official Beszel instructions; this repository does not run the agent
in Docker or grant the Hub access to Docker sockets.

Publish the Hub through a reverse proxy or Cloudflare Tunnel if remote access is
needed. Keep the Hub port private otherwise.
