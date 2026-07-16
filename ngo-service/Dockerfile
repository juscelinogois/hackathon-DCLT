# syntax=docker/dockerfile:1

# ---------- Stage 1: build das dependências ----------
FROM python:3.12-slim AS builder

WORKDIR /app

# Dependências de sistema só necessárias para compilar alguns pacotes
# Python (ex: psycopg2). Ficam só nesta stage, não vão pra imagem final.
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*

# Copia só o requirements.txt primeiro: o cache de camada do Docker só
# invalida essa camada quando as dependências mudam, não a cada
# alteração de código-fonte.
COPY requirements.txt .

RUN pip install --no-cache-dir --user "setuptools<82" "Werkzeug==2.2.3" \
    && pip install --no-cache-dir --user -r requirements.txt

# ---------- Stage 2: imagem final, mínima ----------
FROM python:3.12-slim AS runtime

# Dependência de runtime do psycopg2 (libpq) - só a lib, sem headers
# de compilação, então bem mais leve que a stage de build.
RUN apt-get update && apt-get install -y --no-install-recommends \
    libpq5 \
    && rm -rf /var/lib/apt/lists/* \
    && useradd --create-home --shell /bin/bash appuser

WORKDIR /app

# Pacotes Python instalados no builder (modo --user)
COPY --from=builder /root/.local /home/appuser/.local
COPY . .

RUN chown -R appuser:appuser /app
USER appuser

ENV PATH=/home/appuser/.local/bin:$PATH \
    PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

EXPOSE 8081

# Healthcheck simples - ajuda o Kubernetes (liveness/readiness) e a
# observabilidade a saberem se o serviço está de pé.
HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
    CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8081/')" || exit 1

CMD ["gunicorn", "--bind", "0.0.0.0:8081", "--workers", "2", "--threads", "4", "--timeout", "30", "app:app"]
