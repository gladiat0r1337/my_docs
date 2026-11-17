---
tags: [devops, docker, docker-compose, linux, networking, troubleshooting, cheatsheet]
---

# 🐳 Docker & Docker Compose — шпаргалка

> **Ментальная модель:** `Dockerfile` описывает, **как собрать образ** → **image** служит шаблоном → **container** — запущенный (или остановленный) экземпляр образа → `compose.yaml` описывает, **как запускать несколько сервисов вместе**. Образ сам по себе не запускается.

## 1. Образы (images)

```bash
docker build -t my-app:v1 .            # собрать из Dockerfile; . — контекст сборки
docker build --no-cache -t my-app:v1 . # не использовать кэш слоёв при сборке
docker build -f Dockerfile.dev -t my-app:dev .  # другой Dockerfile
docker image ls                       # локальные образы (аналог docker images)
docker pull nginx:alpine              # скачать образ
docker tag my-app:v1 registry.example.com/team/my-app:v1
docker push registry.example.com/team/my-app:v1
docker image inspect my-app:v1         # метаданные образа
docker rmi my-app:v1                  # удалить локальный образ, если возможно
```

- `-t` / `--tag` задаёт имя и тег; `-f` / `--file` — путь к Dockerfile.
- `.` в `docker build ... .` — **контекст сборки**, а не обязательно местонахождение Dockerfile.
- `--no-cache` отключает использование кэша слоёв при сборке, но не означает автоматического обновления всех базовых образов. Для проверки обновлённой базы пригодится `--pull`.
- `docker push` отправляет образ с **указанным тегом**; для приватного реестра может потребоваться `docker login`.
- `latest` — обычный тег, а не гарантия самой новой или стабильной версии. Для предсказуемого деплоя используй конкретный тег или digest.

## 2. Контейнеры

```bash
docker run -d --name my-nginx -p 127.0.0.1:8080:80 nginx:alpine
docker ps                              # работающие контейнеры
docker ps -a                           # все, включая остановленные
docker stop my-nginx                   # корректная остановка
docker start my-nginx                  # запустить существующий остановленный
docker restart my-nginx                # перезапустить
docker rm my-nginx                     # удалить остановленный
docker run --rm alpine:3.20 echo hello # одноразовый контейнер
```

Флаги `docker run`:

| Флаг | Назначение |
|---|---|
| `-d` / `--detach` | фоновый режим |
| `--name NAME` | имя контейнера |
| `-p HOST_IP:HOST_PORT:CONTAINER_PORT` | публикация порта на хосте; без `HOST_IP` часто на всех интерфейсах |
| `-v SOURCE:TARGET[:ro]` | bind mount (путь на хосте) или именованный том; `:ro` — только чтение |
| `-e KEY=VALUE` | переменная окружения внутри контейнера |
| `--env-file FILE` | передать переменные из файла внутрь контейнера |
| `--rm` | автоматически удалить контейнер после завершения |
| `--network NETWORK` | подключить контейнер к сети |
| `--restart unless-stopped` | перезапуск после сбоя/перезагрузки Docker, кроме случаев ручной остановки |

> ⚠️ `-p 8080:80` может открыть сервис снаружи хоста. `-p 127.0.0.1:8080:80` ограничивает доступ loopback хоста (обычно полезно, когда Nginx запущен на самой VM). `EXPOSE` в Dockerfile **не публикует** порт.

`docker stop` посылает настроенный сигнал остановки (обычно SIGTERM), ждёт grace period (по умолчанию обычно 10 секунд для Linux), затем может послать SIGKILL. `docker rm -f` — принудительное удаление, используй осторожно.

## 3. Логи, диагностика и ресурсы

```bash
docker logs --tail 100 my-nginx        # последние 100 строк stdout/stderr
docker logs -f my-nginx               # следить за новыми логами
docker exec -it my-nginx sh           # оболочка внутри работающего контейнера
docker exec my-nginx env              # переменные окружения: могут содержать секреты!
docker inspect my-nginx               # технические данные в JSON; тоже могут быть секреты
docker stats --no-stream              # разовый снимок CPU/RAM

docker inspect -f '{{.State.Status}}' my-nginx
docker inspect -f '{{.State.ExitCode}}' my-nginx
docker inspect -f '{{json .NetworkSettings.Networks}}' my-nginx
```

