# Install python with uv pre-installed based on Debian Bookworm: https://docs.astral.sh/uv/guides/integration/docker/
FROM ghcr.io/astral-sh/uv:bookworm-slim

# Set environment variables
ENV PYTHONUNBUFFERED=1 \
    UV_COMPILE_BYTECODE=1 \
    PATH="/app/.venv/bin:$PATH"

# Install & update system dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    curl nano supervisord && \
    apt-get purge -y --auto-remove -o APT::AutoRemove::RecommendsImportant=false && \
    rm -rf /var/lib/apt/lists/*

# Set the working directory
WORKDIR /app

# Install project dependencies using uv
COPY uv.lock pyproject.toml ./
RUN uv sync --locked --no-install-project --no-dev

# Copy all files to the working directory & sync dependencies for last time
COPY . .
RUN uv sync --locked --no-dev

# Expose the port and run the application with a specific command based on your needs or write your command
EXPOSE 80

# Option 1) run only Litestar
# CMD ["litestar", "run", "--host", "0.0.0.0", "--port", "80"]

# Option 2) run Litestar with Celery & Celery Beat using supervisord: https://supervisord.org/
CMD ["supervisord", "-c", "/app/supervisord.conf"]