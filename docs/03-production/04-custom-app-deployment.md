# Custom App Deployment for Production

This guide walks you through deploying your custom Frappe application to production using Docker. Custom apps must be **built into the Docker image** before deployment - they cannot be installed into running containers due to Docker's immutable nature.

## Overview

To deploy a custom app in production, you need to:

1. Define your custom app in `apps.json`
2. Build a custom Docker image with your app included
3. Configure environment variables to use your custom image
4. Deploy using docker compose
5. Create sites and install your app

## Prerequisites

- git
- docker or podman
- docker compose v2 or podman compose
- Your custom app repository (e.g., on GitHub, GitLab, or Bitbucket)
- Production server or environment ready

## Step 1: Define Custom Apps

Create an `apps.json` file in the repository root to specify which apps to include in your image:

```json
[
  {
    "url": "https://github.com/frappe/erpnext",
    "branch": "version-15"
  },
  {
    "url": "https://github.com/yourusername/your-custom-app",
    "branch": "main"
  }
]
```

### apps.json Structure

Each app entry requires:

- `url`: Git repository URL for the app
- `branch`: Branch name to use

**Note:** Include all apps you need - both standard apps (like ERPNext) and your custom apps. The Frappe framework itself is always included automatically.

### Private Repositories

If your custom app is in a private repository, you have two options:

**Option 1: Use SSH keys**

```json
[
  {
    "url": "git@github.com:yourusername/your-custom-app.git",
    "branch": "main"
  }
]
```

Then pass your SSH key during build:

```bash
docker build \
  --secret id=ssh,src=$HOME/.ssh/id_rsa \
  --build-arg=APPS_JSON_BASE64=$APPS_JSON_BASE64 \
  --tag=custom:latest \
  --file=images/layered/Containerfile .
```

**Option 2: Use Personal Access Token**

Include the token in the URL (not recommended for production - use secrets management):

```json
[
  {
    "url": "https://username:token@github.com/yourusername/your-custom-app.git",
    "branch": "main"
  }
]
```

## Step 2: Build Custom Docker Image

### Generate Base64 Encoded apps.json

Convert your `apps.json` to base64:

```bash
export APPS_JSON_BASE64=$(base64 -w 0 apps.json)
```

### Choose Image Type

Use the **layered** image for production (faster builds, uses pre-built base images):

```bash
docker build \
  --build-arg=FRAPPE_PATH=https://github.com/frappe/frappe \
  --build-arg=FRAPPE_BRANCH=version-15 \
  --build-arg=APPS_JSON_BASE64=$APPS_JSON_BASE64 \
  --tag=mycompany/custom-app:1.0 \
  --file=images/layered/Containerfile .
```

**Important Build Args:**

| Arg                | Purpose                                    | Example                                  |
| ------------------ | ------------------------------------------ | ---------------------------------------- |
| FRAPPE_PATH        | Frappe framework repository URL            | `https://github.com/frappe/frappe`       |
| FRAPPE_BRANCH      | Frappe framework version                   | `version-15`                             |
| APPS_JSON_BASE64   | Base64-encoded apps.json                   | `$(base64 -w 0 apps.json)`               |
| PYTHON_VERSION     | Python version (optional)                  | `3.11`                                   |
| NODE_VERSION       | Node.js version (optional)                 | `18.19.0`                                |

### Verify Image Build

Check that your image was created successfully:

```bash
docker images | grep custom-app
```

## Step 3: Configure Environment

Create a custom environment file:

```bash
cp example.env production.env
```

Edit `production.env` and set these **required** variables:

```env
# Image Configuration
CUSTOM_IMAGE=mycompany/custom-app
CUSTOM_TAG=1.0
PULL_POLICY=missing

# Database Configuration
DB_PASSWORD=your-secure-db-password
DB_HOST=mariadb-database
DB_PORT=3306

# Site Configuration
SITES_RULE=Host(`yourdomain.com`)

# Letsencrypt (for HTTPS)
LETSENCRYPT_EMAIL=admin@yourdomain.com
```

**Critical Variables Explained:**

- `CUSTOM_IMAGE`: Name of your custom Docker image
- `CUSTOM_TAG`: Version tag of your image
- `PULL_POLICY=missing`: Prevents Docker from trying to pull the locally-built image
- `DB_PASSWORD`: Secure password for MariaDB
- `SITES_RULE`: Domain routing rule for Traefik proxy
- `LETSENCRYPT_EMAIL`: Email for SSL certificate notifications

