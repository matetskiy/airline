# Airline System — Dockerized (Angular + FastAPI + Postgres)

Этот README описывает, как запустить и развёртывать проект **Angular (frontend) + FastAPI (backend) + PostgreSQL (db)** с помощью Docker.
Конфигурация учитывает проксирование API через Nginx и начальную инициализацию БД (создание таблиц и учётной записи администратора).

> Стек: Angular → сборка в статику → Nginx, FastAPI (Uvicorn), PostgreSQL.

---

## 📁 Структура репозитория

```text
projwctii/
  backend/                  # FastAPI-приложение (backend/app/...)
    app/
      main.py
      models.py
      db.py
      routers/...
    requirements.txt
    Dockerfile              # из набора файлов
    init_db.py              # из набора файлов (инициализация БД, создание админа)
  frontend/                 # Angular-проект (airline-client)
    src/...
    angular.json
    package.json
    Dockerfile              # из набора файлов
    .docker/
      nginx.conf            # Nginx-конфиг для SPA + /api прокси
    .dockerignore
  docker-compose.yml        # из набора файлов (frontend + backend + db + migrate)
```

> Если у тебя в репозитории ещё есть Next.js — в рамках этого README фронтенд запускается именно **Angular**-проектом.

---

## 🚀 Быстрый старт (Prod / Docker)

Требуется: Docker, Docker Compose.

1. Убедись, что добавлены файлы из набора (Dockerfile’ы, `docker-compose.yml`, `nginx.conf`, `init_db.py`).  
2. В корне `projwctii/` запусти:
   ```bash
   docker compose up --build -d
   ```
   Это запустит:
   - **db** — PostgreSQL `postgres:16-alpine`
   - **migrate** — единоразовый скрипт инициализации БД и создания админа
   - **backend** — FastAPI на `:8000` (виден только в сети docker-compose)
   - **frontend** — Nginx, раздаёт Angular статику и проксирует `/api` → `backend:8000`

3. Открой в браузере:
   - Frontend: http://localhost:8080  
   - Здоровье API (через прокси): http://localhost:8080/api/health

### 🔐 Дефолтные переменные окружения (можно менять в `docker-compose.yml`)

```yaml
DATABASE_URL: "postgresql+psycopg2://postgres:postgres@db:5432/airline_db"

# для миграционного сервиса (создание админа)
ADMIN_EMAIL: "admin@example.com"
ADMIN_PASSWORD: "admin123"
ADMIN_NAME: "Administrator"
```

> Данные Postgres сохраняются в томе `dbdata`.

---

## 🛠️ Dev-режим (HMR для Angular)

Если нужен живой дев-сервер Angular (быстрые правки без сборки прод-статики):

1. Используй `docker-compose.dev.yml` (если добавлял ранее) и запусти:
   ```bash
   docker compose -f docker-compose.dev.yml up
   ```
2. Открой http://localhost:4200  
3. Для backend можно одновременно запустить локально:
   ```bash
   uvicorn backend.app.main:app --reload --host 0.0.0.0 --port 8000
   ```

---

## 🌐 Сеть и проксирование

- **Angular** должен обращаться к API по относительному пути `**/api**`, а не по `http://localhost:8000`.
  - Минимальная правка в `api.service.ts`:
    ```ts
    // было:
    const API = 'http://localhost:8000';
    // стало:
    const API = '/api';
    ```
  - Рекомендуемо вынести в `environments`:
    - `environment.ts` → `apiUrl: 'http://localhost:8000'` (dev)
    - `environment.prod.ts` → `apiUrl: '/api'` (prod через Nginx)
- В `frontend/.docker/nginx.conf` настроено:
  ```nginx
  location / {
    try_files $uri /index.html; # SPA fallback
  }
  location /api/ {
    rewrite ^/api/(.*)$ /$1 break;
    proxy_pass http://backend:8000;
    # + заголовки X-Forwarded-* ...
  }
  ```

---

## 🧱 Миграции/инициализация БД

Сервис **`migrate`** в `docker-compose.yml` выполняет `python backend/init_db.py`:
- создаёт таблицы из `models.py` через `Base.metadata.create_all(...)`;
- создаёт пользователя-администратора, если его ещё нет (email/пароль берутся из переменных окружения).

Можно запустить отдельно:
```bash
docker compose up --build migrate
```

---

## 🔧 Типичные проблемы и их решения

### 1) `npm ci` падает с `EUSAGE` (нет `package-lock.json`)
- Либо **добавь lock-файл** (локально: `npm install` → закоммить `package-lock.json`),  
- Либо в `frontend/Dockerfile` замени `npm ci` → `npm install`:
  ```dockerfile
  RUN npm install --legacy-peer-deps
  ```

### 2) Ошибка Angular budgets: `anyComponentStyle exceeded maximum budget`
- Подними лимиты в `angular.json`:
  ```json
  {
    "type": "anyComponentStyle",
    "maximumWarning": "20kb",
    "maximumError": "30kb"
  }
  ```
- Или временно собрать dev-конфигурацией:
  ```bash
  npm run build -- --configuration development
  ```
- Или сократи крупные CSS (вынеси общие стили в `src/styles.css`, убери тяжёлые фоновые изображения и т.п.).

### 3) CORS/URL
- Используй относительный путь `/api` во фронте — Nginx проксирует в backend. Тогда CORS не нужен на уровне браузера.
- В FastAPI CORS уже открыт, но при проксировании он не мешает и может быть ужат при желании.

### 4) Полная очистка Docker
Полностью удалить контейнеры/образы/сети/тома:
```bash
docker system prune -a --volumes -f
```
> Будь осторожен: удалит **все** тома (включая данные БД).

---

## 🧰 Полезные команды

```bash
# старт/пересборка
docker compose up --build -d
docker compose build frontend --no-cache
docker compose restart frontend

# логи
docker compose logs -f backend
docker compose logs -f frontend
docker compose logs -f db

# остановка/удаление
docker compose down
docker compose down -v   # + удаление томов (данные БД)

# открыть шелл в контейнере
docker compose exec backend sh
docker compose exec frontend sh
```

---

## 📄 Лицензия

Добавь сюда условия лицензии проекта (например, MIT).

---

**Готово!** Если хочешь, я подгоню README под твой стиль/брендинг и добавлю бейджи, скриншоты и раздел «Contributing».

