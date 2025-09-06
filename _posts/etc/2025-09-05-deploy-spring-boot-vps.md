---
title: "CI/CD-пайплайн: автодеплой Spring Boot приложения на VPS через GitHub Actions"
layout: post
date: 2025-09-05 12:00:00 +0300
categories: [DevOps, Spring Boot]
tags: [spring-boot, github-actions, vps, cicd, deploy, pipeline, automation, systemd, nginx, ssl]
---

В этой статье разберем, как развернуть Spring Boot приложение на **VPS** и настроить **автодеплой** из GitHub. В качестве примера используем небольшой сервис - [**Weather Map Demo**](https://github.com/AlekseyBykov/pets.weather-map-demo) - и опубликуем его на поддомене [weather.abykov.dev](https://weather.abykov.dev).

Арендуем сервер и настроим поддомен, `systemd`-сервис, HTTPS через **Nginx/Let's Encrypt** и **CI/CD**-пайплайн на **GitHub Actions**.

## Что используем

- VPS на Ubuntu 24.04;
- Домен [abykov.dev](https://abykov.dev) с поддоменом [weather.abykov.dev](https://weather.abykov.dev);
- Java 21, Spring Boot 3, Maven;
- Nginx + Let's Encrypt (HTTPS);
- `systemd` для запуска jar как сервиса;
- GitHub Actions для сборки и деплоя.

## Что сделаем пошагово

- Арендуем VPS и привяжем поддомен `weather.abykov.dev` к IP сервера (DNS A-запись);
- Подготовим окружение на сервере: установим Java, создадим рабочие директории;
- Запустим приложение вручную и настроим его как `systemd`-сервис (переживает перезагрузку);
- Включим Nginx как reverse-proxy и выпустим бесплатный SSL-сертификат Let's Encrypt;
- Настроим GitHub Actions: Maven-сборка, копирование JAR на VPS по SSH и рестарт сервиса - автоматически при пуше в ветку `main`;

На выходе получим приложение, доступное по HTTPS на собственном домене, обновляемое одним коммитом в репозиторий.

## Аренда VPS и домена

Для демонстрационного проекта - Spring Boot-приложения Weather Map Demo - арендуем VPS у хостинг-провайдера и приобретем домен `abykov.dev`.
Для обеспечения удобного доступа к приложению в панели управления DNS создадим `A`-запись поддомена `weather.abykov.dev`, указывающую на публичный IP-адрес VPS.

Заходим в DNS-панель, добавляем `A`-запись для поддомена, указывая на IP VPS:

![Пример настройки A-записи для поддомена](/assets/img/www-subdomain-dns-records.png){: .shadow .rounded }

- **Имя**: `weather.abykov.dev`
- **Тип**: `A`
- **Значение**: `89.111.171.25` (IP адрес VPS)
- **TTL**: `3600`

Теперь при обращении к `www.weather.abykov.dev` трафик будет направляться на наш VPS.

## Настройка VPS

После приобретения VPS первым этапом является подготовка окружения - обновление системы и установка необходимых пакетов.

```bash
# обновляем систему
sudo apt update && sudo apt upgrade -y 

# устанавливаем Java (например, OpenJDK 21)
sudo apt install openjdk-21-jdk -y 

# создаем рабочую директорию под приложение
sudo mkdir -p /opt/weather-demo
sudo chown $USER:$USER /opt/weather-demo

```
В эту директорию (`/opt/weather-demo/`) будет загружаться JAR-файл приложения.

## Запуск Spring Boot вручную

После локальной сборки артефакт необходимо передать на сервер с помощью `scp`:
```bash
scp \
  api/target/api-0.0.1-SNAPSHOT.jar \
  root@<IP_сервера>:/opt/weather-demo/weather-demo.jar
```
Затем выполнить запуск:
```bash
cd /opt/weather-demo
java -jar weather-demo.jar
```
> ⚠️ Для успешного запуска требуется **fat jar** (он же **runnable jar** или **uber-jar**). Такой JAR включает в себя все зависимости и формируется 
> при использовании `spring-boot-maven-plugin`. Также это обязательное требование для GitHub Pages при работе с кастомными доменами.

В данном виде приложение работает, однако при перезапуске сервера процесс завершится. Чтобы обеспечить автоматический запуск и перезапуск при сбое, используется настройка `systemd`-сервиса.

## Автозапуск через systemd

Чтобы приложение продолжало работать после перезагрузки сервера и автоматически перезапускалось при сбоях, его следует оформить как сервис `systemd`.

Создадим unit-файл `/etc/systemd/system/weather-demo.service`:

```bash
[Unit]
Description=Weather Demo Spring Boot App
After=network.target

[Service]
User=root
WorkingDirectory=/opt/weather-demo
ExecStart=/usr/bin/java -jar /opt/weather-demo/weather-demo.jar
SuccessExitStatus=143
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

Далее активируем и запускаем сервис:
```bash
sudo systemctl daemon-reload
sudo systemctl enable weather-demo
sudo systemctl start weather-demo
```

Проверить состояние можно командой:
```bash
systemctl status weather-demo
```
Для просмотра логов приложения используется `journalctl`:
```bash
journalctl -u weather-demo -f
```

Теперь приложение работает как полноценный сервис: оно стартует вместе с системой, перезапускается при сбое и не зависит от активной сессии в терминале.


## Настройка доступа по домену и HTTPS

По умолчанию приложение на Spring Boot запускается на порту `8080` и доступно только по IP-адресу сервера. Чтобы открыть его по доменному имени без указания порта, применяется один из двух подходов.

### Вариант 1. Spring Boot на 80 порту

В `application.properties` можно задать:

```bash
server.address=0.0.0.0
server.port=80
```

После перезапуска сервис станет доступен по адресу: `http://weather.abykov.dev`

Однако у такого решения есть недостаток: при размещении нескольких приложений придется вручную распределять порты.

### Вариант 2. Nginx как reverse-proxy (рекомендуется)

Более универсальный подход - использовать **Nginx** в качестве **обратного прокси**: Spring Boot продолжает работать на `8080`, а Nginx принимает запросы на `80` и перенаправляет их внутрь.

#### Что такое обратное проксирование?

Стоит пояснить термин. Существует два вида прокси-серверов:

- **Forward proxy (прямой прокси)** - работает на стороне клиента (стоит *перед пользователем*).  Пользователь отправляет запрос не напрямую к сайту, а через прокси. Пример: корпоративный прокси-сервер, который фильтрует трафик сотрудников или скрывает их IP.
- **Reverse proxy (обратный прокси)** - работает на стороне сервера (стоит *перед сервером*). Пользователь обращается к единому адресу (например, `weather.abykov.dev`), а прокси перенаправляет запрос во внутренние сервисы (например, на Spring Boot приложение на порту `8080`).

Именно поэтому его называют "обратным" - он скрывает внутренние сервисы за единым фасадом.

Визуально схема выглядит так:
```bash
Клиент -> Forward Proxy -> Интернет (сайты)
Клиент -> Reverse Proxy -> Внутренние сервисы (API, приложения)
```

В нашем случае Nginx выполняет роль `reverse proxy`: он принимает HTTP-запросы на `80`-м порту и проксирует их к приложению Spring Boot на порту `8080`.

#### Установка и настройка Nginx:

```bash
sudo apt install nginx -y
```

Создание конфигурации `/etc/nginx/sites-available/weather.abykov.dev`:

```bash
server {
    listen 80;
    server_name weather.abykov.dev;

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```
Активация сайта и проверка конфигурации::

```bash
sudo ln -s /etc/nginx/sites-available/weather.abykov.dev /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

Теперь приложение открывается по адресу http://weather.abykov.dev.

## HTTPS через Let's Encrypt

Для защиты трафика необходимо настроить HTTPS. Самый удобный способ - использовать **Let's Encrypt**, который позволяет бесплатно выпустить TLS-сертификат.

Устанавливаем `certbot` и плагин для Nginx:
```bash
sudo apt install certbot python3-certbot-nginx -y
```

Запрашиваем сертификат для поддомена:
```bash
sudo certbot --nginx -d weather.abykov.dev
```

После выполнения команда certbot автоматически:
- Обновит конфигурацию Nginx;
- Добавит директиву `listen 443 ssl`;
- Пропишет пути к сертификату и ключу;
- Настроит редирект с HTTP → HTTPS.

Теперь сайт доступен по защищенному протоколу: `https://weather.abykov.dev`.

Сертификат Let's Encrypt действует 90 дней, но certbot настраивает автоматическое продление. Проверить можно так:

```bash
sudo systemctl status certbot.timer
```
> ⚠️ Даже если у основного домена (`abykov.dev`) уже был установлен SSL-сертификат (например, AlphaSSL),  он не распространяется автоматически на 
> поддомены. Для `weather.abykov.dev` требуется отдельный сертификат или wildcard-сертификат (`*.abykov.dev`). В нашем случае был использован Let's 
> Encrypt, так как это бесплатное и удобное решение.

## Автоматический деплой через GitHub Actions

Развернутое вручную приложение решает задачу, но требует постоянного копирования JAR-файла и перезапуска сервиса вручную при каждом обновлении кода. Это неудобно и не подходит для продуктивной разработки.

Лучшее решение - настроить CI/CD-пайплайн на GitHub Actions, который будет:

- Собирать проект при каждом пуше в ветку main;
- Передавать собранный JAR на VPS по SSH;
- Перезапускать systemd-сервис.

Конфигурация GitHub Actions

Для этого в проекте создаем файл `.github/workflows/deploy.yml`:

```yaml
name: Deploy to VPS

on:
  push:
    branches:
      - main   # Автодеплой будет запускаться только при пуше в main

jobs:
  deploy:
    runs-on: ubuntu-latest   # GitHub Actions будет использовать виртуалку с Ubuntu

    steps:
      # 1. Клонируем репозиторий
      - name: Checkout code
        uses: actions/checkout@v4

      # 2. Устанавливаем JDK 21 (нужен для сборки Spring Boot проекта)
      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'

      # 3. Собираем проект через Maven (без тестов для ускорения)
      - name: Build with Maven
        run: mvn -B clean package -DskipTests

      # 4. Проверяем, что index.html попал в JAR (отладочный шаг)
      - name: Verify index.html inside JAR
        run: |
          echo "Check if index.html is inside JAR..."
          jar tf api/target/api-0.0.1-SNAPSHOT.jar \
            | grep index.html || echo "index.html not found!"
          echo "----- index.html content -----"
          unzip -p api/target/api-0.0.1-SNAPSHOT.jar \
            BOOT-INF/classes/templates/index.html \
            || echo "No index.html inside JAR"
          echo "------------------------------"

      # 5. Копируем JAR на VPS по SSH
      - name: Copy JAR to VPS
        uses: appleboy/scp-action@v0.1.7
        with:
          host: ${{ secrets.VPS_HOST }}        # IP адрес VPS
          username: ${{ secrets.VPS_USER }}    # Пользователь (обычно root)
          key: ${{ secrets.VPS_SSH_KEY }}      # SSH ключ из GitHub Secrets
          source: "api/target/api-0.0.1-SNAPSHOT.jar"
          target: "/opt/weather-demo/"
          overwrite: true

      # 6. Перемещаем JAR в правильное место на сервере
      - name: Move JAR into place
        uses: appleboy/ssh-action@v1.0.0
        with:
          host: ${{ secrets.VPS_HOST }}
          username: ${{ secrets.VPS_USER }}
          key: ${{ secrets.VPS_SSH_KEY }}
          script: |
            mv /opt/weather-demo/api/target/api-0.0.1-SNAPSHOT.jar \
               /opt/weather-demo/weather-demo.jar

      # 7. Перезапускаем systemd-сервис
      - name: Restart service
        uses: appleboy/ssh-action@v1.0.0
        with:
          host: ${{ secrets.VPS_HOST }}
          username: ${{ secrets.VPS_USER }}
          key: ${{ secrets.VPS_SSH_KEY }}
          script: |
            sudo systemctl stop weather-demo || true
            sudo systemctl start weather-demo
            sudo systemctl status weather-demo --no-pager

```

### Подготовка SSH-ключей для GitHub Actions

Для безопасного копирования артефактов на сервер через GitHub Actions используется аутентификация по SSH-ключам. Нам понадобится сгенерировать пару ключей: приватный (для GitHub) и публичный (для VPS).

#### Генерация ключей

На локальной машине выполним:
```bash
ssh-keygen -t ed25519 -C "github-actions" -f github_actions
```
Здесь:
- `-t ed25519` - современный и безопасный алгоритм;
- `-C "github-actions"` - комментарий, чтобы было понятно, для чего ключ;
- `-f github_actions` - имя файлов (получатся `github_actions` и `github_actions.pub`).

В результате будут созданы два файла:

- `github_actions` - **приватный ключ** (его кладем в GitHub Secrets);
- `github_actions.pub` - **публичный ключ** (его добавляем на VPS).

#### Установка ключа на VPS

Подключаемся к серверу:

```bash
ssh root@<IP_сервера>
```

И добавляем публичный ключ в файл `~/.ssh/authorized_keys`:
```bash
cat github_actions.pub >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

Теперь GitHub Actions сможет подключаться по SSH без пароля.

### Secrets в GitHub

Чтобы pipeline имел доступ к VPS, необходимо добавить следующие секреты в настройках репозитория GitHub (**Settings** → **Secrets and variables** → **Actions**):

- `VPS_HOST` - IP-адрес сервера;
- `VPS_USER` - пользователь для подключения (например, `root`);
- `VPS_SSH_KEY` - приватный SSH-ключ для доступа.

После этого пайплайн сможет без проблем подключаться к серверу и деплоить обновления.

### Результат

После всех настроек, при каждом коммите в ветку `main` GitHub Actions автоматически соберет JAR, загрузит его на VPS в `/opt/weather-demo/`, перезапустит сервис `weather-demo`.

Таким образом, весь деплой сводится к одному действию - `git push`.

### Дополнительно: уведомления о деплое

После настройки CI/CD полезно получать уведомления о том, успешно ли прошел деплой. GitHub Actions поддерживает интеграции с популярными мессенджерами (Slack, Telegram, Matrix и др).

#### Пример: уведомления в Matrix

```bash
      - name: Notify Matrix room
        uses: matrix-org/matrix-message-action@v1
        with:
          homeserver: "https://matrix.org"
          access_token: ${{ secrets.MATRIX_ACCESS_TOKEN }}
          room_id: "!yourRoomId:matrix.org"
          message: |
            ✅ Deploy completed successfully on weather.abykov.dev

```

Здесь:

- `MATRIX_ACCESS_TOKEN` хранится в GitHub Secrets и генерируется в Matrix (для бота или пользователя).
- `room_id` - уникальный идентификатор комнаты, его можно найти в настройках Matrix.

#### Другие варианты

- Slack: через [slackapi/slack-github-action](https://github.com/slackapi/slack-github-action);
- Telegram: через [appleboy/telegram-action](https://github.com/appleboy/telegram-action);
- Email: через стандартный `actions/send-mail`

Таким образом, можно получать уведомления о каждом деплое прямо в рабочий чат.