See [env-variables.md](../02-setup/04-env-variables.md) for complete list of available variables.

## Step 4: Deploy with Docker Compose

### Production with HTTPS (Recommended)

Generate the final compose file with all necessary services:

```bash
docker compose --env-file production.env \
  -f compose.yaml \
  -f overrides/compose.mariadb.yaml \
  -f overrides/compose.redis.yaml \
  -f overrides/compose.https.yaml \
  config > docker-compose.production.yaml
```

Deploy:

```bash
docker compose --project-name production \
  -f docker-compose.production.yaml \
  up -d
```

### Alternative: HTTP Only (Testing/Internal)

For testing or internal networks without SSL:

```bash
docker compose --env-file production.env \
  -f compose.yaml \
  -f overrides/compose.mariadb.yaml \
  -f overrides/compose.redis.yaml \
  -f overrides/compose.noproxy.yaml \
  config > docker-compose.production.yaml
```

This exposes the application on port 8080.

## Step 5: Create Site and Install Custom App

Wait for all containers to start:

```bash
docker compose --project-name production \
  -f docker-compose.production.yaml \
  ps
```

### Create New Site

Create a new site with your custom app:

```bash
docker compose --project-name production \
  -f docker-compose.production.yaml \
  exec backend bench new-site yourdomain.com \
    --mariadb-user-host-login-scope=% \
    --admin-password your-admin-password \
    --db-root-password your-secure-db-password \
    --install-app your_custom_app
```

