# redis_service_prod

Общий (shared) Redis для всех проектов на этом сервере:
`vera_agent_service` и любых будущих. Один инстанс, один docker-compose —
вместо того, чтобы каждый проект поднимал свой Redis отдельно. Тот же
принцип, что и в соседнем `rabbitmq_service_prod`.

## Образ: `redis/redis-stack:7.4.0-v8`

Не голый `redis`, а Stack-сборка — потому что `vera_agent_service`
использует RediSearch (`langgraph-checkpoint-redis`, чекпоинты
LangGraph, `FT._LIST` и т.п. — обычный Redis эти команды не понимает).
Заодно в комплекте — RedisInsight (UI) на порту `8001`, без отдельного
контейнера.

Актуальное ядро Redis (8.x, `redis:8.8-alpine`) уже умеет то же самое
из коробки (поиск/JSON смержены в core с Redis 8) и активнее
развивается — но переезд на него требует отдельной проверки
совместимости FT.*-команд с `langgraph-checkpoint-redis`. Это
осознанно отложено, не задача этого репозитория.

## Архитектурное решение: адресация по host:port, без общей docker-сети

Тот же паттерн, что и в `rabbitmq_service_prod`: потребители
подключаются по `host:port` (`REDIS_HOST`/`REDIS_PORT` в их `.env`), а
не через общую внешнюю docker-сеть. Каждый проект остаётся независимым
docker-compose, переезд Redis на другой сервер — правка `.env`
потребителя, а не сетевой топологии.

## Запуск

```
docker compose up -d
```

Проверить, что сервис здоров:

```
docker compose ps
docker exec shared_redis redis-cli --no-auth-warning -a "$REDIS_PASSWORD" ping
```

RedisInsight (UI): `http://localhost:8001` (порт опубликован только на
`127.0.0.1` — см. ниже). При первом заходе — добавить подключение
вручную: host `localhost`, port `6379`, пароль из `.env`
(`REDIS_PASSWORD`).

## Как подключить проект-потребитель

Прописать в `.env` потребителя тот же `REDIS_PASSWORD`, что и в `.env`
этого репозитория, и адрес:

- **Процесс/контейнер на этом же хосте** — `host.docker.internal:6379`
  (Docker Desktop поддерживает нативно; для Linux-хоста в compose
  потребителя нужна запись `extra_hosts: - "host.docker.internal:host-gateway"`,
  как уже сделано в `vera_agent_service` для RabbitMQ).
- **Скрипт/тесты прямо на хосте (не в контейнере)** — `localhost:6379`.
- **Другой физический сервер (реальный прод)** — DNS-имя/IP этого
  сервера вместо `host.docker.internal`.

Пример (`vera_agent_service/.env`):

```
REDIS_HOST=host.docker.internal
REDIS_PORT=6379
REDIS_PASSWORD=<пароль>
REDIS_DB=0
```

## Изоляция между проектами

Redis не даёт полноценных vhost'ов, как RabbitMQ, но есть логические
базы `0..15` (`SELECT`/`REDIS_DB`) — этого достаточно для изоляции
keyspace (включая RediSearch-индексы, они тоже per-DB) между разными
проектами на одном инстансе:

- `vera_agent_service` — `REDIS_DB=0`;
- следующий проект — `REDIS_DB=1`, и т.д.

Если проекту нужна более строгая изоляция (свой пароль/ACL, не общий
`REDIS_PASSWORD`) — завести отдельного ACL-пользователя:

```
docker exec shared_redis redis-cli --no-auth-warning -a "$REDIS_PASSWORD" \
  ACL SETUSER my_project_user on ">пароль" "~my_project:*" "+@all" -"@admin"
```

## Резервное копирование

Данные — в именованном volume `redis_data` (RDB + AOF, `--appendonly yes`
в `docker-compose.yml`). Бэкап:

```
docker run --rm -v redis_service_prod_redis_data:/data -v "$PWD":/backup alpine \
  tar czf /backup/redis_data_backup.tar.gz -C /data .
```

(имя volume может отличаться префиксом проекта — проверить `docker volume ls`).

## Безопасность / прод-чеклист

- [x] `REDIS_PASSWORD` в `.env` — сгенерированный пароль, авторизация
      обязательна (`--requirepass`), не дефолтный пустой пароль, как
      было в `vera_agent_service/.env.example` изначально.
- [x] Порт `8001` (RedisInsight) забинден на `127.0.0.1`, наружу не
      публикуется — доступ через SSH-туннель при необходимости.
- [ ] Ограничить доступ к порту `6379` файрволом только с IP серверов,
      которым он реально нужен (актуально при выкатке на реальный
      публичный сервер — на этой машине пока не требуется).
- [ ] Для проекта с чужеродными данными — отдельный `REDIS_DB` или ACL-
      пользователь (см. выше), не шарить общий `REDIS_PASSWORD` с
      полным доступом бездумно.
