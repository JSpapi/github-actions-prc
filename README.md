# GitHub Actions Practice

React + TypeScript + Vite project for practicing GitHub Actions with Docker and Telegram notifications.

## Local Development

```bash
nvm use           # switches to Node 18.19.1 (reads .nvmrc)
npm install
npm run dev       # http://localhost:5173
```

## Docker

### Build & run locally

```bash
docker compose up --build        # builds image and starts on http://localhost:3000
docker compose up -d             # detached mode (background)
docker compose down              # stop and remove containers
```

### Rebuild after code changes

```bash
docker compose up --build -d     # rebuild + restart in background
```

### Just build the image (without compose)

```bash
docker build -t github-actions-prc .
docker run -p 3000:80 github-actions-prc
```

---

## How Docker Files Work Together

### `Dockerfile` — Multi-stage build

```
Stage 1 (build):   node:18-alpine → npm ci → npm run build → produces /app/dist
Stage 2 (production): nginx:alpine → copies /app/dist → serves on port 80
```

- **Why multi-stage?** The final image only contains nginx + static files (~25MB), not Node.js + node_modules (~500MB+)
- **Why nginx?** Production-grade static file server with SPA routing support

### `nginx.conf` — SPA routing

- `try_files $uri $uri/ /index.html` — all unknown routes serve `index.html` (React Router support)
- Static assets get 1-year cache headers for performance

### `docker-compose.yml` — Orchestration

- Maps container port `80` (nginx) → host port `3000`
- `restart: unless-stopped` — auto-restarts if the container crashes

### `.dockerignore` — Build optimization

- Excludes `node_modules`, `.git`, etc. from the Docker build context
- Makes builds faster and images cleaner

---

## GitHub Actions Practice Plan

You will create two workflow files in `.github/workflows/`:

### 1. `notifications.yml` — Telegram Notifications

**Triggers:** push (any branch), PR opened/closed

**What to do:**
1. Create a Telegram bot via `@BotFather` → save the bot token
2. Create a Telegram group → add the bot → get the chat ID
3. Store in GitHub repo → Settings → Secrets and variables → Actions:
   - `TELEGRAM_TOKEN` = bot token from BotFather
   - `TELEGRAM_ID` = chat ID (negative number for groups)
4. Create `.github/workflows/notifications.yml` that sends messages on push/PR events

**Getting the chat ID:**
- Add bot to your group
- Send a message in the group
- Visit: `https://api.telegram.org/bot<YOUR_TOKEN>/getUpdates`
- Find `"chat":{"id":-XXXXXXXXX}` — that negative number is your chat ID

**Key action:** `appleboy/telegram-action@master`

### 2. `publish.yml` — Docker Build + Telegram Deploy Status

**Triggers:** push to `main` and `dev` branches

**What to do:**
1. Build Docker image using your `Dockerfile`
2. (Optional) Push image to Docker Hub or GitHub Container Registry (ghcr.io)
3. Send Telegram notification on success/failure with deploy duration

**For Docker Hub:**
- Create account at hub.docker.com
- Store secrets: `DOCKERHUB_USERNAME`, `DOCKERHUB_TOKEN`
- Use `docker/login-action` + `docker/build-push-action`

**For GitHub Container Registry (free, no extra account needed):**
- Use `ghcr.io` as registry
- Login with `${{ github.actor }}` and `${{ secrets.GITHUB_TOKEN }}` (auto-provided)

**Example publish workflow structure:**
```yaml
name: Publish
on:
  push:
    branches: [main, dev]

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set start time
        run: echo "START_TIME=$(date +%s)" >> $GITHUB_ENV

      - name: Login to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ghcr.io/${{ github.repository }}:${{ github.ref_name }}

      - name: Calculate duration
        if: always()
        run: |
          END_TIME=$(date +%s)
          DURATION=$(( END_TIME - START_TIME ))
          MINUTES=$(( DURATION / 60 ))
          SECONDS=$(( DURATION % 60 ))
          echo "DEPLOY_DURATION=${MINUTES}m ${SECONDS}s" >> $GITHUB_ENV

      - name: Notify success
        if: success()
        uses: appleboy/telegram-action@master
        with:
          to: ${{ secrets.TELEGRAM_ID }}
          token: ${{ secrets.TELEGRAM_TOKEN }}
          format: html
          message: |
            ✅ <b>Deploy successful!</b>
            ⏱ Duration: ${{ env.DEPLOY_DURATION }}
            👤 <b>${{ github.actor }}</b> pushed to <b>${{ github.ref_name }}</b>
            📦 <b>${{ github.repository }}</b>

      - name: Notify failure
        if: failure()
        uses: appleboy/telegram-action@master
        with:
          to: ${{ secrets.TELEGRAM_ID }}
          token: ${{ secrets.TELEGRAM_TOKEN }}
          format: html
          message: |
            ❌ <b>Deploy failed!</b>
            👤 <b>${{ github.actor }}</b> pushed to <b>${{ github.ref_name }}</b>
            🔗 <a href="https://github.com/${{ github.repository }}/actions/runs/${{ github.run_id }}">View logs</a>
```

> **Note:** The example above uses GitHub Container Registry (ghcr.io) which is free and requires no extra account. `GITHUB_TOKEN` is automatically available in every workflow.

---

## Secrets Cheatsheet

| Secret | Where to get it | Used by |
|--------|----------------|---------|
| `TELEGRAM_TOKEN` | @BotFather on Telegram | notifications.yml, publish.yml |
| `TELEGRAM_ID` | Bot API `/getUpdates` | notifications.yml, publish.yml |
| `GITHUB_TOKEN` | Auto-provided by GitHub | publish.yml (ghcr.io login) |

Add secrets at: **Repo → Settings → Secrets and variables → Actions → New repository secret**
