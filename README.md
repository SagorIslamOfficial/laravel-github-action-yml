## Deploy Laravel App to VPS using GitHub Actions

This repository contains a GitHub Actions workflow file `deploy.yml` that builds a Laravel application and deploys it to a VPS via SSH.

This README explains what the workflow does, which secrets it needs, how to use it, how to test it safely, and common troubleshooting steps.

## What this workflow does

- Trigger: runs on push to the `main` branch (see `on.push.branches` in `deploy.yml`).
- Builds PHP and Node.js dependencies and assets:
  - Checks out the repository
  - Sets up PHP (specified version in the workflow)
  - Installs Composer dependencies
  - Sets up Node.js and builds frontend assets (npm ci, npm run build)
- Prepares the target directory on the VPS (creates `/var/www/html`, sets ownership and permissions)
- Syncs repository files to the VPS (excludes `node_modules`, `.git`, `storage`, `bootstrap/cache`)
- Runs server-side deployment tasks over SSH on the VPS:
  - Copies `.env.example` to `.env` if missing and injects secrets (APP*URL, DB*\* values)
  - Generates app key
  - Creates storage directories
  - Runs `php artisan migrate --force` and caches (config, route, view)
  - Optimizes and creates symbolic links for storage
  - Adjusts ownership/permissions for web server user and reloads nginx

> Note: The workflow runs migrations (`--force`) on the server. Ensure you have backups and are comfortable with automated migrations.

## Required GitHub Secrets

Create these secrets in your repository (Settings → Secrets & variables → Actions):

- `VPS_HOST` — the server IP or hostname
- `VPS_USER` — SSH username used to connect (e.g., `deploy` or `ubuntu`)
- `SSH_PRIVATE_KEY` — private key (PEM) for `VPS_USER` with passwordless access from GitHub Actions
- `APP_URL` — your application URL (used to set `APP_URL` in `.env` when first creating it)
- `DB_DATABASE` — database name to write into `.env` when first creating it
- `DB_USERNAME` — database username
- `DB_PASSWORD` — database password

Optional (not currently in the workflow but helpful):

- `SSH_PORT` — if your server uses a non-default SSH port (you'd need to update the action to use it)

Security note: keep the private key and database credentials secret and use a deploy-only user with limited privileges.

## How to use

1. Add the required secrets to your repository (see above).
2. Confirm the workflow file `deploy.yml` is in `.github/workflows/` (in this repo it is at the repo root; move it if you use standard placement).
3. Ensure the `VPS_USER` on the server has permissions to write to `/var/www/html` or that `sudo` is available for necessary commands (the workflow uses `sudo` for some operations).
4. Push to the `main` branch to trigger deployment.

If you prefer to trigger manually during testing, you can temporarily change the `on` section in `deploy.yml` to allow `workflow_dispatch` (manual trigger). Example snippet you can add for testing:

```yaml
on:
  workflow_dispatch:
  push:
    branches: [main]
```

Then trigger the workflow from the Actions tab.

## Step-by-step mapping to `deploy.yml`

Below is a high-level mapping of what each step in the workflow does and why it's there.

- Checkout code — clones your repo so Actions can build and deploy it.
- Setup PHP — installs the PHP runtime and common extensions used by Laravel.
- Composer cache/install — caches Composer files and installs dependencies without dev packages.
- Setup Node.js & Build — installs Node, runs `npm ci`, and builds frontend assets with `npm run build`.
- Prepare deployment directory — via SSH: creates `/var/www/html`, fixes ownership and permissions so files can be written by the deploy user.
- Sync files — uses `easingthemes/ssh-deploy` to rsync from Actions runner to your VPS target path. The action uses the provided private key and will delete remote files that no longer exist locally (because of `--delete`).
- Run deployment commands — via SSH: environment setup (`.env` creation), migrations, caching, `storage:link`, permission fixes, and nginx reload.

## Testing safely

- Do not run migrations automatically on production without backups. Consider setting up a staging environment first.
- Test the workflow against a staging server (use a different `VPS_HOST`/`VPS_USER` and secrets).
- To test without affecting DB, comment out or remove the `php artisan migrate --force` line temporarily.
- Use `workflow_dispatch` for manual control during initial testing (see snippet above).

## Troubleshooting

- SSH connection failure:
  - Ensure `SSH_PRIVATE_KEY` matches a public key present in `~/.ssh/authorized_keys` for `VPS_USER` on the server.
  - Confirm `VPS_HOST` and `VPS_USER` are correct.
- Permission denied when writing to `/var/www/html`:
  - Ensure the `VPS_USER` has write permissions, or adjust `sudo chown` and `chmod` commands on the server.
- Migrations fail:
  - Check the database credentials in the server `.env`.
  - Run migrations manually on the server to inspect the error before allowing the automated workflow to run them.
- rsync deletes files unexpectedly:
  - The `ssh-deploy` step uses `--delete`. Review the `EXCLUDE` variable in the workflow to ensure important directories are excluded from sync.

If logs are needed, open the Actions run in GitHub and inspect each step's output. The SSH actions print remote script stdout/stderr which helps diagnose issues.
