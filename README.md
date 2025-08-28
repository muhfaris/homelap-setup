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

```mermaid
graph TD
    subgraph "Your Local Machine"
        A["<br><b>You</b><br>"] -- "1. Run ansible-playbook" --> B(vars/hello.yaml + playbook command)
    end

    subgraph "Homelab Server"
        C[Ansible `service` Role] -->|Uses| D(Jinja2 Templates)
        C -->|2. Generates| E(services/hello.yaml)
        C -->|2. Generates| F(dynamic/routers/hello.yaml)
        E -->|4. Deploy| G(Docker Swarm)
        F -->|3. Hot Reload| H(Traefik)
    end

    I["<br><b>End User</b><br>"]

    B -- SSH --> C
    G --> H
    H -->|5. Access Service| I
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
- `command`: Custom command to run in the container. Required for `service_config`.
- `service_config`: A dictionary of structured configuration data to be mounted as a file.
- `service_config_format`: The format of the configuration file (`yaml` or `json`). Defaults to `yaml`.

### Updating Services

Modify your vars file and redeploy:
```bash
ansible-playbook -i inventory.ini playbooks/add-service.yaml --extra-vars @vars/hello.yml
```

Changes to routing configuration trigger hot reloads. Changes to image or replicas trigger rolling updates.

### Advanced Service Configuration

For applications requiring structured configuration, you can use the `service_config` feature. This allows you to inject a configuration file directly into your service container.

#### How It Works
When you define a `service_config` object in your service's variable file, the system:
1. Creates a Docker Swarm config containing your structured data (in YAML or JSON format).
2. Mounts this config as a file into your service container at `/<service-name>.config.<format>`.
3. Appends a `--config` flag to your service's `command`, pointing to the mounted config file.

#### Example
Here’s an example of a service that uses a YAML configuration:
```yaml
# vars/my-app.yml
name: my-app
image: my-org/my-app:latest
port: 8080
host: apps.yourdomain.com
path: /my-app
command: "node server.js"
service_config:
  database:
    host: db.internal
    port: 5432
  api_keys:
    - key: "key1"
      value: "value1"
```

In this example:
- A configuration file will be mounted at `/my-app.config.yaml`.
- The service's command will be executed as: `node server.js --config /my-app.config.yaml`.

**Note**: Your application must be able to parse the configuration file passed via the `--config` flag.

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
