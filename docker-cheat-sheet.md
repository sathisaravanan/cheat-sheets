# 🐳 Docker & Docker Compose Cheat Sheet

A comprehensive reference for everyday Docker work — images, containers, networking, volumes, Compose, and cleanup.

---

## 1. Info, Version & Help

```bash
docker version                 # Client + server version
docker info                    # System-wide info (containers, images, driver)
docker --help                  # List all top-level commands
docker <command> --help        # Help for a specific command
docker system df               # Disk usage (images, containers, volumes)
docker login                   # Log in to a registry (Docker Hub by default)
docker login registry.example.com
docker logout
```

---

## 2. Images

### Pull / List / Inspect
```bash
docker pull nginx                     # Pull latest tag
docker pull nginx:1.27                # Pull a specific tag
docker images                         # List local images
docker image ls                       # Same as above
docker image ls -a                    # Include intermediate images
docker image inspect nginx            # Full metadata (JSON)
docker history nginx                  # Show image layers
```

### Build
```bash
docker build -t myapp:1.0 .                          # Build from ./Dockerfile
docker build -t myapp:1.0 -f Dockerfile.prod .       # Custom Dockerfile
docker build --no-cache -t myapp:1.0 .               # Ignore build cache
docker build --build-arg VERSION=1.0 -t myapp .      # Pass build args
docker build --target builder -t myapp:build .       # Stop at a build stage
docker buildx build --platform linux/amd64,linux/arm64 -t myapp .   # Multi-arch
```

### Tag / Push / Save / Load
```bash
docker tag myapp:1.0 user/myapp:1.0             # Retag an image
docker push user/myapp:1.0                       # Push to registry
docker save -o myapp.tar myapp:1.0               # Export image to tar
docker load -i myapp.tar                         # Import image from tar
```

### Remove
```bash
docker rmi nginx                    # Remove an image
docker rmi -f nginx                 # Force remove
docker image prune                  # Remove dangling images
docker image prune -a               # Remove ALL unused images
```

---

## 3. Containers — Running

```bash
docker run nginx                              # Run (foreground)
docker run -d nginx                           # Run detached (background)
docker run -it ubuntu bash                    # Interactive terminal
docker run --name web nginx                   # Name the container
docker run -d -p 8080:80 nginx                # Map host:container ports
docker run -d -P nginx                        # Map all exposed ports randomly
docker run -e ENV=prod nginx                  # Set an env variable
docker run --env-file .env nginx              # Load env vars from a file
docker run -v /host/path:/container/path nginx      # Bind mount
docker run -v mydata:/data nginx                    # Named volume
docker run --rm nginx                         # Auto-remove on exit
docker run --network mynet nginx              # Attach to a network
docker run --restart unless-stopped nginx     # Restart policy
docker run -m 512m --cpus 1.5 nginx           # Resource limits
docker run -w /app node:20 npm start          # Set working dir
docker run -u 1000:1000 nginx                 # Run as user:group
docker run -d --health-cmd="curl -f localhost || exit 1" nginx   # Healthcheck
```

**Common `run` flags at a glance**

| Flag | Meaning |
|------|---------|
| `-d` | Detached (background) |
| `-it` | Interactive + TTY |
| `-p` | Publish port `host:container` |
| `-v` | Mount volume / bind mount |
| `-e` | Environment variable |
| `--name` | Container name |
| `--rm` | Remove container when it exits |
| `--network` | Connect to a network |
| `--restart` | Restart policy (`no`, `on-failure`, `always`, `unless-stopped`) |

---

## 4. Containers — Manage

```bash
docker ps                          # Running containers
docker ps -a                       # All containers (incl. stopped)
docker ps -q                       # Only container IDs
docker ps --filter "status=exited"
docker start web                   # Start a stopped container
docker stop web                    # Graceful stop (SIGTERM)
docker restart web                 # Restart
docker kill web                    # Force stop (SIGKILL)
docker pause web                   # Pause processes
docker unpause web                 # Resume
docker rename web web2             # Rename
docker rm web                      # Remove a stopped container
docker rm -f web                   # Force remove (running)
docker rm $(docker ps -aq)         # Remove ALL containers
docker update --memory 1g web      # Update resource limits live
```

---

## 5. Inspecting & Debugging Containers