- `docker exec` работает только с **работающим** контейнером. `sh` обычно доступнее, чем `bash`.
- `-i` оставляет STDIN открытым; `-t` выделяет псевдотерминал.
- `docker logs` читает stdout/stderr при поддерживаемом драйвере логирования; логи, записанные только в файл внутри контейнера, могут туда не попасть.
- `Up` ≠ `healthy` ≠ «бизнес-функция работает».
- Не публикуй вывод `env`, полного `inspect` и `docker compose config` без проверки: там могут быть пароли и токены.

## 4. Docker Compose

> Современный синтаксис: `docker compose` (с пробелом). По умолчанию Compose обнаруживает `compose.yaml`/`compose.yml` и поддерживает старые имена `docker-compose.yml`/`.yaml`.

```bash
docker compose config --quiet         # проверить итоговую модель, не печатая конфиг
docker compose config --services      # имена сервисов
docker compose ps -a                  # контейнеры проекта, включая остановленные
docker compose up -d                  # создать/запустить стек в фоне
docker compose up -d --build app      # пересобрать сервис app, если есть build
docker compose up -d --no-deps app    # не запускать зависимости app
docker compose up -d --force-recreate app  # пересоздать контейнер app
docker compose up -d --wait           # дождаться running/healthy (при наличии healthcheck)
docker compose logs -f --tail 100 app # логи сервиса app, не имя контейнера
docker compose exec app sh            # оболочка в запущенном сервисе
docker compose pull                   # получить образы из реестра
docker compose restart app            # перезапуск без применения изменений конфигурации
docker compose down                   # удалить контейнеры и сети проекта
docker compose -f compose.yaml -p mailservice-dev ps  # явный файл и имя проекта
```

- `up` создаёт контейнеры при необходимости и пересоздаёт при изменении образа/конфигурации. `restart` **не применяет** обновлённые переменные окружения или Compose-конфигурацию.
- `--build` имеет смысл для сервисов с `build:`. При деплое готового образа из Registry чаще используется `image:` и пересборка на VM не нужна.
- В `docker compose logs app` указывается **имя сервиса** из YAML (`app`), а в `docker logs` — имя/ID **контейнера**.
- `docker compose down -v` удаляет **именованные тома, объявленные в Compose**, и привязанные анонимные тома. **Опасно для БД.** External volumes не удаляются.
- `docker compose -p dev` и `-p test` помогают разделять имена ресурсов проектов, но не дают полноценной изоляции безопасности.

## 5. Dockerfile: пример multi-stage для Go

**Файл `Dockerfile`** (отдельно от `compose.yaml`):

```dockerfile
# пример: укажи поддерживаемую версию Go для своего приложения
FROM golang:1.26-alpine AS builder
WORKDIR /build
COPY go.mod go.sum ./
RUN go mod download
COPY . .
# ./cmd/app — пример каталога, в котором находится main package
RUN CGO_ENABLED=0 GOOS=linux go build -o /build/app ./cmd/app

FROM alpine:3.23
RUN adduser -D appuser
WORKDIR /app
COPY --from=builder /build/app ./app
USER appuser
EXPOSE 8080
CMD ["./app"]
```

- `FROM` — базовый образ; `AS builder` — имя этапа.
- `WORKDIR` — рабочая директория внутри образа.
- `COPY` — копирование файлов из **контекста** сборки.
- `RUN` — команда **во время сборки образа**.
- `COPY --from=builder` — перенос файла из этапа builder в финальный образ.
- `USER` — пользователь, от имени которого запускается процесс.
- `EXPOSE` — декларация порта, **не** делает `-p`.
- `CMD` — команда **при запуске контейнера**; JSON/exec-форма помогает корректной обработке сигналов, если приложение — основной процесс.
- `CGO_ENABLED=0` отключает cgo; результат и потребность в системных библиотеках зависят от программы.
- Multi-stage часто уменьшает образ, но конкретный размер заранее не гарантирован. Версии образов обновляй по совместимости и поддержке.

