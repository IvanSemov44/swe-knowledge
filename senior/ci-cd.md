# CI/CD

<!-- last-reviewed: 2026-04-09 | next-review: 2026-05-09 | confidence: b -->

---

## What CI/CD Is

**CI (Continuous Integration):** Every push runs automated checks — lint, type check, tests, build. Catches problems before they reach production.

**CD (Continuous Delivery/Deployment):** After CI passes, the artifact is automatically deployed to an environment.

---

## Pipeline Order (Why It Matters)

Run cheapest checks first. Fail fast.

```
push to main
    ↓
1. Lint + format check     (ESLint, Prettier)  — fastest, catch style
    ↓
2. Type check              (tsc --noEmit)       — catch type errors
    ↓
3. Tests                   (unit + integration) — catch logic errors
    ↓
4. Build                   (dotnet publish / npm run build) — produce artifact
    ↓
5. Docker build + push     (tag with commit SHA)
    ↓
6. Deploy                  (pull image, restart containers)
```

No point running tests on code that doesn't compile. No point building code that fails tests.

---

## GitHub Actions — Basic Workflow

```yaml
name: CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  ci:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '8.0'

      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install frontend deps
        run: npm ci
        working-directory: ./frontend

      - name: Lint
        run: npm run lint
        working-directory: ./frontend

      - name: Type check
        run: npx tsc --noEmit
        working-directory: ./frontend

      - name: Run backend tests
        run: dotnet test --no-build

      - name: Build frontend
        run: npm run build
        working-directory: ./frontend

      - name: Build Docker image
        run: docker build -t ghcr.io/${{ github.repository }}/api:${{ github.sha }} .

      - name: Push to registry
        run: |
          echo ${{ secrets.GITHUB_TOKEN }} | docker login ghcr.io -u ${{ github.actor }} --password-stdin
          docker push ghcr.io/${{ github.repository }}/api:${{ github.sha }}

  deploy:
    needs: ci
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'

    steps:
      - name: Deploy to server
        run: |
          ssh user@${{ secrets.SERVER_IP }} "
            docker pull ghcr.io/${{ github.repository }}/api:${{ github.sha }} &&
            docker-compose up -d
          "
```

---

## Key Concepts

### Commit SHA as image tag

```yaml
docker build -t my-api:${{ github.sha }} .
```

Every image is tagged with the exact commit that built it. You can always roll back to a previous commit SHA. Never tag with `latest` in production — you lose traceability.

### Secrets

Never hardcode passwords, tokens, or keys in workflow files. Store them in **GitHub Secrets** (repo Settings → Secrets) and reference via `${{ secrets.MY_SECRET }}`.

```yaml
environment:
  SA_PASSWORD: ${{ secrets.DB_PASSWORD }}
```

### jobs vs steps

- **job:** runs on a fresh VM (ubuntu-latest). Jobs run in parallel by default.
- **step:** runs inside a job, sequentially. Share the same filesystem.
- `needs: ci` — makes deploy job wait for ci job to pass.

### `on: pull_request`

Run CI on PRs too — not just on push to main. Catch failures before merge, not after.

---

## Pre-commit Hooks vs CI

| | Pre-commit hook | CI pipeline |
|---|---|---|
| When | Before local commit | After push |
| Purpose | Fast feedback for developer | Enforce for everyone |
| Can be skipped | Yes (`--no-verify`) | No |

Run both. Pre-commit hooks are a convenience. CI is the gate.

---

## Deployment Strategies

### Basic (what most small projects use)
Pull new Docker image, restart containers with docker-compose.

```bash
docker pull my-api:new-sha
docker-compose up -d
```

Brief downtime during restart.

### Blue/Green
Run two identical environments (blue = current, green = new). Switch traffic to green after health check passes. Instant rollback = switch back to blue.

```
Load Balancer → Blue (v1)
              → Green (v2) ← new deploy goes here first
```

### Rolling
Update instances one at a time. No downtime but briefly runs two versions simultaneously.

---

## Gaps to Fill Next

- [ ] Caching dependencies in GitHub Actions (actions/cache)
- [ ] Matrix builds (test on multiple Node/dotnet versions)
- [ ] Environment-specific deployments (staging vs production)
- [ ] Health checks before marking deploy successful
- [ ] Rollback strategy

---

## Connected To

- `mid/docker.md` — Docker image build is the core CI artifact
- `mid/docker.md` — Container registries (GHCR, Docker Hub, ACR)
