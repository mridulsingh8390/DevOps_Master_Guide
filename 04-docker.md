# Docker Complete Guide (Beginner to Advanced, DCA-Oriented)
## For DevOps Engineers (Ubuntu 24.04, local/on-prem)

> This file is detailed and practical.
> Each section includes:
> - What/Why
> - Commands
> - Explanation
> - Verification
> - Common mistakes and fixes
> - Practice tasks

---

## Table of Contents

1. What is Docker and why it matters  
2. Install Docker Engine + CLI + Compose  
3. Docker architecture basics  
4. Image lifecycle commands  
5. Container lifecycle commands  
6. Logs, exec, inspect, stats, events  
7. Volumes and persistent data  
8. Docker networks  
9. Docker Compose (single-host orchestration)  
10. Docker Swarm essentials  
11. Docker secrets/configs (Swarm)  
12. Save/load/export/import  
13. Dockerfile complete guide  
14. Build best practices and security  
15. Troubleshooting  
16. Practice lab tasks  
17. Daily cheat sheet

---

## 1) What is Docker and Why it Matters

## What
Docker packages apps and dependencies into portable containers.

## Why
- Works consistently across environments
- Faster than full VMs
- Simplifies CI/CD
- Easy rollback with image tags

---

## 2) Install Docker Engine + CLI + Compose

## 2.1 Install prerequisites
```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg
```

## 2.2 Add official Docker GPG key + repo
```bash
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
"deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

## 2.3 Install Docker components
```bash
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

## 2.4 Start and enable service
```bash
sudo systemctl enable --now docker
sudo systemctl status docker
```

## 2.5 Run docker without sudo
```bash
sudo usermod -aG docker $USER
newgrp docker
```

## Verify
```bash
docker version
docker info
docker run --rm hello-world
```

---

## 3) Docker Architecture Basics

- **Docker daemon (`dockerd`)**: backend service
- **Docker CLI (`docker`)**: user command tool
- **Image**: read-only template
- **Container**: runtime instance of image
- **Registry**: image store (Docker Hub/private registry)

---

## 4) Image Lifecycle Commands

## Pull image
```bash
docker pull nginx:latest
```

## List images
```bash
docker images
docker image ls
```

## Build image
```bash
docker build -t myapp:v1 .
```

## Tag image
```bash
docker tag myapp:v1 myrepo/myapp:v1
```

## Push image
```bash
docker push myrepo/myapp:v1
```

## Remove image
```bash
docker rmi myapp:v1
```

## Cleanup
```bash
docker image prune -f
docker image prune -a -f
```

---

## 5) Container Lifecycle Commands

## Run container (detached)
```bash
docker run -d --name web -p 8080:80 nginx
```

## Run interactive shell
```bash
docker run -it --rm ubuntu:24.04 bash
```

## List containers
```bash
docker ps
docker ps -a
```

## Start/stop/restart
```bash
docker start web
docker stop web
docker restart web
```

## Remove container
```bash
docker rm web
docker rm -f web
```

## Rename container
```bash
docker rename web web-prod
```

---

## 6) Logs, Exec, Inspect, Stats, Events

## Logs
```bash
docker logs web
docker logs -f web
docker logs --tail 100 web
```

## Exec into running container
```bash
docker exec -it web sh
docker exec -it web bash
```

## Inspect metadata
```bash
docker inspect web
docker inspect --format '{{.NetworkSettings.IPAddress}}' web
```

## Resource usage
```bash
docker stats
```

## Real-time daemon events
```bash
docker events
```

---

## 7) Volumes and Persistent Data

## Why volumes?
Container filesystem is ephemeral. Volumes persist data across restarts/removal.

## Create/list/inspect/remove volume
```bash
docker volume create app-vol
docker volume ls
docker volume inspect app-vol
docker volume rm app-vol
```

## Mount named volume
```bash
docker run -d --name db \
  -e POSTGRES_PASSWORD=secret \
  -v pgdata:/var/lib/postgresql/data \
  postgres:16
```

## Bind mount (host path)
```bash
docker run -d --name nginx-bind \
  -v /home/user/site:/usr/share/nginx/html:ro \
  -p 8081:80 nginx
```

---

## 8) Docker Networks

## List/create network
```bash
docker network ls
docker network create app-net
```

## Run containers on same network
```bash
docker run -d --name app1 --network app-net nginx
docker run -d --name app2 --network app-net busybox sleep 3600
```

## Inspect network
```bash
docker network inspect app-net
```

## Connect/disconnect running container
```bash
docker network connect app-net web
docker network disconnect app-net web
```

Network drivers (common):
- `bridge` (default single-host)
- `host` (shares host network namespace)
- `none` (no networking)
- `overlay` (Swarm multi-host)

---

## 9) Docker Compose (Single-host Orchestration)

## 9.1 Example `compose.yaml`
```yaml
services:
  web:
    image: nginx:latest
    ports:
      - "8080:80"
    depends_on:
      - redis

  redis:
    image: redis:7
    volumes:
      - redisdata:/data

volumes:
  redisdata:
```

## 9.2 Commands
```bash
docker compose up -d
docker compose ps
docker compose logs -f
docker compose down
docker compose down -v
docker compose config
```

## Why Compose?
Manage multi-container apps via single YAML.

---

## 10) Docker Swarm Essentials

## Init swarm
```bash
docker swarm init
```

