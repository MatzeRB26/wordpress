# WordPress with Docker Compose

WordPress and MariaDB in Docker Compose, ready to run in a few steps.

## Table of Contents

- [Quickstart](#quickstart)
  - [Prerequisites](#prerequisites)
  - [Quickstart Guide](#quickstart-guide)
- [Usage](#usage)
  - [Configuration Overview](#configuration-overview)
  - [Secrets (`.env`)](#secrets-env)
  - [Non-Critical Settings](#non-critical-settings)
  - [Services](#services)
  - [Network](#network)
  - [Volumes and Persistence](#volumes-and-persistence)
  - [Restart Policy](#restart-policy)
  - [Common Commands](#common-commands)
- [Verification](#verification)
- [Security Notes](#security-notes)
- [Additional Information](#additional-information)
  - [Description](#description)
  - [Repository Contents](#repository-contents)
  - [Extras](#extras)

## Quickstart

### Prerequisites

- [Docker Engine](https://docs.docker.com/engine/install/) with the [Docker Compose plugin](https://docs.docker.com/compose/install/)
- Git with an [SSH key added to GitHub](https://docs.github.com/en/authentication/connecting-to-github-with-ssh)
- Port `8080` reachable from your browser

Works on a server (e.g. a cloud VM) or locally on `localhost`.

### Quickstart Guide

1. Clone the repository:

   ```bash
   git clone git@github.com:MatzeRB26/wordpress.git
   cd wordpress
   ```

2. Create your `.env` file from the template:

   ```bash
   cp .env.example .env
   ```

3. Replace all placeholders in `.env` with your own values:

   ```bash
   nano .env
   ```

4. Start the setup:

   ```bash
   docker compose up -d
   ```

Open `http://<your-server-ip>:8080` (or `http://localhost:8080`) and complete the WordPress installer to create your **admin user**.

## Usage

This section explains how the setup is configured and which parts you can change.

### Configuration Overview

| Layer | Where | Content |
|---|---|---|
| Credentials | `.env` (local only, git-ignored) | Database user and passwords |
| Non-critical settings | `docker-compose.yaml` (`environment` block) | Database host, database name, table prefix, port |

Each service only receives the variables listed explicitly in its `environment` block. The `.env` file is never passed into a container as a whole.

### Secrets (`.env`)

Copy `.env.example` to `.env` and replace every placeholder:

| Variable | Description |
|---|---|
| `MARIADB_ROOT_PASSWORD` | Password of the MariaDB `root` user |
| `MARIADB_USER` | Database user. WordPress connects with this user as well. |
| `MARIADB_PASSWORD` | Password of that database user |

Tip: generate strong random passwords and fill them in automatically:

```bash
cp .env.example .env
DB_PW=$(openssl rand -hex 20); ROOT_PW=$(openssl rand -hex 20)
sed -i "s/change-me-root-password/${ROOT_PW}/; s/change-me-db-password/${DB_PW}/; s/change-me-db-user/wpuser/" .env
grep -c change-me .env   # should print 0
```

> **Note:** MariaDB only reads these values when the database volume is created for the first time. If you change the credentials later, change them inside the database as well, or start over with a fresh volume (see [Volumes and Persistence](#volumes-and-persistence)).

### Non-Critical Settings

These variables are optional. Add any of them to your `.env` file to change the default:

| Variable | Default | Effect |
|---|---|---|
| `WORDPRESS_PORT` | `8080` | Host port WordPress is published on |
| `DB_NAME` | `wordpress` | Name of the database (used by both services) |
| `WORDPRESS_TABLE_PREFIX` | `wp_` | Prefix for all WordPress database tables |

Example: to run WordPress on port `9000` with a custom table prefix, add the following to `.env`:

```env
WORDPRESS_PORT=9000
WORDPRESS_TABLE_PREFIX=site1_
```

Then recreate the containers:

```bash
docker compose up -d
```

> Change `DB_NAME` and `WORDPRESS_TABLE_PREFIX` **before** the first start. Changing them afterwards makes WordPress look for a database or tables that do not exist, and the installer appears again.

To check the final configuration with all variables resolved:

```bash
docker compose config
```

The image versions are fixed in `docker-compose.yaml` (`mariadb:11.4`, `wordpress:php8.3-apache`). To upgrade, change the tag there.

### Services

| Service | Image | Purpose |
|---|---|---|
| `db` | `mariadb:11.4` | Stores all WordPress content. Not published to the host, only reachable inside the Docker network. |
| `wordpress` | `wordpress:php8.3-apache` | Serves the site on port `8080` and connects to the database via the hostname `db`. |

The `db` service has a **healthcheck**. Because of `depends_on: condition: service_healthy`, WordPress only starts once the database accepts connections.

### Network

Both services are attached to the bridge network `wordpress_net`. Docker Compose would put them in a shared network automatically; it is defined explicitly to make this visible. WordPress reaches the database at `db:3306`.

### Volumes and Persistence

| Volume | Mount point | Content |
|---|---|---|
| `db_data` | `/var/lib/mysql` | All database files |
| `wp_data` | `/var/www/html` | WordPress core, `wp-config.php`, themes, plugins, uploads |

Named volumes survive `docker compose down` and host reboots.

```bash
docker volume ls
```

> ⚠️ `docker compose down -v` **deletes the volumes and all data**. Only use it for a completely fresh installation.

### Restart Policy

Both services use `restart: unless-stopped`. Docker restarts a container automatically if it crashes and after a host reboot. Only containers you stop manually stay stopped.

### Common Commands

| Command | Effect |
|---|---|
| `docker compose up -d` | Start or apply configuration changes |
| `docker compose ps` | Show status (`db` should be `healthy`) |
| `docker compose logs -f` | Follow the logs |
| `docker compose restart` | Restart the services |
| `docker compose down` | Stop and remove containers (data is kept) |
| `docker compose pull` | Pull the newest images for the configured tags |

## Verification

The setup was tested on a cloud VM:

1. **Reachability:** WordPress is reachable at `http://<your-server-ip>:8080`.
2. **Login:** You can log in with the admin user created in the installer.
3. **Persistence:** After `docker compose down` and `docker compose up -d`, posts and settings are still there.
4. **Automatic restart:** After killing the container's main process, Docker restarts it:

   ```bash
   docker inspect -f '{{.RestartCount}}' wordpress-wordpress-1
   ```

   ```bash
   sudo kill -9 $(docker inspect -f '{{.State.Pid}}' wordpress-wordpress-1)
   ```

   ```bash
   docker compose ps
   docker inspect -f '{{.RestartCount}}' wordpress-wordpress-1   # increased by 1
   ```

   `docker stop` is not suitable for this test, because Docker does not restart a manually stopped container.

## Security Notes

- No passwords, tokens, usernames, IP addresses, or SSH keys are stored in this repository.
- Credentials live in the git-ignored `.env` file. Only the placeholder template `.env.example` is committed.
- Each container only receives the variables it needs. The `.env` file is not loaded into the containers as a whole.
- The database is not published to the host.
- Do not use `admin` as the WordPress username, and choose a strong password.

## Additional Information

### Description

This repository deploys WordPress together with a MariaDB database using Docker Compose. It focuses on **configuring** containers rather than building a custom image and uses the official images [`wordpress`](https://hub.docker.com/_/wordpress) and [`mariadb`](https://hub.docker.com/_/mariadb).

### Repository Contents

| File | Description |
|---|---|
| `docker-compose.yaml` | The `wordpress` and `db` services, the network, and the volumes |
| `.env.example` | Template for the `.env` file (placeholders only) |
| `.gitignore` | Excludes secrets, database dumps, logs, and editor/OS files |
| `README.md` | This documentation |
| [`CHECKLIST.md`](CHECKLIST.md) | Project submission checklist (Developer Akademie) |

### Extras

- Database healthcheck with `depends_on: condition: service_healthy` for a reliable startup order
- Additional volume `wp_data` so uploads, themes, plugins, and `wp-config.php` persist as well
- Database not exposed to the host network
