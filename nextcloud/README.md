# Nextcloud на Windows Server 2025 + Docker + ZeroTier

Инструкция по развёртыванию Nextcloud в Docker на Windows Server 2025 с доступом через ZeroTier VPN. Данные хранятся на диске с ReFS + дедупликация.

## Стек

- **Windows Server 2025** + WSL2
- **Docker** (через WSL2)
- **Nextcloud 29** (Apache)
- **MariaDB 11**
- **Redis 7**
- **ZeroTier** — удалённый доступ
- **ReFS** диск с дедупликацией — хранилище данных

---

## Структура файлов

```
C:\nextcloud\
├── docker-compose.yml
├── .env
└── data\
    └── db\          ← база данных MariaDB
```

Данные Nextcloud хранятся на отдельном диске: `D:\nextcloud-data`

---

## Требования

- Windows Server 2025 с включённым WSL2
- Docker установлен и доступен из WSL
- ZeroTier установлен, сервер добавлен в сеть
- Диск D:\ с ReFS и включённой дедупликацией

---

## Установка

### 1. Создание папок

В PowerShell (от администратора):

```powershell
mkdir C:\nextcloud
mkdir C:\nextcloud\data\db
mkdir D:\nextcloud-data

icacls D:\nextcloud-data /grant "*S-1-1-0:(OI)(CI)F"
```

### 2. Создание .env

```env
# MariaDB
MYSQL_ROOT_PASSWORD=RootStrongPass987!
MYSQL_DATABASE=nextcloud
MYSQL_USER=nextcloud
MYSQL_PASSWORD=NcDbPass456!

# Nextcloud Admin
NEXTCLOUD_ADMIN_USER=admin
NEXTCLOUD_ADMIN_PASSWORD=AdminPass123!

# Trusted domains — ZeroTier IP сервера
NEXTCLOUD_TRUSTED_DOMAINS=172.172.172.161

# Paths (Unix-style для WSL)
DB_DATA_PATH=/mnt/c/nextcloud/data/db

# Port
NC_PORT=8080

# PHP
PHP_MEMORY_LIMIT=1024M
PHP_UPLOAD_LIMIT=16G
```

> ⚠️ Замените IP и пароли на свои.

### 3. Создание docker-compose.yml

```yaml
services:

  db:
    image: mariadb:11
    container_name: nextcloud_db
    restart: unless-stopped
    # --innodb-use-native-aio=0 и fsync нужны для WSL2 (io_uring заблокирован ядром)
    command: --transaction-isolation=READ-COMMITTED --log-bin=binlog --binlog-format=ROW --innodb-use-native-aio=0 --innodb-flush-method=fsync
    volumes:
      - ${DB_DATA_PATH}:/var/lib/mysql
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
    healthcheck:
      test: ["CMD", "mariadb-admin", "ping", "-h", "localhost", "-u", "nextcloud", "-p${MYSQL_PASSWORD}"]
      interval: 10s
      timeout: 5s
      retries: 10
      start_period: 60s
    networks:
      - nextcloud_net

  redis:
    image: redis:7-alpine
    container_name: nextcloud_redis
    restart: unless-stopped
    networks:
      - nextcloud_net

  app:
    image: nextcloud:29.0.16-apache
    container_name: nextcloud_app
    restart: unless-stopped
    ports:
      - "0.0.0.0:${NC_PORT}:80"
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
    volumes:
      - nextcloud_data:/var/www/html/data
      - nextcloud_config:/var/www/html/config
      - nextcloud_apps:/var/www/html/custom_apps
      - /mnt/d:/mnt/d_drive        # весь диск D:\ для External Storage
    environment:
      MYSQL_HOST: db
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
      NEXTCLOUD_ADMIN_USER: ${NEXTCLOUD_ADMIN_USER}
      NEXTCLOUD_ADMIN_PASSWORD: ${NEXTCLOUD_ADMIN_PASSWORD}
      NEXTCLOUD_TRUSTED_DOMAINS: ${NEXTCLOUD_TRUSTED_DOMAINS}
      REDIS_HOST: redis
      PHP_MEMORY_LIMIT: ${PHP_MEMORY_LIMIT}
      PHP_UPLOAD_LIMIT: ${PHP_UPLOAD_LIMIT}
      NEXTCLOUD_DATA_DIR: /var/www/html/data
    networks:
      - nextcloud_net

  cron:
    image: nextcloud:29.0.16-apache
    container_name: nextcloud_cron
    restart: unless-stopped
    volumes:
      - nextcloud_data:/var/www/html/data
      - nextcloud_config:/var/www/html/config
      - nextcloud_apps:/var/www/html/custom_apps
      - /mnt/d:/mnt/d_drive
    entrypoint: /cron.sh
    depends_on:
      - app
    networks:
      - nextcloud_net

networks:
  nextcloud_net:
    driver: bridge

volumes:
  nextcloud_config:
  nextcloud_apps:
  # Data через named volume с bind на D:\ — решает проблему прав NTFS
  nextcloud_data:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /mnt/d/nextcloud-data
```

### 4. Монтирование диска D:\ в WSL с правильными правами

Данный шаг **критически важен** — NTFS не поддерживает Unix-права, поэтому диск нужно перемонтировать с uid=33 (www-data):

```bash
sudo umount /mnt/d
sudo mount -t drvfs D: /mnt/d -o metadata,uid=33,gid=33,fmask=007,dmask=007
```

Чтобы монтирование сохранялось после перезагрузки WSL, добавьте в `/etc/fstab` внутри WSL:

```
D: /mnt/d drvfs metadata,uid=33,gid=33,fmask=007,dmask=007 0 0
```

### 5. Запуск