## View nodes
```bash
docker node ls
```

## Create service
```bash
docker service create --name web --replicas 2 -p 8080:80 nginx
```

## Service management
```bash
docker service ls
docker service ps web
docker service scale web=4
docker service update --image nginx:alpine web
docker service rollback web
docker service rm web
```

## Leave swarm
```bash
docker swarm leave --force
```

---

## 11) Docker Secrets and Configs (Swarm)

## Secret
```bash
echo "MyDBPassword123" | docker secret create db_password -
docker secret ls
docker secret rm db_password
```

## Config
```bash
echo "APP_ENV=prod" > app.env
docker config create app_env app.env
docker config ls
docker config rm app_env
```

---

## 12) Save/Load/Export/Import

## Save and load image (image-level)
```bash
docker save -o myapp_v1.tar myapp:v1
docker load -i myapp_v1.tar
```

## Export/import container filesystem
```bash
docker export web -o web_fs.tar
cat web_fs.tar | docker import - web-imported:latest
```

Difference:
- `save/load`: preserves image layers and metadata
- `export/import`: plain container filesystem snapshot

---

## 13) Dockerfile Complete Guide

## 13.1 Core instructions
- `FROM`: base image
- `WORKDIR`: set working directory
- `COPY` / `ADD`: copy files
- `RUN`: build-time commands
- `ENV`: environment variable
- `ARG`: build-time variable
- `EXPOSE`: document app port
- `CMD`: default runtime command
- `ENTRYPOINT`: fixed executable
- `USER`: run as non-root
- `HEALTHCHECK`: runtime health probe

---

## 13.2 Python Dockerfile
```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 5000
CMD ["python", "app.py"]
```

## 13.3 Node.js Dockerfile
```dockerfile
FROM node:20-alpine
WORKDIR /usr/src/app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
EXPOSE 3000
CMD ["node", "server.js"]
```

## 13.4 Java multi-stage
```dockerfile
FROM maven:3.9.8-eclipse-temurin-17 AS builder
WORKDIR /build
COPY pom.xml .
COPY src ./src
RUN mvn clean package -DskipTests

FROM eclipse-temurin:17-jre
WORKDIR /app
COPY --from=builder /build/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java","-jar","app.jar"]
```

## 13.5 Go multi-stage
```dockerfile
FROM golang:1.22 AS builder
WORKDIR /src
COPY . .
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -o app main.go

FROM gcr.io/distroless/static:nonroot
COPY --from=builder /src/app /app
USER nonroot:nonroot
EXPOSE 8080
ENTRYPOINT ["/app"]
```

## 13.6 Nginx static site
```dockerfile
FROM nginx:alpine
COPY ./site /usr/share/nginx/html
EXPOSE 80
CMD ["nginx","-g","daemon off;"]
```

## 13.7 ARG vs ENV example
```dockerfile
FROM alpine:3.20
ARG APP_VERSION=1.0.0
ENV APP_ENV=dev
RUN echo "Build version: ${APP_VERSION}" > /version.txt
CMD ["sh","-c","echo Runtime env is ${APP_ENV}; sleep 3600"]
```

## 13.8 Non-root user example
```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY . .
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
CMD ["node","server.js"]
```

## 13.9 `.dockerignore`
```gitignore
.git
.gitignore
node_modules/
__pycache__/
*.log
.env
dist/
build/
```

---

## 14) Build Best Practices and Security

1. Use minimal base images (`alpine`, `slim`, distroless)
2. Use multi-stage builds to reduce final image size
3. Pin image versions (`nginx:1.27` not `latest` in prod)
4. Do not bake secrets into image
5. Use non-root user where possible
6. Reduce layers and cleanup package cache
7. Scan images (e.g., Trivy/Grype)

---

## 15) Troubleshooting

## 15.1 Docker daemon issues
```bash
sudo systemctl status docker
sudo journalctl -u docker -f
```

## 15.2 Permission denied on docker socket
```bash
sudo usermod -aG docker $USER
newgrp docker
```

## 15.3 Port already in use
```bash
sudo ss -tulnp | grep 8080
```
Change host port mapping.

## 15.4 Container exits immediately
```bash
docker logs <container>
docker inspect <container>
```

## 15.5 Disk full from images/containers
```bash
docker system df
docker system prune -a --volumes
```
(Use carefully; removes unused resources.)

---

## 16) Practice Lab Tasks

1. Install Docker and run hello-world  
2. Build Python app image and run container  
3. Create named volume and prove data persists after container recreation  
4. Create custom bridge network and test container-to-container DNS  
5. Run 2-service stack using Compose  
6. Initialize Swarm and deploy scalable service  
7. Build multi-stage Java or Go image  
8. Add healthcheck and non-root user in Dockerfile  

---

## 17) Daily Docker Cheat Sheet

```bash
# Info
docker version
docker info
docker ps -a
docker images

# Build/run
docker build -t myapp:v1 .
docker run -d --name myapp -p 5000:5000 myapp:v1

# Debug
docker logs -f myapp
docker exec -it myapp sh
docker inspect myapp

# Cleanup
docker container prune -f
docker image prune -f
docker volume prune -f
```

---

## Final Notes

- Containers are immutable runtime units; rebuild image for app changes.
- Compose is ideal for local dev; Swarm/Kubernetes for orchestration.
- Learn Dockerfile optimization early—it impacts cost, speed, and security.