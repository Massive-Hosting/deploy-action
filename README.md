# Deploy Action

A GitHub Action for deploying code to [Massive Hosting](https://massive-hosting.com) via SSH + rsync. Works with any webapp runtime (PHP, Node.js, Python, static sites) and supports optional build steps and post-deploy commands.

## How It Works

```
GitHub Runner ──SSH+rsync──▶ Tenant Node ──▶ ~/apps/{webapp}/
```

1. Installs your SSH private key for the session
2. Optionally runs a local build command (e.g. `npm run build`)
3. Syncs files to your webapp directory via `rsync --delete`
4. Optionally runs post-deploy commands on the server (e.g. migrations, cache clearing)
5. Cleans up the SSH key

## Quick Start

```yaml
name: Deploy
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: massive-hosting/deploy-action@v1
        with:
          ssh-key: ${{ secrets.SSH_PRIVATE_KEY }}
          host: your-tenant.example.com
          webapp: your-webapp-id
```

## Setup

### 1. Generate an SSH Key

```bash
ssh-keygen -t ed25519 -C "deploy" -f deploy_key -N ""
```

### 2. Add the Public Key to Your Hosting Account

Go to your control panel → SSH Keys → Add the contents of `deploy_key.pub`.

### 3. Add the Private Key as a GitHub Secret

Go to your repository → Settings → Secrets and variables → Actions → New repository secret:
- **Name**: `SSH_PRIVATE_KEY`
- **Value**: Contents of `deploy_key` (the private key)

### 4. Find Your SSH Host and Webapp ID

In the control panel, go to your webapp → Deploy tab. The templates there have your host and webapp ID pre-filled.

Your SSH host follows the pattern `{tenant-id}.{base-hostname}`, e.g. `abc123.ssh.example.com`.

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `ssh-key` | **Yes** | — | SSH private key for authentication |
| `host` | **Yes** | — | SSH hostname (e.g. `tenant-id.example.com`) |
| `webapp` | **Yes** | — | Webapp directory name under `~/apps/` |
| `source` | No | `.` | Local directory to deploy |
| `exclude` | No | `.git .github node_modules` | Space-separated rsync exclude patterns |
| `build` | No | — | Build command to run locally before deploying |
| `run-after` | No | — | Commands to run on the server after deploy |

## Examples

### PHP / Laravel

```yaml
name: Deploy
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: massive-hosting/deploy-action@v1
        with:
          ssh-key: ${{ secrets.SSH_PRIVATE_KEY }}
          host: tenant-id.ssh.example.com
          webapp: my-laravel-app
          exclude: .git .github node_modules tests
          run-after: |
            cd ~/apps/my-laravel-app
            composer install --no-dev --optimize-autoloader
            php artisan migrate --force
            php artisan config:cache
            php artisan route:cache
            php artisan view:cache
```

### Node.js

```yaml
name: Deploy
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: massive-hosting/deploy-action@v1
        with:
          ssh-key: ${{ secrets.SSH_PRIVATE_KEY }}
          host: tenant-id.ssh.example.com
          webapp: my-node-app
          build: npm ci && npm run build
          exclude: .git .github node_modules
          run-after: |
            cd ~/apps/my-node-app && npm ci --production
```

### Next.js (Standalone)

```yaml
name: Deploy
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
      - run: npm ci && npm run build
      - uses: massive-hosting/deploy-action@v1
        with:
          ssh-key: ${{ secrets.SSH_PRIVATE_KEY }}
          host: tenant-id.ssh.example.com
          webapp: my-nextjs-app
          source: .next/standalone
          run-after: |
            cp -r .next/static ~/apps/my-nextjs-app/.next/static
```

### Django / Python

```yaml
name: Deploy
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: massive-hosting/deploy-action@v1
        with:
          ssh-key: ${{ secrets.SSH_PRIVATE_KEY }}
          host: tenant-id.ssh.example.com
          webapp: my-django-app
          exclude: .git .github __pycache__ .venv
          run-after: |
            cd ~/apps/my-django-app
            pip install -r requirements.txt
            python manage.py migrate
            python manage.py collectstatic --noinput
```

### WordPress

```yaml
name: Deploy
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: massive-hosting/deploy-action@v1
        with:
          ssh-key: ${{ secrets.SSH_PRIVATE_KEY }}
          host: tenant-id.ssh.example.com
          webapp: my-wordpress-site
          exclude: .git .github wp-content/uploads
```

### Static Site with Build

```yaml
name: Deploy
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: massive-hosting/deploy-action@v1
        with:
          ssh-key: ${{ secrets.SSH_PRIVATE_KEY }}
          host: tenant-id.ssh.example.com
          webapp: my-static-site
          build: npm ci && npm run build
          source: dist
```

### Deploy Only on Tags

```yaml
name: Deploy Release
on:
  push:
    tags: ['v*']

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: massive-hosting/deploy-action@v1
        with:
          ssh-key: ${{ secrets.SSH_PRIVATE_KEY }}
          host: tenant-id.ssh.example.com
          webapp: my-app
```

### Staging + Production

```yaml
name: Deploy
on:
  push:
    branches: [main, develop]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: massive-hosting/deploy-action@v1
        with:
          ssh-key: ${{ secrets.SSH_PRIVATE_KEY }}
          host: ${{ github.ref == 'refs/heads/main' && 'prod-tenant.ssh.example.com' || 'staging-tenant.ssh.example.com' }}
          webapp: ${{ github.ref == 'refs/heads/main' && 'prod-app' || 'staging-app' }}
```

## How rsync Works

The action uses `rsync -azv --delete` which means:

- **`-a`** — Archive mode (preserves permissions, timestamps, symlinks)
- **`-z`** — Compress data during transfer
- **`-v`** — Verbose output (visible in workflow logs)
- **`--delete`** — Remove files on the server that don't exist locally

The `--delete` flag ensures your server exactly mirrors your repository. Files listed in `exclude` are not deleted or uploaded.

## Troubleshooting

### "Permission denied (publickey)"

- Verify the SSH key is added to your hosting account (control panel → SSH Keys)
- Check that `SSH_PRIVATE_KEY` secret contains the full private key including `-----BEGIN` and `-----END` lines
- Make sure SSH access is enabled for your tenant

### "rsync: connection unexpectedly closed"

- Check that the `host` input matches your tenant's SSH hostname
- Verify your tenant is active and not suspended

### Files not updating

- Remember `rsync --delete` only affects the webapp directory (`~/apps/{webapp}/`)
- Excluded patterns are neither uploaded nor deleted — if you previously deployed a file and then add it to excludes, it remains on the server
- Check the rsync output in workflow logs for details

### Post-deploy commands failing

- Commands in `run-after` execute via SSH on the server, not locally
- Working directory is `~` (home), not the webapp directory — always `cd` first
- Use `set -e` in run-after if you want the step to fail on any error

## License

MIT
