# Сборщик логов

Набор скриптов и конфигураций для автоматизации задач системного администрирования: сборщик ошибок из системного лога и полное окружение для его запуска (Docker, systemd, nginx).

## Состав проекта

| Файл | Назначение |
|------|-----------|
| `script.sh` | Сборщик логов: читает `/var/log/syslog`, отбирает строки с `error`/`fail` и сохраняет их в `report.txt` |
| `sample_output.txt` | Пример вывода работы скрипта |
| `Dockerfile` | Образ на базе Ubuntu 22.04 со скриптом и HTTP-сервером (Python 3, порт 8080) |
| `docker-compose.yml` | Запуск контейнера с монтированием `/var/log` в режиме read-only |
| `my-app.service` | systemd-юнит для автозапуска контейнера с приложением |
| `nginx-my-app.conf` | Конфигурация nginx: редирект HTTP → HTTPS и обратный прокси на приложение |

## Как это работает

1. `script.sh` ищет в `/var/log/syslog` строки со словами `error` или `fail` (без учёта регистра) и записывает их в `report.txt`.
2. Docker-контейнер запускает скрипт и поднимает HTTP-сервер на порту `8080`.
3. systemd-юнит обеспечивает автозапуск контейнера после загрузки Docker и перезапуск при сбоях.
4. nginx принимает внешние запросы по HTTPS и проксирует их на приложение (`127.0.0.1:8080`), весь HTTP-трафик перенаправляется на HTTPS.

## Требования

- Docker и Docker Compose
- Для развёртывания на хосте: systemd, nginx, SSL-сертификат (`my-app.crt` / `my-app.key`)

## Запуск

### Локально (только скрипт)

```bash
chmod +x script.sh
./script.sh
cat report.txt
```

### В Docker

```bash
docker compose up --build -d
```

Контейнер монтирует `/var/log` хоста в режиме read-only, поэтому скрипт анализирует реальный системный лог.

### Как systemd-сервис

```bash
# создать и запустить контейнер с именем my-app
docker build -t my-app .
docker create --name my-app -v /var/log:/var/log:ro my-app

# установить и включить юнит
sudo cp my-app.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now my-app
```

### Настройка nginx

```bash
sudo cp nginx-my-app.conf /etc/nginx/sites-available/my-app
sudo ln -s /etc/nginx/sites-available/my-app /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

Предварительно поместите сертификат в `/etc/ssl/certs/my-app.crt` и ключ в `/etc/ssl/private/my-app.key`.
