# Matrix (Continuwuity) + LiveKit на VPS

Папка дополняет пошаговый план установки: **Docker Compose** для **LiveKit**, **lk-jwt-service** и (опционально) **coturn**, плюс шаблоны конфигов и замечания по надёжности.

## Что вынесено в Docker Compose

| Компонент        | Где запускать | Почему |
|------------------|---------------|--------|
| **LiveKit**      | `docker-compose.yml` | Официальный образ, проще обновлять, `network_mode: host` для WebRTC |
| **lk-jwt**       | `docker-compose.yml` | Те же секреты, что в LiveKit, удобно через `.env` |
| **coturn**       | `docker-compose.coturn.yml` | `network_mode: host` — те же порты 3478/5349, что у «apt install coturn», без проброса UDP-диапазонов |
| **Continuwuity** | Хост (`.deb` + systemd) | Меньше сюрпризов с путями и данными; проще бэкап `/var/lib/conduwuit` |
| **Nginx + TLS**  | Хост | Привычный certbot, один вход на 443 |

**Не поднимайте одновременно** контейнер `coturn` и системный `coturn`/`turnserver` из пакета — порты заняты.

### Coturn в Docker

1. Выпустите сертификат Let’s Encrypt (как в основном плане), чтобы на хосте существовали `fullchain.pem` и `privkey.pem`.
2. `cp config/turnserver.conf.example config/turnserver.conf` — укажите `realm`, `static-auth-secret` (`openssl rand -hex 32`), пути `cert`/`pkey` на **ваш** каталог внутри `/etc/letsencrypt/live/...` (в контейнере тот же путь, каталог монтируется с хоста).
3. В `.env` задайте `LETSENCRYPT_DIR=/etc/letsencrypt` (если отличается).
4. Если контейнер не читает ключи, на хосте часто нужно дать чтение каталогу Let’s Encrypt (группа `ssl-cert`, `chmod` на `archive`/`live` — см. доки дистрибутива; это та же проблема, что и у coturn из `apt`).
5. Запуск **вместе** с LiveKit:

```bash
docker compose -f docker-compose.yml -f docker-compose.coturn.yml up -d
```

Только TURN: `docker compose -f docker-compose.coturn.yml up -d`

Логи: `docker logs -f coturn`

## Быстрый старт (часть стека в Compose)

1. На VPS уже должны работать: Continuwuity (порт `6167` на loopback), Nginx с прокси как в плане, SSL.
2. Скопируйте репозиторий/папку на сервер (или только файлы из `matrix-vps-deploy/`).
3. `cp .env.example .env` — заполните ключи и домены.
4. `cp config/livekit.yaml.example config/livekit.yaml` — вставьте ту же пару **key: secret**, что в `.env`.
5. Из этой директории: `docker compose up -d`
6. Проверка: `curl -sS https://matrix.yourdomain.ru/livekit/jwt/healthz`

Обновление образов: `docker compose pull && docker compose up -d`

## Улучшения относительно «сырого» плана

1. **Опечатки в командах** — в оригинале местами слиплись `bash` и команда (`bashfallocate`, `bashapt`). В шелле должны быть отдельные строки.
2. **1 ГБ RAM** — зависит от числа пользователей и звонков; swap обязателен. LiveKit на слабом VPS: узкий диапазон `rtc.port_range_*`, ограничение одновременных участников.
3. **Пин версий образов** — в `docker-compose.yml` указаны конкретные теги; не используйте неограниченный `:latest` в проде без осознанного CI.
4. **Секреты** — `registration_token`, LiveKit keys, TURN secret только в конфигах на сервере или в `.env`, не в git.
5. **Firewall** — открыть как минимум: `80/tcp`, `443/tcp`, `3478/udp` (TURN), диапазон UDP для LiveKit (как в `livekit.yaml`, например `50100-50200/udp`), при необходимости `7881/tcp` (RTC TCP). Точный список лучше сверить с [документацией LiveKit](https://docs.livekit.io/) под вашу схему за NAT.
6. **Nginx и статика `/account`** — для одного файла надёжнее `alias …/account.html`, чем `root` + `try_files` (меньше путаницы с путями). Фрагмент: `config/nginx-snippet-matrix.conf`.
7. **Путь к бинарнику Continuwuity** — после `dpkg -i` проверьте `which conduwuit` или `dpkg -L continuwuity`; в unit-файле должен совпадать реальный путь (не всегда `/usr/local/bin/conduwuit`).
8. **Coturn и TLS** — пути к `fullchain.pem` / `privkey.pem` должны существовать до рестарта coturn после первого успешного `certbot`; пользователь `turnserver` должен читать каталог Let’s Encrypt (на Debian часто нужна группа `ssl-cert` и права).
9. **Бэкапы** — регулярно копируйте `/var/lib/conduwuit` и `/etc/conduwuit/conduwuit.toml` (и `.env` этой папки, если используете Compose).
10. **DNS** — для клиентов Matrix важны корректные `/.well-known/matrix/*`; при смене поддоменов обновляйте JSON в Nginx синхронно с `server_name` в conduwuit.

## Переменные окружения

См. `.env.example`. `LIVEKIT_URL` — публичный **WSS** до SFU через ваш Nginx (как в оригинальном плане).

## Файлы

| Файл | Назначение |
|------|------------|
| `docker-compose.yml` | LiveKit (host network) + lk-jwt |
| `docker-compose.coturn.yml` | Coturn (host network), конфиг и `/etc/letsencrypt` с хоста |
| `config/livekit.yaml.example` | Шаблон конфига LiveKit |
| `config/turnserver.conf.example` | Шаблон coturn |
| `config/nginx-snippet-matrix.conf` | Фрагмент `location` для Matrix + LiveKit + `/account` |
| `.env.example` | Шаблон секретов и доменов |

Страницу `account.html` из вашего плана положите на сервер в `/var/www/html/account.html` и подставьте свой домен во все вхождения.

## Исходный сценарий (кратко)

Подготовка Debian → DNS `A` на `@` и `matrix` → Continuwuity → Nginx → Certbot → coturn (**apt или** `docker-compose.coturn.yml`) → **Compose: LiveKit + lk-jwt** → первый пользователь → `/account` → проверка `systemctl` / `docker ps`.

---

English: Continuwuity and Nginx stay on the host. **LiveKit**, **lk-jwt-service**, and **optionally coturn** run via Compose (coturn uses host networking). Pin tags and open firewall ports per LiveKit/TURN docs.
