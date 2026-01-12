# Домашнее задание к лекции «Docker Compose» — запуск Django+DRF проекта в Docker
### ОСНОВА ЗАДАНИЯ
### HW7-6_DJANGO-CRUD_in_DRF

Этот документ описывает, как запустить учебный проект **Django + DRF** в **Docker Compose** (Docker Desktop на Windows) в связке из **трёх контейнеров**:

- **postgres** — база данных PostgreSQL
- **backend** — Django, запущенный через **Gunicorn**
- **nginx** — reverse-proxy + отдача статики/медиа

Также добавлены:
- `healthcheck` для Postgres и backend
- базовое усиление безопасности: `no-new-privileges`, `read_only` для nginx, запуск nginx под non-root, `tmpfs` для временных директорий

---

## 1) Требования

- **Docker Desktop for Windows** (режим WSL2 приветствуется)
- `docker compose` доступен в терминале (PowerShell / CMD)

Проверка:
```bash
docker --version
docker compose version
```

---

## 2) Структура дополнений в проекте

В каталоге проекта (там, где лежит `docker-compose.yaml`) должны присутствовать:

```text
EX1+ADDIONS-Stocks_products/
├─ docker-compose.yaml
├─ Dockerfile
├─ .env
└─ nginx/
   ├─ nginx.conf
   └─ conf.d/
      └─ app.conf
```

---

## 3) Переменные окружения (.env)

Создай файл `.env` рядом с `docker-compose.yaml`.

Пример:
```env
# --- Postgres ---
POSTGRES_DB=netology_stocks_products
POSTGRES_USER=netology_stocks
POSTGRES_PASSWORD=change_me_please

# --- Django ---
DJANGO_SECRET_KEY=change_me_to_real_secret_key
DJANGO_DEBUG=0
DJANGO_ALLOWED_HOSTS=localhost,127.0.0.1
DJANGO_WSGI_MODULE=stocks_products.wsgi
```

### Как сгенерировать DJANGO_SECRET_KEY
Через контейнер backend (если локально Django не установлен):
```bash
docker compose run --rm backend python -c "from django.core.management.utils import get_random_secret_key; print(get_random_secret_key())"
```

---

## 4) Настройки Django (settings.py)

Чтобы проект работал в контейнерах, значения должны читаться из переменных окружения.

Рекомендуемые фрагменты:

```python
import os

SECRET_KEY = os.getenv("DJANGO_SECRET_KEY", "unsafe-dev-key")
DEBUG = os.getenv("DJANGO_DEBUG", "0") == "1"

ALLOWED_HOSTS = [
    h.strip() for h in os.getenv(
        "DJANGO_ALLOWED_HOSTS", "localhost,127.0.0.1"
    ).split(",") if h.strip()
]

# Для работы за reverse-proxy (полезно при HTTPS в будущем)
USE_X_FORWARDED_HOST = True
SECURE_PROXY_SSL_HEADER = ("HTTP_X_FORWARDED_PROTO", "https")
```

> Важно: в файле `settings.py` должно быть **ровно одно** определение `ALLOWED_HOSTS`, иначе Django будет брать последнее и игнорировать env.

---

## 5) Конфигурация Nginx

### nginx/nginx.conf
Особенность для `read_only: true`: все временные каталоги и PID переносим в `/tmp` (tmpfs), чтобы nginx мог писать туда без доступа к файловой системе контейнера.

Пример:
```nginx
worker_processes  1;
pid /tmp/nginx.pid;

events { worker_connections 1024; }

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    sendfile on;
    keepalive_timeout 65;

    client_body_temp_path /tmp/client_body_temp;
    proxy_temp_path       /tmp/proxy_temp;
    fastcgi_temp_path     /tmp/fastcgi_temp;
    uwsgi_temp_path       /tmp/uwsgi_temp;
    scgi_temp_path        /tmp/scgi_temp;

    include /etc/nginx/conf.d/*.conf;
}
```

### nginx/conf.d/app.conf
Nginx слушает **8080** внутри контейнера (так как nginx запускается **non-root**), а наружу пробрасывается порт **80**.

