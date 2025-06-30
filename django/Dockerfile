# Install python with uv pre-installed based on Debian Bookworm: https://docs.astral.sh/uv/guides/integration/docker/
FROM ghcr.io/astral-sh/uv:bookworm-slim

# Set environment variables
ENV PYTHONUNBUFFERED=1 \
    UV_COMPILE_BYTECODE=1 \
    PATH="/app/.venv/bin:$PATH"

# Install & update system dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    gettext curl nano supervisord && \
    apt-get purge -y --auto-remove -o APT::AutoRemove::RecommendsImportant=false && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# Set the working directory
WORKDIR /app

# Install project dependencies using uv
COPY uv.lock pyproject.toml ./
RUN uv sync --locked --no-install-project --no-dev

# Copy all files to the working directory & sync dependencies for last time
COPY . .
RUN uv sync --locked --no-dev

# Generate & compile translations, then collect static files
RUN python manage.py makemessages -a --no-location && \
    python manage.py compilemessages && \
    python manage.py collectstatic --noinput

# Expose the port and run the application with a specific command based on your needs or write your command
EXPOSE 80

# Option 1) run only Django with Gunicorn, Uvicorn, Granian & etc
# CMD ["granian" "--interface" "wsgi" "--host" "0.0.0.0" "--port" "80" "config.wsgi:application"]

# Option 2) run Django with Celery & Celery Beat using supervisord: https://supervisord.org/
CMD ["supervisord", "-c", "/app/supervisord.conf"]