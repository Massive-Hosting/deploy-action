# Deploy Action

GitHub Action for deploying code to your hosting account via SSH + rsync.

## Usage

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
          website: your-website-id
```

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `ssh-key` | Yes | - | SSH private key for authentication |
| `host` | Yes | - | SSH hostname (e.g. `tenant-id.example.com`) |
| `website` | Yes | - | Website directory name under `~/sites/` |
| `source` | No | `.` | Local directory to deploy |
| `exclude` | No | `.git .github node_modules` | Space-separated rsync exclude patterns |
| `run-after` | No | - | Post-deploy commands to run via SSH |
| `build` | No | - | Build command to run locally before deploy |

## Examples

### Laravel

```yaml
- uses: massive-hosting/deploy-action@v1
  with:
    ssh-key: ${{ secrets.SSH_PRIVATE_KEY }}
    host: your-tenant.example.com
    website: your-website-id
    exclude: .git .github node_modules tests
    run-after: |
      cd ~/sites/your-website-id && composer install --no-dev --optimize-autoloader
      php artisan migrate --force
      php artisan config:cache
      php artisan route:cache
      php artisan view:cache
```

### Node.js with build step

```yaml
- uses: massive-hosting/deploy-action@v1
  with:
    ssh-key: ${{ secrets.SSH_PRIVATE_KEY }}
    host: your-tenant.example.com
    website: your-website-id
    build: npm ci && npm run build
    exclude: .git .github node_modules
    run-after: |
      cd ~/sites/your-website-id && npm ci --production
```

### Django

```yaml
- uses: massive-hosting/deploy-action@v1
  with:
    ssh-key: ${{ secrets.SSH_PRIVATE_KEY }}
    host: your-tenant.example.com
    website: your-website-id
    exclude: .git .github __pycache__ .venv
    run-after: |
      cd ~/sites/your-website-id && pip install -r requirements.txt
      python manage.py migrate
      python manage.py collectstatic --noinput
```

## Setup

1. Generate an SSH key pair: `ssh-keygen -t ed25519 -f deploy_key`
2. Add the public key to your hosting account (SSH Keys section in the control panel)
3. Add the private key as a GitHub repository secret named `SSH_PRIVATE_KEY`
4. Copy a workflow template from the Deploy tab in your website's control panel
