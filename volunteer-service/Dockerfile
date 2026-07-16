# syntax=docker/dockerfile:1

# ---------- Stage 1: build das dependências ----------
FROM python:3.12-slim AS builder

WORKDIR /app

COPY requirements.txt .

# Este serviço usa DynamoDB (boto3), então normalmente não precisa de
# libs de compilação como o ngo-service (que usa psycopg2). Ainda
# assim, instalamos --user para manter o mesmo padrão de cópia entre
# stages.
RUN pip install --no-cache-dir --user "setuptools<82" "Werkzeug==2.2.3" \
    && pip install --no-cache-dir --user -r requirements.txt

# ---------- Stage 2: imagem final, mínima ----------
FROM python:3.12-slim AS runtime

RUN useradd --create-home --shell /bin/bash appuser

WORKDIR /app

COPY --from=builder /root/.local /home/appuser/.local
COPY . .

RUN chown -R appuser:appuser /app
USER appuser

ENV PATH=/home/appuser/.local/bin:$PATH \
    PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

EXPOSE 8083

HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
    CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8083/')" || exit 1

CMD ["gunicorn", "--bind", "0.0.0.0:8083", "--workers", "2", "--threads", "4", "--timeout", "30", "app:app"]
