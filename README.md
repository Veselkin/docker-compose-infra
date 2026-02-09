# Docker Compose Infrastructure

Коллекция Docker-стеков для развертывания полноценной веб-инфраструктуры. Все сервисы интегрированы с Traefik v3 для автоматического получения SSL-сертификатов (Let's Encrypt).

## Состав инфраструктуры

- **Traefik** — Edge-router, SSL прокси и балансировщик.
- **Nginx** — Статические сайты и быстрый фронтенд.
- **Apache + PHP 8.2 (FPM)** — Гибкий хостинг для MODX и других CMS.
- **Database (MariaDB)** — Основное хранилище данных + phpMyAdmin.
- **Icecast** — Стриминговый сервер для радиовещания.
- **Zabbix** — Комплексный мониторинг системы и контейнеров.

## Быстрый старт

### 1. Подготовка системы

Перед запуском убедитесь, что установлены Docker и Docker Compose. Также необходимо создать общую сеть:

```bash
docker network create traefik_proxy
```

### 2. Подготовка папок и прав

Создайте необходимые директории для хранения данных и логов:

```bash
mkdir -p ~/traefik/conf ~/traefik/letsencrypt ~/traefik/logs \
         ~/sites/nginx ~/sites/apache \
         ~/mariadb/data ~/php/sessions ~/icecast/logs \
         ~/zabbix/mysql_data ~/zabbix/zabbix_server_data

# Установка прав для корректной записи
chmod 600 ~/traefik/letsencrypt/acme.json
chmod -R 777 ~/php/sessions
```

### 3. Настройка переменных

В каждой папке сервиса скопируйте файл .env.example в .env и заполните свои данные:

```bash
cp ./traefik/.env.example ./traefik/.env
# И так далее для каждого стека
```

## Описание сервисов

### Traefik

Основная точка входа. Дашборд доступен по пути `/dashboard/` с Basic Auth защитой.

Запуск:
```bash
cd traefik && docker compose up -d
```

### Веб-серверы

**Nginx:** Использует лейблы для 7+ доменов. Конфиги лежат в `./nginx/conf.d/`.

**Apache:** Настроен на работу с PHP-FPM через FastCGI.

### PHP-FPM (Custom Build)

Собран на базе `php:8.2-fpm` с расширениями: gd, intl, zip, pdo_mysql, mbstring. Оптимизирован под MODX.

### Мониторинг (Zabbix)

Веб-интерфейс доступен по пути `your-domain.com/zabbix`.

Логин по умолчанию: `Admin / zabbix` (не забудьте сменить!).

## Безопасность

- **Секреты:** Все `.env` файлы добавлены в `.gitignore` и не попадают в репозиторий.
- **Basic Auth:** Критичные панели (Traefik, phpMyAdmin, Zabbix) защищены дополнительным слоем авторизации на уровне прокси.
- **Изоляция:** Базы данных находятся во внутренних сетях и не доступны напрямую из интернета.

## Обслуживание

Просмотр логов:
```bash
docker compose logs -f [service_name]
```

Обновление образов:
```bash
docker compose pull && docker compose up -d
```

Проверка конфига Traefik:
```bash
docker exec -it traefik traefik healthcheck
```
