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
