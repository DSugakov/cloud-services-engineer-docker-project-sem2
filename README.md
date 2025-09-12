# Проектная работа: Docker-контейнеризация и хранение данных

Этот репозиторий содержит учебное приложение, разделённое на два сервиса:
- backend — API на Go (порт 8081)
- frontend — SPA на Vue 3, собирается и отдаётся nginx (порт 8080 внутри контейнера, снаружи также 8080)

Цель работы — упаковать приложение в Docker-образы, оркестрировать через Docker Compose, а также внедрить базовые практики безопасности и сканирование уязвимостей (Trivy) в CI.

## Стек
- Go 1.17 (backend)
- Vue 3 + Vue CLI 4 (frontend)
- Nginx (отдача статики SPA)
- Docker, Docker Compose
- GitHub Actions (CI)
- Trivy (сканирование уязвимостей образов)

## Структура репозитория
```
backend/                # исходники бэкенда (Go)
  Dockerfile            # multi-stage сборка → alpine runtime + healthcheck
  .dockerignore         # исключения для сборки
frontend/               # исходники фронтенда (Vue)
  Dockerfile            # сборка на node → runtime на nginx:8080 (non-root)
  nginx.conf            # безопасный конфиг nginx для SPA (listen 8080)
  .dockerignore         # исключения для сборки
.github/workflows/
  deploy.yaml           # CI: сборка и пуш образов + Trivy-сканирование

docker-compose.yml      # оркестрация backend+frontend
```

## Быстрый старт (Docker Compose)
Требуется установленный Docker и Docker Compose v2.

```bash
# Продакшн (с балансировщиком)
docker compose --profile prod build
docker compose --profile prod up -d

# Разработка (горячая перезагрузка)
docker compose --profile dev up -d

# Горизонтальное масштабирование
docker compose --profile scale up -d --scale frontend=3 --scale backend=2

# Просмотр логов
docker compose logs -f
```

После запуска:
- Frontend: http://localhost:80 (через nginx-lb) или http://localhost:8080 (dev)
- Backend API: http://localhost:8081
  - Healthcheck: http://localhost:8081/health (200 OK)
  - Metrics (Prometheus): http://localhost:8081/metrics

**Размеры образов (примерные):**
- Backend: ~15MB (alpine + статический бинарник)
- Frontend: ~25MB (nginx:alpine + собранные статические файлы)
- Nginx LB: ~15MB (nginx:alpine)

Остановить и удалить контейнеры:
```bash
docker compose down
```

## Конфигурация фронтенда (API URL)
Фронтенд использует переменную окружения сборки `VUE_APP_API_URL` (см. `frontend/src/services/api.service.ts`).
В `docker-compose.yml` она пробрасывается как build-arg и указывает на внутреннее имя сервиса бэкенда:
```
VUE_APP_API_URL=http://backend:8081
```
Если нужно подтянуть API с другого адреса при сборке фронтенда локально, можно сделать так:
```bash
# Пример локальной сборки фронтенда c кастомным API URL
docker build --build-arg VUE_APP_API_URL=http://localhost:8081 -t my-frontend ./frontend
```

## Детали реализации Docker
### Backend (Go)
- Multi-stage: сборка на `golang:1.17-alpine`, рантайм — `alpine:3.19` c непривилегированным пользователем `app`.
- Бинарник сборки: `CGO_ENABLED=0`, флаги `-trimpath -ldflags "-s -w"` для уменьшения размера.
- HEALTHCHECK запрашивает `http://127.0.0.1:8081/health` (wget), порт 8081.

### Frontend (Vue + Nginx)
- Multi-stage: сборка на `node:16-alpine`, рантайм — `nginx:1.25-alpine`.
- Nginx слушает 80 и запускается от пользователя `www` (не root) — в dev-режиме используется порт 8080.
- Минимальный безопасный конфиг nginx в `frontend/nginx.conf` (SPA routing, базовые security headers, gzip).