**Файл `.dockerignore`** (не отправлять лишнее в контекст):

```gitignore
.git
.env
*.log
node_modules
.DS_Store
```

## 6. `compose.yaml`: app + PostgreSQL + Redis

**Учебный пример**: приложение должно реально поддерживать переменные `DB_HOST`, `DB_USER`, `DB_PASSWORD`, `DB_NAME`, `REDIS_HOST` и слушать порт 8080. Для Spring Boot имена переменных другие (см. ниже).

```yaml
services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    restart: unless-stopped
    ports:
      - "127.0.0.1:8080:8080"  # Nginx работает на той же VM
    environment:
      DB_HOST: postgres
      DB_USER: ${DB_USER:?DB_USER is required}
      DB_PASSWORD: ${DB_PASS:?DB_PASS is required}
      DB_NAME: ${DB_NAME:?DB_NAME is required}
      REDIS_HOST: redis
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_started
    networks:
      - backend-net

  postgres:
    image: postgres:16-alpine
    restart: unless-stopped
    environment:
      POSTGRES_USER: ${DB_USER:?DB_USER is required}
      POSTGRES_PASSWORD: ${DB_PASS:?DB_PASS is required}
      POSTGRES_DB: ${DB_NAME:?DB_NAME is required}
    volumes:
      - pg_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U \"$$POSTGRES_USER\" -d \"$$POSTGRES_DB\""]
      interval: 5s
      timeout: 5s
      retries: 5
      start_period: 10s
    networks:
      - backend-net

  redis:
    image: redis:7-alpine
    restart: unless-stopped
    command: ["redis-server", "--appendonly", "yes"]
    volumes:
      - redis_data:/data
    networks:
      - backend-net

volumes:
  pg_data:
  redis_data:

networks:
  backend-net:
    driver: bridge
```

**Отдельный файл `.env.example`** — только учебные значения, без реальных секретов:

```dotenv
DB_USER=mailservice
DB_PASS=replace_me_for_local_lab
DB_NAME=mailservice
```

- Реальный `.env` добавь в `.gitignore`, не коммить секреты. В проде используй принятый командой способ управления секретами.
- `depends_on: service_healthy` ждёт успешного **healthcheck**, но не гарантирует готовность всех бизнес-операций. `service_started` ждёт запуска, не готовности Redis.
- `pg_isready` выше проверяет готовность PostgreSQL локально **в контейнере БД**. Он не подтверждает правильность пароля и доступность по сети **из app**.
- `$$POSTGRES_USER` нужно, чтобы Compose не заменил `$POSTGRES_USER` при чтении YAML, а передал `$POSTGRES_USER` внутрь контейнера.
- `postgres` и `redis` доступны внутри общей Compose-сети по именам сервисов; не обязательно публиковать их порты на хосте.
- Обычная `bridge`-сеть — это **сегментация контейнеров**, не полноценный DMZ/изолированный контур; исходящий доступ обычно возможен. `internal: true` сильнее ограничивает внешнюю связность, но требует продумывания маршрутов, обновлений и доступа сервисов.
- Для уже инициализированного тома PostgreSQL переменные `POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_DB` **не переинициализируют** существующую БД/пользователя.
- `container_name` необязателен; без него Compose сам создаёт имена с префиксом проекта. Жёсткое имя может мешать масштабированию и нескольким окружениям.
- Для деплоя образов из Registry вместо `build:` у `app` используй, например, `image: registry.example.com/team/mailservice:v1`.

**Если приложение на Spring Boot**, вместо `DB_*` для datasource обычно нужны:

```yaml
environment:
  SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/mailservice
  SPRING_DATASOURCE_USERNAME: ${DB_USER:?required}
  SPRING_DATASOURCE_PASSWORD: ${DB_PASS:?required}
```

## 7. `.env` vs `env_file` vs `environment`