```bash
docker logs web                    # View logs
docker logs -f web                 # Follow logs (live)
docker logs --tail 100 web         # Last 100 lines
docker logs -t web                 # With timestamps
docker exec -it web bash           # Shell into running container
docker exec web ls /app            # Run one-off command
docker inspect web                 # Full JSON metadata
docker inspect -f '{{.State.Status}}' web    # Extract a single field
docker top web                     # Running processes inside
docker stats                       # Live resource usage (all)
docker stats web                   # Live stats for one container
docker port web                    # Show port mappings
docker diff web                    # Files changed vs image
docker cp web:/app/file.txt .      # Copy from container to host
docker cp ./file.txt web:/app/     # Copy from host to container
docker attach web                  # Attach to main process (careful!)
docker wait web                    # Block until container stops
docker export web -o web.tar       # Export container filesystem
docker commit web myimage:snapshot # Create image from container
```

---

## 6. Networking

```bash
docker network ls                              # List networks
docker network create mynet                    # Create a network (bridge)
docker network create -d overlay myoverlay     # Overlay (swarm)
docker network inspect mynet                   # Details
docker network connect mynet web               # Attach container to network
docker network disconnect mynet web            # Detach
docker network rm mynet                        # Remove network
docker network prune                           # Remove unused networks
```

**Built-in network drivers:** `bridge` (default), `host`, `none`, `overlay`, `macvlan`.

---

## 7. Volumes & Data

```bash
docker volume ls                       # List volumes
docker volume create mydata            # Create a named volume
docker volume inspect mydata           # Details
docker volume rm mydata                # Remove a volume
docker volume prune                    # Remove unused volumes
```

- **Named volume:**  `-v mydata:/data`
- **Bind mount:**    `-v /host/path:/container/path`
- **Read-only:**     `-v mydata:/data:ro`
- **tmpfs (memory):** `--tmpfs /tmp`

---

## 8. System Cleanup

```bash
docker system df                   # Show disk usage
docker system prune                # Remove stopped containers, dangling images, unused networks
docker system prune -a             # Also remove ALL unused images
docker system prune -a --volumes   # Also remove unused volumes ⚠️
docker container prune             # Remove stopped containers
docker image prune -a              # Remove unused images
docker volume prune                # Remove unused volumes
docker network prune               # Remove unused networks
docker builder prune               # Clear build cache
```

> ⚠️ `prune` commands are destructive. Double-check before running on shared machines.

---

## 9. Dockerfile Essentials

```dockerfile
# Base image
FROM node:20-alpine

# Metadata
LABEL maintainer="you@example.com"

# Build-time variable
ARG APP_VERSION=1.0

# Runtime environment variable
ENV NODE_ENV=production

# Working directory
WORKDIR /app

# Copy files
COPY package*.json ./
RUN npm ci --omit=dev
COPY . .

# Expose a port (documentation only)
EXPOSE 3000

# Volume mount point
VOLUME ["/data"]

# Health check
HEALTHCHECK --interval=30s --timeout=3s CMD curl -f http://localhost:3000 || exit 1

# Default command (exec form preferred)
CMD ["node", "server.js"]

# ENTRYPOINT + CMD combo
# ENTRYPOINT ["node"]
# CMD ["server.js"]
```

**Key instructions:** `FROM`, `RUN`, `CMD`, `ENTRYPOINT`, `COPY`, `ADD`, `WORKDIR`, `ENV`, `ARG`, `EXPOSE`, `VOLUME`, `USER`, `LABEL`, `HEALTHCHECK`, `ONBUILD`, `SHELL`, `STOPSIGNAL`.

**Multi-stage build example**
```dockerfile
FROM golang:1.22 AS builder
WORKDIR /src
COPY . .
RUN go build -o app

FROM alpine:latest
COPY --from=builder /src/app /app
CMD ["/app"]
```

**`.dockerignore`** — exclude files from the build context:
```
node_modules
.git
*.log
.env
```

---

## 10. Docker Compose

> Modern CLI is `docker compose` (v2, space). Older standalone is `docker-compose` (hyphen). Commands are otherwise the same.

### Lifecycle
```bash
docker compose up                  # Create + start services (foreground)
docker compose up -d               # Detached
docker compose up --build          # Rebuild images before starting
docker compose up --force-recreate # Recreate containers
docker compose down                # Stop + remove containers/networks
docker compose down -v             # Also remove named volumes ⚠️
docker compose down --rmi all      # Also remove images
docker compose start               # Start existing services
docker compose stop                # Stop without removing
docker compose restart             # Restart services
docker compose pause / unpause
```

