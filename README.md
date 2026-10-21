# Telegram API — запуск и эксплуатация

Сервис `logistics-telegram-api` разворачивается через Docker Compose из готового образа
`nizomiddinkhoja/truck-tj-telegram-api` (Docker Hub). Исходники и сборка образа — в
`logistics-telegram-api`, здесь только запуск.

| Сервис         | Назначение                                                        | Порт на хосте |
|----------------|-------------------------------------------------------------------|---------------|
| `telegram-api` | Node.js: сообщения в чаты, ответы на чеки, коды подтверждения (Gateway) | `3100`  |

## Требования

- Docker 24+ и Docker Compose v2 (`docker compose`, не `docker-compose`)
- Свободный порт `3100`
- Файл `.env` в директории проекта (см. `.env.example`)
- Доступ к образу на Docker Hub (`docker login`, если репозиторий приватный)

## Настройка `.env`

```bash
cp .env.example .env
openssl rand -hex 32          # → API_KEY
nano .env
chmod 600 .env
```

| Переменная | Описание |
|---|---|
| `TAG` | Версия образа, например `v1.0.0`. По умолчанию `latest` |
| `API_KEY` | Ключ для заголовка `X-Api-Key`. Тот же, что `TELEGRAM_SERVICE_KEY` в logistics-api |
| `TELEGRAM_BOT_TOKEN` | Токен бота (сообщения в чаты, реакции на чеки) |
| `TELEGRAM_GATEWAY_TOKEN` | Токен Telegram Gateway (коды подтверждения) |
| `TELEGRAM_CODE_CHANNEL_USERNAME` | Канал-отправитель кодов. Пусто — коды приходят от «Verification Codes» |
| `TELEGRAM_*_ID` | ID чатов: каналы TJ/UZ, модераторы, мониторинг, balance_group |

`HOST` и `PORT` в `.env` не указывайте: `docker-compose.yml` задаёт их сам (`0.0.0.0:3100`).

`.env` содержит токены и не коммитится (`.gitignore`).

## Запуск

### Первый запуск

```bash
# 1. Скачать образ и поднять сервис
docker compose pull
docker compose up -d

# 2. Дождаться статуса healthy (10–30 секунд)
docker compose ps
# STATUS: Up ... (healthy)

# 3. Проверить, что слушает все интерфейсы
docker logs telegram-api | grep listening
# ожидаем: listening on http://0.0.0.0:3100

# 4. Проверить ответ
curl http://127.0.0.1:3100/health
# ожидаем: {"ok":true}
```

### Подключение logistics-api

В `.env` logistics-api:

```
TELEGRAM_SERVICE_URL=http://127.0.0.1:3100        # если на этом же сервере
# TELEGRAM_SERVICE_URL=http://<ip-сервера>:3100   # если на другом
TELEGRAM_SERVICE_KEY=<тот же, что API_KEY>
```

Затем в logistics-api:

```bash
php artisan config:cache
sudo supervisorctl restart all   # воркеры очереди подхватят новый конфиг
```

Проверка всей цепочки (сообщение уйдёт в чат мониторинга):

```bash
php artisan tinker --execute='print_r(\App\Services\Telegram\TelegramApiClient::sendToMonitoring("Проверка telegram-api"));'
# ожидаем: [ok] => 1
```

### Повторный запуск

```bash
docker compose up -d
```

Если меняли `.env` или `docker-compose.yml` — пересоздайте контейнер
(`docker compose restart` **не** перечитывает `.env`):

```bash
docker compose up -d --force-recreate telegram-api
```

### Обновление версии

```bash
# 1. В .env поменять TAG, например TAG=v1.0.1
nano .env

# 2. Скачать и пересоздать
docker compose pull
docker compose up -d
```

Откат — вернуть прежний `TAG` в `.env` и повторить шаг 2.

### Остановка

```bash
docker compose down
```

Данных у сервиса нет, удалять нечего.

## Проверка состояния

### Контейнер

```bash
docker compose ps
docker inspect telegram-api --format 'status={{.State.Status}} health={{.State.Health.Status}} restarts={{.RestartCount}}'
```

Ожидаемо: `status=running health=healthy`, `restarts` не растёт.

### Порт

```bash
docker ps --filter name=telegram-api --format '{{.Ports}}'
# ожидаем: 0.0.0.0:3100->3100/tcp
```

### Логи

Каждая строка — JSON, у запросов от logistics-api есть `logUUID` (тот же, что в `laravel.log` и таблице `logs`):

```bash
docker logs --tail 200 telegram-api
docker logs -f telegram-api
docker logs telegram-api 2>&1 | grep <logUUID>
```

### Переменные в контейнере

```bash
docker exec telegram-api printenv | grep -E 'CHAT_ID|CHANNEL|HOST|PORT'
```

Токены и `API_KEY` так не выводите.

## Troubleshooting

### Контейнер сразу останавливается: `Environment variable ... is required`

Не заполнен `API_KEY`, `TELEGRAM_BOT_TOKEN` или `TELEGRAM_GATEWAY_TOKEN` в `.env`.

```bash
docker logs --tail 20 telegram-api
nano .env
docker compose up -d --force-recreate telegram-api
```

### В логе `listening on http://127.0.0.1:3100`, снаружи не отвечает

В `.env` есть `HOST=127.0.0.1`, а контейнер запущен не через этот `docker-compose.yml`
(например, через `docker run --env-file .env`).

```bash
sed -i '/^HOST=/d; /^PORT=/d' .env
docker rm -f telegram-api
docker compose up -d
```

### `port is already allocated` / `address already in use`

Порт 3100 занят — чаще всего сервисом, запущенным раньше через supervisor или `npm start`:

```bash
sudo ss -ltnp | grep 3100
sudo supervisorctl stop telegram-api   # если запускали через supervisor
```

### logistics-api получает 401

`TELEGRAM_SERVICE_KEY` в logistics-api не совпадает с `API_KEY` здесь. Исправьте и выполните
в logistics-api `php artisan config:cache`.

### Сообщения не доходят, в логе `Telegram API sendMessage failed`

Смотрите `description` в строке лога:

- `chat not found` — неверный `TELEGRAM_*_ID` или бот не добавлен в чат;
- `bot was kicked` / `not enough rights` — бот удалён из чата или не админ канала;
- `Unauthorized` — неверный `TELEGRAM_BOT_TOKEN`.

### Ограничить доступ к порту по IP

Docker обходит правила `ufw`: опубликованный порт доступен всем. Пустить только сервер logistics-api:

```bash
sudo iptables -I DOCKER-USER -p tcp -m conntrack --ctorigdstport 3100 -j DROP
sudo iptables -I DOCKER-USER -p tcp -m conntrack --ctorigdstport 3100 -s <IP logistics-api> -j ACCEPT
sudo netfilter-persistent save   # пакет iptables-persistent
```
