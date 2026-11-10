# WordPress with Docker Compose

A containerized WordPress setup with a MariaDB database, managed declaratively with Docker Compose. The setup can be reproduced on any Docker host (e.g. a cloud VM) with only a few configuration steps.

## Table of Contents

- [Description](#description)
- [Repository Contents](#repository-contents)
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
- [Extras](#extras)

## Description

The purpose of this repository is to deploy a WordPress instance together with its backing service, a MariaDB database, using Docker Compose. The project focuses on **configuring** a containerized application rather than building a custom image. It uses the official images [`wordpress`](https://hub.docker.com/_/wordpress) and [`mariadb`](https://hub.docker.com/_/mariadb).

Main contents:

- A `docker-compose.yaml` with two services, `wordpress` and `db`, connected through a shared network
- Named volumes so that database content and WordPress files persist across restarts
- A strict separation of non-critical configuration (in the compose file) and credentials (in a local `.env` file that is never committed)

## Repository Contents

| File | Description |
|---|---|
| `docker-compose.yaml` | Defines the `wordpress` and `db` services, the network, and the volumes |
| `.env.example` | Template for the `.env` file containing all required credentials (placeholders only) |
| `.gitignore` | Excludes secrets (`.env`), database dumps, logs, virtual environments, and editor/OS files |
| `README.md` | This documentation |
| [`CHECKLIST.md`](CHECKLIST.md) | Project submission checklist (Developer Akademie) with all fulfilled requirements and tests |

## Quickstart

The fastest way to get WordPress up and running. For details on configuration, see [Usage](#usage).

### Prerequisites

- A Linux host (e.g. a cloud VM) with SSH access
- [Docker Engine](https://docs.docker.com/engine/install/) with the [Docker Compose plugin](https://docs.docker.com/compose/install/) (`docker compose`)
- Git
- Port `8080` reachable from your browser (open it in your firewall if necessary, e.g. `sudo ufw allow 8080/tcp`)

### Quickstart Guide

```bash
# 1. Clone the repository
git clone https://github.com/MatzeRB26/wordpress.git
cd wordpress

# 2. Create your .env file from the template
cp .env.example .env

# 3. Replace all placeholders in .env with your own values
nano .env

# 4. Start the setup
docker compose up -d

# 5. Check that both containers are running (db should be "healthy")
docker compose ps
```

Open `http://<your-server-ip>:8080` in your browser and complete the WordPress installer. This is where you create your **admin user**.

## Usage

This section explains in detail how the setup is configured and which parts you can modify to get different results.

### Configuration Overview

The configuration is split into two layers:

| Layer | Where | Content |
|---|---|---|
| Credentials | `.env` (local only, git-ignored) | Database users and passwords |
| Non-critical settings | `docker-compose.yaml` (`environment` block) | Database host, database name, table prefix, port, image versions |

All non-critical settings use the `${VARIABLE:-default}` notation. They work out of the box with sensible defaults, but each of them can be overridden by adding the variable to your `.env` file. You do not need to edit the compose file.

### Secrets (`.env`)

Copy `.env.example` to `.env` and replace every placeholder:

| Variable | Used by | Description |
|---|---|---|
| `MARIADB_ROOT_PASSWORD` | `db` | Password of the MariaDB `root` user |
| `MARIADB_USER` | `db` | Database user that WordPress connects with |
| `MARIADB_PASSWORD` | `db` | Password of that database user |
| `WORDPRESS_DB_USER` | `wordpress` | Must be identical to `MARIADB_USER` |
| `WORDPRESS_DB_PASSWORD` | `wordpress` | Must be identical to `MARIADB_PASSWORD` |

The values appear twice because the MariaDB and WordPress images expect different variable names.

Tip: generate strong random passwords and fill them in automatically:

```bash
cp .env.example .env
DB_PW=$(openssl rand -hex 20); ROOT_PW=$(openssl rand -hex 20)
sed -i "s/change-me-root-password/${ROOT_PW}/; s/change-me-db-password/${DB_PW}/g; s/change-me-db-user/wpuser/g" .env
grep -c change-me .env   # should print 0
```

> **Note:** MariaDB only reads these values when the database volume is created for the first time. If you change the database credentials later, you have to change them inside the database as well, or start over with a fresh volume (see [Volumes and Persistence](#volumes-and-persistence)).

### Non-Critical Settings

These variables are optional. Add any of them to your `.env` file to change the default behavior:

| Variable | Default | Effect |
|---|---|---|
| `WORDPRESS_PORT` | `8080` | Host port WordPress is published on (`http://<host>:<port>`) |
| `DB_NAME` | `wordpress` | Name of the database (used by both services) |
| `WORDPRESS_TABLE_PREFIX` | `wp_` | Prefix for all WordPress database tables |
| `WORDPRESS_VERSION` | `php8.3-apache` | Tag of the `wordpress` image. The default pins the PHP version and always provides the latest WordPress release. |
| `MARIADB_VERSION` | `11.4` | Tag of the `mariadb` image (11.4 is a long-term support release) |

Example: to run WordPress on port `9000` with a custom table prefix, add the following to `.env` and recreate the containers:

```env
WORDPRESS_PORT=9000
WORDPRESS_TABLE_PREFIX=site1_
```

```bash
docker compose up -d
```

> Change `DB_NAME` and `WORDPRESS_TABLE_PREFIX` **before** the first start. Changing them afterwards makes WordPress look for a database or tables that do not exist, and the installer appears again.

You can check the final configuration, with all variables resolved, before starting:

```bash
docker compose config
```

### Services

| Service | Image | Purpose |
|---|---|---|
| `db` | `mariadb` | Stores all WordPress content (posts, users, settings). It is **not** published to the host and is only reachable inside the Docker network. |
| `wordpress` | `wordpress` (Apache + PHP) | Serves the WordPress site on port `8080` and connects to the database via the hostname `db`. |

The `db` service has a **healthcheck**. Because of `depends_on: condition: service_healthy`, the `wordpress` service only starts once the database is ready to accept connections.

### Network

Both services are attached to the user-defined bridge network `wordpress_net`. Docker's internal DNS resolves service names, so WordPress reaches the database at `db:3306`. The database port is not exposed outside this network.

### Volumes and Persistence

| Volume | Mount point | Content |
|---|---|---|
| `db_data` | `/var/lib/mysql` | All database files |
| `wp_data` | `/var/www/html` | WordPress core, `wp-config.php`, themes, plugins, uploads |

Named volumes are managed by Docker and survive `docker compose down` and host reboots.

```bash
docker volume ls                       # list volumes
docker volume inspect wordpress_db_data
```

> ⚠️ `docker compose down -v` **deletes the volumes and all data**. Only use it if you want a completely fresh installation.

### Restart Policy

Both services use `restart: unless-stopped`. If a container terminates because of an error, Docker restarts it automatically. After a host reboot, the containers start again by themselves. Only containers you stop manually (`docker compose stop`) stay stopped.

### Common Commands

```bash
docker compose up -d          # start (or apply configuration changes)
docker compose ps             # show status
docker compose logs -f        # follow the logs of all services
docker compose logs db        # logs of a single service
docker compose restart        # restart the services
docker compose down           # stop and remove containers (data is kept)
docker compose pull && docker compose up -d   # update to the newest images
```

## Verification

The setup was tested on a cloud VM with the following checks:

1. **Reachability:** WordPress is reachable at `http://<your-server-ip>:8080`.
2. **Login:** You can log in with the admin credentials created in the installer and navigate the dashboard.
3. **Persistence:** Create a post, then run `docker compose down` followed by `docker compose up -d`. The post and all settings are still there and the installer does not appear again.
4. **Automatic restart:** Simulate a crash by killing the container's main process, then check that Docker restarted it:

   ```bash
   docker inspect -f '{{.RestartCount}}' wordpress-wordpress-1   # e.g. 0
   sudo kill -9 $(docker inspect -f '{{.State.Pid}}' wordpress-wordpress-1)
   sleep 5
   docker compose ps                                             # wordpress is "Up" again
   docker inspect -f '{{.RestartCount}}' wordpress-wordpress-1   # increased by 1
   ```

   `docker stop` is not suitable for this test, because Docker treats a manual stop as intentional and does not restart the container.

## Security Notes

- No passwords, tokens, usernames, IP addresses, or SSH keys are stored in this repository.
- Credentials are passed to the containers via the git-ignored `.env` file (`env_file`). Only the placeholder template `.env.example` is committed.
- The database is not published to the host. It is only reachable from the `wordpress` container through the internal network.
- Do not use `admin` as the WordPress username, and choose a strong password in the installer.
- Use up-to-date images (`docker compose pull`) to receive security fixes.

## Extras

- **Database healthcheck** with `depends_on: condition: service_healthy` for a reliable startup order
- **Additional volume `wp_data`** so that uploads, themes, plugins, and `wp-config.php` persist as well
- **Configurable image versions and port** through environment variables with defaults
- **Database not exposed** to the host network