### Build & Images
```bash
docker compose build               # Build all services
docker compose build web           # Build one service
docker compose build --no-cache
docker compose pull                # Pull service images
docker compose push                # Push service images
```

### Inspect & Debug
```bash
docker compose ps                  # List service containers
docker compose ps -a               # Include stopped
docker compose logs                # All logs
docker compose logs -f             # Follow
docker compose logs -f web         # Follow one service
docker compose top                 # Running processes
docker compose config              # Validate + view merged config
docker compose config --services   # List service names
docker compose images              # Images used by services
docker compose events              # Stream real-time events
```

### Exec & Run
```bash
docker compose exec web bash       # Shell into a running service
docker compose exec web env        # Run command in running service
docker compose run --rm web bash   # New one-off container
docker compose run --rm web npm test
```

### Scaling & Files
```bash
docker compose up -d --scale web=3          # Run 3 replicas of "web"
docker compose -f docker-compose.prod.yml up -d          # Custom file
docker compose -f base.yml -f override.yml up -d         # Merge multiple files
docker compose --env-file .env.prod up -d               # Custom env file
docker compose -p myproject up -d           # Custom project name
```

### Example `docker-compose.yml`
```yaml
services:
  web:
    build: .
    # image: myapp:1.0
    ports:
      - "8080:80"
    environment:
      - NODE_ENV=production
    env_file:
      - .env
    depends_on:
      db:
        condition: service_healthy
    volumes:
      - ./src:/app/src
    networks:
      - appnet
    restart: unless-stopped

  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: mydb
    volumes:
      - dbdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - appnet

volumes:
  dbdata:

networks:
  appnet:
    driver: bridge
```

---

## 11. Docker Swarm (Orchestration)

```bash
docker swarm init                          # Initialize a swarm (manager)
docker swarm join --token <token> <ip>:2377    # Join as worker/manager
docker swarm join-token worker             # Show worker join token
docker swarm leave                         # Leave the swarm
docker node ls                             # List nodes
docker service create --name web -p 80:80 nginx    # Create a service
docker service ls                          # List services
docker service ps web                      # List tasks for a service
docker service scale web=5                 # Scale replicas
docker service update --image nginx:1.27 web       # Rolling update
docker service rm web                      # Remove a service
docker stack deploy -c docker-compose.yml mystack  # Deploy a stack
docker stack ls / services / rm
```

---

## 12. Registry (Docker Hub & Private)

```bash
docker login                                        # Docker Hub
docker login myregistry.example.com:5000            # Private registry
docker tag myapp:1.0 myregistry.example.com/myapp:1.0
docker push myregistry.example.com/myapp:1.0
docker pull myregistry.example.com/myapp:1.0
docker search nginx                                 # Search Docker Hub
```

Run a local registry:
```bash
docker run -d -p 5000:5000 --name registry registry:2
```

---

## 13. Handy One-Liners

```bash
# Stop all running containers
docker stop $(docker ps -q)

# Remove all containers
docker rm -f $(docker ps -aq)

# Remove all images
docker rmi -f $(docker images -q)

# Remove dangling (untagged) images
docker rmi $(docker images -f "dangling=true" -q)

# Follow logs of the most recently started container
docker logs -f $(docker ps -lq)

# Shell into the last-created container
docker exec -it $(docker ps -lq) sh

# Get a container's IP address
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' web

# Show all container names
docker ps -a --format '{{.Names}}'

# Total reclaimable space
docker system df -v

# Kill everything and clean up (use with care ⚠️)
docker rm -f $(docker ps -aq); docker system prune -af --volumes
```

---

## 14. Formatting Cheats (`--format`)

```bash
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
docker images --format "{{.Repository}}:{{.Tag}} -> {{.Size}}"
docker inspect --format '{{.State.Running}}' web
```

Common placeholders: `.ID`, `.Names`, `.Image`, `.Status`, `.Ports`, `.Command`, `.CreatedAt`, `.Size`.

---

### Quick mental model
- **Image** = blueprint (read-only). **Container** = running instance of an image.
- **`build`** makes images, **`run`** makes containers, **`compose`** runs multi-container apps.
- **Volumes** persist data; **bind mounts** map host folders; **networks** let containers talk.

Happy shipping! 🚀
