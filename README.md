# Pro Fold Apps

A personal app ecosystem built around a Raspberry Pi 4B, Tailscale, and a Pixel 10 Pro Fold. The idea: self-hosted services running on the Pi, discoverable and usable from a foldable Android app that acts as a tiled window manager for your infrastructure.

## Architecture

```
Pixel 10 Pro Fold                    Tailscale Network
┌─────────────────┐                 ┌──────────────────────┐
│  My Fold Pods   │◄── encrypted ──►│  Raspberry Pi 4B     │
│  (Android app)  │    WireGuard    │  ├─ Pod API (:8090)  │
│                 │                 │  ├─ Pi-hole (:8080)   │
│  Tiled device   │                 │  └─ Podman containers │
│  hub + service  │                 └──────────────────────┘
│  browser        │                          ▲
└─────────────────┘                          │
                                    ┌────────┴───────────┐
                                    │  Agents (:8090)    │
                                    │  ├─ MacBook Pro    │
                                    │  ├─ bazzite        │
                                    │  └─ (any device)   │
                                    └────────────────────┘
```

**How it works:** The Pi runs a central API that tracks all Tailscale devices. Each device can install a lightweight agent (`curl -sL http://<pi>/install | bash`) that reports its local containers and health. The Android app polls the Pi API and renders a device grid — tap a device to see its services, tap a service to open its web UI.

## Workspace Structure

```
pro-fold-apps/
├── my-fold-pods/              # Pod dashboard app (own repo + CI)
├── templates/
│   └── pro-fold-apps-template/  # Jetpack Compose foldable starter
└── descriptions/              # Vision docs, session notes (local only)
```

Each app lives in its own subdirectory with its own git repo and CI pipeline. The template provides a starting point for new Pro Fold apps with Material 3, WindowSizeClass foldable support, and a GitHub Actions workflow for building signed APKs.

## Components

### My Fold Pods (v1.5)
The first app. A Tailscale device hub that shows all devices on the network, their services, and health status. Built with Kotlin + Jetpack Compose + Material 3.

- **Repo:** [Cmtovar/my-fold-pods](https://github.com/Cmtovar/my-fold-pods)
- **Package:** `com.profold.pods`
- **CI:** GitHub Actions builds a signed release APK on every push to main

### Pod API
A Python HTTP server running on the Pi (port 8090) that serves as the central hub.

| Endpoint | Description |
|----------|-------------|
| `GET /api/devices` | All Tailscale devices with probe status and services |
| `GET /api/services` | Local Podman containers |
| `GET /api/health` | CPU, memory, disk, uptime |
| `GET /api/config` | Service config (web paths, labels) |
| `POST /api/config` | Update config |
| `GET /install` | Universal agent install script |

The API probes online peers in background threads and caches results for 15 seconds, so the app gets instant responses.

### Agent System
A one-liner install that makes any Tailscale device part of the ecosystem:

```bash
curl -sL http://100.97.40.66:8090/install | bash
```

Auto-detects OS and container runtime. Sets up a persistent service (launchd on macOS, systemd on Linux) that reports local containers on port 8090. Install once, survives reboots.

### Pi-hole
DNS-level ad blocking for the entire Tailscale network. Runs as a Podman container on the Pi, configured as the global Tailscale DNS nameserver.

## Network

All traffic runs over Tailscale (WireGuard encrypted). No ports exposed to the public internet.

| Device | Tailscale IP | Role |
|--------|-------------|------|
| rpi4b | 100.97.40.66 | Central hub (Pi API + services) |
| MacBook Pro | 100.82.154.19 | Dev machine, agent installed |
| Pixel 10 Pro Fold | 100.92.0.108 | Runs the app |
| bazzite | 100.88.5.90 | Gaming desktop (agent not yet set up) |

## Roadmap

The long-term vision is a tiled window manager inside the app — a library of services where you can long-press to peek, drag to snap into split view, and manage containers directly.

| Version | Feature |
|---------|---------|
| v1.x | Foundation — device grid, service browser, agent system |
| v2.0 | Peek & preview (long-press to glance at a service) |
| v3.0 | Tiled window manager (drag-to-snap split view on unfolded screen) |
| v4.0 | Container management (start/stop/restart, logs, system health) |
| v5.0 | Cached state, animations, outage notifications |

## Development

### Prerequisites
- Android Studio (for app development)
- A Tailscale account with devices on the same tailnet
- SSH access to the Pi: `ssh cmtovar5@100.97.40.66`

### Creating a new app
Copy `templates/pro-fold-apps-template/` to a new subdirectory, rename the package, and start building. The template includes Compose scaffolding, foldable support, and a CI workflow.

### Releasing
Push to `main` in an app's repo. GitHub Actions builds a signed APK and creates a release. Download the APK on the phone from GitHub Releases.