### Docker Compose
- **Профили:**
  - `prod`: продакшн с балансировщиком nginx (порт 80)
  - `dev`: разработка с горячей перезагрузкой (порты 8080/8081)
  - `scale`: горизонтальное масштабирование с балансировкой
- **Сервисы:** `backend`, `frontend`, `nginx-lb`, `backend-dev`, `frontend-dev`
- **Сеть:** `app` (bridge)
- **Порты:**
  - `80:80` — nginx-lb (prod/scale)
  - `8080:8080` — frontend-dev
  - `8081:8081` — backend/backend-dev
- Безопасные параметры контейнеров:
  - `read_only: true` — корневая ФС только для чтения
  - `security_opt: ["no-new-privileges:true"]`
  - `cap_drop: ["ALL"]` — убраны все capabilities
  - `tmpfs` для временных директорий (nginx cache/run, /tmp)
- Ограничения ресурсов (best effort):
  - лимиты CPU и памяти через `deploy.resources.limits` (поддержка зависит от окружения)
- Профили:
  - профиль `prod`: `--profile prod` при build/up
- **Volumes:**
  - `frontend_logs` монтируется в `/var/log/nginx` для сохранения логов между перезапусками
- **Secrets:**
  - `api_key` из файла `./secrets/api_key.txt` (демонстрация Docker Secrets)
- **Масштабирование:**
  - `docker compose --profile scale up --scale frontend=3 --scale backend=2`
  - nginx-lb автоматически балансирует нагрузку между репликами

## Безопасность
В проекте применены практики, уменьшающие площадь атаки:
- Лёгкие базовые образы (alpine)
- Запуск процессов от непривилегированных пользователей (`app`, `www`)
- Read-only файловая система + tmpfs для временных путей
- `no-new-privileges` и `cap_drop: ALL` в контейнерах
- Открыты только необходимые порты
- Исключение лишних файлов из контекста сборки через `.dockerignore`
- Сканирование уязвимостей образов Trivy в CI

> Важно: не храните секреты в Dockerfile/репозитории. Используйте секреты GitHub Actions/хранилища секретов.

## CI/CD (GitHub Actions)
Workflow: `.github/workflows/deploy.yaml`
- Сборка и пуш образов в Docker Hub (теги `latest`) для бэкенда и фронтенда
- Логин в Docker Hub через секреты `DOCKER_USER`, `DOCKER_PASSWORD`
- Trivy-сканирование обоих образов, публикация отчётов как artifacts

Пример секретов, которые должны быть заданы в репозитории GitHub:
- `DOCKER_USER`
- `DOCKER_PASSWORD`

## Разработка локально (опционально)
Можно запускать только бэкенд локально (без Docker):
```bash
cd backend
go mod download
go run ./cmd/api
# API будет доступно на :8081
```

Сборка фронтенда локально (потребуется Node.js):
```bash
cd frontend
npm ci
VUE_APP_API_URL=http://localhost:8081 npm run build
# локальная разработка
VUE_APP_API_URL=http://localhost:8081 npm run serve
```

## Устранение проблем
- Порты заняты: измените проброс портов в `docker-compose.yml` или освободите занятые
- API недоступно с фронтенда: проверьте `VUE_APP_API_URL` (должно быть `http://backend:8081` в Compose)
- Ошибки сборки npm: очистите кэш `npm cache clean --force` и пересоберите
- Trivy находит уязвимости: анализируйте отчёт артефактов, обновляйте базовые образы/зависимости

## Оценка по чек-листу
- Docker: multi-stage для обоих сервисов, лёгкие базовые образы, non-root, порты корректные, лишние инструменты отсутствуют, теги есть, `.dockerignore` добавлен, healthcheck настроен (frontend+backend).
- Docker Compose: определены оба сервиса, depends_on, внутренняя сеть, политика перезапуска, лимиты ресурсов, профили, volume для логов nginx.
- Безопасность: non-root, ограничение прав, read-only, cap_drop, сканирование Trivy, отсутствие секретов в образах.
