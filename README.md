# WordPress Docker Deployment

A containerized WordPress setup running on Docker Compose, with a MariaDB database,
persistent storage and automatic container restart. Everything needed to run the site
is defined in a single `docker-compose.yaml`; all environment-specific values live in a
local `.env` file that is never committed.

---

## Table of Contents

- [Quickstart](#quickstart)
  - [Prerequisites](#prerequisites)
  - [Steps](#steps)
- [Usage](#usage)
  - [Environment Variables](#environment-variables)
  - [Changing the Port](#changing-the-port)
  - [Changing Database Credentials](#changing-database-credentials)
  - [Changing Image Versions](#changing-image-versions)
  - [Managing the Stack](#managing-the-stack)
  - [Viewing Logs](#viewing-logs)
- [Deployment on a Server](#deployment-on-a-server)
- [Testing](#testing)
- [Troubleshooting](#troubleshooting)
- [Overview](#overview)
- [How It Works](#how-it-works)
- [Repository Contents](#repository-contents)

---

## Quickstart

### Prerequisites

- Docker Engine 20.10 or newer
- Docker Compose v2 (included in current Docker installations as `docker compose`)
- A free TCP port on the host (`8080` by default)

Verify your installation:

```bash
docker --version
docker compose version
```

### Steps

**1. Clone the repository**

```bash
git clone <repository-url>
cd wordpress-project
```

**2. Create your environment file**

```bash
cp .env.example .env
```

**3. Set real values in `.env`**

Open `.env` and replace every placeholder. Use two different, strong passwords for
`MYSQL_PASSWORD` and `MYSQL_ROOT_PASSWORD`. Never commit this file.

**4. Start the stack**

```bash
docker compose up -d
```

The first run pulls both images and initialises the database, which takes a moment.

**5. Check that both services are up**

```bash
docker compose ps
```

Both `db` and `wordpress` should show status `Up`. If `wordpress` shows `Restarting`,
wait about 20 seconds — it retries until the database is ready — and check again.

**6. Complete the WordPress installation**

Open <http://localhost:8080> in a browser. WordPress will show its setup wizard, where
you choose the site title and create the **administrator account** (username, password,
email). These credentials are stored in the database, not in any configuration file.

For security, avoid `admin` as the username.

---

## Usage

### Environment Variables

All configuration is read from `.env`, which Docker Compose loads automatically when it
sits next to `docker-compose.yaml`. The values are referenced in the compose file using
`${VARIABLE}` notation.

| Variable              | Example     | Sensitive | Description                                                                             |
| --------------------- | ----------- | --------- | --------------------------------------------------------------------------------------- |
| `MYSQL_DATABASE`      | `wordpress` | no        | Name of the database created on first start and used by WordPress.                        |
| `MYSQL_USER`          | `wp_user`   | no        | Database user created on first start. WordPress connects with this account.               |
| `MYSQL_PASSWORD`      | —           | **yes**   | Password for `MYSQL_USER`. Used by both services so they agree on the same credentials.   |
| `MYSQL_ROOT_PASSWORD` | —           | **yes**   | Password for the MariaDB `root` account. Administrative access to the database only.      |
| `WORDPRESS_PORT`      | `8080`      | no        | Host port that WordPress is published on. Mapped to container port 80.                    |

Note that `MYSQL_ROOT_PASSWORD` is the database administrator, which is unrelated to the
WordPress administrator account created through the web installer.

### Changing the Port

Edit `WORDPRESS_PORT` in `.env`, then recreate the containers:

```bash
docker compose up -d
```

The `docker-compose.yaml` needs no modification — the port is referenced as
`"${WORDPRESS_PORT}:80"`. Use this when 8080 is already taken on the host, or to run
several instances side by side.

Check whether a port is free before starting:

```bash
lsof -iTCP:8080 -sTCP:LISTEN   # macOS / Linux
```

### Changing Database Credentials

The database user and password are created **once**, when the `db_data` volume is
initialised on first start. Editing `.env` afterwards will not change the existing
database — WordPress would then fail to authenticate.

To apply new credentials on a fresh installation:

```bash
docker compose down -v     # WARNING: deletes all data
# edit .env
docker compose up -d
```

On an existing installation that must keep its data, change the password inside MariaDB
instead, and then update `.env` to match:

```bash
docker compose exec db mariadb -u root -p
```

```sql
ALTER USER 'wp_user'@'%' IDENTIFIED BY 'new_password';
FLUSH PRIVILEGES;
```

Then restart so WordPress picks up the new value:

```bash
docker compose up -d
```

### Changing Image Versions

Both images are pinned to a major version in `docker-compose.yaml`
(`wordpress:6-php8.3-apache`, `mariadb:11`). This keeps builds reproducible while still
receiving patch updates. To move to a different version, edit the `image:` line of the
service and run:

```bash
docker compose pull
docker compose up -d
```

Back up your volumes before a major database upgrade.

### Managing the Stack

| Command                       | Effect                                                    |
| ----------------------------- | ---------------------------------------------------------- |
| `docker compose up -d`        | Create and start the containers in the background.          |
| `docker compose stop`         | Stop the containers; they and the volumes remain.           |
| `docker compose start`        | Start previously stopped containers.                        |
| `docker compose restart`      | Restart both services.                                      |
| `docker compose down`         | Stop and **remove** the containers. Volumes are kept.       |
| `docker compose down -v`      | Remove the containers **and delete all data**. Irreversible. |
| `docker compose ps`           | List running containers.                                    |
| `docker compose ps -a`        | List all containers, including stopped ones.                |

The distinction between `down` and `down -v` matters: `down` is safe and keeps your
site, `down -v` wipes the database and the WordPress files.

### Viewing Logs

```bash
docker compose logs -f            # both services, follow
docker compose logs wordpress     # one service
docker compose logs --tail 50 db  # last 50 lines
```

---

## Deployment on a Server

The setup is identical on a remote host; only the URL differs.

**1. Install Docker** on the server (Docker Engine plus the Compose plugin). On Ubuntu,
Docker runs as a systemd service and starts automatically at boot, which — combined with
`restart: unless-stopped` — brings the site back up after a reboot without manual steps.

**2. Clone the repository and create `.env`** exactly as in the Quickstart. Generate new
passwords for the server; do not reuse your local ones.

**3. Set `WORDPRESS_PORT=8080`** in `.env` so the site is reachable on the expected port.

**4. Start the stack:**

```bash
docker compose up -d
```

**5. Open the firewall** for the chosen port. On Ubuntu with `ufw`:

```bash
sudo ufw allow 8080/tcp
```

If your provider has its own firewall or security group, allow the port there as well.

**6. Run the installer** at `http://<server-ip>:8080` and create the administrator
account.

For a public site, put a reverse proxy with TLS in front of WordPress and stop
publishing port 8080 directly.

---

## Testing

The following checks verify that the setup behaves correctly.

### 1. Accessibility

```bash
docker compose ps
```

`wordpress` shows `0.0.0.0:8080->80/tcp`, `db` shows only `3306/tcp` without a host
address — confirming the database is not exposed to the outside.

Open <http://localhost:8080>; the site loads.

### 2. Administrator Login

Go to <http://localhost:8080/wp-admin> and log in with the account created during the
installation. The dashboard loads and the site can be navigated.

### 3. Data Persistence

Create a post in the WordPress admin, then remove the containers entirely and start
them again:

```bash
docker compose down
docker compose ps          # no containers listed
docker volume ls           # db_data and wp_data still exist
docker compose up -d
```

Reload <http://localhost:8080>. The post and your login are still there, and the setup
wizard does **not** reappear — the data lives in the volumes, not in the containers.

### 4. Automatic Restart

Terminate the main process inside the WordPress container to simulate a crash:

```bash
docker compose exec wordpress kill 1
docker compose ps
```

The container reappears with an old `CREATED` timestamp but a `STATUS` of only a few
seconds — Docker restarted the same container automatically.

Note that `docker kill <container>` is **not** a valid test here. Docker treats a
container stopped from the outside as intentionally stopped, and `unless-stopped` will
correctly leave it down. Only a process failing inside the container counts as a crash.

---

## Overview

This repository provides a reproducible, self-hosted WordPress installation for
development and small production deployments. It runs two services:

| Service     | Image                        | Role                                             |
| ----------- | ---------------------------- | ------------------------------------------------ |
| `wordpress` | `wordpress:6-php8.3-apache`  | WordPress with Apache and PHP, exposed to the host |
| `db`        | `mariadb:11`                 | MariaDB database, reachable only inside Docker    |

Key properties:

- **No secrets in the repository.** Passwords are supplied through a local `.env` file
  that is excluded from version control. The repository only contains a template.
- **Data survives container removal.** Named volumes hold the database and the WordPress
  content directory, so posts, users, uploads and plugins persist across restarts and
  recreations of the containers.
- **Self-healing.** Both services use `restart: unless-stopped`, so a crashed container
  is restarted automatically, including after a host reboot.
- **Private database.** MariaDB publishes no host port. It is reachable only from the
  `wordpress` container over the shared Docker network.

---

## How It Works

```
┌──────────────────── network: wordpress_net ────────────────────┐
│                                                                │
│   ┌─────────────────┐                  ┌─────────────────┐     │
│   │    wordpress    │ ──── db:3306 ──▶ │       db        │     │
│   │  Apache + PHP   │                  │    MariaDB 11   │     │
│   └────────┬────────┘                  └────────┬────────┘     │
│            │ :80                                │              │
└────────────┼────────────────────────────────────┼──────────────┘
             │                                    │
    ┌────────▼─────────┐                 ┌────────▼────────┐
    │   host :8080     │                 │ volume: db_data │
    │  (your browser)  │                 └─────────────────┘
    └──────────────────┘
             │
    ┌────────▼────────┐
    │ volume: wp_data │
    └─────────────────┘
```

Both containers join the user-defined network `wordpress_net`. Docker provides internal
DNS on that network, which resolves each **service name** to its container. WordPress
therefore reaches the database at the hostname `db` — this is why `WORDPRESS_DB_HOST` is
set to `db:3306` and why the database needs no published port.

Only the `wordpress` service maps a port to the host (`${WORDPRESS_PORT}:80`). Apache
always listens on port 80 inside the container; the host-side port is configurable.

Two named volumes provide persistence:

- `db_data` → `/var/lib/mysql` — the MariaDB data directory (posts, users, settings)
- `wp_data` → `/var/www/html` — the WordPress installation (uploads, themes, plugins)

---

## Repository Contents

| File                                         | Description                                                                                              |
| -------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| [`docker-compose.yaml`](docker-compose.yaml) | Defines both services, the named volumes, and the shared network. The only file needed to run the stack.   |
| [`.env.example`](.env.example)               | Template for the environment file. Copy it to `.env` and fill in real values. Contains placeholders only.   |
| [`.gitignore`](.gitignore)                   | Excludes the local `.env` from version control so credentials never reach the repository.                   |
| [`README.md`](README.md)                     | This document.                                                                                             |

Created locally, **not** part of the repository:

| File   | Description                                                                                      |
| ------ | -------------------------------------------------------------------------------------------------- |
| `.env` | Your real configuration, including database passwords. Created from `.env.example` during setup.   |

---

## Troubleshooting

**`Cannot connect to the Docker daemon` / `no such file or directory`**
The Docker daemon is not running. Start Docker Desktop or OrbStack on macOS, or
`sudo systemctl start docker` on Linux. Verify with `docker info`.

**`Bind for 0.0.0.0:8080 failed: port is already allocated`**
Another process holds the port. Find it with `lsof -iTCP:8080 -sTCP:LISTEN`, then either
stop that process or set a different `WORDPRESS_PORT` in `.env`.

**WordPress shows "Error establishing a database connection"**
Usually a timing issue on first start; the container retries automatically. If it
persists, the credentials in `.env` no longer match those stored in the `db_data`
volume — see [Changing Database Credentials](#changing-database-credentials). Check
`docker compose logs db` for details.

**`variable is not set` warnings when starting**
`.env` is missing or incomplete. Confirm it exists next to `docker-compose.yaml` and
defines every variable listed in `.env.example`.

**The setup wizard appears again after a restart**
The volumes were removed, most likely by `docker compose down -v`. The previous data
cannot be recovered.
