# Homelab Infrastructure Automation

Ansible-based homelab infrastructure automation system using Docker Swarm for container orchestration and Traefik for reverse proxy with dynamic routing.

## Table of Contents

- [Features](#features)
- [Quick Start](#quick-start)
  - [Prerequisites](#prerequisites)
  - [Configuration](#configuration)
  - [Initial Setup](#initial-setup)
- [Service Management](#service-management)
  - [Adding Services](#adding-services)
  - [Service Variables](#service-variables)
  - [Updating Services](#updating-services)
- [Directory Structure](#directory-structure)
- [Architecture](#architecture)

## Features

- **Docker Swarm Orchestration**: Automatic swarm initialization and overlay networking
- **Traefik Reverse Proxy**: Dynamic file-based routing with automatic service discovery
- **Template-driven Deployment**: Consistent service deployment using Jinja2 templates
- **Hot Reload**: Route updates without service restarts
- **Automated Maintenance**: Log rotation and system cleanup via cron jobs

## How It Works

This system is designed to automate the deployment of services to a Docker Swarm cluster. Here’s a visual overview of the workflow when you add a new service:

```
                                     ┌─────────────────────────────┐
                                     │      Your Local Machine     │
                                     └─────────────────────────────┘
                                                  │
                                                  │ 1. Run Ansible Playbook
                                                  │
                                                  ▼
     ┌───────────────────┐        ┌───────────────────────────────────┐
     │  vars/hello.yaml  │        │       ansible-playbook -i ...       │
     └───────────────────┘        └───────────────────────────────────┘
              │                                      │ ssh
              └──────────────────────────────────────┘
                                                     │
                                                     ▼
┌───────────────────────────────────────────────────────────────────────────────────────────┐
│                                       Homelab Server                                        │
│                                                                                           │
│   ┌──────────────────────────┐     ┌────────────────────────┐      ┌─────────────────────┐   │
│   │  Ansible `service` Role  │ ────▶   Templates (Jinja2)   │      │    Docker Swarm     │   │
│   └──────────────────────────┘     └────────────────────────┘      └─────────────────────┘   │
│                │                                                         ▲                │
│                │ 2. Generate Configs                                     │ 4. Deploy      │
│                ▼                                                         │                │
│   ┌───────────────────────────┐      ┌──────────────────────────┐          │                │
│   │ services/hello.yaml       │      │ dynamic/routers/         │          │                │
│   │ (Docker Service Def)      │      │ hello.yaml               │          │                │
│   └───────────────────────────┘      │ (Traefik Route)          │          │                │
│                │                     └──────────────────────────┘          │                │
│                └───────────────────────────────────────────────────────────┘                │
│                                              │                                            │
│                                              │ 3. Hot Reload                              │
│                                              ▼                                            │
│                                     ┌──────────────────┐                                  │
│                                     │     Traefik      │                                  │
│                                     └──────────────────┘                                  │
│                                              │                                            │
└──────────────────────────────────────────────┼────────────────────────────────────────────┘
                                               │
                                               │ 5. User Access (e.g., yourdomain.com/hello)
                                               │
                                               ▼
                                     ┌──────────────────┐
                                     │    End User      │
                                     └──────────────────┘
```

**Workflow Steps:**

1.  **Run Playbook**: You execute the `ansible-playbook` command from your local machine, targeting your homelab server. You pass a service definition file (e.g., `vars/hello.yaml`) containing all the necessary variables for your new service.
2.  **Generate Configs**: Ansible connects to the server via SSH. The `service` role uses Jinja2 templates to generate two critical files based on your variables:
    *   A Docker Compose file in `/opt/homelab/services/` that defines your application as a Docker Swarm service.
    *   A Traefik routing configuration file in `/opt/homelab/dynamic/routers/` that tells Traefik how to route traffic to your new service.
3.  **Hot Reload**: Traefik is configured to use Docker for service discovery and a file provider for dynamic routing. It automatically detects the new routing file and reloads its configuration on the fly, without requiring a restart.
4.  **Deploy**: The Ansible playbook uses the `docker_stack` module to deploy your service to the Docker Swarm cluster. Docker Swarm ensures the service is running according to your definition (e.g., correct image, replicas, etc.).
5.  **Access Service**: The service is now live and accessible to end-users through the Traefik reverse proxy at the host and path you defined.

## Quick Start

### Prerequisites

- Ansible installed on control machine
- SSH access to target server (configure in `inventory.ini`)
- Docker installed on target server

### Configuration

Copy and configure the inventory file:
```bash
cp example.inventory.ini inventory.ini
# Edit inventory.ini with your server details (host, user, SSH key path)
```

**⚠️ Security Note**: Keep `inventory.ini` secure as it contains sensitive connection details.

### Initial Setup

Bootstrap the entire homelab infrastructure:
```bash
ansible-playbook -i inventory.ini playbooks/initialize.yaml -e "main_domain=yourdomain.com"
```

This will:
- Create necessary directories
- Initialize Docker Swarm
- Set up overlay network
- Deploy Traefik with dynamic routing (dashboard accessible at yourdomain.com/traefik/dashboard)
- Configure log rotation and maintenance cron jobs

**Required:** Replace `yourdomain.com` with your actual domain where Traefik will be accessible.

## Service Management

### Adding Services

Create a vars file for your service in the `vars/` directory:

```yaml
# vars/hello.yml
name: hello
image: devmuhfaris/hello-world
port: 80
host: apps.yourdomain.com
path: /hello
replicas: 1
```

For services with static assets (Next.js, React, etc.), add assets routing:
```yaml
# vars/webapp.yml
name: webapp
image: yourorg/webapp:latest
port: 3000
host: apps.yourdomain.com
path: /webapp
replicas: 2
priority: 2000
assets_paths:
  - "/_next/"
  - "/favicon.ico"
  - "/robots.txt"
  - "/static/"
```

Deploy the service:
```bash
ansible-playbook -i inventory.ini playbooks/add-service.yaml --extra-vars @vars/hello.yml
```

### Service Variables

**Required:**
- `name`: Service identifier
- `image`: Docker image reference  
- `port`: Internal service port
- `host`: External hostname
- `path`: URL path for routing

**Optional:**
- `replicas`: Number of service replicas (default: 1)
- `priority`: Traefik routing priority (default: 2000)
- `assets_paths`: List of asset paths for separate routing
- `assets_priority`: Priority for assets router (default: priority + 1000)
- `cpu`: CPU allocation per replica (default: "0.25")
- `memory`: Memory allocation per replica (default: "64M")

### Updating Services

Modify your vars file and redeploy:
```bash
ansible-playbook -i inventory.ini playbooks/add-service.yaml --extra-vars @vars/hello.yml
```

Changes to routing configuration trigger hot reloads. Changes to image or replicas trigger rolling updates.

## Directory Structure

```bash
/opt/homelab/                   # Root directory 
├── dynamic                     # Traefik dynamic routing
│   └── routers
│       └── hello.yaml 
├── hl-traefik.yaml
└── services                    # Docker Compose files
    └── hello.yaml

/var/log/traefik/               # Traefik logs
├── access.log                  # Access logs  
├── traefik.log                 # Application logs
```


## Architecture

The system uses a modular Ansible role structure:

- **common**: Infrastructure bootstrap and maintenance
- **traefik**: Reverse proxy configuration and dynamic routing
- **service**: Template-driven service deployment

Services are deployed as Docker Swarm services with automatic Traefik integration for routing and load balancing.
