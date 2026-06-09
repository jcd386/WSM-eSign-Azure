# WSM eSign PDF service — container image
# Published to ghcr.io/jcd386/wsm-esign-api (linux/amd64).
FROM python:3.12-slim

RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        libreoffice libreoffice-writer default-jre && \
    apt-get clean && rm -rf /var/lib/apt/lists/*

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY esign_app.py .

RUN useradd -r -s /bin/false appuser
USER appuser

EXPOSE 8000
CMD ["uvicorn", "esign_app:app", "--host", "0.0.0.0", "--port", "8000"]
