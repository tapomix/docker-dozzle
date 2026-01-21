# Tapomix / Docker - Dozzle

[Dozzle](https://dozzle.dev/) is a lightweight, real-time log viewer for Docker containers. It streams logs directly from the Docker daemon without storing them on disk.

This project configures Dozzle to run behind [Traefik](https://traefik.io/traefik) and connect to Docker via a socket proxy for enhanced security.

## Installation

Clone this repository:

```bash
git clone https://github.com/tapomix/docker-dozzle.git dozzle
```

## Configuration

Copy the environment template file and edit it with your values:

```bash
cp .env.dist .env
```

### Environment Variables

| Variable | Description |
| -------- | ----------- |
| `CONTAINER_NAME` | Name of the Docker container |
| `SERVICE_VERSION` | Docker image version to use (default: *`latest`*) |
| `SOCKET_PROXY_NET` | Network for socket proxy connection |
| `TRAEFIK_HOST` | Hostname for Traefik routing (default: *`${CONTAINER_NAME}.domain.ext`*) |
| `TRAEFIK_NET` | Traefik network name (default: *`traefik-net`*) |
| `TRAEFIK_PORT` | Port exposed by the service for Traefik (default: *`8080`*) |
| `TZ` | Timezone (default: *`Etc/UTC`*) |
| `USER_ID` | User ID for file permissions (default: *`1000`*) |
| `GROUP_ID` | Group ID for file permissions (default: *`1000`*) |

> **Note:** `USER_ID` and `GROUP_ID` are used instead of `UID` and `GID` because the latter are reserved shell variables in bash. Using `UID`/`GID` in `.env` would cause issues when sourcing the file in shell scripts, as these variables cannot be overwritten.

## Usage

### Socket Proxy Configuration (required)

Dozzle connects to Docker via a socket proxy for security reasons. You **must** create a `compose.override.yaml` file to specify the remote host(s).

Create the file:

```yaml
// compose.override.yaml
services:
  dozzle:
    environment:
      DOZZLE_REMOTE_HOST: tcp://socket-proxy:2375
```

#### Multiple hosts

To monitor containers from multiple Docker hosts, separate them with a comma. You can add a label after `|` for each host:

```yaml
// compose.override.yaml
services:
  dozzle:
    environment:
      DOZZLE_REMOTE_HOST: tcp://socket-proxy-local:2375|local,tcp://socket-proxy-vps:2375|vps
```

> **Note:** Each socket proxy must be accessible via the network defined in `SOCKET_PROXY_NET`. If you have multiple proxies on different networks, add them to the compose file.

#### Socket proxy permissions

The socket proxy must allow the following endpoints for Dozzle to work properly:

- `CONTAINERS=1` - Required to list and access container logs
- `INFO=1` - Required for Docker host information (UI crashes without it)

If you enable container actions with `DOZZLE_ENABLE_ACTIONS=true`, you also need:

- `ALLOW_START=1`
- `ALLOW_STOP=1`
- `ALLOW_RESTART=1`

If you enable shell access with `DOZZLE_ENABLE_SHELL=true`, you also need:

- `POST=1` - Required for write operations
- `EXEC=1` - Required for exec endpoints (create, attach, resize)

### Authentication (optional)

To enable authentication, you need to configure an auth provider and create a users file.

#### 1. Add auth configuration

```yaml
// compose.override.yaml
services:
  dozzle:
    environment:
      # ...
      DOZZLE_AUTH_PROVIDER: simple

    # ...
    secrets:
      - source: auth-users
        target: /data/users.yml

secrets:
  auth-users:
    file: .docker/.secrets/users.yaml
```

#### 2. Create the users file

Create the directory and file:

```bash
mkdir -p .docker/.secrets
```

Create the secret file with your users:

```yaml
// .docker/.secrets/users.yaml
users:
  admin: # = login
    name: "Admin"
    password: "$2a$10$..."  # bcrypt hash
    email: admin@example.com
    roles: all

  viewer: # = login
    name: "Viewer"
    password: "$2a$10$..."  # bcrypt hash
    email: viewer@example.com
    roles: none
```

See official documentation for [available roles](https://dozzle.dev/guide/authentication#setting-specific-roles-for-users).

#### 3. Generate password hashes

Use the built-in Dozzle command to generate bcrypt hashes:

```bash
docker run --rm amir20/dozzle generate admin --password your-secure-password
```

Or use an online bcrypt generator / htpasswd tool.

### Security Features

This configuration includes several security hardening measures:

- **Read-only filesystem**: Container runs with `read_only: true`
- **No new privileges**: Prevents privilege escalation with `no-new-privileges`
- **Dropped capabilities**: All Linux capabilities are dropped with `cap_drop: ALL`
- **Resource limits**: CPU, memory, and PID limits are enforced
- **Socket proxy**: No direct access to Docker socket
- **Actions disabled**: `DOZZLE_ENABLE_ACTIONS=false` prevents container actions
- **Shell disabled**: `DOZZLE_ENABLE_SHELL=false` prevents shell access

## Alternatives

- [dtop](https://github.com/amir20/dtop) - Terminal-based Docker container viewer by the same developer as Dozzle, provides a `htop`-like interface
- [Portainer](https://www.portainer.io/) - Full-featured Docker management UI with log viewing and container management
- [Lazydocker](https://github.com/jesseduffield/lazydocker) - Terminal UI for Docker with logs, stats, and container management

## Resources

- Dozzle documentation: <https://dozzle.dev/guide/getting-started>
- Dozzle GitHub: <https://github.com/amir20/dozzle>
