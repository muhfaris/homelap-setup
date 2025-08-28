# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## AI Agent Workflow

This section provides a guide for an AI agent to follow when performing common tasks.

### General Approach
1.  **Understand the Goal:** Read the user's request carefully to understand what needs to be done.
2.  **Consult this Document:** This file is your primary source of truth. Refer to it before exploring the codebase.
3.  **Formulate a Plan:** Based on the user's request and the information in this document, create a step-by-step plan.
4.  **Execute the Plan:** Follow your plan, using the commands and file structures described below.
5.  **Verify Changes:** After making changes, always verify that they have been applied correctly.

### Workflow: Adding a New Service
1.  **Create a Service Definition File:** Create a new `.yaml` file in the `vars/` directory (e.g., `vars/new-service.yaml`).
2.  **Define the Service:** Add the required variables (`name`, `image`, `port`, `host`, `path`) and any optional variables to the file.
3.  **Run the `add-service` Playbook:** Execute the following command, replacing `new-service.yaml` with your filename:
    ```bash
    ansible-playbook -i inventory.ini playbooks/add-service.yaml --extra-vars @vars/new-service.yaml
    ```

### Workflow: Updating an Existing Service
1.  **Locate the Service Definition File:** Find the corresponding `.yaml` file for the service in the `vars/` directory.
2.  **Modify the Definition:** Update the file with the new configuration (e.g., change the `image` tag, increase `replicas`).
3.  **Run the `update-service` Playbook:** Execute the following command, replacing `service-name.yaml` with the correct filename:
    ```bash
    ansible-playbook -i inventory.ini playbooks/update-service.yaml --extra-vars @vars/service-name.yaml
    ```

## Architecture Overview

This is an Ansible-based homelab infrastructure automation system using Docker Swarm. The architecture consists of:

- **Ansible Configuration**: Standard ansible.cfg with roles and inventory setup
- **Target Infrastructure**: Single server deployment to `server.com`
- **Container Orchestration**: Docker Swarm with overlay networking
- **Reverse Proxy**: Traefik with dynamic file-based routing
- **Service Structure**: Template-driven service deployment with automatic routing

## Key Components

### Roles Architecture
- `common`: Handles initial server bootstrap tasks like creating directories, initializing Docker Swarm, setting up the overlay network, and creating cron jobs for maintenance.
- `shared`: This role does not contain tasks but provides a crucial handler:
  - `Redeploy homelab stack`: This handler is triggered by the `service` and `traefik` roles. It gathers all the generated configuration files (`hl-traefik.yaml` and all service files in `services/`) and applies them to the Docker Swarm stack. This is the core mechanism for deploying and updating the entire infrastructure.
- `traefik`: Sets up the Traefik reverse proxy service. It uses `hl-traefik.yaml.j2` to create the main Traefik configuration.
- `service`: A generic role to deploy a new service. It uses the `service-swarm.yaml.j2` and `route-file.yaml.j2` templates to create the necessary configuration files for a given service.

### Directory Structure on Target
- `/opt/homelab/`: Root directory for all homelab services
- `/opt/homelab/services/`: Docker Compose files for individual services
- `/opt/homelab/dynamic/routers/`: Traefik dynamic routing configuration
- `/var/log/traefik/`: Traefik access and application logs

## Common Commands

### Initial Deployment

#### 1. Configuration
First, set up your inventory file. Copy the example inventory file and edit it with your server's details.

```bash
cp example.inventory.ini inventory.ini
# Edit inventory.ini with your server's host, user, and SSH key path
```

#### 2. Execution
Bootstrap the entire homelab infrastructure with the following command.

**Note:** The `main_domain` variable is required and the playbook will fail if it's not provided. This should be the domain where the Traefik dashboard will be accessible.

```bash
ansible-playbook -i inventory.ini playbooks/initialize.yaml -e "main_domain=yourdomain.com"
```

### Adding Services
Deploy a new service by creating a `service-name.yaml` file in the `vars/` directory and running the `add-service.yaml` playbook.

