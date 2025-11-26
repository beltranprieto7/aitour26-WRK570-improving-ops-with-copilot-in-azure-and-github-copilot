### Secure multi-stage Dockerfile for ai-service
# Builder: install build dependencies and Python packages
FROM python:3.12.7-slim AS builder

WORKDIR /app
ARG APP_VERSION=0.1.0
ENV PYTHONDONTWRITEBYTECODE=1 \
	PYTHONUNBUFFERED=1

# Install build dependencies (only in builder stage)
RUN apt-get update && \
	apt-get install -y --no-install-recommends build-essential gcc && \
	rm -rf /var/lib/apt/lists/*

# Copy requirements first to leverage Docker cache
COPY requirements.txt .

# Use pip cache mount when building with BuildKit to speed up builds
RUN --mount=type=cache,target=/root/.cache/pip \
	pip install --user --no-cache-dir -r requirements.txt

# Copy application sources into builder
COPY . /app


# Final runtime image: keep surface area small
FROM python:3.12.7-slim
WORKDIR /app
ARG APP_VERSION=0.1.0
ENV APP_VERSION=$APP_VERSION \
	PYTHONDONTWRITEBYTECODE=1 \
	PYTHONUNBUFFERED=1 \
	PATH=/root/.local/bin:$PATH

# Create a non-root user and group
RUN useradd --create-home --shell /bin/bash appuser

# Copy installed Python packages from builder
COPY --from=builder /root/.local /root/.local

# Copy application files and set ownership to non-root user
COPY --from=builder --chown=appuser:appuser /app /app

EXPOSE 5001

# Switch to non-root user
USER appuser

# Healthcheck: quick TCP connect to port 5001
HEALTHCHECK --interval=30s --timeout=5s --start-period=5s --retries=3 \
  CMD python -c "import socket,sys; s=socket.socket(); s.settimeout(1); \ 
try:\n    s.connect(('127.0.0.1',5001)); s.close(); sys.exit(0)\nexcept:\n    sys.exit(1)"

# Use exec form so signals are received correctly
ENTRYPOINT ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "5001"]