```bash
wsl
cd /mnt/c/nextcloud
docker-compose up -d
docker logs -f nextcloud_app
```

Ждём строки `Nextcloud was successfully installed`.

### 6. Исправление прав на кеш (после первого запуска)

Из-за особенностей NTFS нужно вручную создать папку кеша:

```bash
docker exec -u root nextcloud_app mkdir -p /var/www/html/data/cache
docker exec -u root nextcloud_app chown -R www-data:www-data /var/www/html/data/cache
docker exec -u www-data nextcloud_app php occ maintenance:repair
```

---

## Настройка сети (ZeroTier + portproxy)

Docker в WSL2 слушает только на `127.0.0.1`. Чтобы ZeroTier-адрес был доступен, настраиваем проброс порта в PowerShell (от администратора):

```powershell
# Узнать IP WSL
wsl -e bash -c "ip addr show eth0 | grep 'inet '"

# Создать правило portproxy
netsh interface portproxy add v4tov4 `
  listenaddress=172.172.172.161 `
  listenport=8080 `
  connectaddress=<WSL_IP> `
  connectport=8080

# Правила брандмауэра
New-NetFirewallRule -DisplayName "Nextcloud HTTP" -Direction Inbound -Protocol TCP -LocalPort 8080 -Action Allow
New-NetFirewallRule -DisplayName "Nextcloud HTTP Outbound" -Direction Outbound -Protocol TCP -LocalPort 8080 -Action Allow
```

> ⚠️ IP адрес WSL меняется при каждом перезапуске — правило portproxy нужно обновлять. Можно автоматизировать через Task Scheduler.

---

## Подключение диска D:\ как External Storage

После входа в Nextcloud:

```bash
# Включить приложение
docker exec -u www-data nextcloud_app php occ app:enable files_external

# Добавить хранилище
docker exec -u www-data nextcloud_app php occ files_external:create \
  "Диск D" local null::null --config datadir=/mnt/d_drive
```

Или через веб-интерфейс: **Настройки → Администрирование → Внешние хранилища**

| Поле | Значение |
|---|---|
| Имя папки | `Диск D` |
| Тип | `Локальный` |
| Путь | `/mnt/d_drive` |

---

## Доступ

| | |
|---|---|
| URL | `http://<ZeroTier-IP>:8080` |
| Логин | значение `NEXTCLOUD_ADMIN_USER` |
| Пароль | значение `NEXTCLOUD_ADMIN_PASSWORD` |

---

## Полная переустановка

Если что-то пошло не так и нужно начать с нуля — выполнять строго по порядку.

### Шаг 1 — Остановить и удалить контейнеры и volumes

```bash
wsl
cd /mnt/c/nextcloud

# Останавливаем и удаляем контейнеры
docker-compose down

# Удаляем named volumes (config, apps, data)
docker volume rm nextcloud_nextcloud_config nextcloud_nextcloud_apps nextcloud_nextcloud_data 2>/dev/null

# Проверяем что volumes удалены
docker volume ls | grep nextcloud
```

### Шаг 2 — Очистить данные БД

```bash
rm -rf /mnt/c/nextcloud/data/db/*
```

### Шаг 3 — Очистить данные Nextcloud на диске D:\

```bash
# От root через контейнер (своими правами не удалить — uid=33)
docker run --rm -v /mnt/d/nextcloud-data:/data alpine sh -c "rm -rf /data/*  /data/.* 2>/dev/null; ls -la /data"
```

### Шаг 4 — Перемонтировать диск D:\ с правильными правами

```bash
sudo umount /mnt/d
sudo mount -t drvfs D: /mnt/d -o metadata,uid=33,gid=33,fmask=007,dmask=007
```

### Шаг 5 — Запустить заново

```bash
docker-compose up -d
docker logs -f nextcloud_app
```

Ждём `Nextcloud was successfully installed`, затем исправляем кеш:

```bash
docker exec -u root nextcloud_app mkdir -p /var/www/html/data/cache
docker exec -u root nextcloud_app chown -R www-data:www-data /var/www/html/data/cache
docker exec -u www-data nextcloud_app php occ maintenance:repair
```

> ⚠️ **Все данные пользователей будут удалены.** Сделайте резервную копию `D:\nextcloud-data` перед переустановкой если нужно сохранить файлы.

---

## Полезные команды

```bash
# Статус
docker-compose ps

# Логи
docker logs -f nextcloud_app

# OCC команды
docker exec -u www-data nextcloud_app php occ status
docker exec -u www-data nextcloud_app php occ maintenance:mode --off

# Обновление
docker-compose pull
docker-compose up -d

# Статус дедупликации ReFS (PowerShell)
Get-DedupStatus -Volume D:
```

---

## Известные проблемы и решения

| Проблема | Причина | Решение |
|---|---|---|
| `io_uring` ошибка MariaDB | WSL2 блокирует io_uring | `--innodb-use-native-aio=0 --innodb-flush-method=fsync` |
| Права на config/data (uid 1000) | NTFS не поддерживает Unix-права | Named volumes для config/apps, drvfs с uid=33 для data |
| Docker слушает только 127.0.0.1 | WSL2 NAT | `netsh portproxy` |
| 500 ошибка после входа | Нет прав на папку cache | `mkdir /var/www/html/data/cache` + chown |
| `maintenance:install` not defined | Старый autoconfig.php | Удалить volume nextcloud_config и пересоздать |

---

## Заметки по ReFS + дедупликация

Дедупликация работает прозрачно на уровне Windows Server — контейнер просто пишет файлы, ОС дедуплицирует в фоне. Особенно эффективна для дублирующихся файлов пользователей (фото, документы).

```powershell
# Статус дедупликации
Get-DedupStatus -Volume D:
Get-DedupJob
```
