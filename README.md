# n8n Local Development Setup on Docker (Windows 10)

This document outlines the setup for a local n8n instance using Docker Compose, with persistent PostgreSQL storage and configured for access via Tailscale.

## 1. Prerequisites

*   **Docker Desktop for Windows:** Ensure it is installed and running.
*   **Tailscale:** Ensure it is installed and running on this development machine.

## 2. Directory Structure

All files and data will be relative to the directory containing this `README.md` and `docker-compose.yml` (expected to be `d:/app_live_n8n`).

*   `./docker-compose.yml`: The Docker Compose configuration file.
*   `./n8n_data/`: This directory will be created automatically on your host to store persistent n8n data (workflows, credentials, etc.).
*   `postgres_data_vol`: This is a Docker named volume (managed by Docker, not directly in this directory) that stores persistent PostgreSQL database files. The actual name Docker uses will be prefixed, e.g., `app_live_n8n_postgres_data_vol`.

## 3. Configuration

### 3.1. `docker-compose.yml`

The following `docker-compose.yml` file defines the n8n and PostgreSQL services:

```yaml
services:
  postgres:
    image: postgres:15
    restart: always
    environment:
      - POSTGRES_USER=n8nuser
      - POSTGRES_PASSWORD=n8nDefaultDevPassword123! # Documented as is, recommend changing
      - POSTGRES_DB=n8ndb
    volumes:
      - postgres_data_vol:/var/lib/postgresql/data
    networks:
      - n8n-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U n8nuser -d n8ndb"]
      interval: 10s
      timeout: 5s
      retries: 5

  n8n:
    image: n8nio/n8n
    restart: always
    ports:
      - "127.0.0.1:5678:5678"
    environment:
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_PORT=5432
      - DB_POSTGRESDB_DATABASE=n8ndb
      - DB_POSTGRESDB_USER=n8nuser
      - DB_POSTGRESDB_PASSWORD=n8nDefaultDevPassword123!
      - GENERIC_TIMEZONE=America/Chicago # USER ACTION REQUIRED: Verify or change
      - WEBHOOK_URL=http://100.113.4.39:5678 # USER ACTION REQUIRED: Verify or change
      # - N8N_DIAGNOSTICS_ENABLED=false # Uncomment to disable telemetry
    volumes:
      - ./n8n_data:/home/node/.n8n
    depends_on:
      postgres:
        condition: service_healthy
    networks:
      - n8n-network

networks:
  n8n-network:
    driver: bridge

volumes:
  postgres_data_vol:
```

### 3.2. Environment Variables & Secrets

These are the environment variables used in the `docker-compose.yml`.

**PostgreSQL Service (`postgres`):**

*   `POSTGRES_USER`: `n8nuser`
*   `POSTGRES_PASSWORD`: `n8nDefaultDevPassword123!`
    *   **Note:** For development. It's good practice to change this to a unique, strong password.
*   `POSTGRES_DB`: `n8ndb`

**n8n Service (`n8n`):**

*   `DB_TYPE`: `postgresdb`
*   `DB_POSTGRESDB_HOST`: `postgres`
*   `DB_POSTGRESDB_PORT`: `5432`
*   `DB_POSTGRESDB_DATABASE`: `n8ndb`
*   `DB_POSTGRESDB_USER`: `n8nuser`
*   `DB_POSTGRESDB_PASSWORD`: `n8nDefaultDevPassword123!`
*   `GENERIC_TIMEZONE`: `America/Chicago`
    *   **USER ACTION REQUIRED:** Verify this is your correct timezone. If not, change it. Find valid timezones [here](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones).
*   `WEBHOOK_URL`: `http://100.113.4.39:5678`
    *   **USER ACTION REQUIRED:** Verify this is your machine's correct Tailscale IP address or MagicDNS hostname and port. This is critical for webhook functionality.
*   `N8N_DIAGNOSTICS_ENABLED`: (Optional, commented out) Set to `false` to disable n8n telemetry.

## 4. Setup Instructions

1.  **Ensure Docker Desktop is running.**
2.  **Verify `docker-compose.yml`** content (from section 3.1) is saved in your project directory (`d:/app_live_n8n`).
3.  **Verify Placeholders in `docker-compose.yml`:**
    *   Ensure `GENERIC_TIMEZONE` is correct for your location.
    *   Ensure `WEBHOOK_URL` is correct with your Tailscale IP/hostname.
    *   (Recommended) Change `POSTGRES_PASSWORD` if desired.
4.  **Open a terminal** (Command Prompt, PowerShell, or Windows Terminal) and navigate to your project directory (`d:/app_live_n8n`).
5.  **Start the services:**
    ```bash
    docker-compose up -d
    ```
    The `-d` flag runs the containers in detached mode. If they are already running, this command will recreate them if changes were made to the `docker-compose.yml` or images.
6.  **Access n8n:**
    *   **Locally on the host machine:** `http://localhost:5678`
    *   **Via Tailscale (from any device on your Tailscale network):** `http://100.113.4.39:5678` (or your verified Tailscale URL)

## 5. Maintenance & Operations

### 5.1. Viewing Logs

To view logs for a specific service:
```bash
docker-compose logs -f n8n
docker-compose logs -f postgres
```
Press `Ctrl+C` to stop following logs.

### 5.2. Stopping Services

To stop and remove the containers (data in named volumes like `postgres_data_vol` and host-bound volumes like `./n8n_data` will persist):
```bash
docker-compose down
```
To also remove named volumes (like `postgres_data_vol`), use `docker-compose down -v`. This will delete the PostgreSQL database.

### 5.3. Starting Services

If previously stopped:
```bash
docker-compose up -d
```

### 5.4. Updating n8n or PostgreSQL

1.  Pull the latest images specified in your `docker-compose.yml`:
    ```bash
    docker-compose pull
    ```
2.  Recreate the containers with the new images:
    ```bash
    docker-compose up -d --remove-orphans
    ```
    It's always a good idea to back up your data before significant updates.

### 5.5. Backups

**Persistent data for n8n is stored in `./n8n_data` on your host. PostgreSQL data is stored in a Docker named volume (e.g., `app_live_n8n_postgres_data_vol`).**

*   **n8n Data (`./n8n_data`):** This folder on your host contains your workflows, credentials, etc. Regularly back up this entire folder.
*   **PostgreSQL Data (Named Volume):**
    *   To use `pg_dump` (recommended for live backups):
        ```bash
        docker-compose exec -T postgres pg_dump -U n8nuser -d n8ndb > n8n_database_backup.sql
        ```
        This command creates a `n8n_database_backup.sql` file on your host in the current directory.
    *   To restore, you would typically use `psql`.
    *   For backing up the raw named volume data, refer to Docker documentation (e.g., running a temporary container to tar the volume contents).

## 6. Troubleshooting

*   **Port Conflicts:** If port `5678` is already in use on your host, n8n will fail to start. Change the host port in `docker-compose.yml` (e.g., `"127.0.0.1:5679:5678"`) and update your `WEBHOOK_URL` accordingly.
*   **"service "postgres" is unhealthy" or n8n connection issues:**
    *   Check `docker-compose logs -f postgres`.
    *   Ensure `POSTGRES_USER`, `POSTGRES_PASSWORD`, and `POSTGRES_DB` in the `postgres` service environment match the `DB_POSTGRESDB_*` variables in the `n8n` service.
