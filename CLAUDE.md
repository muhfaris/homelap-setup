# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Architecture Overview

This is an Ansible-based homelab infrastructure automation system using Docker Swarm. The architecture consists of:

- **Ansible Configuration**: Standard ansible.cfg with roles and inventory setup
- **Target Infrastructure**: Single server deployment to `server.com`
- **Container Orchestration**: Docker Swarm with overlay networking
- **Reverse Proxy**: Traefik with dynamic file-based routing
- **Service Structure**: Template-driven service deployment with automatic routing

## Key Components

### Roles Architecture
- `common`: Bootstrap tasks (directories, Docker Swarm init, networking, cron jobs)
- `traefik`: Reverse proxy setup with dynamic routing capabilities
- `service`: Template-driven service deployment with validation

### Directory Structure on Target
- `/opt/homelab/`: Root directory for all homelab services
- `/opt/homelab/services/`: Docker Compose files for individual services
- `/opt/homelab/dynamic/routers/`: Traefik dynamic routing configuration
- `/var/log/traefik/`: Traefik access and application logs

## Common Commands

### Initial Deployment
Bootstrap the entire homelab infrastructure:
```bash
ansible-playbook -i inventory.ini playbooks/initialize.yaml -e "main_domain=yourdomain.com"
```

Required variables:
- `main_domain`: Main domain where Traefik dashboard will be accessible (validated - playbook will fail if not provided)

### Adding Services
Deploy a new service with automatic routing using vars files:
```bash
ansible-playbook -i inventory.ini playbooks/add-service.yaml --extra-vars @vars/SERVICE_NAME.yml
```

Create a vars file for your service (e.g., `vars/hello.yml`):
```yaml
name: hello
image: devmuhfaris/hello-world
port: 80
host: apps.yourdomain.com
path: /hello
replicas: 1
```

Then deploy:
```bash
ansible-playbook -i inventory.ini playbooks/add-service.yaml --extra-vars @vars/hello.yml
```

### Service Updates
Update existing services (supports hot-reload for routing changes):
```bash
ansible-playbook -i inventory.ini playbooks/add-service.yaml --extra-vars @vars/hello.yml
```

## Service Template System

Services require these mandatory variables:
- `name`: Service identifier
- `image`: Docker image reference
- `port`: Internal service port
- `host`: External hostname
- `path`: URL path for routing

Optional variables:
- `replicas`: Number of service replicas (default: 1)
- `priority`: Traefik routing priority (default: 2000)
- `assets_paths`: List of asset paths for separate routing (e.g., `["/_next/", "/favicon.ico", "/robots.txt", "/static/"]`)
- `assets_priority`: Priority for assets router (default: priority + 1000)

## Infrastructure Features

### Automation
- **Docker Swarm**: Automatic initialization and overlay network (`web`) creation
- **Log Management**: Automatic logrotate for Traefik logs (14-day retention)
- **Maintenance**: Weekly Docker system pruning via cron

### Service Deployment
- **Hot Reload**: Traefik routing updates without service restarts
- **Rolling Updates**: Automatic when image/replica changes detected
- **Validation**: Required variable validation before deployment

## Templates
Key Jinja2 templates:
- `hl-traefik.yaml.j2`: Main Traefik service configuration
- `service-swarm.yaml.j2`: Docker Swarm service definitions
- `route-file.yaml.j2`: Dynamic Traefik routing rules

## Variable Management

### Service Configuration
Store service-specific variables in `vars/` directory using YAML files.

Basic service example:
```yaml
# vars/hello.yml
name: hello
image: devmuhfaris/hello-world
port: 80
host: apps.yourdomain.com
path: /hello
replicas: 1
```

Service with assets routing:
```yaml
# vars/webapp.yml
name: webapp
image: yourorg/webapp:latest
port: 3000
host: apps.yourdomain.com
path: /webapp
assets_paths:
  - "/_next/"
  - "/favicon.ico"
  - "/robots.txt"
  - "/static/"
```

Deploy with: `--extra-vars @vars/hello.yml`
