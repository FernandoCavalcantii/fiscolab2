FROM python:3.11-slim

# Set environment variables
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

# Set work directory
WORKDIR /app

# Install system dependencies
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        postgresql-client \
        build-essential \
        libpq-dev \
        tesseract-ocr \
        tesseract-ocr-por \
        poppler-utils \
    && rm -rf /var/lib/apt/lists/*

# Install Python dependencies
COPY requirements.txt .
RUN pip install --upgrade pip setuptools wheel \
    && pip install --no-cache-dir -r requirements.txt --timeout 120 --retries 5

# Install CPU-only PyTorch and sentence-transformers to reduce image size
RUN pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cpu
RUN pip install sentence-transformers>=2.7.0 huggingface_hub>=0.20.0

# Clean up to reduce image size
RUN pip cache purge
RUN apt-get clean
RUN rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

# Note: sentence-transformers will be tested at runtime

# Note: Pre-download moved to runtime to avoid build issues

# Copy project
COPY . .

# Create optimized startup script
RUN echo '#!/bin/bash\n\
# Start model download in background\n\
echo "Starting model download in background..."\n\
python -c "from sentence_transformers import SentenceTransformer; SentenceTransformer('\''neuralmind/bert-base-portuguese-cased'\'')" &\n\
# Wait for postgres\n\
until pg_isready -h postgres -p 5432 -U $POSTGRES_USER; do\n\
  echo "Waiting for postgres..."\n\
  sleep 2\n\
done\n\
echo "PostgreSQL started"\n\
exec "$@"' > /wait-for-postgres.sh && chmod +x /wait-for-postgres.sh

# Expose port
EXPOSE $PORT

# Run the application
CMD ["/wait-for-postgres.sh", "sh", "-c", "echo 'Starting Django application...' && python manage.py migrate && echo 'Migrations completed' && python manage.py seed_admin && echo 'Admin seeded' && echo 'Starting server on port $PORT' && python manage.py runserver 0.0.0.0:$PORT"]
