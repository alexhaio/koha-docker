# koha-docker

Modern, hardened Multi-Arch (`linux/amd64`, `linux/arm64`) Docker container for [Koha ILS](https://koha-community.org/) based on Debian Bookworm and `s6-overlay`.

## Available Tags & Versions

All images are published to the GitHub Container Registry (`ghcr.io/alexhaio/koha`):

| Tag | Koha Release Track | Description | Multi-Arch |
| --- | --- | --- | --- |
| `25.05` | `25.05` | Koha 25.05 release series | `amd64`, `arm64` |
| `25.11` | `25.11` | Koha 25.11 release series | `amd64`, `arm64` |
| `stable` / `latest` | `stable` | Official current stable Koha release | `amd64`, `arm64` |
| `26.05` | `26.05` | Koha 26.05 development / preview | `amd64`, `arm64` |

## Key Improvements & Bugfixes

Compared to upstream `teogramm/koha`:
* **Multi-Arch Support:** Full native support for `linux/amd64` and `linux/arm64`.
* **STOMP ERROR Ping-Pong Loop Fix:** Patched `Net::Stomp` connect handling and non-`MESSAGE` frame processing in `background_jobs_worker.pl` to eliminate 100% CPU loops between Koha and RabbitMQ.
* **s6 Respawn Throttling:** Added `finish` scripts with delay for `worker` and `indexer` daemons to prevent high-CPU crash-looping before DB schema initialization.
* **Safe Credential Injection:** Replaced fragile `sed` password substitution with `xmlstarlet` to support passwords containing special characters (`/`, `&`, `\`, `<`, `>`).
* **Reverse Proxy HTTPS Support:** Automatic `SetEnvIf X-Forwarded-Proto "^https$" HTTPS=on` for proper Apache CGI and Plack HTTPS detection behind Nginx / Traefik.
* **Fixed Typo:** Fixed `koha_conf.xml` filename check in `02-setup-koha.sh` to prevent accidental configuration rewrites on container restarts.

## Ports

| Port | Protocol | Description |
| --- | --- | --- |
| `8080` | HTTP | Koha OPAC (Public Catalog, served via Plack) |
| `8081` | HTTP | Koha Staff Client / Intranet (Administration, served via Plack) |
| `2100` | TCP | Zebra Z39.50 search server |
| `6001` | TCP | Zebra SRU search server |

## Environment Variables

| Variable | Default | Description |
| --- | --- | --- |
| `MYSQL_SERVER` | `db` | Hostname or IP of the MariaDB/MySQL database server |
| `DB_NAME` | `koha_default` | Database name for the Koha instance |
| `MYSQL_USER` | `koha_default` | Database username |
| `MYSQL_PASSWORD` | *(required)* | Database password |
| `KOHA_LANGS` | *(empty)* | Space-separated list of language packages to install (e.g. `de-DE fr-FR`) |
| `ZEBRA_MARC_FORMAT` | `marc21` | MARC cataloging format (`marc21`, `unimarc`, or `normarc`) |
| `MEMCACHED_SERVERS` | `memcached:11211` | Memcached cache server address (`host:port`) |
| `USE_MEMCACHED` | `yes` | Enable Memcached session/metadata caching |
| `MB_HOST` | `rabbitmq` | RabbitMQ host for background jobs worker (STOMP protocol) |
| `MB_PORT` | `61613` | RabbitMQ STOMP port |
| `MB_USER` | `guest` | RabbitMQ username |
| `MB_PASS` | `guest` | RabbitMQ password |
| `USE_ELASTICSEARCH` | `false` | Set to `true` to use Elasticsearch instead of Zebra |
| `ELASTICSEARCH_HOST` | *(empty)* | Elasticsearch host (e.g. `elasticsearch:9200`) |
| `USE_Z3950` | `1` | Enable Zebra Z39.50 server inside container |

## Volumes & Persistence

| Path inside Container | Description |
| --- | --- |
| `/var/lib/koha` | Persistent instance directory containing Zebra search indexes, plugins, state, and upload files. |
| `/var/lib/koha/transfer` | Recommended mount point for bulk data migrations, MARC XML imports, or SQL backups. |

## Docker Compose Example

```yaml
version: "3.9"

services:
  koha:
    image: ghcr.io/alexhaio/koha:25.05
    container_name: koha
    restart: always
    ports:
      - "8080:8080"
      - "8081:8081"
    cap_add:
      - SYS_NICE
      - DAC_READ_SEARCH
    environment:
      - MYSQL_SERVER=koha-db
      - DB_NAME=koha_default
      - MYSQL_USER=koha_default
      - MYSQL_PASSWORD=secret_db_pass
      - USE_MEMCACHED=yes
      - MEMCACHED_SERVERS=koha-memcached:11211
      - MB_HOST=koha-rabbitmq
      - MB_PORT=61613
      - MB_USER=koha_rabbit
      - MB_PASS=secret_rabbit_pass
      - KOHA_LANGS=de-DE
    volumes:
      - koha-data:/var/lib/koha
    depends_on:
      koha-db:
        condition: service_healthy
      koha-rabbitmq:
        condition: service_healthy

  koha-db:
    image: mariadb:11
    container_name: koha-db
    restart: always
    environment:
      MARIADB_ROOT_PASSWORD: secret_root_pass
      MARIADB_DATABASE: koha_default
      MARIADB_USER: koha_default
      MARIADB_PASSWORD: secret_db_pass
    volumes:
      - koha-db-data:/var/lib/mysql
    healthcheck:
      test: ["CMD", "healthcheck.sh", "--connect", "--innodb_initialized"]
      interval: 10s
      timeout: 5s
      retries: 5

  koha-memcached:
    image: memcached:alpine
    container_name: koha-memcached
    restart: always

  koha-rabbitmq:
    image: rabbitmq:3-management-alpine
    container_name: koha-rabbitmq
    restart: always
    environment:
      RABBITMQ_DEFAULT_USER: koha_rabbit
      RABBITMQ_DEFAULT_PASS: secret_rabbit_pass
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "check_running"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  koha-data:
  koha-db-data:
```

> [!NOTE]
> **Linux Capabilities:** Koha Apache uses `mpm_itk` to isolate Koha instances under dedicated UIDs/GIDs (`default-koha`). The capabilities `SYS_NICE` and `DAC_READ_SEARCH` are required so Apache can perform user/group transitions within the container.

---

## Container Administration & CLI Tools

All standard Koha Debian administrative scripts are available directly inside the container (default instance name is `default`).

To access the container shell:
```bash
docker exec -it koha /bin/bash
```

### Key CLI Commands

* **Instance Shell:**
  ```bash
  koha-shell default
  ```
  Opens a subshell as the instance user (`default-koha`) with all Perl environment variables (`KOHA_CONF`, `PERL5LIB`) preloaded.

* **Direct Database CLI:**
  ```bash
  koha-mysql default
  ```
  Connects to the MariaDB instance without requiring manual credentials.

* **Restart Plack / Starman Workers:**
  ```bash
  koha-plack --restart default
  koha-plack --status default
  ```

* **Rebuild Zebra Search Index:**
  ```bash
  koha-rebuild-zebra -v -f default
  ```

* **Rebuild Elasticsearch Index (if configured):**
  ```bash
  koha-elasticsearch --rebuild -v default
  ```

* **Run Database Schema Upgrades:**
  ```bash
  koha-upgrade-schema default
  ```

* **Manage Language Packs:**
  ```bash
  koha-translate -l            # List installed languages
  koha-translate -i de-DE      # Install German translation
  koha-translate -u de-DE      # Update German translation
  ```

---

## Remote Maintenance Shell Access (Optional Pattern)

If external support providers or administrators need maintenance access to Koha without having shell access to the Docker host system, a restricted jump configuration can be implemented on the host:

1. Create a dedicated unprivileged user on the host (e.g. `koha-support`) without `sudo` rights.
2. In `~koha-support/.ssh/authorized_keys`, restrict the SSH key:
   ```text
   command="/usr/local/sbin/koha-container-shell",no-port-forwarding,no-X11-forwarding,no-agent-forwarding ssh-ed25519 AAAAC3NzaC1... maintenance-key
   ```
3. Use a minimal wrapper script (`/usr/local/sbin/koha-container-shell`) to route commands directly into the container:
   ```bash
   #!/bin/bash
   case "$SSH_ORIGINAL_COMMAND" in
     "restart") docker restart koha ;;
     "logs")    docker logs -f --tail 200 koha ;;
     "status")  docker ps --filter "name=koha" ;;
     "")        exec docker exec -it koha /bin/bash ;;
     *)         exec docker exec -it koha /bin/bash -c "$SSH_ORIGINAL_COMMAND" ;;
   esac
   ```

This grants full Koha management capabilities inside the container while keeping the host system completely isolated.

---

## Credits

Based on the original foundation from [teogramm/koha-docker](https://github.com/teogramm/koha-docker) and [koha-community](https://gitlab.com/koha-community/docker/koha-docker).
