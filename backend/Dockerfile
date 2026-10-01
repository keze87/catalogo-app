# Etapa 1: Builder
FROM python:3.12 AS builder
WORKDIR /app

COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt

# Etapa 2: Runtime
FROM python:3.12-slim
WORKDIR /app

ARG APP_VERSION=v1
ENV APP_VERSION=${APP_VERSION}
ENV PYTHONUNBUFFERED=1

RUN useradd -m -u 1000 appuser

COPY --from=builder --chown=appuser:appuser /root/.local /home/appuser/.local
COPY --chown=appuser:appuser . .

ENV PATH="/home/appuser/.local/bin:${PATH}"

USER appuser

EXPOSE 8000
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
