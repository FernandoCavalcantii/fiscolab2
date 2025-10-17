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

# Explicitly install sentence-transformers with specific version
RUN pip install sentence-transformers==2.2.2

# Debug: Check what was installed
RUN echo "=== Checking installed packages ==="
RUN pip list | head -20
RUN echo "=== Looking for sentence-transformers ==="
RUN pip list | grep sentence-transformers || echo "sentence-transformers not found"
RUN echo "=== Trying to import sentence-transformers ==="
RUN echo 'try:\n    import sentence_transformers\n    print("sentence_transformers imported successfully")\nexcept ImportError as e:\n    print("Import error:", e)' > /tmp/test_import.py && python /tmp/test_import.py

# Note: Pre-download moved to runtime to avoid build issues

# Copy project
COPY . .

# Create a script to wait for postgres and pre-download model
RUN echo '#!/bin/bash\n\
echo "Pre-downloading sentence transformer model..."\n\
python -c "from sentence_transformers import SentenceTransformer; SentenceTransformer('\''neuralmind/bert-base-portuguese-cased'\'')" || echo "Model pre-download failed, will download on first use"\n\
until pg_isready -h postgres -p 5432 -U $POSTGRES_USER; do\n\
  echo "Waiting for postgres..."\n\
  sleep 2\n\
done\n\
echo "PostgreSQL started"\n\
exec "$@"' > /wait-for-postgres.sh && chmod +x /wait-for-postgres.sh

# Expose port
EXPOSE 8000

# Run the application
CMD ["/wait-for-postgres.sh", "sh", "-c", "python manage.py migrate && python manage.py seed_admin && python manage.py runserver 0.0.0.0:8000"]
