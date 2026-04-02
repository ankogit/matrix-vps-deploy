# Matrix (Continuwuity) + LiveKit на VPS

Полный стек Matrix-сервера через Docker Compose: **Continuwuity**, **LiveKit**, **lk-jwt-service**, **coturn** и статик-сервер для страницы аккаунта.

## Архитектура

| Компонент | Образ | Порт | Сеть |
|-----------|-------|------|------|
| **Continuwuity** | `ghcr.io/continuwuity/continuwuity:latest` | `6167` | bridge |
| **LiveKit** | `livekit/livekit-server:v1.9.4` | `7884` (HTTP), `7885` (RTC TCP), `50200-50300/udp` | host |
| **lk-jwt** | `ghcr.io/element-hq/lk-jwt-service:0.4.2` | `8080` | bridge |
| **coturn** | `coturn/coturn:latest` | `3478/udp`, `5349/tcp` | host |
| **static** | `nginx:alpine` | `8888` | bridge |

Nginx и SSL — на отдельном шлюзе, который проксирует трафик на этот сервер.

## Быстрый старт

1. `cp .env.example .env` — заполните домены и ключи
2. `cp config/conduwuit.toml.example config/conduwuit.toml` — подставьте домен и токен регистрации
3. `cp config/livekit.yaml.example config/livekit.yaml` — вставьте ту же пару key/secret из `.env`
4. `cp config/turnserver.conf.example config/turnserver.conf` — подставьте домен и TURN_SHARED_SECRET
5. `docker compose up -d`

### Генерация секретов

```bash
openssl rand -hex 12   # LIVEKIT_API_KEY
openssl rand -hex 32   # LIVEKIT_API_SECRET, TURN_SHARED_SECRET, registration_token
```

### Первый пользователь

При первом запуске Continuwuity выдаёт временный токен для создания первого аккаунта:

```bash
docker logs conduwuit 2>&1 | grep "registration token"
```

После создания первого аккаунта начнёт работать токен из `conduwuit.toml`.

## Порты для проксирования

Внешний nginx (`matrix.nonza.ru` → этот сервер):

| Путь | Порт | Протокол |
|------|------|----------|
| `/` | `6167` | HTTP |
| `/livekit/jwt/` | `8080` | HTTP |
| `/livekit/sfu/` | `7884` | WebSocket |
| `/account` | `8888` | HTTP (статика) |

Открыть в файрволе напрямую (не через nginx):

| Порт | Протокол | Назначение |
|------|----------|------------|
| `7885` | TCP | LiveKit RTC TCP |
| `50200-50300` | UDP | LiveKit RTC UDP |
| `3478` | UDP | TURN |
| `5349` | TCP | TURNS |

## Файлы

| Файл | Назначение |
|------|------------|
| `docker-compose.yml` | Весь стек |
| `config/conduwuit.toml.example` | Шаблон конфига Matrix-сервера |
| `config/livekit.yaml.example` | Шаблон конфига LiveKit |
| `config/turnserver.conf.example` | Шаблон конфига coturn |
| `config/nginx-matrix.nonza.ru.conf` | Конфиг внешнего nginx-шлюза |
| `.env.example` | Шаблон переменных окружения |
| `account.html` | Страница регистрации и смены пароля |

## Страница регистрации (`/account`)

Веб-страница для регистрации и смены пароля — нужна потому что Element X не поддерживает эти операции без OIDC/MAS. Работает через Matrix Client API напрямую.

Файл `account.html` лежит на сервере в `/var/www/html/account.html` и отдаётся контейнером `static` (nginx:alpine) на порту `8888`. Внешний nginx проксирует `/account` → `8888`.

Доступна по адресу: `https://matrix.nonza.ru/account`

## Обновление

```bash
docker compose pull && docker compose up -d
```

## Заметки

- **Порты LiveKit**: если на сервере уже занят `7880/7881` другим инстансом, используйте `7884/7885` и UDP-диапазон `50200-50300` (как в текущем конфиге).
- **livekit + network_mode: host**: нельзя использовать Docker-сеть между livekit и другими сервисами — lk-jwt подключается к livekit по публичному URL.
- **coturn без TLS**: если нет локального certbot, coturn работает только на UDP 3478. TURNS (5349) требует сертификатов.
- **Бэкапы**: регулярно копируйте Docker volume `matrix_conduwuit-data` и `/home/infra/matrix/.env`.
