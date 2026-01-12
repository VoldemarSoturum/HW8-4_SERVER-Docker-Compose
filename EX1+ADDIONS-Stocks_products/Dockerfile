FROM python:3.11-slim

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

WORKDIR /app

# gosu нужен для переключения на non-root после подготовки
RUN apt-get update && apt-get install -y --no-install-recommends \
    postgresql-client \
    build-essential \
    libpq-dev \
    gosu \
 && rm -rf /var/lib/apt/lists/*

COPY requirements.txt /app/requirements.txt
RUN pip install --upgrade pip && pip install --no-cache-dir -r /app/requirements.txt

COPY . /app

# Создаём пользователя приложения
RUN useradd -m -u 1000 -s /bin/bash appuser \
 && mkdir -p /app/staticfiles /app/media \
 && chown -R appuser:appuser /app

EXPOSE 8000

CMD ["sh", "-c", "\
  echo \"Waiting for Postgres...\" && \
  until PGPASSWORD=\"$POSTGRES_PASSWORD\" pg_isready -h \"$POSTGRES_HOST\" -p \"$POSTGRES_PORT\" -U \"$POSTGRES_USER\" -d \"$POSTGRES_DB\"; do \
    sleep 1; \
  done && \
  echo \"Postgres is ready\" && \
  \
  # ВАЖНО: named volumes могут быть root:root — приводим права
  chown -R appuser:appuser /app/staticfiles /app/media || true && \
  \
  # дальше работаем как non-root
  gosu appuser python manage.py migrate --noinput && \
  gosu appuser python manage.py collectstatic --noinput || true && \
  gosu appuser gunicorn ${DJANGO_WSGI_MODULE}:application --bind 0.0.0.0:8000 --workers 3 --timeout 60 \
"]