```nginx
upstream django_upstream {
    server backend:8000;
}

server {
    listen 8080;

    client_max_body_size 20m;

    # Красивая страница на /
    location = / {
        default_type text/html;
        return 200 '<!doctype html>
<html lang="ru"><head><meta charset="utf-8"><meta name="viewport" content="width=device-width,initial-scale=1">
<title>Stocks Products</title>
<style>
body{font-family:system-ui,-apple-system,Segoe UI,Roboto,Arial,sans-serif;background:#0b1220;color:#e5e7eb;margin:0}
.wrap{min-height:100vh;display:flex;align-items:center;justify-content:center;padding:24px}
.card{width:min(720px,100%);background:#111a2e;border:1px solid #1f2a44;border-radius:16px;padding:28px;box-shadow:0 10px 30px rgba(0,0,0,.35)}
h1{margin:0 0 10px;font-size:26px}
p{margin:0 0 18px;color:#cbd5e1;line-height:1.5}
.grid{display:grid;grid-template-columns:repeat(3,minmax(0,1fr));gap:12px;margin-top:14px}
a.btn{display:flex;align-items:center;justify-content:center;text-decoration:none;background:#1f2a44;border:1px solid #2b3a5c;color:#e5e7eb;padding:14px 12px;border-radius:12px;font-weight:600}
a.btn:hover{background:#273455}
.muted{margin-top:14px;font-size:13px;color:#94a3b8}
code{background:#0b1220;border:1px solid #1f2a44;padding:2px 6px;border-radius:8px}
@media(max-width:640px){.grid{grid-template-columns:1fr}}
</style></head>
<body><div class="wrap"><div class="card">
<h1>Stocks Products</h1><p>Выбери, куда перейти:</p>
<div class="grid">
<a class="btn" href="/api/v1/">API</a>
<a class="btn" href="/admin/">Admin</a>
<a class="btn" href="/api/v1/stocks/">Stocks</a>
</div>
<div class="muted">Запущено через <code>Nginx → Gunicorn → Django</code></div>
</div></div></body></html>';
    }

    location /static/ {
        alias /var/www/static/;
        expires 30d;
        add_header Cache-Control "public";
    }

    location /media/ {
        alias /var/www/media/;
    }

    location / {
        proxy_pass http://django_upstream;
        proxy_set_header Host $http_host;
        proxy_set_header X-Forwarded-Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

---

## 6) Запуск проекта

Из каталога `EX1+ADDIONS-Stocks_products`:

### Первый запуск (сборка + старт)
```bash
docker compose up -d --build
```

### Проверка состояния
```bash
docker compose ps
```

### Просмотр логов
```bash
docker compose logs -f --tail=100
```

### Остановка
```bash
docker compose down
```

---

## 7) Доступ к сервисам

- Главная страница (Nginx): `http://localhost/`
- API root: `http://localhost/api/v1/`
- Пример endpoint: `http://localhost/api/v1/stocks/`
- Админка: `http://localhost/admin/`

---

## 8) Безопасность: что сделано

- `no-new-privileges:true` для всех сервисов — запрещает эскалацию привилегий
- Nginx запускается как non-root (`user: "101:101"`)
- Nginx `read_only: true` + `tmpfs` для `/tmp`, `/var/run`, `/var/cache/nginx`
- PID перенесён в `/tmp/nginx.pid`
- Postgres и backend **не публикуются наружу** (только nginx имеет `ports:`)

---

## 9) Типовые проблемы и решения

### 9.1. Nginx: `unknown directive "﻿worker_processes"`
Файл `nginx.conf` сохранён в **UTF-8 with BOM**. Пересохрани как **UTF-8 (без BOM)**.

### 9.2. Nginx падает: `open() "/var/run/nginx.pid" failed`
При `read_only` nginx не может писать PID. Решение:
- `pid /tmp/nginx.pid;` в `nginx.conf`
- `tmpfs: /tmp` (и при необходимости `/var/run`)

### 9.3. Django отдаёт 400 (DisallowedHost)
Проверь:
- в `settings.py` не должно быть второго `ALLOWED_HOSTS`
- `DJANGO_ALLOWED_HOSTS` в `.env` содержит `localhost,127.0.0.1`
- Django читает env через `os.getenv(...)`

---

## 10) Итог

Проект запускается одной командой:
```bash
docker compose up -d --build
```

и доступен по адресу:
`http://localhost/`