**Replace:**
- `yourdomain.com`: Your actual domain name
- `your-admin-password`: Administrator password for the site
- `your-secure-db-password`: Database root password (same as in .env)
- `your_custom_app`: Name of your custom app (as defined in your app's `hooks.py`)

### Install App to Existing Site

If you already have a site and want to install your custom app:

```bash
docker compose --project-name production \
  -f docker-compose.production.yaml \
  exec backend bench --site yourdomain.com install-app your_custom_app
```

## Step 6: Verify Deployment

### Check Site Status

```bash
docker compose --project-name production \
  -f docker-compose.production.yaml \
  exec backend bench --site yourdomain.com list-apps
```

### Check Logs

```bash
# All services
docker compose --project-name production -f docker-compose.production.yaml logs

# Specific service
docker compose --project-name production -f docker-compose.production.yaml logs backend

# Follow logs in real-time
docker compose --project-name production -f docker-compose.production.yaml logs -f
```

### Access Your Site

Open your browser and navigate to:
- HTTPS setup: `https://yourdomain.com`
- HTTP setup: `http://your-server-ip:8080`

Login with:
- Username: `Administrator`
- Password: The admin password you set during site creation

## Updating Your Custom App

When you make changes to your custom app, follow these steps:

### 1. Update Your App Code

Commit and push changes to your app repository.

### 2. Rebuild Docker Image

```bash
# Update apps.json if needed (e.g., new branch or version)
export APPS_JSON_BASE64=$(base64 -w 0 apps.json)

# Rebuild with new tag
docker build \
  --build-arg=FRAPPE_PATH=https://github.com/frappe/frappe \
  --build-arg=FRAPPE_BRANCH=version-15 \
  --build-arg=APPS_JSON_BASE64=$APPS_JSON_BASE64 \
  --tag=mycompany/custom-app:1.1 \
  --file=images/layered/Containerfile .
```

### 3. Update Environment

Edit `production.env`:

```env
CUSTOM_TAG=1.1
```

### 4. Regenerate Compose File

```bash
docker compose --env-file production.env \
  -f compose.yaml \
  -f overrides/compose.mariadb.yaml \
  -f overrides/compose.redis.yaml \
  -f overrides/compose.https.yaml \
  config > docker-compose.production.yaml
```

### 5. Update Services

```bash
# Stop current containers
docker compose --project-name production -f docker-compose.production.yaml down

# Start with new image
docker compose --project-name production -f docker-compose.production.yaml up -d
```

### 6. Migrate Site

Run migrations to update database schema:

```bash
docker compose --project-name production \
  -f docker-compose.production.yaml \
  exec backend bench --site yourdomain.com migrate
```

## Production Best Practices

### Image Management

1. **Version Tagging**: Use semantic versioning for your images (e.g., `1.0.0`, `1.1.0`)
2. **Image Registry**: Push images to a registry (Docker Hub, GitLab Registry, AWS ECR):
   ```bash
   docker push mycompany/custom-app:1.0
   ```
3. **Automated Builds**: Set up CI/CD to build images automatically

### Security

1. **Secrets Management**: Never commit passwords or tokens to git
2. **Use Environment Files**: Keep sensitive data in `.env` files (add to `.gitignore`)
3. **Regular Updates**: Keep Frappe and dependencies updated
4. **Backups**: Implement regular database and file backups (see [backup-strategy.md](02-backup-strategy.md))

### Monitoring

1. **Health Checks**: Monitor container health
   ```bash
   docker compose --project-name production -f docker-compose.production.yaml ps
   ```
2. **Logs**: Set up log aggregation for production
3. **Alerts**: Configure alerts for container failures

### Performance

1. **Resource Limits**: Set memory and CPU limits in compose files
2. **Worker Scaling**: Adjust the number of background workers based on load
3. **Database Tuning**: Optimize MariaDB configuration for production workloads

## Troubleshooting

### Image Build Fails

**Problem**: Build fails when fetching your custom app

**Solution**: 
- Verify repository URL and branch name in `apps.json`
- Check network connectivity
- For private repos, ensure SSH keys or tokens are configured correctly

### Site Creation Fails

**Problem**: `bench new-site` command fails

**Solution**:
- Check database connectivity: Verify `DB_HOST`, `DB_PORT`, and `DB_PASSWORD`
- Ensure MariaDB container is running: `docker compose ps`
- Check logs: `docker compose logs mariadb-database`

### App Not Listed

**Problem**: Custom app doesn't appear when running `list-apps`

**Solution**:
- Verify app was included in image: `docker compose exec backend ls apps/`
- Check app name in `hooks.py` matches what you're trying to install
- Rebuild image if app wasn't included

### Cannot Access Site

**Problem**: Site is not accessible from browser

**Solution**:
- For HTTPS: Verify DNS points to your server
- Check `SITES_RULE` matches your domain
- Verify Traefik is running: `docker compose ps`
- Check Traefik logs: `docker compose logs frontend`

## Complete Example Workflow

Here's a complete example deploying a custom app called "my_inventory_app":

```bash
# 1. Clone frappe_docker
git clone https://github.com/frappe/frappe_docker
cd frappe_docker

# 2. Create apps.json
cat > apps.json << EOF
[
  {
    "url": "https://github.com/frappe/erpnext",
    "branch": "version-15"
  },
  {
    "url": "https://github.com/mycompany/my_inventory_app",
    "branch": "production"
  }
]
EOF

# 3. Build image
export APPS_JSON_BASE64=$(base64 -w 0 apps.json)
docker build \
  --build-arg=FRAPPE_PATH=https://github.com/frappe/frappe \
  --build-arg=FRAPPE_BRANCH=version-15 \
  --build-arg=APPS_JSON_BASE64=$APPS_JSON_BASE64 \
  --tag=mycompany/inventory:1.0 \
  --file=images/layered/Containerfile .

# 4. Create environment file
cp example.env production.env
# Edit production.env with your values

# 5. Generate compose file
docker compose --env-file production.env \
  -f compose.yaml \
  -f overrides/compose.mariadb.yaml \
  -f overrides/compose.redis.yaml \
  -f overrides/compose.https.yaml \
  config > docker-compose.production.yaml

# 6. Deploy
docker compose --project-name inventory-prod \
  -f docker-compose.production.yaml up -d

# 7. Create site
docker compose --project-name inventory-prod \
  -f docker-compose.production.yaml \
  exec backend bench new-site inventory.mycompany.com \
    --mariadb-user-host-login-scope=% \
    --admin-password 'StrongPassword123!' \
    --db-root-password 'DbRootPassword456!' \
    --install-app erpnext \
    --install-app my_inventory_app

# 8. Verify
docker compose --project-name inventory-prod \
  -f docker-compose.production.yaml \
  exec backend bench --site inventory.mycompany.com list-apps
```

## Additional Resources

- [Container Setup Overview](../02-setup/01-overview.md) - Understanding Docker image types
- [Build Setup](../02-setup/02-build-setup.md) - Detailed build instructions
- [Environment Variables](../02-setup/04-env-variables.md) - Complete variable reference
- [Site Operations](../04-operations/01-site-operations.md) - Managing sites
- [Backup Strategy](02-backup-strategy.md) - Backup and restore
- [Multi-tenancy](03-multi-tenancy.md) - Running multiple sites
- [Single Server Example](../02-setup/07-single-server-example.md) - Complete production setup

---

**Back:** [Multi-tenancy ←](03-multi-tenancy.md)
