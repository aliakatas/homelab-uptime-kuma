# 🏠 Homelab Uptime Kuma

Availability monitoring, status pages, and alerts for homelab services using [Uptime Kuma](https://github.com/louislam/uptime-kuma).

## 📌 Overview

This repository contains a Podman Compose deployment and a systemd user service for keeping monitoring reproducible. Use it to track internal and external services, review uptime trends, and receive outage notifications.

## 🧱 Stack

- **Uptime Kuma** — service monitoring and status tracking
- **Podman / Podman Compose** — container runtime and deployment management
- **Traefik** — reverse proxy and HTTPS termination, managed separately
- **Persistent storage** — `./data` mounted at `/app/data`

The included Compose file uses `docker.io/elestio/uptime-kuma:latest`. The registry hostname identifies the image source; no Docker daemon is required.

## 🚀 Getting Started

### 1) Clone this repository

Install Podman and `podman-compose` using your distribution's packages, then:

```bash
git clone https://github.com/aliakatas/homelab-uptime-kuma.git
cd homelab-uptime-kuma
```

### 2) Start Uptime Kuma

Create the external network if it does not already exist, then start the included deployment:

```bash
podman network exists proxy || podman network create proxy
mkdir -p data
podman-compose up -d
```

Open **http://127.0.0.1:13001** on the Podman host and create your administrator account. The published port is intentionally bound to loopback, not the LAN.

For a standalone Podman run instead of Compose, use the same storage, network, and port mapping:

```bash
podman run -d \
  --name uptime-kuma \
  --restart=unless-stopped \
  --network proxy \
  --cap-add=NET_RAW \
  -p 127.0.0.1:13001:3001 \
  -v "$PWD/data:/app/data" \
  docker.io/elestio/uptime-kuma:latest
```

Run only one of these alternatives at a time. `NET_RAW` supports ping monitors; omit it if you do not need them. On SELinux-enforcing hosts, add `:Z` to the data bind mount in whichever deployment you use, and ensure the directory is writable by the container.

### 3) Connect Traefik

The repository does not configure Traefik routes or certificates. In your separately managed Traefik configuration, add an HTTPS router for your chosen hostname and point its service at:

- **Containerized Traefik:** `http://uptime-kuma:3001`, with Traefik attached to the same Podman `proxy` network.
- **Traefik running directly on the Podman host:** `http://127.0.0.1:13001`.

Use Traefik's file provider for explicit routing without granting access to the Podman API socket. Configure TLS certificates and any access-control middleware in Traefik before exposing the dashboard. Containers must share the same Podman user/network context; a container's own loopback address cannot reach the host's loopback-bound port.

### 4) Configure monitors

Add monitors for your router, NAS, hypervisors, home automation, DNS, and public endpoints. Check that each target is reachable from inside the container, especially when using rootless Podman or monitoring LAN-only services.

## 🔔 Notifications

Configure and test at least one notification channel for critical alerts. Uptime Kuma supports options such as Discord, Telegram, email, and Slack. Keep notification credentials out of version control.

## 🗂️ Repository Structure

```text
.
├── README.md
├── compose.yml                     # Podman Compose deployment
├── uptime-kuma-container.service    # systemd user service template
└── data/                           # Runtime data, ignored by Git
```

For startup at boot, adapt the service's `WorkingDirectory` to your checkout, install it under `~/.config/systemd/user/`, and verify its `podman-compose` executable paths. Run `systemctl --user daemon-reload` and `systemctl --user enable --now uptime-kuma-container.service`. Enable user lingering if it must run without a login session. Use this service to manage the Compose deployment, not the standalone alternative.

## 🔒 Security Notes

- Prefer rootless Podman and grant only the capabilities your monitors need.
- Keep the application port loopback-only; use Traefik with HTTPS and appropriate access controls for remote access.
- Use a strong Uptime Kuma administrator password and restrict dashboard access to trusted users.
- Never commit tokens, passwords, private endpoints, or certificate keys. If you introduce an `.env` file, add it to `.gitignore` before storing secrets in it.
- Do not expose the Podman API socket to untrusted containers or the network.

## 🛠️ Maintenance

- Back up `./data` securely before upgrades. Stop Uptime Kuma while copying its database for a consistent backup, then restart it and periodically test restores.
- Review image release notes and compatibility before running `podman-compose pull` followed by `podman-compose up -d`. The current `latest` tag is mutable; pin a tested tag or digest for reproducible upgrades.
- Keep Podman, Podman Compose, and Traefik updated; check TLS certificate renewal.
- Review failed monitors and container logs (`podman logs uptime-kuma`), and test notification delivery regularly.

## 📝 Roadmap

- [ ] Document Traefik routing configuration
- [ ] Document monitor categories and notification setup
- [ ] Add a tested backup/restore procedure
- [ ] Add dashboard examples without sensitive details

## 📄 License

No license file is currently included.

---

Built for reliability in a self-hosted homelab 🧡
