FROM python:3.12-slim

WORKDIR /app

# Copy requirements first for better caching
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application files explicitly
COPY app.py .
COPY crypto_tools.py .
# COPY .env .          ← ELIMINAR esta línea (la API key ahora viene del K8s Secret)

EXPOSE 80


CMD ["python", "app.py"]

