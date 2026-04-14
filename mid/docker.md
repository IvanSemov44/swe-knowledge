# Docker

<!-- last-reviewed: 2026-04-09 | next-review: 2026-05-09 | confidence: b -->

---

## What It Is

Docker packages your app and everything it needs (runtime, config, dependencies) into a **container** — a portable, isolated unit that runs the same everywhere.

---

## Core Concepts

| Concept | What it is |
|---|---|
| **Image** | Blueprint. Read-only snapshot of your app + environment |
| **Container** | Running instance of an image |
| **Dockerfile** | Instructions to build an image |
| **docker-compose** | Tool to define and run multi-container apps |
| **Volume** | Persistent storage that survives container restarts |
| **Network** | How containers communicate with each other |

---

## Dockerfile (.NET 8 API)

```dockerfile
# Build stage
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY . .
RUN dotnet publish -c Release -o /app/publish

# Runtime stage
FROM mcr.microsoft.com/dotnet/aspnet:8.0
WORKDIR /app
COPY --from=build /app/publish .
ENTRYPOINT ["dotnet", "MyApi.dll"]
```

**Multi-stage build:** SDK image (large) only used to compile. Final image uses the smaller ASP.NET runtime image. Keeps image size small.

---

## docker-compose (.NET API + SQL Server)

```yaml
services:

  sqlserver:
    image: mcr.microsoft.com/mssql/server:2022-latest
    environment:
      SA_PASSWORD: "YourStrong!Passw0rd"
      ACCEPT_EULA: "Y"
    ports:
      - "1433:1433"
    volumes:
      - sqldata:/var/opt/mssql

  api:
    build: .
    environment:
      ASPNETCORE_ENVIRONMENT: Development
      ConnectionStrings__Default: "Server=sqlserver;Database=MyDb;User Id=sa;Password=YourStrong!Passw0rd;"
    ports:
      - "5000:8080"
    depends_on:
      - sqlserver

volumes:
  sqldata:
```

### Critical rule: service name = hostname

Containers reach each other by **service name**. If `sqlserver` is renamed to `db`, the connection string must use `Server=db`. No IP addresses needed — docker-compose creates a default network automatically.

### depends_on caveat

`depends_on` only waits for the container to **start**, not for SQL Server to be **ready**. SQL Server takes a few seconds to initialize. In production, add a healthcheck. In dev, it's acceptable as-is.

---

## Ports: host:container

```yaml
ports:
  - "5000:8080"   # traffic on host:5000 → container:8080
```

Left = your machine. Right = inside the container.

---

## Volumes

```yaml
volumes:
  - sqldata:/var/opt/mssql   # named volume — data persists across restarts
  - ./logs:/app/logs          # bind mount — maps host folder into container
```

Without a volume, all data inside a container is wiped on restart.

---

## Useful Commands

```bash
docker-compose up -d          # start all services in background
docker-compose down           # stop and remove containers
docker-compose down -v        # stop + delete volumes (wipes db data)
docker-compose logs -f api    # tail logs for a specific service
docker ps                     # list running containers
docker exec -it <id> bash     # shell into a running container
```

---

## Gaps to Fill Next

- [ ] Networking deep dive (bridge, host, overlay)
- [ ] Container registries (Docker Hub, GitHub Container Registry, ACR)
- [ ] Multi-stage build optimizations
- [ ] Healthchecks (depends_on + condition: service_healthy)
- [ ] Docker in CI (GitHub Actions build + push)

---

## Connected To

- `senior/ci-cd.md` — Docker build step in GitHub Actions pipeline
- `my-stack/aspnet.md` — ASPNETCORE_ENVIRONMENT, configuration in containers
