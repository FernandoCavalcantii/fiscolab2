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

# Debug: Check if sentence-transformers is installed
RUN pip list | grep sentence-transformers || echo "sentence-transformers not found"
RUN python -c "try: import sentence_transformers; print('sentence_transformers imported successfully'); except ImportError as e: print('Import error:', e)"

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