Example `vars/hello.yaml`:
```yaml
name: hello
image: devmuhfaris/hello-world
port: 80
host: apps.muhfaris.com
path: /hello
replicas: 1
cpu: "0.5"
memory: "512M"
```

Deploy the service:
```bash
ansible-playbook -i inventory.ini playbooks/add-service.yaml --extra-vars @vars/hello.yaml
```

### Service Updates
To update an existing service (e.g., change the image version, replicas, or other parameters), modify the corresponding file in the `vars/` directory and run the `update-service.yaml` playbook.

```bash
ansible-playbook -i inventory.ini playbooks/update-service.yaml --extra-vars @vars/hello.yaml
```
This playbook uses a "stop-first" update strategy.

### Updating Traefik
To update the main Traefik configuration (e.g., change ports, labels), run the `update-traefik.yaml` playbook.
```bash
ansible-playbook -i inventory.ini playbooks/update-traefik.yaml
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
- `cpu`: CPU allocation per replica (default: "0.25")
- `memory`: Memory allocation per replica (default: "64M")

## Infrastructure Features

### Automation
- **Docker Swarm**: Automatic initialization and overlay network (`web`) creation
- **Log Management**: Automatic logrotate for Traefik logs (14-day retention)
- **Maintenance**: Weekly Docker system pruning via cron

### Service Deployment
- **Hot Reload**: Traefik routing updates without service restarts
- **Rolling Updates**: Automatic when image/replica changes detected
- **Validation**: Required variable validation before deployment

## Debugging

If a service fails to deploy or is not accessible, here are some steps to troubleshoot:

1.  **Check the Stack Status:**
    SSH into the target server and check the status of the main stack.
    ```bash
    docker stack ps hl
    ```
    This command will show you the status of all services in the `hl` stack. Look for any services that have a "Shutdown" or "Rejected" state.

2.  **View Service Logs:**
    To view the logs for a specific service, use the following command on the target server:
    ```bash
    docker service logs hl_<service_name>
    ```
    For example, for the `hello` service, you would run `docker service logs hl_hello`.

3.  **Check Traefik Logs:**
    Traefik's access and application logs are located in `/var/log/traefik/` on the target server.
    - `access.log`: Shows all incoming requests.
    - `traefik.log`: Shows Traefik's own logs, including any errors with routing configuration.

4.  **Inspect Generated Files:**
    The Ansible playbooks generate configuration files on the target server. You can inspect these to ensure they were rendered correctly.
    - Service definitions: `/opt/homelab/services/`
    - Traefik dynamic routing: `/opt/homelab/dynamic/routers/`

## Templates

The `service` and `traefik` roles use Jinja2 templates to generate configuration files on the target server.

- **`hl-traefik.yaml.j2`**:
  - **Role:** `traefik`
  - **Output:** `/opt/homelab/hl-traefik.yaml`
  - **Purpose:** Defines the main Traefik service, including its network, ports, and command-line arguments. This is the core configuration for the reverse proxy.

- **`service-swarm.yaml.j2`**:
  - **Role:** `service`
  - **Output:** `/opt/homelab/services/{{ name }}.yaml`
  - **Purpose:** Generates a Docker Compose file for a specific service. It defines the Docker Swarm service, including the image, replicas, resource limits, and Traefik labels for routing.

- **`route-file.yaml.j2`**:
  - **Role:** `service`
  - **Output:** `/opt/homelab/dynamic/routers/{{ name }}.yaml`
  - **Purpose:** Creates a Traefik dynamic configuration file for a service. This file defines the router, middleware, and service for Traefik's file provider, enabling dynamic routing without restarting Traefik.

## Variable Management

### Service Configuration
Store service-specific variables in `vars/` directory using YAML files (e.g., `vars/hello.yaml`).

Basic service example:
```yaml
# vars/hello.yaml
name: hello
image: devmuhfaris/hello-world
port: 80
host: apps.muhfaris.com
path: /hello
replicas: 1
```

Service with assets routing:
```yaml
# vars/webapp.yaml
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

Deploy with: `--extra-vars @vars/hello.yaml`
