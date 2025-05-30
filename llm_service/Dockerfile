FROM python:3.12-slim

# Set non-interactive to avoid user prompts
ENV DEBIAN_FRONTEND=noninteractive

# Create a non-root user
RUN groupadd -r appuser && useradd -r -g appuser -d /home/appuser -m appuser

WORKDIR /app

# Install system dependencies for llama-cpp-python with security updates
RUN apt-get update && apt-get install -y \
  build-essential \
  cmake \
  curl \
  ca-certificates \
  && apt-get upgrade -y \
  && apt-get clean \
  && rm -rf /var/lib/apt/lists/*

# Copy requirements first for better caching
COPY requirements.txt .

# Configure pip for better network handling and install basic dependencies first
RUN pip install --upgrade pip && \
  pip install --no-cache-dir --timeout 300 --retries 3 \
  fastapi uvicorn pymongo motor python-dotenv pydantic starlette \
  langchain langchain-community huggingface-hub

# Install llama-cpp-python with CPU optimization for Linux containers
# Use CPU-optimized build since Metal GPU is not available in Linux containers
ENV CMAKE_ARGS="-DLLAMA_BLAS=ON -DLLAMA_BLAS_VENDOR=OpenBLAS"
ENV FORCE_CMAKE=1

# Install OpenBLAS for CPU optimization
RUN apt-get update && apt-get install -y \
  libopenblas-dev \
  && apt-get clean \
  && rm -rf /var/lib/apt/lists/*

RUN pip install --no-cache-dir --timeout 600 --retries 3 \
  llama-cpp-python==0.3.2 && \
  # Remove pip cache
  rm -rf /root/.cache/pip/*

# Create directories for model storage and source code with proper permissions
RUN mkdir -p /app/models /app/src && \
  chown -R appuser:appuser /app

# Copy application code to src directory
COPY . /app/src/

# Copy entrypoint script
COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh && \
  chown appuser:appuser /entrypoint.sh

# Expose the API port
EXPOSE 8080

# Add health check (simple response check once the service is running)
HEALTHCHECK --interval=60s --timeout=10s --start-period=30s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1

# Switch to non-root user for running the application
USER appuser

# Change the working directory to where the source code is mounted
WORKDIR /app/src

ENTRYPOINT ["/entrypoint.sh"]
CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8080"]
