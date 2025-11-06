# Docker Quick Guide (No ECS)

This README covers **build → tag → push → run → stop** for a Node.js app using Docker.  
It also provides production-ready templates for **Dockerfile**, **.dockerignore**, and **docker-compose**.
Replace names like `your-docker-id` and `myapp` with your own.

---

## Prerequisites
- Docker Engine 24+ (or Docker Desktop)
- Optional: Docker Hub account (or your private registry)

---

## 1) Quick Start — Build & Run

```bash
# Build the image from the Dockerfile in the current folder
docker build -t myapp:v1.0.0 .

# Run the container in the background, map port 3000, load env vars from .env
docker run -d --name myapp   -p 3000:3000   --env-file .env   -v myapp_data:/app/storage   --restart unless-stopped   myapp:v1.0.0

# Tail logs
docker logs -f myapp

# Health check (if your app exposes /health)
curl http://localhost:3000/health
```

---

## 2) Tag & Push to Docker Hub (Optional)

```bash
# Tag for your Docker Hub repo
docker tag myapp:v1.0.0 your-docker-id/myapp:v1.0.0
docker tag myapp:v1.0.0 your-docker-id/myapp:latest

# Login & push
docker login -u your-docker-id
docker push your-docker-id/myapp:v1.0.0
docker push your-docker-id/myapp:latest
```

> To pull on a server:
```bash
docker pull your-docker-id/myapp:latest
```

---

## 3) Stop / Remove / Clean Up

```bash
# Stop then remove the container
docker stop myapp && docker rm myapp

# Remove an image (careful)
docker rmi myapp:v1.0.0

# Disk usage & prune
docker system df
docker system prune -af  # deletes unused containers/images/networks/build cache
```

---

## 4) Production-ready Dockerfile (Multi-stage, Non-root, Healthcheck, Volume)

Create `Dockerfile`:

```dockerfile
# ---------- build stage ----------
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build && npm prune --omit=dev

# ---------- runtime stage ----------
FROM node:20-alpine
# create non-root user for security
RUN addgroup -S app && adduser -S app -G app
WORKDIR /app
ENV NODE_ENV=production

# copy only what we need
COPY --from=build /app/node_modules ./node_modules
COPY --from=build /app/dist ./dist

# persistent data directory (can be mounted)
VOLUME ["/app/storage"]

# healthcheck (expects your app to expose GET /health returning 200)
HEALTHCHECK --interval=30s --timeout=3s --start-period=10s   CMD wget -qO- http://127.0.0.1:3000/health || exit 1

EXPOSE 3000
USER app
CMD ["node","dist/server.js"]
```

Expose a health endpoint in your app, e.g. Express:

```ts
app.get('/health', (_req, res) => res.status(200).json({ ok: true }));
```

---

## 5) .dockerignore (Reduce image size & speed up builds)

Create `.dockerignore`:

```
node_modules
.git
dist/*.map
Dockerfile*
.dockerignore
.env
*.log
coverage
```

---

## 6) docker-compose (API + Redis + MySQL)

Create `docker-compose.yml`:

```yaml
version: "3.9"
services:
  api:
    image: your-docker-id/myapp:v1.0.0  # or use: build: .
    container_name: myapp
    ports:
      - "3000:3000"
    env_file: .env
    depends_on:
      - redis
      - db
    volumes:
      - myapp_data:/app/storage
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    container_name: redis
    ports:
      - "6379:6379"
    command: ["redis-server", "--appendonly", "yes"]
    volumes:
      - redis_data:/data
    restart: unless-stopped

  db:
    image: mysql:8.0
    container_name: mysql
    environment:
      MYSQL_ROOT_PASSWORD: yourpass
      MYSQL_DATABASE: app
    ports:
      - "3306:3306"
    command: ["--default-authentication-plugin=mysql_native_password"]
    volumes:
      - mysql_data:/var/lib/mysql
    restart: unless-stopped

volumes:
  myapp_data:
  redis_data:
  mysql_data:
```

Usage:

```bash
docker compose up -d          # start all services
docker compose logs -f api    # tail API logs
docker compose down           # stop & remove containers (volumes are kept)
```

---

## 7) Dev Mode (Bind Mount for hot-reload)

```bash
docker run -it --rm   -p 3000:3000   -v $(pwd):/app   -w /app   node:20-alpine sh

# inside the container:
npm i
npm run dev   # e.g. nodemon / ts-node-dev
```

---

## 8) Tips

- Prefer **multi-stage** builds so runtime images are small & secure.
- Run as a **non-root user** in production.
- Keep secrets out of the image; pass via **env vars** or an **env file**.
- Enable `--restart unless-stopped` for services on single servers.
- Add **probes/healthchecks** so your orchestrator or monitoring can detect failures.
- Use **volumes** for persistent data (`/app/storage` example above).

---

Happy shipping! 🚀