| Механизм | Что делает |
|---|---|
| `.env` рядом с проектом | источник значений `${VAR}` для **подстановки в Compose** (с учётом приоритетов) |
| `env_file: .env` внутри сервиса | передаёт переменные **внутрь контейнера** |
| `environment:` | явно задаёт переменные **внутри контейнера**; можно использовать `${VAR}` |
| `docker compose --env-file .env.test up -d` | выбирает файл для подстановки при запуске Compose |
| `docker run --env-file .env ...` | передаёт переменные непосредственно контейнеру |

> ⚠️ `docker compose config` и `docker inspect` могут раскрывать подставленные секреты; не вставляй их полный вывод в тикеты и CI-логи.

## 8. Сети и тома

```bash
docker network ls                      # Docker-сети
docker network inspect NETWORK         # подключённые контейнеры и параметры сети
docker volume ls                       # тома
docker volume inspect VOLUME           # информация о конкретном томе
docker system df                       # использование места Docker
```

- Контейнеры одной пользовательской bridge-сети обычно находят друг друга по **именам сервисов/сетевым алиасам**, а не через `localhost`.
- `127.0.0.1` **в контейнере** означает сам контейнер; `127.0.0.1` **на VM** означает VM.
- `ports:` публикует порт на хосте. Для общения двух сервисов в одной Docker-сети публикация порта обычно **не нужна**.
- **Bind mount**: явно выбранный путь хоста, например `./data:/data`.
- **Named volume**: управляемое Docker хранилище, например `pg_data:/var/lib/postgresql/data`.
- Данные тома переживают пересоздание контейнера. Том **не заменяет резервные копии**.

## 9. Диагностика: от симптома к причине

| Симптом | Что проверять |
|---|---|
| 502 Bad Gateway | Nginx error log → состояние upstream → контейнер → логи приложения |
| Контейнер `Exited (1)` | `docker logs`, exit code, env/config, зависимости |
| `Connection refused` к БД | слушает ли порт, верны ли адрес и имя сервиса, доступен ли TCP |
| PostgreSQL `accepting connections` локально | это **не** доказательство TCP-доступа из другого контейнера |
| `password authentication failed` | реальные учётные данные, метод аутентификации, состояние старого volume |
| Приложение `Up`, но не работает | healthcheck, HTTP smoke test, логи, бизнес-эндпоинты |
| 404 на `/` | существует ли маршрут `/`, не переписал ли путь Nginx |

```bash
# Проверка HTTP через Nginx
curl -i http://127.0.0.1:8080/actuator/health  # только если этот порт ведёт к нужному endpoint

# Если Nginx на VM слушает порт 80
curl -i http://127.0.0.1/actuator/health

# Проверка доступности порта (там, где установлен nc)
nc -vz HOST 5432

# Проверка DNS-имени (там, где доступен getent)
getent hosts postgres

# Проверка готовности Postgres через TCP loopback внутри БД
docker compose exec postgres pg_isready -h 127.0.0.1 -p 5432
```

**Алгоритм:** наблюдаем симптом → формируем гипотезу → выбираем проверку, которая её подтверждает/опровергает → исправляем только установленную причину → проверяем HTTP и бизнес-функцию → фиксируем выводы.

## 10. Очистка — осторожно на рабочей VM

```bash
docker system df             # сначала оценить дисковое пространство
docker image prune           # удалить dangling-образы
docker container prune       # удалить остановленные контейнеры
docker system prune          # удалить неиспользуемые контейнеры/сети/dangling images/cache
docker system prune -a       # также удалить ВСЕ неиспользуемые образы
docker system prune --volumes # также удалить неиспользуемые АНОНИМНЫЕ тома
docker volume prune -a       # удалить неиспользуемые тома, в том числе ИМЕНОВАННЫЕ
```

**Не запускай `prune` и `compose down -v` на общей/рабочей VM без понимания того, какие ресурсы и данные будут удалены.**

## 11. Полезные ссылки

- [Docker CLI reference](https://docs.docker.com/reference/cli/docker/)
- [Docker Compose reference](https://docs.docker.com/compose/)
- [Docker networks](https://docs.docker.com/engine/network/)
- [Compose environment variables](https://docs.docker.com/compose/how-tos/environment-variables/)
- [MIT Missing Semester 2026](https://missing.csail.mit.edu/2026/)
