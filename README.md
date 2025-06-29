# Python Backend Docker Deployment Templates

A collection of pragmatic and efficient Docker configurations for Python backend applications, specifically tailored for **Django** and **Litestar** projects. These templates are designed for simplicity, ease of deployment on single servers or vertically scaled environments, and efficient multi-process management.

## Table of Contents

* [Why This Approach?](#why-this-approach)
* [Features](#features)
* [Directory Structure](#directory-structure)
* [Getting Started](#getting-started)
    * [Prerequisites](#prerequisites)
    * [General Usage](#general-usage)
    * [Django Deployment](#django-deployment)
    * [Litestar Deployment](#litestar-deployment)
* [Configuration and Environment Variables](#configuration-and-environment-variables)
* [Troubleshooting Common Issues](#troubleshooting-common-issues)

---

## Why This Approach?

In small-to-medium projects and agile teams with frequent deployments, simplicity and rapid iteration are key. This repository provides Docker configurations that prioritize:

* **Simplicity:** Fewer moving parts, easier to understand, build, and debug.
* **Cost-Effectiveness:** Optimized for deployment on single, vertically scaled servers, avoiding the overhead and complexity (and cost) of distributed orchestration systems (like Kubernetes) when not truly needed.
* **Efficiency:** Utilizes `uv` for lightning-fast dependency management and `supervisord` for robust multi-process management within a single container.
* **Flexibility:** Allows running your application in different modes (e.g., with or without background task workers) from a single Docker image, both in single-container and multi-container setups.

This approach is ideal for projects that can be effectively scaled vertically or for initial deployments before horizontal scaling becomes a critical necessity.

## Features

* **Framework-Specific Templates:** Dedicated Dockerfiles and Docker Compose files for Django and Litestar applications.
* **`uv` Integration:** Uses `uv` for highly efficient and reproducible dependency installation.
* **`supervisord` for Single-Container Multi-Process Management:** Easily manage your web server (Granian, Uvicorn, Gunicorn) alongside Celery workers and Celery beat within a single container.
* **Docker Compose for Multi-Service Orchestration:** Define and run your web application, Celery workers, and Celery beat in separate, linked containers, ideal for production-like environments connecting to external databases and message brokers.
* **Flexible Runtime Options:** Configure your container(s) to run the web application only, or include background task processors, by simply changing the `docker run` command or `docker compose` configuration.
* **Optimized Builds:** Best practices for Docker caching and minimizing image size, including the use of `.dockerignore`.

## Directory Structure


<pre><code>
├── django
│   ├── .dockerignore
│   ├── .env
│   ├── docker-compose.yml
│   ├── Dockerfile
│   └── supervisord.conf
├── litestar
│   ├── .dockerignore
│   ├── .env
│   ├── docker-compose.yml
│   ├── Dockerfile
│   └── supervisord.conf
└── README.md
</code></pre>


Each framework directory contains:
* `Dockerfile`: The main Dockerfile for building the application image for both single-container and multi-container setups.
* `supervisord.conf`: A Supervisor configuration file to manage multiple processes (web server, Celery worker, Celery beat) within a **single container**.
* `docker-compose.yml`: A Docker Compose file for orchestrating multiple services (web server, Celery worker, Celery beat) in separate containers, connecting to external databases and message brokers.
* `.dockerignore`: A file that tells the Docker build process which files and directories to ignore when creating the image, ensuring a smaller and cleaner build.
* `.env`: An example `.env` file for use with `docker-compose.yml`, outlining required environment variables for external services.


## Getting Started

### Prerequisites

* **Docker:** Ensure Docker Desktop or Docker Engine is installed and running on your system.
* Your Python project (Django or Litestar) with `pyproject.toml` and `uv.lock` files.

### General Usage

To use these templates:

1.  **Choose your framework's directory** (`django/` or `litestar/`).
2.  **Copy the necessary files** (`Dockerfile`, `supervisord.conf`, `docker-compose.yml`, `.env`, and **`.dockerignore`**) from that directory into the **root** of your specific Django or Litestar project.
3.  **Ensure a `.dockerignore` file** is present and properly configured in your project's root to prevent unnecessary files (like `.git`, `.venv`, `.env` – especially for sensitive data, `__pycache__`, etc.) from being copied into your Docker image.

### Django Deployment

The `django/Dockerfile` builds the application image, and you can deploy it in two main ways:

#### Option A: Single Container with Supervisord (Web + Celery + Celery Beat in one container)

This option uses the `supervisord.conf` to manage all processes within a single container.

1.  **Preparation:**
    * **For Django + Celery + Celery Beat (Default):** Use the `supervisord.conf` provided in `django/`. Ensure your Celery app is importable (e.g., `config.celery`).
    * **For Django Only (No Celery/Beat):** Use the single command `CMD ["granian" "--interface" "wsgi" "--host" "0.0.0.0" "--port" "80" "config.wsgi:application"]` in your Dockerfile and remove the `CMD ["supervisord", "-c", "/app/supervisord.conf"]` command. Your Dockerfile must contain *only* one `CMD` instruction.

2.  **Build the Docker Image:**
    Navigate to your Django project's root directory in your terminal and run:
    ```bash
    docker build -t your-django-app:latest .
    ```
    Replace `your-django-app` with a meaningful name for your image.

3.  **Run the Docker Container:**
    * **Run Django with or without Celery (Default `CMD` from Dockerfile):**
        ```bash
        docker run -d -p 80:80 --name my-django-full-app your-django-app:latest
        ```

    * **Run Django Only (Override default `CMD` from Dockerfile - bypasses Supervisord):**
        ```bash
        docker run -d -p 80:80 --name my-django-full-app your-django-app:latest \
            granian --interface wsgi --host 0.0.0.0 --port 80 config.wsgi:application
        ```

    * **Run Django with Celery (Override default `CMD` from Dockerfile - bypasses Django only):**
        ```bash
        docker run -d -p 80:80 --name my-django-full-app your-django-app:latest \
            supervisord -c /app/supervisord.conf
        ```

#### Option B: Multi-Container with Docker Compose (Web + Celery + Celery Beat in separate containers)

This option uses `docker-compose.yml` to define and run your application services, connecting to external, managed database and message broker services (not defined in this compose file).

1.  **Prepare `.env` file:**
    Copy `django/.env` to your Django project's root. **Fill in the actual connection details for your external database and message broker.** Ensure this `.env` file is in your project's `.dockerignore`!

    ```dotenv
    # Example .env content
    # granian
    GRANIAN_LOG_ACCESS_ENABLED=True
    GRANIAN_LOG_ACCESS_FMT='[%(time)s] %(addr)s - "%(method)s %(scheme)s://%(addr)s%(path)s%(query_string)s %(protocol)s" %(status)d %(dt_ms).3f'
    GRANIAN_WORKERS=2

    # celery
    CELERY_BROKER_URL=redis://your-managed-redis-host:6379/1
    CELERY_RESULT_BACKEND=redis://your-managed-redis-host:6379/2

    # django
    DATABASE_URL=postgresql://user:password@your-managed-db-host:5432/your_db_name
    DJANGO_SECRET_KEY=your_super_secret_key
    DJANGO_DEBUG=False
    ALLOWED_HOSTS=yourdomain.com
    ```

2.  **Build and Run with Docker Compose:**
    Navigate to your Django project's root directory (where `docker-compose.yml` and `.env` are) and run:
    ```bash
    docker compose up --build -d
    ```
    * `--build`: Builds (or rebuilds) your application image.
    * `-d`: Runs services in detached (background) mode.

3.  **Check Status and Logs:**
    * View running services: `docker compose ps`
    * Stream logs from all services: `docker compose logs -f`
    * Stream logs from a specific service: `docker compose logs -f django`

4.  **Stop Services:**
    * To stop services: `docker compose stop`
    * To stop and remove containers/networks (keeping named volumes for data persistence): `docker compose down`
    * To also remove named volumes (like `celerybeat_data`): `docker compose down -v`

### Litestar Deployment

The `litestar/Dockerfile` and `litestar/docker-compose.yml` are configured similarly for your Litestar project.

#### Option A: Single Container with Supervisord (Web + Celery + Celery Beat in one container)

1.  **Preparation:**
    * **For Litestar + Celery + Celery Beat (Default):** Use the `supervisord.conf` provided in `litestar/`. Ensure your Celery app is importable (e.g., `config.celery`).
    * **For Litestar Only (No Celery/Beat):** Use the single command `CMD ["granian" "--interface" "wsgi" "--host" "0.0.0.0" "--port" "80" "config.wsgi:application"]` in your Dockerfile and remove the `CMD ["supervisord", "-c", "/app/supervisord.conf"]` command. Your Dockerfile must contain *only* one `CMD` instruction.

3.  **Build the Docker Image:**
    Navigate to your Litestar project's root directory in your terminal and run:
    ```bash
    docker build -t your-litestar-app:latest .
    ```
    Replace `your-litestar-app` with a meaningful name for your image.

4.  **Run the Docker Container:**
    * **Run Litestar with Celery (Default `CMD` from Dockerfile):**
        ```bash
        docker run -d -p 80:80 --name my-litestar-full-app your-litestar-app:latest
        ```

    * **Run Litestar Only (Override default `CMD` from Dockerfile - bypasses Supervisord):**
        ```bash
        docker run -d -p 80:80 --name my-litestar-full-app your-litestar-app:latest \
            litestar run --host 0.0.0.0 --port 80
        ```

    * **Run Litestar with Celery (Override default `CMD` from Dockerfile - bypasses Django only):**
        ```bash
        docker run -d -p 80:80 --name my-litestar-full-app your-litestar-app:latest \
            supervisord -c /app/supervisord.conf
        ```

#### Option B: Multi-Container with Docker Compose (Web + Celery + Celery Beat in separate containers)

This option uses `docker-compose.yml` to define and run your application services, connecting to external, managed database and message broker services (not defined in this compose file).

1.  **Prepare `.env` file:**
    Copy `litestar/.env` to your Django project's root. **Fill in the actual connection details for your external database and message broker.** Ensure this `.env` file is in your project's `.dockerignore`!

    ```dotenv
    # Example .env content
    # granian
    GRANIAN_LOG_ACCESS_ENABLED=True
    GRANIAN_LOG_ACCESS_FMT='[%(time)s] %(addr)s - "%(method)s %(scheme)s://%(addr)s%(path)s%(query_string)s %(protocol)s" %(status)d %(dt_ms).3f'
    GRANIAN_WORKERS=2

    # celery
    CELERY_BROKER_URL=redis://your-managed-redis-host:6379/1
    CELERY_RESULT_BACKEND=redis://your-managed-redis-host:6379/2

    # litestar
    DATABASE_URL=postgresql://user:password@your-managed-db-host:5432/your_db_name
    LITESTAR_SECRET_KEY=your_super_secret_key
    LITESTAR_DEBUG=False
    ALLOWED_HOSTS=yourdomain.com
    ```

2.  **Build and Run with Docker Compose:**
    Navigate to your Django project's root directory (where `docker-compose.yml` and `.env` are) and run:
    ```bash
    docker compose up --build -d
    ```
    * `--build`: Builds (or rebuilds) your application image.
    * `-d`: Runs services in detached (background) mode.
    ```

3.  **Check Status and Logs:**
    * View running services: `docker compose ps`
    * Stream logs from all services: `docker compose logs -f`
    * Stream logs from a specific service: `docker compose logs -f litestar`

4.  **Stop Services:**
    * To stop services: `docker compose stop`
    * To stop and remove containers/networks (keeping named volumes for data persistence): `docker compose down`
    * To also remove named volumes (like `celerybeat_data`): `docker compose down -v`

## Configuration and Environment Variables

* **Application-Specific Variables:** For sensitive data like database credentials, API keys, or environment-specific settings (e.g., `DJANGO_DEBUG=False`), **do NOT hardcode them in your Dockerfile.**
* **Local Development:** Use a `.env` file (which should be in your `.dockerignore`) in your project's root. Your application can read these using libraries like `python-dotenv`.
* **Production Deployment:** For Docker Compose, the `env_file` directive is used. For direct `docker run`, pass these variables at runtime using the `-e` flag. In cloud environments, production secrets should ideally be managed via secure services (e.g., AWS Secrets Manager, Google Cloud Secret Manager, Azure Key Vault) and retrieved by your application at startup.
    ```bash
    # Example: Passing a variable directly to docker run
    docker run -d -p 80:80 -e DATABASE_URL="postgres://user:pass@host:port/db" your-app:latest
    ```
    Your application code (e.g., `os.getenv('DATABASE_URL')` in Django `settings.py` or Litestar `config.py`) should read these environment variables.

## Troubleshooting Common Issues

* **`Error response from daemon: Ports are not available: ... address already in use`**
    This means the port Docker is trying to expose on your host machine (e.g., `80`) is already being used by another application or container.
    * **Solution 1: Stop the conflicting process.** Identify what's using the port (`sudo lsof -i :80` on Linux/macOS, `netstat -ano | findstr :80` on Windows) and stop it.
    * **Solution 2: Change the host port mapping.** In your `docker-compose.yml`, change `ports: - "80:80"` to `ports: - "8000:80"` (or any other free port). Then access your application at `http://localhost:8000`.

* **Container Exits Immediately/Doesn't Start:**
    * Check container logs: `docker logs <container_id_or_name>` or `docker compose logs <service_name>`
    * Look for errors related to missing dependencies, incorrect commands, or application startup failures.
    * Ensure all necessary files (like `supervisord.conf`, `uv.lock`, `pyproject.toml`, application code, and **`.dockerignore`**) are correctly copied into the image.

* **Application Can't Connect to External Database/Broker:**
    * Verify the `DATABASE_URL` or `CELERY_BROKER_URL` environment variables in your `.env` file are correct (host, port, credentials).
    * Ensure your cloud-managed services are publicly accessible or accessible from the network where your Docker host is running. Check firewall rules or security groups.