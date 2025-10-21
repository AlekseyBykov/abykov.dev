---
title: "SQL*Plus на Ubuntu: полный рабочий сетап через Oracle Instant Client"
layout: post
date: 2025-10-21 12:00:00 +0300
categories: [oracle, linux, devops]
tags: [sqlplus, oracle, ubuntu, database, tooling]
---

**SQLPlus** остаeтся востребованным инструментом для администрирования Oracle Database и автозапуска SQL-скриптов в CI/CD и DevOps-пайплайнах. Несмотря на распространeнность Oracle, официальная установка SQLPlus в Linux, в частности Ubuntu, часто вызывает трудности из-за зависимостей и особенностей Oracle Instant Client.

Разберем пошагово, как установить SQL*Plus на Ubuntu 22.04 и 24.04. В примерах используется актуальная версия Instant Client 21.13.

## Зачем нужен SQL*Plus

SQL*Plus — это минималистичная консольная утилита Oracle. Она полезна в сценариях:

- Выполнение SQL- и PL/SQL-скриптов на серверах;
- Миграции схем и данных;
- Автоматизация через Bash и CI/CD (GitLab, Jenkins, TeamCity);
- Отладка подключения к Oracle (проверка listener, доступности сервисов);
- Управление пользователями и ролями;
- Запуск инсталляционных скриптов (`*.sql`) в деплоях.

Если нужно подключить Oracle в инфраструктуру — SQL*Plus почти всегда нужен.

## Варианты установки

В Linux есть три способа поставить SQL*Plus:

| Вариант установки | Описание | Когда использовать |
|-------------------|----------|---------------------|
| **Oracle Instant Client**<br>(рекомендуется) | Официальный способ от Oracle.<br>Минимальный набор библиотек.<br>Стабильная работа в продакшене. | ✅ Серверы<br>✅ DevOps<br>✅ CI/CD |
| **Oracle SQLcl** | Современная альтернатива SQL\*Plus.<br>Поддерживает историю команд, алиасы <br>и SQL форматирование.<br>Требует Java (JDK/JRE). | 💡 Локальная разработка<br>💡 Административные задачи |
| **PPA/форки из интернета** | Неофициальные сборки SQL\*Plus.<br>Можно найти в сторонних репозиториях. | ⚠️ Риск зависимостей<br>⚠️ Проблемы обновлений |


В этой статье используется **официальный способ через Oracle Instant Client** — надeжно и подходит для продакшена.

## Установка Oracle Instant Client и SQL*Plus

Oracle Instant Client содержит всe, что нужно для подключения к базе: базовые сетевые библиотеки и консоль SQL*Plus. Для Ubuntu пакетов нет, поэтому ставим вручную.

### Установка зависимостей

```bash
sudo apt update
sudo apt install wget unzip libaio-dev -y
```
`libaio-dev` — обязательная зависимость. Раньше использовался `libaio1`, но в новых версиях Ubuntu пакет переименовали.

### Загрузка Instant Client

Переходим в каталог `~/Downloads` и скачиваем актуальные архивы (версия 21.13):

```bash
cd ~/Downloads

wget https://download.oracle.com/otn_software/linux/instantclient/2113000/instantclient-basiclite-linux.x64-21.13.0.0.0dbru.zip \
     -O instantclient-basiclite.zip

wget https://download.oracle.com/otn_software/linux/instantclient/2113000/instantclient-sqlplus-linux.x64-21.13.0.0.0dbru.zip \
     -O instantclient-sqlplus.zip
```
Архивы можно обновить при выходе новой версии — структура ссылок у Oracle стабильная.

### Распаковка и размещение

```bash
unzip instantclient-basiclite-linux.x64-21.13.0.0.0dbru.zip
unzip instantclient-sqlplus-linux.x64-21.13.0.0.0dbru.zip

sudo mkdir -p /opt/oracle
sudo mv instantclient_21_13 /opt/oracle/
```
`/opt/oracle` — стандартное место для системных инструментов Oracle, удобно использовать при обновлениях.

### Настройка переменных окружения

Добавляем путь к библиотекам в `~/.bashrc`, чтобы sqlplus запускался из любой директории:

```bash
echo "export LD_LIBRARY_PATH=/opt/oracle/instantclient_21_13" >> ~/.bashrc
echo "export PATH=\$PATH:/opt/oracle/instantclient_21_13" >> ~/.bashrc
source ~/.bashrc
```
### Проверка установки
```bash
sqlplus -v
```
Если появляется следующая ошибка, значит Oracle ищет старую версию `libaio`:
```bash
sqlplus: error while loading shared libraries:
libaio.so.1: cannot open shared object file
```
Исправляем симлинком:
```bash
sudo ln -s /usr/lib/x86_64-linux-gnu/libaio.so /usr/lib/x86_64-linux-gnu/libaio.so.1
```
Проверяем снова:
```bash
sqlplus -v

SQL*Plus: Release 21.0.0.0.0 - Production
Version 21.13.0.0.0
```

SQL*Plus установлен и готов к работе.

## Подключение к базе данных

SQL*Plus поддерживает подключение по протоколу Oracle Net (TCP). Проще всего подключаться в формате **EZConnect**, без `tnsnames.ora`:

```bash
sqlplus USER/PASS@//HOSTNAME:1521/SERVICE_NAME
```
Пример:
```bash
sqlplus scott/tiger@//192.168.1.10:1521/ORCLCDB
```
Если требуется указать кодировку клиента (например, UTF-8), указываем переменную `NLS_LANG`:
```bash
NLS_LANG=.AL32UTF8 sqlplus scott/tiger@//db.example.com:1521/ORCLCDB
```

## Выполнение SQL-скриптов

SQL*Plus часто используют для автоматизированного выполнения SQL-скриптов и деплоя схем. Есть два основных режима работы: обычный и пакетный (без интерактивности).

### Выполнение одного скрипта
Если скрипт лежит в корне:
```bash
sqlplus user/pass@//host:1521/service @script.sql
```

Для подкаталога:
```bash
sqlplus user/pass@//host:1521/service @scripts/init_schema.sql
```

### Пакетный режим (для CI/CD)

Пакетный режим позволяет передавать сразу несколько команд:
```bash
sqlplus -s user/pass@//host:1521/service <<EOF
@schema/init.sql
@schema/data.sql
EXIT;
EOF
```

Ключ `-s` отключает лишние SQL*Plus сообщения, что удобно для CI-логов.

### Выполнение SQL из pipeline

Пример выполнения SQL прямо из shell-команды (например, GitLab CI):
```bash
echo "SELECT sysdate FROM dual;" | sqlplus -s user/pass@//host:1521/service
```

## Завершение сессии

Важно завершать сеанс sqlplus, иначе CI job может «залипнуть», ожидая ввода:
```bash
EXIT;
```
