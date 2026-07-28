# Контейнерное окружение для Битрикс

Сборка для запуска сайта на продуктах 1С-Битрикс (`Управление сайтом`, `Битрикс24 Коробка`) в контейнерах [Docker](https://www.docker.com/).

Программы, необходимые для работы сайта, запускаются в контейнерах (`Docker Containers`) по шаблону из образов (`Docker Images`). Данные хранятся в томах (`Docker Volumes`). Контейнеры связаны между собой сетью (`Docker Network`) и управляются через `Docker Compose`.

Документ построен как **пошаговая инструкция по переносу существующего сайта в эту сборку**: начинаем с резервных копий (файлы + база данных), заканчиваем настроенным и проверенным сайтом. Дальше идут разделы про установку сайта с нуля, регулярное резервное копирование и справочник по окружению.

> [!CAUTION]
> Внимание! Порядок шагов важен. Не начинайте перенос, не имея на руках проверенных резервных копий файлов и базы данных исходного сайта.

<a id="contents"></a>
# Содержание

* [Часть 1. Что нужно знать до старта](#part1)
  * [Состав сборки](#stack)
  * [Требования к хосту](#hostrequirements)
  * [Как читать команды](#howtoreadcommands)
* [Часть 2. Пошаговый перенос сайта](#part2)
  * [Шаг 1. Резервная копия файлов сайта](#step1)
  * [Шаг 2. Резервная копия базы данных](#step2)
  * [Шаг 3. Проверка и перенос резервных копий](#step3)
  * [Шаг 4. Docker и получение сборки на новом хосте](#step4)
  * [Шаг 5. Настройка сборки до первого запуска](#step5)
  * [Шаг 6. Первый запуск контейнеров](#step6)
  * [Шаг 7. Загрузка файлов сайта в контейнер](#step7)
  * [Шаг 8. Создание базы данных и импорт дампа](#step8)
  * [Шаг 9. Подключение сайта к базе данных](#step9)
  * [Шаг 10. Очистка кеша и настроек старого окружения](#step10)
  * [Шаг 11. Первое открытие сайта и проверка](#step11)
  * [Шаг 12. Настройки модулей, лицензия и обновления](#step12)
  * [Шаг 13. Cron и агенты](#step13)
  * [Шаг 14. Временные файлы вне корня сайта](#step14)
  * [Шаг 15. Почта](#step15)
  * [Шаг 16. Кеш и сессии в memcached или redis](#step16)
  * [Шаг 17. Поиск через Sphinx](#step17)
  * [Шаг 18. Финальный чек-лист](#step18)
* [Часть 3. Установка сайта с нуля](#part3)
  * [BitrixSetup / Restore](#bitrixsetupphprestorephp)
  * [Установка дистрибутива](#installdistro)
  * [Восстановление из резервной копии Битрикс](#restorebackup)
* [Часть 4. Регулярное резервное копирование](#part4)
  * [Бекап средствами Битрикс по cron](#backupbybitrix)
  * [Дамп базы данных с хоста](#backupmysqldump)
  * [Резервная копия томов Docker](#backupvolumes)
  * [Проверка восстановления](#backuprestoretest)
* [Часть 5. Справочник по окружению](#part5)
  * [Управление контейнерами](#management)
  * [Публикация через Traefik](#traefik)
  * [Работа с файлами сайта из папки](#worksitefolder)
  * [Консоль сервисов](#servicesconsole)
  * [PHP](#php)
  * [Nginx](#nginx)
  * [Проверка системы и тест производительности](#sitecheckerandperfmon)
  * [Версии ПО](#softwareversions)
  * [Кастомизация](#customization)
  * [Сборка или скачивание Docker образов](#dockerimages)
  * [Особенности ОС, сертифицированных ФСТЭК](#fstkos)
* [Часть 6. Диагностика типовых проблем](#part6)

<a id="part1"></a>
# Часть 1. Что нужно знать до старта

<a id="stack"></a>
## Состав сборки

Файл `docker-compose.yml` описывает сеть `dev` (подсеть `10.10.10.0/24`), тома и семь сервисов:

| Сервис | Контейнер | Образ | Назначение |
|---|---|---|---|
| `nginx` | `dev_nginx` | `quay.io/bitrix24/nginx:1.30.4-v1-alpine` | веб-сервер, публикуется наружу через Traefik |
| `php` | `dev_php` | `quay.io/bitrix24/php:8.2.32-fpm-v1-alpine` | PHP-FPM, обработка кода сайта |
| `cron` | `dev_cron` | `quay.io/bitrix24/php:8.2.32-fpm-v1-alpine` | периодические задания, агенты Битрикс |
| `mysql` | `dev_mysql` | `quay.io/bitrix24/percona-server:8.0.46-v1-rhel` | база данных |
| `memcached` | `dev_memcached` | `memcached:1.6.45-alpine` | кеш и/или сессии |
| `redis` | `dev_redis` | `redis:8.2.7-alpine` | кеш и/или сессии |
| `sphinx` | `dev_sphinx` | `quay.io/bitrix24/sphinx:2.2.11-v2-alpine` | полнотекстовый поиск |

**Файлы сайта лежат в каталоге `www/` репозитория** и подключены в контейнеры `php`, `cron` и `nginx` как `/opt/www` (bind mount). С ними можно работать напрямую с хоста - редактором, IDE, git - без захода в контейнер. Подробности и права доступа - в разделе [Работа с файлами сайта из папки](#worksitefolder).

Остальные данные лежат в именованных томах Docker (префикс тома - значение `COMPOSE_PROJECT_NAME` из файла `.env`, по умолчанию `dev`):

| Том | Точка монтирования | Что хранит |
|---|---|---|
| `dev_mysql_lib_data` | `/var/lib/mysql` в `mysql` | файлы базы данных |
| `dev_mysql_log_data` | `/var/log/mysql` в `mysql` | логи базы данных |
| `dev_bxtemp_data` | `/opt/.bx_temp` в `php`, `cron`, `nginx` | временные файлы Битрикс вне корня сайта |
| `dev_msmtp_data` | `/opt/msmtp` в `php`, `cron` | настройки отправки почты (`.msmtprc`) |
| `dev_browscap_data` | `/opt/browscap` в `php`, `cron` | база Browser Capabilities |
| `dev_geoip2_data` | `/opt/geoip2` в `php`, `cron` | базы GeoIP2 (`*.mmdb`) |
| `dev_redis_data` | `/data` в `redis` | дампы redis |

> [!CAUTION]
> Внимание! База данных живет в томе Docker, а не в каталоге репозитория. Команда `docker compose down -v` удаляет тома вместе с базой (файлы сайта в каталоге `www/` при этом останутся). Для остановки используйте `docker compose stop`.

Конфигурационные файлы подключаются с хоста из каталога `confs/` и переживают пересоздание контейнеров:

```bash
www/                 -> /opt/www/              (контейнеры php, cron, nginx) - корень сайта
confs/nginx/         -> /etc/nginx/            (контейнер nginx)
confs/php82/etc/     -> /usr/local/etc/        (контейнеры php и cron)
confs/redis/redis.conf   -> /usr/local/etc/redis/redis.conf
confs/sphinx/sphinx.conf -> /opt/sphinx/conf/sphinx.conf
confs/mysql/         -> конфигурация базы, зашита в образ bitrix24/percona-server
```

Переменные окружения хранятся в файлах:

```bash
.env       - имя проекта, часовой пояс и параметры Traefik, подключен ко всем сервисам
.env_sql   - пароль root базы MySQL, подключен к сервису mysql
.env_php   - версия и тип окружения VMBitrix, подключен к сервисам php и cron
.env_redis - шаблон пароля redis, в текущей сборке ни к одному сервису не подключен (см. Шаг 5)
```

Содержимое `.env`:

```bash
COMPOSE_PROJECT_NAME=dev        # префикс имен контейнеров и томов
TZ=Europe/Moscow                # часовой пояс контейнеров
SITE_DOMAIN=dev.example.com     # домен сайта для роутера Traefik
TRAEFIK_NETWORK=proxy           # имя внешней сети, в которой работает Traefik
TRAEFIK_CERTRESOLVER=le         # имя certresolver-а из конфигурации Traefik
```

> [!IMPORTANT]
> Сайт публикуется наружу **через Traefik**: у сервиса `nginx` нет проброшенного порта, он подключен к внешней сети Traefik и размечен labels. Настройка - в [Шаге 5](#step5), разбор схемы - в разделе [Публикация через Traefik](#traefik).
>
> В этой сборке нет сервисов `postgres`, `push` (pub/sub), `ssl` и `lego`, а в `confs/nginx/conf.d/default.conf` оставлены только HTTP-серверы (порты `80` и `8893`) - TLS терминирует Traefik. Если нужны PostgreSQL, push-сервер или встроенный выпуск SSL-сертификатов - их конфигурации есть в истории git этого репозитория.

<a id="hostrequirements"></a>
## Требования к хосту

- `Docker` и `Docker Compose` v2 (проверка: `docker version` и `docker compose version`);
- свободное место: **размер файлов сайта + размер базы + запас минимум ×2** на время распаковки и импорта дампа;
- оперативная память: минимум 4 ГБ, комфортно от 8 ГБ (`innodb_buffer_pool_size` по умолчанию `1024M`, `maxmemory` redis - `942mb`);
- запущенный Traefik и внешняя сеть Docker, к которой он подключен (по умолчанию `proxy`); порты `80/tcp` и `443/tcp` на фаерволе открывает и слушает сам Traefik.

Установка Docker зависит от ОС хоста:
- `Docker Desktop on Windows`: https://docs.docker.com/desktop/setup/install/windows-install/
- `Docker Desktop on Mac`: https://docs.docker.com/desktop/setup/install/mac-install/
- `Docker Desktop on Linux`: https://docs.docker.com/desktop/setup/install/linux/
- `Docker Engine` (сервер без графической среды): https://docs.docker.com/engine/install/
- `Docker Compose` (если требуется отдельно): https://docs.docker.com/compose/

Для ОС, сертифицированных ФСТЭК, смотрите раздел [Особенности ОС, сертифицированных ФСТЭК](#fstkos).

<a id="howtoreadcommands"></a>
## Как читать команды

Команды в инструкции разделены на два типа:

1. **На исходном сервере** (откуда переносим сайт) - обычный `bash` Linux-сервера.
2. **На новом хосте** (где поднимаем сборку) - команды `docker compose`, выполняются **из каталога репозитория**, рядом с файлом `docker-compose.yml`.

Команды `docker compose ***` одинаковы в `bash` и в `PowerShell`. Отличается только запись путей и переменных: в примерах для Windows используется `.\file.tar.gz`, для Linux/macOS - `./file.tar.gz`.

Конструкции с конвейером (`|`), подстановкой (`$(...)`) и `zcat`/`gunzip` в `PowerShell` не работают. Поэтому во всех шагах, где такие конструкции нужны, команда выполняется **внутри контейнера** - это работает одинаково на всех ОС.

> [!CAUTION]
> Внимание! При использовании ОС ALT 10.x или ALT 11.x compose поставляется отдельным исполняемым файлом, команды нужно писать через тире: `docker-compose ***`.

<a id="part2"></a>
# Часть 2. Пошаговый перенос сайта

Шаги 1-3 выполняются на исходном сервере, шаги 4-18 - на новом хосте со сборкой.

<a id="step1"></a>
## Шаг 1. Резервная копия файлов сайта

Работаем на исходном сервере. Определяем корень сайта (для BitrixVM это обычно `/home/bitrix/www`) и его размер:

```bash
du -sh /home/bitrix/www
df -h
```

Убеждаемся, что свободного места хватает для архива (в среднем архив занимает 40-70% от размера каталога).

Создаем каталог для резервных копий:

```bash
mkdir -p /root/backup
```

Архивируем файлы сайта, исключая кеш и временные каталоги - они не переносятся и только раздувают архив:

```bash
cd /home/bitrix/www
tar -czf /root/backup/www_$(date +%Y%m%d).tar.gz \
  --exclude='./bitrix/backup/*' \
  --exclude='./bitrix/cache/*' \
  --exclude='./bitrix/managed_cache/*' \
  --exclude='./bitrix/stack_cache/*' \
  --exclude='./bitrix/html_pages/*' \
  --exclude='./bitrix/tmp/*' \
  --exclude='./upload/resize_cache/*' \
  --exclude='./upload/tmp/*' \
  .
```

> [!IMPORTANT]
> Каталог `upload` (без `resize_cache` и `tmp`) исключать <b>нельзя</b> - это файлы пользователей сайта.
>
> Файлы `bitrix/.settings.php`, `bitrix/php_interface/dbconn.php` и `bitrix/license_key.php` тоже должны попасть в архив: из них берутся текущие настройки, которые мы будем править на шаге 9.

Проверяем архив и запоминаем его контрольную сумму:

```bash
tar -tzf /root/backup/www_*.tar.gz > /dev/null && echo "archive OK"
ls -lh /root/backup/
sha256sum /root/backup/www_*.tar.gz
```

Заодно сохраняем список версий исходного окружения - пригодится при выборе версий контейнеров на шаге 5:

```bash
php -v
mysql --version
nginx -v
```

<a id="step2"></a>
## Шаг 2. Резервная копия базы данных

Продолжаем работать на исходном сервере.

**2.1. Узнаем параметры подключения к базе.** Они лежат в файле `bitrix/php_interface/dbconn.php` и/или в `bitrix/.settings.php`:

```bash
grep -E "DBHost|DBName|DBLogin|DBPassword" /home/bitrix/www/bitrix/php_interface/dbconn.php
```

```bash
php -r '$c = include "/home/bitrix/www/bitrix/.settings.php"; print_r($c["connections"]["value"]["default"]);'
```

Записываем: хост, имя базы, логин, пароль. Имя базы у продуктов Битрикс чаще всего `sitemanager`.

**2.2. Смотрим размер базы, кодировку и типы таблиц:**

```bash
mysql -u root -p -e "
SELECT table_schema, ROUND(SUM(data_length+index_length)/1024/1024) AS size_mb, COUNT(*) AS tables
FROM information_schema.tables WHERE table_schema='sitemanager' GROUP BY table_schema;
SELECT engine, COUNT(*) FROM information_schema.tables WHERE table_schema='sitemanager' GROUP BY engine;
SHOW CREATE DATABASE sitemanager;"
```

Запоминаем три вещи:
- **количество таблиц** - по нему будем проверять полноту импорта на шаге 8;
- **кодировку и сравнение (collation) базы** из `SHOW CREATE DATABASE` - с ними будем создавать базу на новом хосте;
- **типы таблиц (engine)** - если есть таблицы `MyISAM`, читаем предупреждение ниже.

**2.3. Снимаем дамп.**

Для базы целиком на `InnoDB` (обычный случай) дамп снимается без блокировки сайта:

```bash
mkdir -p /root/backup
mysqldump -u root -p \
  --single-transaction --quick --hex-blob \
  --routines --triggers --events \
  --default-character-set=utf8mb4 \
  --set-gtid-purged=OFF --no-tablespaces \
  sitemanager | gzip -1 > /root/backup/db_$(date +%Y%m%d).sql.gz
```

Пояснения к опциям:
- `--single-transaction` - согласованный снимок без блокировки таблиц `InnoDB`;
- `--quick` - выгрузка построчно, без загрузки таблицы в память целиком;
- `--hex-blob` - корректный перенос двоичных полей;
- `--routines --triggers --events` - процедуры, триггеры и события базы;
- `--set-gtid-purged=OFF` - убирает из дампа GTID-метки исходного сервера (в MariaDB такой опции нет, уберите ее);
- `--no-tablespaces` - нужна для MySQL 5.7.31+/8.x, если у пользователя нет привилегии `PROCESS`.

> [!CAUTION]
> Внимание! Если в базе есть таблицы `MyISAM`, опция `--single-transaction` не гарантирует их целостность.
>
> В этом случае либо переводим сайт в режим технических работ и снимаем дамп с `--lock-all-tables`, либо предварительно конвертируем таблицы в `InnoDB`.

Если дамп снимается со старого сервера более новым клиентом `mysqldump` 8.x, добавьте `--column-statistics=0`.

**2.4. Проверяем дамп.** У корректного дампа последняя строка содержит `Dump completed`:

```bash
zcat /root/backup/db_*.sql.gz | tail -1
zcat /root/backup/db_*.sql.gz | grep -c "^CREATE TABLE"
ls -lh /root/backup/
sha256sum /root/backup/db_*.sql.gz
```

Количество `CREATE TABLE` должно совпадать с количеством таблиц из пункта 2.2.

> [!CAUTION]
> Внимание! Дамп, у которого нет строки `Dump completed`, обрезан. Переносить такой дамп нельзя - снимите его заново.

<a id="step3"></a>
## Шаг 3. Проверка и перенос резервных копий

На руках должно быть два файла:

```bash
www_YYYYMMDD.tar.gz  - файлы сайта
db_YYYYMMDD.sql.gz   - дамп базы данных
```

Копируем их на новый хост, например по `scp` (выполняется на новом хосте):

```bash
scp root@old-server:/root/backup/www_*.tar.gz .
scp root@old-server:/root/backup/db_*.sql.gz .
```

Для Windows подойдет `scp` из состава OpenSSH или любой SFTP-клиент.

После копирования **обязательно** сверяем контрольные суммы с теми, что получили на шагах 1 и 2:

- Linux/macOS:
```bash
sha256sum www_*.tar.gz db_*.sql.gz
```

- Windows PowerShell:
```powershell
Get-FileHash .\www_*.tar.gz, .\db_*.sql.gz -Algorithm SHA256
```

Суммы совпали - идем дальше. Не совпали - копируем файлы заново.

<a id="step4"></a>
## Шаг 4. Docker и получение сборки на новом хосте

Устанавливаем `Docker` и `Docker Compose` по документации для вашей ОС (ссылки в разделе [Требования к хосту](#hostrequirements)) и проверяем:

```bash
docker version
docker compose version
```

Получаем сборку и переходим в ее каталог:

```bash
git clone <URL репозитория> env-docker
cd env-docker
```

Все дальнейшие команды `docker compose` выполняются из этого каталога.

Предварительно скачиваем образы (необязательно, но так первый запуск пройдет быстрее):

```bash
docker compose pull
```

<a id="step5"></a>
## Шаг 5. Настройка сборки до первого запуска

> [!CAUTION]
> Внимание! Все правки этого шага делаются <b>до</b> первого запуска контейнеров. Пароль базы данных после создания тома `dev_mysql_lib_data` меняется уже средствами MySQL, а не через файл `.env_sql`.

**5.1. Имя проекта и часовой пояс.** Файл `.env`:

```bash
COMPOSE_PROJECT_NAME=dev
TZ=Europe/Moscow
```

`COMPOSE_PROJECT_NAME` задает префикс имен контейнеров (`dev_nginx`, `dev_mysql`, ...) и томов (`dev_www_data`, ...). Меняйте его, если на одном хосте будет несколько сайтов на этой сборке.

Часовой пояс дублируется для PHP в файле `confs/php82/etc/php/conf.d/timezone.ini`:

```bash
date.timezone = Europe/Moscow
```

Значения в обоих файлах должны совпадать. Если на шаге 5.4 выбрана другая версия PHP, правится файл соответствующего каталога `confs/php83`, `confs/php84` или `confs/php85`.

**5.2. Пароль root для MySQL.** Генерируем уникальный пароль:

```bash
docker pull alpine:3.23
docker container run --rm --name mysql_password_generate alpine:3.23 sh -c "(cat /dev/urandom | tr -dc A-Za-z0-9\?\!\@\-\_\+\%\(\)\{\}\[\]\= | head -c 16) | tr -d '\' | tr -d '^' && echo ''"
```

В файле `.env_sql` заменяем шаблон `CHANGE_MYSQL_ROOT_PASSWORD_HERE` на полученное значение:

```bash
MYSQL_ROOT_PASSWORD="ваш_пароль"
```

Сохраняем пароль в менеджере паролей - он понадобится на шаге 8 и при любом обслуживании базы.

**5.3. Пароль Redis (по желанию).** По умолчанию `redis` работает без пароля и доступен только внутри сети `dev` (порт наружу не проброшен). Если пароль все же нужен, генерируем его:

```bash
docker container run --rm --name redis_password_generate alpine:3.23 sh -c "(cat /dev/urandom | tr -dc A-Za-z0-9\.\,\?\!\@\$\-\+\=\(\)\{\}\[\] | head -c 16) | tr -d '\' | tr -d '^' && echo ''"
```

В файле `confs/redis/redis.conf` убираем `#` в первой строке и подставляем пароль:

```bash
requirepass ваш_пароль
```

Этот же пароль позже указывается в блоках `cache`/`session` файла `/bitrix/.settings.php` (см. [Шаг 16](#step16)).

> [!NOTE]
> Файл `.env_redis` в текущей сборке не подключен ни к одному сервису (он использовался push-сервером). Пароль redis задается только в `confs/redis/redis.conf`.

**5.4. Версии PHP и MySQL.** По умолчанию: PHP `8.2.32`, Percona Server `8.0.46`. Доступны PHP `8.3.x`, `8.4.x`, `8.5.x` и Percona Server `8.4.x`. Переключение - комментированием строк в `docker-compose.yml`, подробности в разделе [Версии ПО](#softwareversions).

Ориентируйтесь на версии исходного сервера (собранные на шаге 1) и на требования вашей версии продукта: переносить сайт лучше на ту же или соседнюю версию PHP, а обновляться уже после успешного переезда.

**5.5. Публикация сайта через Traefik.** Порт наружу сборка не пробрасывает - запросы к сайту приходят из Traefik по внутренней сети.

В файле `.env` указываем домен сайта, имя внешней сети Traefik и имя certresolver-а:

```bash
SITE_DOMAIN=site.example.com
TRAEFIK_NETWORK=proxy
TRAEFIK_CERTRESOLVER=le
```

Убеждаемся, что внешняя сеть существует и Traefik подключен к ней:

```bash
docker network ls | grep proxy
```

Если сети нет, создаем ее и подключаем к ней Traefik:

```bash
docker network create proxy
```

Проверяем, что домен из `SITE_DOMAIN` указывает A-записью на IP хоста с Traefik - иначе выпуск сертификата не пройдет.

Подробный разбор labels, требований к Traefik и типовых ошибок - в разделе [Публикация через Traefik](#traefik).

> [!NOTE]
> Если Traefik пока нет, а сайт нужно открыть напрямую, раскомментируйте в `docker-compose.yml` у сервиса `nginx` блок:
> ```yaml
>     ports:
>       - "8588:80"
> ```
> и откройте порт на фаерволе: `ufw allow 8588` или `firewall-cmd --add-port=8588/tcp --permanent && firewall-cmd --reload`.

**5.6. Параметры MySQL под размер вашей базы.** Файл `confs/mysql/etc/my.cnf.d/my.cnf`, значения по умолчанию:

```bash
max_allowed_packet = 16M
innodb_buffer_pool_size = 1024M
max_connections = 43
```

Если база больше 2-3 ГБ или в ней есть крупные двоичные поля, до первого запуска увеличиваем:

```bash
max_allowed_packet = 128M
innodb_buffer_pool_size = 2048M
```

`innodb_buffer_pool_size` разумно держать в пределах 50-70% доступной хосту памяти. Слишком маленький `max_allowed_packet` - самая частая причина обрыва импорта дампа на шаге 8.

<a id="step6"></a>
## Шаг 6. Первый запуск контейнеров

Запускаем все контейнеры в фоне:

```bash
docker compose up -d
```

Проверяем статус - все сервисы должны быть в состоянии `Up`/`running`:

```bash
docker compose ps
```

Смотрим логи, ошибок быть не должно:

```bash
docker compose logs --tail=50
```

Отдельно убеждаемся, что база инициализировалась (в логе должна быть строка вида `ready for connections`):

```bash
docker compose logs mysql | tail -20
```

Проверяем, что тома созданы:

```bash
docker volume ls
```

Проверяем, что nginx отвечает (в ответ на пустой корень ожидаемы 403/404 - сайта еще нет):

```bash
docker compose exec nginx sh -c "wget -S -O /dev/null http://127.0.0.1/ 2>&1 | head -5"
```

Проверяем, что контейнер `nginx` подключен к сети Traefik и Traefik его увидел:

```bash
docker inspect dev_nginx --format "{{json .NetworkSettings.Networks}}"
docker compose logs --tail=20 nginx
```

В списке сетей должны быть обе: `dev_dev` и `proxy`. Если Traefik запущен с дашбордом, роутер сайта появится в разделе `HTTP Routers` с именем `dev@docker`.

<a id="step7"></a>
## Шаг 7. Распаковка файлов сайта в папку

Корень сайта - каталог `www/` репозитория, подключенный в контейнеры `php`, `cron` и `nginx` как `/opt/www`. Архив распаковывается прямо на хосте, копировать что-либо внутрь контейнера не нужно.

**7.1. Распаковываем архив в каталог `www/`:**

- Linux/macOS:
```bash
tar -xzf ./www_YYYYMMDD.tar.gz -C ./www/
```

- Windows PowerShell (`tar` входит в состав Windows 10/11):
```powershell
tar -xzf .\www_YYYYMMDD.tar.gz -C .\www\
```

**7.2. Выставляем владельца.** Внутри контейнеров код сайта работает от пользователя `bitrix` с `uid:gid = 979:979`. На Linux-хосте выставляем владельца на хосте:

```bash
sudo chown -R 979:979 ./www
```

На Windows с Docker Desktop (файлы лежат на диске Windows) владелец не имеет значения - права проксируются, шаг можно пропустить.

Каталог временных файлов остается томом, его владельца выставляем из контейнера:

```bash
docker compose exec --user=root php sh -c "chown -R bitrix:bitrix /opt/.bx_temp"
```

**7.3. Приводим права к обычным**, если на исходном сервере они были нестандартными (на больших сайтах команда выполняется несколько минут):

```bash
docker compose exec --user=root php sh -c "find /opt/www -type d -exec chmod 755 {} \; ; find /opt/www -type f -exec chmod 644 {} \;"
```

**7.4. Проверяем результат** - список файлов должен совпадать при взгляде с хоста и из контейнера:

```bash
ls -la ./www | head -20
docker compose exec --user=bitrix php sh -c "ls -la /opt/www | head -20; du -sh /opt/www"
```

В корне должны быть каталоги `bitrix`, `upload`, файл `index.php` и т.д.

> [!IMPORTANT]
> Каталог `www/` исключен из git через `.gitignore` - файлы сайта в репозиторий сборки не попадают. Если сайт ведется в отдельном git-репозитории, инициализируйте его прямо внутри `www/`.

Особенности работы с папкой (производительность на Windows, права, IDE) - в разделе [Работа с файлами сайта из папки](#worksitefolder).

<a id="step8"></a>
## Шаг 8. Создание базы данных и импорт дампа

**8.1. Создаем базу и пользователя.** Кодировку и сравнение берем из пункта 2.2 (в примере - типовые для Битрикс `utf8mb4` / `utf8mb4_unicode_ci`):

```bash
docker compose exec mysql mysql -u root -p -e "CREATE DATABASE sitemanager CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"
```

```bash
docker compose exec mysql mysql -u root -p -e "CREATE USER 'bitrix'@'%' IDENTIFIED BY 'ПАРОЛЬ_ПОЛЬЗОВАТЕЛЯ_БД'; GRANT ALL PRIVILEGES ON sitemanager.* TO 'bitrix'@'%'; FLUSH PRIVILEGES;"
```

Пароль вводится по запросу - это пароль `root`, заданный на шаге 5.2.

> [!IMPORTANT]
> Кодировка и сравнение новой базы должны совпадать с исходными. Расхождение приводит к "битым" русским буквам и к ошибкам сравнения строк уже после переезда.

**8.2. Копируем дамп внутрь контейнера базы:**

- Windows PowerShell:
```powershell
docker compose cp .\db_YYYYMMDD.sql.gz mysql:/tmp/db.sql.gz
```

- Linux/macOS:
```bash
docker compose cp ./db_YYYYMMDD.sql.gz mysql:/tmp/db.sql.gz
```

**8.3. Импортируем дамп.** Команда выполняется внутри контейнера, поэтому работает одинаково на всех ОС:

```bash
docker compose exec mysql bash -c "gunzip -c /tmp/db.sql.gz | mysql -u root -p --default-character-set=utf8mb4 sitemanager"
```

Импорт крупной базы занимает от нескольких минут до нескольких часов. Прервать процесс на середине нельзя - при обрыве базу нужно удалить (`DROP DATABASE`), создать заново и повторить импорт.

Если импорт завершается ошибкой `MySQL server has gone away` или `Packet too large` - увеличиваем `max_allowed_packet` в `confs/mysql/etc/my.cnf.d/my.cnf` (см. пункт 5.6), перезапускаем базу и импортируем заново:

```bash
docker compose restart mysql
```

**8.4. Проверяем импорт.** Количество таблиц должно совпасть с исходным (пункт 2.2):

```bash
docker compose exec mysql mysql -u root -p -e "SELECT COUNT(*) AS tables FROM information_schema.tables WHERE table_schema='sitemanager'; SELECT COUNT(*) AS users FROM sitemanager.b_user;"
```

**8.5. Удаляем дамп из контейнера** - он больше не нужен и занимает место:

```bash
docker compose exec mysql bash -c "rm -f /tmp/db.sql.gz"
```

<a id="step9"></a>
## Шаг 9. Подключение сайта к базе данных

Файлы сайта перенесены, база импортирована - осталось связать их. Внутри сети сборки база доступна по имени сервиса `mysql`, порт `3306`.

Для правки файлов ставим редактор в контейнере `php` (один раз):

```bash
docker compose exec --user=root php sh -c "apk add mc"
```

**9.1. Файл `/opt/www/bitrix/php_interface/dbconn.php`** (используется классическим ядром):

```bash
docker compose exec --user=bitrix php sh -c "mcedit /opt/www/bitrix/php_interface/dbconn.php"
```

Приводим параметры подключения к виду:

```php
$DBHost = "mysql";
$DBName = "sitemanager";
$DBLogin = "bitrix";
$DBPassword = "ПАРОЛЬ_ПОЛЬЗОВАТЕЛЯ_БД";
```

**9.2. Файл `/opt/www/bitrix/.settings.php`** (используется ядром D7):

```bash
docker compose exec --user=bitrix php sh -c "mcedit /opt/www/bitrix/.settings.php"
```

Приводим блок `connections` к виду:

```php
  'connections' => array(
    'value' => array(
      'default' => array(
        'className' => '\\Bitrix\\Main\\DB\\MysqliConnection',
        'host' => 'mysql',
        'database' => 'sitemanager',
        'login' => 'bitrix',
        'password' => 'ПАРОЛЬ_ПОЛЬЗОВАТЕЛЯ_БД',
        'options' => 2,
      ),
    ),
    'readonly' => true,
  ),
```

**9.3. Проверяем подключение к базе из PHP:**

```bash
docker compose exec --user=bitrix php sh -c "php -r '\$c = new mysqli(\"mysql\", \"bitrix\", \"ПАРОЛЬ_ПОЛЬЗОВАТЕЛЯ_БД\", \"sitemanager\"); echo \$c->connect_error ?: \"connect OK\"; echo PHP_EOL;'"
```

> [!NOTE]
> Если появляется ошибка аутентификации, а сайт работает на старой версии ядра, создайте пользователя базы с устаревшим плагином:
> `CREATE USER 'bitrix'@'%' IDENTIFIED WITH mysql_native_password BY 'ПАРОЛЬ';`
> Учтите: этот плагин объявлен устаревшим в MySQL 8.0.34 и удален в 8.4, поэтому для Percona Server `8.4.x` он недоступен.

<a id="step10"></a>
## Шаг 10. Очистка кеша и настроек старого окружения

Сайт переехал вместе с настройками старого сервера. Их нужно вычистить, иначе сайт будет обращаться к недоступным хостам и путям.

**10.1. Ищем пути старого сервера в конфигурации:**

```bash
docker compose exec --user=bitrix php sh -c "grep -rn '/home/bitrix' /opt/www/bitrix/.settings.php /opt/www/bitrix/.settings_extra.php /opt/www/bitrix/php_interface/ 2>/dev/null"
```

Все найденные пути заменяем на пути сборки: корень сайта - `/opt/www`, каталог временных файлов - `/opt/.bx_temp`.

**10.2. Правим `/opt/www/bitrix/.settings.php`:**

- блоки `cache` и `session`: на время переезда переводим на файлы - удаляем блоки целиком (кеш по умолчанию файловый) либо указываем новые хосты `memcached`/`redis` (см. [Шаг 16](#step16)). Старые адреса вида `127.0.0.1` или IP прежнего сервера оставлять нельзя;
- блок `pull` (push-сервер): в этой сборке push-сервера нет, блок удаляем или выставляем `'nginx' => 'N'`, `'push' => 'N'`, `'websocket' => 'N'`;
- блок `smtp`, пути к лог-файлам, `composer.config_path` - проверяем, что указанные пути существуют в новом окружении.

Файл `/opt/www/bitrix/.settings_extra.php` (если есть) проверяем по тем же правилам - настройки из него перекрывают `.settings.php`.

**10.3. Чистим кеш и сгенерированные файлы:**

```bash
docker compose exec --user=bitrix php sh -c "rm -rf /opt/www/bitrix/cache/* /opt/www/bitrix/managed_cache/* /opt/www/bitrix/stack_cache/* /opt/www/bitrix/html_pages/* /opt/www/upload/resize_cache/*"
```

**10.4. Перезапускаем PHP и cron, чтобы сбросить кеш опкода:**

```bash
docker compose restart php cron
```

<a id="step11"></a>
## Шаг 11. Первое открытие сайта и проверка

**11.1. Определяем адрес.** Сайт открывается по домену из `SITE_DOMAIN`, TLS выдает Traefik:

```bash
https://site.example.com/
```

Обращение по `http://` должно автоматически перенаправляться на `https://` - за это отвечает middleware редиректа в labels сервиса `nginx`.

> [!IMPORTANT]
> <b>НЕ</b> используйте `127.0.0.1` или `localhost` при работе с сайтом - часть функционала Битрикс (проверка домена, ссылки в письмах, авторизация) работает некорректно. Работайте по домену, который указан в `SITE_DOMAIN`.
>
> В примерах ниже используется адрес `https://site.example.com`.

Открываем сайт в браузере. Ожидаемый результат - публичная часть сайта или страница авторизации `/bitrix/admin/`.

**11.2. Если что-то не открылось**, смотрим логи снизу вверх - сначала сам сайт, потом прокси:

```bash
docker compose logs --tail=100 nginx
docker compose logs --tail=100 php
docker logs --tail=100 traefik
```

Полезная проверка: сайт можно запросить напрямую, минуя Traefik, изнутри контейнера. Если так страница отдается, а через домен нет - проблема в Traefik или DNS, а не в сборке:

```bash
docker compose exec nginx sh -c "wget -S -O /dev/null -H 'X-Forwarded-Proto: https' http://127.0.0.1/ 2>&1 | head -10"
```

Типовые ситуации разобраны в разделе [Диагностика типовых проблем](#part6).

**11.3. Базовая проверка конфигурации окружения** скриптом `bitrix_server_test.php`:

```bash
docker compose exec --user=bitrix php sh -c "cd /opt/www && wget https://dev.1c-bitrix.ru/download/scripts/bitrix_server_test.php"
```

Открываем в браузере:

```bash
https://site.example.com/bitrix_server_test.php
```

> [!CAUTION]
> Внимание! После проверки скрипт `bitrix_server_test.php` обязательно удаляем:
> ```bash
> docker compose exec --user=bitrix php sh -c "rm -f /opt/www/bitrix_server_test.php"
> ```

**11.4. Проверка системы** средствами продукта - страница `Проверка системы` (`/bitrix/admin/site_checker.php?lang=ru`), запускаем `Тестирование конфигурации`. На этом этапе допустимы замечания по почте, cron и поиску - их настраиваем на шагах 13, 15 и 17.

<a id="step12"></a>
## Шаг 12. Настройки модулей, лицензия и обновления

**12.1. Главный модуль (main)** - страница `/bitrix/admin/settings.php?lang=ru&mid=main`:

- вкладка `Настройки`:
  - `Название сайта`;
  - `URL сайта (без http://)` - домен из `SITE_DOMAIN`, например `site.example.com` (без порта, сайт работает на стандартном 443);
  - включаем `Быстрая отдача файлов через Nginx` - в сборке она сконфигурирована;
- вкладка `Авторизация`: снимаем `Продлевать сессию при активности посетителя в окне браузера`;
- вкладка `Журнал событий`: отмечаем нужные события;
- вкладка `Система обновлений`: заполняем `Лицензионный ключ` и `Имя сервера, содержащего обновления`.

Сохраняем настройки.

**12.2. Настройки сайта** - страница `Список сайтов` (`/bitrix/admin/site_admin.php?lang=ru`). Проверяем у каждого сайта поля `URL сервера` и `Путь к корневой папке веб-сервера` (должно быть пусто либо `/opt/www`).

**12.3. Обновления.** Переходим на страницу `Обновление платформы` (`/bitrix/admin/update_system.php?lang=ru`), обновляем систему обновлений, принимаем лицензионные соглашения и ставим доступные обновления. Затем - `Обновление решений` (`/bitrix/admin/update_system_partner.php?lang=ru`).

> [!NOTE]
> Обновляйте платформу только после того, как перенесенный сайт открылся и работает. Совмещать переезд и обновление в одном шаге - плохая идея: непонятно, что именно сломалось.

<a id="step13"></a>
## Шаг 13. Cron и агенты

За периодические задания отвечает отдельный контейнер `cron` на образе `bitrix24/php`. По умолчанию он раз в минуту от пользователя `bitrix` выполняет:

```bash
php -f /opt/www/bitrix/modules/main/tools/cron_events.php
```

Чтобы агенты и рассылка почты выполнялись из-под cron, а не по хитам посетителей, добавляем в файл `/opt/www/bitrix/php_interface/dbconn.php` строку:

```php
define("BX_CRONTAB_SUPPORT", true);
```

Редактировать можно из административной части на странице `Управление структурой` (`/bitrix/admin/fileman_admin.php?lang=ru&path=%2F`) или из консоли:

```bash
docker compose exec --user=bitrix php sh -c "mcedit /opt/www/bitrix/php_interface/dbconn.php"
```

Проверяем работу:

```bash
docker compose logs --tail=50 cron
```

и на страницах `Проверка системы` (`/bitrix/admin/site_checker.php?lang=ru`) и `Список агентов` (`/bitrix/admin/agent_list.php?lang=ru`).

Свои задания добавляются в каталоги `/etc/periodic/{1min,15min,hourly,daily,weekly,monthly}` контейнера `cron` - пример есть в разделе [Бекап средствами Битрикс по cron](#backupbybitrix).

<a id="step14"></a>
## Шаг 14. Временные файлы вне корня сайта

Конфигурация `nginx` в сборке уже подготовлена так, чтобы временные файлы хранились вне корня сайта (том `dev_bxtemp_data`, каталог `/opt/.bx_temp`) - это существенно повышает защищенность от известных атак.

Для завершения настройки добавляем в файл `/opt/www/bitrix/php_interface/dbconn.php` строку:

```php
define("BX_TEMPORARY_FILES_DIRECTORY", "/opt/.bx_temp");
```

Проверить настройку можно на странице `Сканер безопасности` (`/bitrix/admin/security_scanner.php?lang=ru`) модуля `Проактивная защита (security)`.

<a id="step15"></a>
## Шаг 15. Почта

Образ `bitrix24/php` содержит mta `msmtp`. Его настройки для аккаунта `default` хранятся в файле `/opt/msmtp/.msmtprc` (том `dev_msmtp_data`).

Пример настроек для локального почтового сервера без TLS:

```bash
host mail.server.local
port 25
from info@mail.server.local
user info@mail.server.local
password password_example
tls off
tls_starttls off
tls_certcheck off
```

Пример полного набора настроек для `Gmail`:

```bash
# smtp account configuration for default
account default
logfile /proc/self/fd/2
host smtp.gmail.com
port 587
from [LOGIN]@gmail.com
user [LOGIN]@gmail.com
password [APP_PASSWORD]
auth login
aliases /etc/aliases
keepbcc off
tls on
tls_starttls on
tls_certcheck on
tls_trust_file /etc/ssl/certs/ca-certificates.crt
protocol smtp
```

Настройки почтовых сервисов приведены в курсе: https://dev.1c-bitrix.ru/learning/course/index.php?COURSE_ID=37&LESSON_ID=12265

Редактируем файл:

```bash
docker compose exec --user=bitrix php sh -c "mcedit /opt/msmtp/.msmtprc"
```

После этого в административной части указываем e-mail отправителя в настройках модулей:
- `Главный модуль (main)` (`/bitrix/admin/settings.php?lang=ru&mid=main`)
- `Email-маркетинг (sender)` (`/bitrix/admin/settings.php?lang=ru&mid=sender`)
- `Интернет-магазин (sale)` (`/bitrix/admin/settings.php?lang=ru&mid=sale`)
- `Подписка, рассылки (subscribe)` (`/bitrix/admin/settings.php?lang=ru&mid=subscribe`)
- `Форум (forum)` (`/bitrix/admin/settings.php?lang=ru&mid=forum`)

Проверяем отправку на странице `Проверка системы` (`/bitrix/admin/site_checker.php?lang=ru`), запустив `Тестирование конфигурации`. Результат отображается в блоке `Дополнительные функции`:

```
Отправка почты - Отправлено. Время отправки: 1.36 сек.
Отправка почтового сообщения больше 64Кб - Отправлено. Время отправки: 1.55 сек.
```

Детали отправки и ошибки видны в логе контейнера `php`:

```bash
docker compose logs --tail=50 php
```

**Альтернатива: SMTP-сервера отправителя** (без msmtp, средствами главного модуля). В файл `/opt/www/bitrix/.settings.php` добавляем блок:

```php
    'smtp' => [
        'value' => [
            'enabled' => true,
            'debug' => true,
            'log_file' => $_SERVER["DOCUMENT_ROOT"]."/bitrix/modules/smtp_mailer.log",
        ],
        'readonly' => false,
    ],
```

Затем на странице `Добавление SMTP подключения` (`/bitrix/admin/smtp_edit.php?lang=ru`) заполняем параметры SMTP и сохраняем. Лог отправки - в файле `/bitrix/modules/smtp_mailer.log`.

Документация: https://dev.1c-bitrix.ru/learning/course/index.php?COURSE_ID=43&LESSON_ID=23612&LESSON_PATH=3913.3516.5062.2795.23612

> [!NOTE]
> `Проверка системы` при тестировании не использует SMTP-сервера отправителя, два почтовых теста будут красными. Это можно проигнорировать.

**Логирование отправки почты в файл** (для диагностики). В `/opt/www/bitrix/php_interface/dbconn.php` добавляем:

```php
define("LOG_FILENAME", $_SERVER["DOCUMENT_ROOT"]."/bitrix/modules/mail_log.txt");
```

В файл `/opt/www/bitrix/php_interface/init.php` (создаем, если его нет) добавляем функцию:

```php
function custom_mail($to, $subject, $message, $additional_headers='', $additional_parameters='')
{
    AddMessage2Log(
        'To: '.$to.PHP_EOL.
        'Subject: '.$subject.PHP_EOL.
        'Message: '.$message.PHP_EOL.
        'Headers: '.$additional_headers.PHP_EOL.
        'Params: '.$additional_parameters.PHP_EOL
    );
    if ($additional_parameters!='')
    {
        return @mail($to, $subject, $message, $additional_headers, $additional_parameters);
    }
    else
    {
        return @mail($to, $subject, $message, $additional_headers);
    }
}
```

Документация: https://dev.1c-bitrix.ru/api_help/main/functions/other/bxmail.php

<a id="step16"></a>
## Шаг 16. Кеш и сессии в memcached или redis

После успешного переезда переводим кеш и сессии с файлов на `memcached` или `redis` - оба контейнера уже запущены и доступны внутри сети по именам `memcached:11211` и `redis:6379`.

Настройки задаются в файле `/opt/www/bitrix/.settings.php`, редактировать удобно через `Управление структурой` (`/bitrix/admin/fileman_admin.php?lang=ru&path=%2F`).

**Кеширование**, вариант `memcache`:

```php
        // cache on memcache
        'cache' => [
                'value' => [
                        'type' => [
                                'class_name' => '\\Bitrix\\Main\\Data\\CacheEngineMemcache',
                                'extension' => 'memcache'
                        ],
                        'memcache' => [
                                'host' => 'memcached',
                                'port' => '11211',
                        ]
                ],
                'sid' => $_SERVER["DOCUMENT_ROOT"]."#01234"
        ],
```

вариант `redis` без пароля:

```php
        // cache on redis
        'cache' => [
                'value' => [
                        'type' => [
                                'class_name' => '\\Bitrix\\Main\\Data\\CacheEngineRedis',
                                'extension' => 'redis'
                        ],
                        'redis' => [
                                'host' => 'redis',
                                'port' => '6379',
                        ]
                ],
                'sid' => $_SERVER["DOCUMENT_ROOT"]."#01234"
        ],
```

вариант `redis` с паролем (пароль из `confs/redis/redis.conf`, см. пункт 5.3):

```php
        // cache on redis with password
        'cache' => [
                'value' => [
                        'type' => [
                                'class_name' => '\\Bitrix\\Main\\Data\\CacheEngineRedis',
                                'extension' => 'redis'
                        ],
                        'redis' => [
                                'host' => 'redis',
                                'port' => '6379',
                                'password' => 'ВАШ_ПАРОЛЬ_REDIS',
                        ]
                ],
                'sid' => $_SERVER["DOCUMENT_ROOT"]."#01234"
        ],
```

**Хранение сессий**, вариант `memcache` (разделенная сессия):

```php
        // sessions on memcache (separated)
        'session' => [
                'value' => [
                        'lifetime' => 14400,
                        'mode' => 'separated',
                        'handlers' => [
                                'kernel' => 'encrypted_cookies',
                                'general' => [
                                        'type' => 'memcache',
                                        'port' => '11211',
                                        'host' => 'memcached',
                                ],
                        ],
                ],
        ],
```

вариант `redis`:

```php
        // sessions on redis
        'session' => [
                'value' => [
                        'mode' => 'default',
                        'handlers' => [
                                'general' => [
                                        'type' => 'redis',
                                        'port' => '6379',
                                        'host' => 'redis',
                                        //'password' => 'ВАШ_ПАРОЛЬ_REDIS',
                                ],
                        ],
                ],
        ],
```

вариант `database`:

```php
        // sessions on database
        'session' => [
                'value' => [
                        'mode' => 'default',
                        'handlers' => [
                                'general' => [
                                        'type' => 'database',
                                ],
                        ],
                ],
        ],
```

Документация:
- кеш: https://dev.1c-bitrix.ru/learning/course/index.php?COURSE_ID=43&CHAPTER_ID=02795&LESSON_PATH=3913.3516.5062.2795#cache
- сессии: https://dev.1c-bitrix.ru/learning/course/index.php?COURSE_ID=43&LESSON_ID=14026

**Альтернатива - модуль Веб-кластер (cluster)**, если он есть в редакции продукта:

- `memcache`: на странице настроек модуля (`/bitrix/admin/settings.php?lang=ru&mid=cluster`) выбираем `Использовать для хранения кеша` - `Memcache`, затем на странице `Подключения к memcached` (`/bitrix/admin/cluster_memcache_list.php?lang=ru&group_id=1`) добавляем подключение: `Сервер` - `memcached`, `Порт` - `11211`, `Процент распределения нагрузки` - `100`, и нажимаем `Начать использовать`;
- `redis`: выбираем `Использовать для хранения кеша` - `Redis`, на странице `Подключения к redis` (`/bitrix/admin/cluster_redis_list.php?lang=ru&group_id=1`) добавляем подключение: `Сервер` - `redis`, `Порт` - `6379`, и нажимаем `Начать использовать`.

Проверить работу кеша можно в консоли сервиса:

```bash
docker compose exec --user=root redis sh -c "redis-cli -h 127.0.0.1 -p 6379 info keyspace"
docker compose exec --user=root memcached sh -c "echo stats | nc 127.0.0.1 11211 | head -20"
```

<a id="step17"></a>
## Шаг 17. Поиск через Sphinx

Полнотекстовый поиск обслуживает контейнер `sphinx` (индекс типа `rt`, конфигурация - `confs/sphinx/sphinx.conf`).

Настройка:
- переходим в настройки модуля `Поиск` (`/bitrix/admin/settings.php?lang=ru&mid=search`);
- вкладка `Морфология`:
  - `Полнотекстовый поиск с помощью` - `Sphinx`;
  - `Строка подключения для управления индексом (протокол MySql)` - `sphinx:9306`;
  - `Идентификатор индекса` - `bitrix` (не меняем).

Сохраняем настройки модуля.

На странице `Переиндексация сайта` (`/bitrix/admin/search_reindex.php?lang=ru`) снимаем галку `Переиндексировать только измененные` и выполняем полную переиндексацию.

> [!IMPORTANT]
> После переезда переиндексация обязательна: индекс Sphinx не переносится вместе с файлами и базой, он пуст.

Документация: https://dev.1c-bitrix.ru/learning/course/index.php?COURSE_ID=35&CHAPTER_ID=04507&LESSON_PATH=3906.4507

<a id="step18"></a>
## Шаг 18. Финальный чек-лист

Проверяем перед тем, как считать переезд завершенным:

- [ ] все контейнеры в состоянии `Up`: `docker compose ps`;
- [ ] в логах нет повторяющихся ошибок: `docker compose logs --tail=100`;
- [ ] публичная часть сайта открывается, картинки и файлы из `/upload/` отдаются;
- [ ] административная часть открывается, авторизация работает;
- [ ] количество таблиц и пользователей в базе совпадает с исходным (пункт 8.4);
- [ ] `Проверка системы` (`/bitrix/admin/site_checker.php?lang=ru`) - без критичных ошибок;
- [ ] в настройках главного модуля указан правильный `URL сайта`;
- [ ] агенты выполняются на cron: `/bitrix/admin/agent_list.php?lang=ru`;
- [ ] почта уходит (тест в `Проверке системы` или письмо самому себе);
- [ ] поиск работает после полной переиндексации;
- [ ] служебные скрипты (`bitrix_server_test.php`, `bitrixsetup.php`, `restore.php`) удалены из корня сайта;
- [ ] архив сайта и дамп базы удалены из контейнеров (`/tmp`), но сохранены в надежном месте вне хоста;
- [ ] настроено регулярное резервное копирование (см. [Часть 4](#part4)).

<a id="part3"></a>
# Часть 3. Установка сайта с нуля

Если переносить нечего и сайт разворачивается заново, шаги 1-3 и 7-10 заменяются на этот раздел. Шаги 4-6 (подготовка сборки и запуск контейнеров) и 12-18 (настройка, cron, почта, кеш, поиск, проверка) выполняются как описано выше.

<a id="bitrixsetupphprestorephp"></a>
## BitrixSetup / Restore

Для установки продуктов 1С-Битрикс используется скрипт `bitrixsetup.php`, для восстановления из резервной копии, созданной средствами Битрикс, - `restore.php`.

Документация:
- `bitrixsetup.php`: https://dev.1c-bitrix.ru/learning/course/index.php?COURSE_ID=135&LESSON_ID=4523&LESSON_PATH=10495.4495.4523
- `restore.php`: https://dev.1c-bitrix.ru/learning/course/index.php?COURSE_ID=135&CHAPTER_ID=02014&LESSON_PATH=10495.4496.2014

Скачиваем нужный скрипт в корень сайта:

```bash
docker compose exec --user=bitrix php sh -c "cd /opt/www && wget https://www.1c-bitrix.ru/download/scripts/bitrixsetup.php"
```

```bash
docker compose exec --user=bitrix php sh -c "cd /opt/www && wget https://www.1c-bitrix.ru/download/scripts/restore.php"
```

Открываем в браузере:

```bash
https://site.example.com/bitrixsetup.php
https://site.example.com/restore.php
```

<a id="installdistro"></a>
## Установка дистрибутива

Через `bitrixsetup.php` скачиваем дистрибутив продукта (демо или лицензионную версию), затем открываем сайт:

```bash
https://site.example.com/
```

и проходим мастер установки. На шаге создания базы указываем:
- `Имя хоста (алиас)` - `mysql`
- `Имя суперпользователя` - `root`
- `Пароль суперпользователя` - значение из файла `.env_sql` (см. пункт 5.2)

Документация по мастеру установки: https://dev.1c-bitrix.ru/learning/course/index.php?COURSE_ID=135&CHAPTER_ID=04522&LESSON_PATH=10495.4495.4522

> [!NOTE]
> Мастер создаст базу от имени `root`. Для боевой эксплуатации после установки создайте отдельного пользователя базы с правами только на базу сайта (команды - в пункте 8.1) и укажите его в `/bitrix/.settings.php` и `/bitrix/php_interface/dbconn.php`.

<a id="restorebackup"></a>
## Восстановление из резервной копии Битрикс

Через `restore.php` восстанавливаем сайт из резервной копии, созданной модулем `Резервное копирование`:
- копия на стороннем сайте - вариант `Скачать резервную копию с другого сайта`, указываем путь к архиву;
- копия в облаке 1С-Битрикс - вариант `Развернуть резервную копию из облака "1С-Битрикс"`, указываем активный лицензионный ключ;
- копия на вашем компьютере - вариант `Загрузить с локального диска`;
- копия уже лежит в корне сайта - появится пункт `Архив загружен в корневую папку сервера`.

Загрузить архив в корень сайта можно так:

```bash
docker compose cp .\backup.tar.gz php:/opt/www/backup.tar.gz
docker compose exec --user=root php sh -c "chown bitrix:bitrix /opt/www/backup.tar.gz"
```

После распаковки файлов скрипт запросит параметры подключения к базе: хост - `mysql`, пользователь и база - те, что созданы в пункте 8.1 (или `root` с паролем из `.env_sql`).

Нажимаем `Восстановить` и дожидаемся завершения. После успешного восстановления нажимаем `Удалить локальную копию и служебные скрипты`.

Далее возвращаемся к [Шагу 10](#step10) - очистка настроек старого окружения нужна и в этом случае.

<a id="part4"></a>
# Часть 4. Регулярное резервное копирование

Сайт переехал - теперь резервные копии нужно снимать уже с этой сборки. Рабочая схема состоит из трех независимых частей: копия средствами Битрикс, отдельный дамп базы и копия томов Docker.

> [!CAUTION]
> Внимание! Резервные копии нужно хранить вне хоста, на котором работает сайт. Копия, лежащая в каталоге `www/` рядом с сайтом, не защищает ни от отказа диска, ни от ошибочного удаления каталога.

<a id="backupbybitrix"></a>
## Бекап средствами Битрикс по cron

Модуль `Резервное копирование` создает архив файлов и дамп базы в каталоге `/bitrix/backup/` и умеет выгружать его в облако 1С-Битрикс.

Настраиваем ежедневный запуск в контейнере `cron`. Заходим в его консоль под `root`:

```bash
docker compose exec --user=root cron sh
```

Ставим редактор и создаем задание `/etc/periodic/daily/backup` (без расширения):

```bash
apk add mc
mcedit /etc/periodic/daily/backup
```

Содержимое файла:

```bash
#!/bin/sh
#
su - bitrix -c 'php -f /opt/www/bitrix/modules/main/tools/backup.php; > /dev/null 2>&1'
#
```

Делаем задание исполняемым и просим `crond` перечитать задания:

```bash
chmod -R a+x /etc/periodic/daily
touch /etc/crontabs/cron.update
exit
```

Итог: раз в сутки (в 2 часа ночи) выполняется резервное копирование. Детали видны в логе контейнера:

```bash
docker compose logs --tail=50 cron
```

Пример вывода:

```
0.19 sec	Backup started to file: /opt/www/bitrix/backup/20250423_113000_full_vnw0wuf76rq6e5je.tar
0.19 sec	Dumping database
14.18 sec	Archiving database dump
14.2 sec	Archiving files
167.01 sec	Finished.
Data size: 2569.42 M
Archive size: 2569.42 M
Time: 166.76 sec
```

Параметры копирования (что включать, куда выгружать, сколько копий хранить) задаются на странице `Резервное копирование` (`/bitrix/admin/dump.php?lang=ru`).

Документация: https://dev.1c-bitrix.ru/learning/course/index.php?COURSE_ID=35&LESSON_ID=4464&LESSON_PATH=3906.4833.4464

> [!IMPORTANT]
> Задание, созданное внутри контейнера, живет только до его пересоздания. После `docker compose down`/`up` или смены версии PHP задание нужно создать заново. Если такое поведение неудобно - используйте способ с дампом с хоста (ниже), он не зависит от жизненного цикла контейнеров.

<a id="backupmysqldump"></a>
## Дамп базы данных с хоста

Независимый от Битрикс способ. Команда выполняется внутри контейнера `mysql`, дамп складывается в каталог `/var/log/mysql` (том `dev_mysql_log_data`), после чего забирается на хост:

```bash
docker compose exec mysql bash -c "mysqldump -u root -p'ПАРОЛЬ_ROOT' --single-transaction --quick --hex-blob --routines --triggers --events --default-character-set=utf8mb4 --set-gtid-purged=OFF --no-tablespaces sitemanager | gzip -1 > /var/log/mysql/db_backup.sql.gz"
```

```bash
docker compose cp mysql:/var/log/mysql/db_backup.sql.gz .\db_backup.sql.gz
docker compose exec mysql bash -c "rm -f /var/log/mysql/db_backup.sql.gz"
```

Проверяем полученный дамп так же, как в пункте 2.4 - последняя строка должна содержать `Dump completed`.

Для автоматического запуска оформляем эти команды скриптом и ставим в планировщик хоста: `cron` на Linux, `Планировщик заданий` на Windows.

<a id="backupvolumes"></a>
## Резервная копия файлов сайта и томов Docker

**Файлы сайта** лежат в каталоге `www/` и архивируются обычным `tar` на хосте, без остановки контейнеров:

- Linux/macOS:
```bash
tar -czf ./www_$(date +%Y%m%d).tar.gz \
  --exclude='./bitrix/cache/*' --exclude='./bitrix/managed_cache/*' \
  --exclude='./bitrix/stack_cache/*' --exclude='./upload/resize_cache/*' \
  -C ./www .
```

- Windows PowerShell:
```powershell
tar -czf ".\www_$(Get-Date -Format yyyyMMdd).tar.gz" --exclude="./bitrix/cache/*" --exclude="./bitrix/managed_cache/*" --exclude="./bitrix/stack_cache/*" --exclude="./upload/resize_cache/*" -C .\www .
```

Так же каталог `www/` можно включить в любую штатную систему бекапа хоста - это обычные файлы.

**Тома Docker** (база, `.bx_temp`, msmtp, browscap, geoip2) архивируются через временный контейнер. Останавливаем контейнеры, чтобы данные не менялись во время копирования:

```bash
docker compose stop
```

Архивируем том базы (`dev` - значение `COMPOSE_PROJECT_NAME`):

- Linux/macOS:
```bash
docker run --rm -v dev_mysql_lib_data:/data -v "$PWD":/backup alpine:3.23 tar -czf /backup/mysql_lib_data.tar.gz -C /data .
```

- Windows PowerShell:
```powershell
docker run --rm -v dev_mysql_lib_data:/data -v "${PWD}:/backup" alpine:3.23 tar -czf /backup/mysql_lib_data.tar.gz -C /data .
```

Запускаем контейнеры обратно:

```bash
docker compose up -d
```

Восстановление тома выполняется обратной командой (том должен быть пустым):

```bash
docker run --rm -v dev_mysql_lib_data:/data -v "$PWD":/backup alpine:3.23 sh -c "cd /data && tar -xzf /backup/mysql_lib_data.tar.gz"
```

> [!CAUTION]
> Внимание! Архив тома `dev_mysql_lib_data` - это копия файлов базы "как есть". Снимать его на работающем контейнере `mysql` нельзя: копия будет несогласованной. Либо останавливаем контейнеры, либо пользуемся дампом `mysqldump`.

<a id="backuprestoretest"></a>
## Проверка восстановления

Резервная копия, из которой ни разу не восстанавливались, резервной копией не считается. Раз в квартал разумно проделать учебное восстановление:

1. копируем сборку в отдельный каталог, в `.env` меняем `COMPOSE_PROJECT_NAME` (например, на `test`) и `SITE_DOMAIN` (например, на `test.example.com`) - иначе роутеры Traefik и имена контейнеров конфликтуют с боевыми;
2. поднимаем контейнеры: `docker compose up -d`;
3. разворачиваем последние копии по шагам 7-11 этой инструкции;
4. проверяем, что сайт открывается и данные на месте;
5. удаляем тестовое окружение: `docker compose down -v` и удаляем каталог копии.

<a id="part5"></a>
# Часть 5. Справочник по окружению

<a id="management"></a>
## Управление контейнерами

Все команды выполняются из каталога проекта.

Запустить все контейнеры в фоне:
```bash
docker compose up -d
```

Список контейнеров и их статус:
```bash
docker compose ps
```

Логи всех контейнеров / одного сервиса:
```bash
docker compose logs
docker compose logs --tail=100 -f nginx
```

Перезапустить один контейнер или все:
```bash
docker compose restart nginx
docker compose restart
```

Остановить все контейнеры (данные сохраняются):
```bash
docker compose stop
```

Остановить и удалить контейнеры (тома сохраняются):
```bash
docker compose down
```

Остановить, удалить контейнеры **и удалить все тома** (сайт и база будут потеряны):
```bash
docker compose down -v
```

Зайти в sh-консоль контейнера:
```bash
docker compose exec nginx sh
```

Использование отдельного compose-файла:
```bash
docker compose -f docker-compose-test.yml ps
```

Документация: https://docs.docker.com/reference/cli/docker/compose/

<a id="traefik"></a>
## Публикация через Traefik

Схема: клиент -> Traefik (порты `80`/`443` хоста, терминирует TLS) -> сеть `proxy` -> контейнер `dev_nginx`, порт `80` -> `php-fpm`.

Сервис `nginx` подключен к двум сетям: внутренней `dev` (там же база, php, redis и т.д.) и внешней сети Traefik (`proxy`). Наружу порт не публикуется, поэтому сайт доступен только через прокси.

### Что нужно со стороны Traefik

- Traefik запущен и подключен к той же внешней сети, что указана в `TRAEFIK_NETWORK`;
- у Traefik настроены entrypoints `web` (`:80`) и `websecure` (`:443`);
- настроен certresolver с именем из `TRAEFIK_CERTRESOLVER` (по умолчанию `le`);
- включен docker provider (`--providers.docker`), Traefik читает labels контейнеров.

Если сети еще нет:

```bash
docker network create proxy
```

### Параметры сборки

В файле `.env`:

```bash
SITE_DOMAIN=site.example.com
TRAEFIK_NETWORK=proxy
TRAEFIK_CERTRESOLVER=le
```

Labels сервиса `nginx` в `docker-compose.yml` (имена роутеров формируются из `COMPOSE_PROJECT_NAME`, поэтому несколько сайтов на одном хосте не конфликтуют):

```yaml
      - "traefik.enable=true"
      - "traefik.docker.network=${TRAEFIK_NETWORK:-proxy}"
      - "traefik.http.routers.${COMPOSE_PROJECT_NAME:-dev}.rule=Host(`${SITE_DOMAIN}`)"
      - "traefik.http.routers.${COMPOSE_PROJECT_NAME:-dev}.entrypoints=websecure"
      - "traefik.http.routers.${COMPOSE_PROJECT_NAME:-dev}.tls.certresolver=${TRAEFIK_CERTRESOLVER:-le}"
      - "traefik.http.services.${COMPOSE_PROJECT_NAME:-dev}.loadbalancer.server.port=80"
      - "traefik.http.routers.${COMPOSE_PROJECT_NAME:-dev}-http.rule=Host(`${SITE_DOMAIN}`)"
      - "traefik.http.routers.${COMPOSE_PROJECT_NAME:-dev}-http.entrypoints=web"
      - "traefik.http.routers.${COMPOSE_PROJECT_NAME:-dev}-http.middlewares=${COMPOSE_PROJECT_NAME:-dev}-https"
      - "traefik.http.middlewares.${COMPOSE_PROJECT_NAME:-dev}-https.redirectscheme.scheme=https"
      - "traefik.http.middlewares.${COMPOSE_PROJECT_NAME:-dev}-https.redirectscheme.permanent=true"
```

Проверить, во что развернулись переменные, можно без запуска контейнеров:

```bash
docker compose config
```

### Что настроено в nginx для работы за прокси

Прокси терминирует TLS, поэтому до сайта запрос доходит по `http`. Чтобы Битрикс не строил ссылки и редиректы на `http` (иначе получаем цикл редиректов) и видел реальный IP клиента, в сборку добавлены:

```bash
confs/nginx/maps/proxy_scheme.conf  - карты $bx_https, $bx_scheme, $bx_server_port из заголовков X-Forwarded-*
confs/nginx/inc/proxy_params.conf   - fastcgi_param HTTPS / REQUEST_SCHEME / SERVER_PORT, подключается после fastcgi_params
confs/nginx/inc/real_ip.conf        - set_real_ip_from для частных сетей + real_ip_header X-Forwarded-For
```

Файл `proxy_params.conf` подключен во всех трех php-локациях (`~ \.php$`, `@php`, `@bitrix`) файла `confs/nginx/conf.d/default.conf` **после** `include fastcgi_params;` - порядок важен, значения переопределяются последним включением.

Проверить, что сайт видит https, можно так:

```bash
docker compose exec nginx sh -c "wget -q -O /dev/null -H 'X-Forwarded-Proto: https' http://127.0.0.1/ && echo request OK"
```

и в самом продукте - в `Проверке системы` не должно быть замечаний по протоколу, а ссылки в письмах и редиректы должны быть на `https://`.

### Порты сервисов

Внутри контейнера `nginx` слушает:
- `80` - сайт, на него Traefik проксирует запросы;
- `8893` - служебный HTTP-сервер (заглушка).

Порты остальных сервисов (`3306` mysql, `6379` redis, `11211` memcached, `9000` php-fpm, `9306`/`9312` sphinx) доступны только внутри сети `dev`. В `docker-compose.yml` они закомментированы - это осознанное решение по безопасности, открывайте их только при реальной необходимости (например, для подключения внешним SQL-клиентом на время отладки).

Для работы без Traefik раскомментируйте у сервиса `nginx` блок `ports` (`"8588:80"`) - сайт станет доступен по `http://IP_хоста:8588/`.

<a id="worksitefolder"></a>
## Работа с файлами сайта из папки

Корень сайта - каталог `www/` репозитория, подключенный как `/opt/www` в контейнеры `php`, `cron` и `nginx`:

```yaml
    volumes:
      - ./www/:/opt/www/
```

Это обычный bind mount: правки файлов с хоста видны в контейнерах сразу, перезапуск не нужен. Можно открыть `www/` в IDE, вести git-репозиторий сайта, запускать линтеры и сборщики фронтенда прямо на хосте.

### Права доступа

Внутри контейнеров процессы работают от пользователя `bitrix` с `uid:gid = 979:979` (задано в `sources/bxphp*/Dockerfile` и `sources/bxnginx*/Dockerfile`).

- **Linux-хост**: владелец файлов на хосте и в контейнере - один и тот же числовой uid. Чтобы сайт мог писать в свои каталоги:
```bash
sudo chown -R 979:979 ./www
```
Чтобы при этом файлы оставались доступными вашему пользователю на запись, добавьте себя в группу `979` или используйте `setfacl`.

- **Windows + Docker Desktop**: права на файлах Windows-диска не отображаются в Linux-контейнер напрямую, все файлы видны как доступные - `chown` не требуется.

- **macOS + Docker Desktop**: работает через VirtioFS/gRPC-FUSE с автоматическим маппингом владельца, `chown` обычно не требуется.

Проверка, что сайт может писать в каталог:

```bash
docker compose exec --user=bitrix php sh -c "touch /opt/www/.write_test && rm /opt/www/.write_test && echo write OK"
```

### Производительность на Windows

Bind mount в каталог на диске `C:` работает заметно медленнее, чем том Docker: Битрикс делает тысячи обращений к файлам на хит. Если сайт открывается медленно:

- перенесите проект в файловую систему WSL2 (например, `\\wsl$\Ubuntu\home\user\env-docker`) и работайте с ним оттуда - разница обычно кратная;
- либо оставьте `www/` bind mount только для разработки, а для нагрузочных проверок временно верните том.

### Возврат к тому Docker

Если понадобится вернуть прежнюю схему, в `docker-compose.yml` возвращаем том в трех сервисах (`php`, `cron`, `nginx`):

```yaml
      - www_data:/opt/www/
```

и объявляем его в разделе `volumes`:

```yaml
  www_data:
    driver: local
```

### Перенос данных из старого тома в папку

Если сайт уже работал на томе `dev_www_data`, содержимое переносится так:

```bash
docker compose stop
```

- Linux/macOS:
```bash
docker run --rm -v dev_www_data:/from -v "$PWD/www":/to alpine:3.23 sh -c "cd /from && cp -a . /to"
```

- Windows PowerShell:
```powershell
docker run --rm -v dev_www_data:/from -v "${PWD}/www:/to" alpine:3.23 sh -c "cd /from && cp -a . /to"
```

```bash
docker compose up -d
```

Проверяем, что сайт открывается, и только после этого удаляем старый том:

```bash
docker volume rm dev_www_data
```

<a id="servicesconsole"></a>
## Консоль сервисов

Общий шаблон входа в консоль контейнера:

```bash
docker compose exec [--user=[пользователь]] [сервис] [оболочка]
```

Где:
- `пользователь` - `bitrix` или `root`; если не указывать, берется поведение по умолчанию;
- `сервис` - имя сервиса из `docker-compose.yml`: `nginx`, `php`, `cron`, `mysql`, `redis`, `memcached`, `sphinx`;
- `оболочка` - `sh`, `bash`, `ash`.

Примеры:

```bash
docker compose exec mysql bash
docker compose exec --user=bitrix php sh
docker compose exec --user=root php sh
docker compose exec --user=root redis sh -c "id"
```

**MySQL** - консоль базы из-под `root`:

```bash
docker compose exec mysql bash -c "mysql -u root -p"
```

Пароль - из файла `.env_sql`. Выход - `exit`.

**Memcache**:

```bash
docker compose exec --user=root memcached sh -c "nc 127.0.0.1 11211"
```

Выполняем запросы (например, `stats`). Выход - `exit`.

**Redis**:

```bash
docker compose exec --user=root redis sh -c "redis-cli -h 127.0.0.1 -p 6379"
```

Выполняем запросы (например, `ping` или `KEYS *`). Выход - `exit`.

**Sphinx** (индекс доступен по протоколу MySQL):

```bash
docker compose exec --user=root sphinx sh -c "mysql -h 127.0.0.1 -P 9306 -e 'SHOW TABLES;'"
```

<a id="php"></a>
## PHP

### Composer

В контейнере `php` доступен [PHP Composer](https://getcomposer.org/).

Предварительно в файл `/opt/www/bitrix/.settings.php` добавляем блок:

```php
        'composer' => [
                'value' => [
                        'config_path' => '/opt/www/bitrix/composer-bx.json'
                ]
        ],
```

Заходим в консоль контейнера и работаем:

```bash
docker compose exec --user=bitrix php sh
composer --version
cd /opt/www/bitrix/
COMPOSER=composer-bx.json composer install
php bitrix.php -h
```

Документация: https://dev.1c-bitrix.ru/learning/course/index.php?COURSE_ID=43&LESSON_ID=11685&LESSON_PATH=3913.3516.4776.2483.11685

### Browser Capabilities

Нужна для `Истории входов` (https://helpdesk.bitrix24.ru/open/16615982/).

Под `root` подключаем ini-файл:

```bash
docker compose exec --user=root php sh -c "apk add curl && echo 'browscap = /opt/browscap/php_browscap.ini' > /usr/local/etc/php/conf.d/browscap.ini"
```

Под `bitrix` загружаем базу в том `dev_browscap_data`:

```bash
docker compose exec --user=bitrix php sh -c "cd /opt/browscap && curl http://browscap.org/stream?q=PHP_BrowsCapINI -o php_browscap.ini"
```

Перезапускаем контейнеры:

```bash
docker compose restart php cron
```

В настройках `Главного модуля (main)` на вкладке `Журнал событий` включаем `Сохранять историю входов с устройств пользователя` и задаем срок хранения.

### GeoIP2

Нужна для геопозиции в `Истории входов`.

Размещаем файлы `GeoLite2-ASN.mmdb`, `GeoLite2-City.mmdb`, `GeoLite2-Country.mmdb` в каталоге `/opt/geoip2` (том `dev_geoip2_data`):

```bash
docker compose cp .\GeoLite2-City.mmdb php:/opt/geoip2/GeoLite2-City.mmdb
docker compose exec --user=root php sh -c "chown -R bitrix:bitrix /opt/geoip2"
```

На странице `Список обработчиков геолокации` (`/bitrix/admin/geoip_handlers_list.php?lang=ru`) редактируем обработчик `GeoIP2`, на вкладке `Дополнительные` указываем тип базы и абсолютный путь к файлу, например `/opt/geoip2/GeoLite2-City.mmdb`.

В настройках `Главного модуля (main)` на вкладке `Журнал событий` включаем `Собирать IP-геоданные для истории входов`.

### Расширения (extensions)

Образ `bitrix24/php` содержит расширения:

```bash
amqp apcu bz2 calendar exif gd gettext igbinary imagick intl ldap mcrypt
memcache memcached msgpack mysqli opcache pdo_mysql pdo_pgsql pgsql pspell
redis shmop sockets sodium ssh2 sysvmsg sysvsem sysvshm xdebug xhprof
xml xmlwriter xsl zip
```

Большинство активны по умолчанию. Для включения или выключения правим файл `/usr/local/etc/php/conf.d/docker-php-ext-***.ini`, где `***` - название расширения:

```bash
docker compose exec --user=root php sh -c "apk add mc"
docker compose exec --user=root php sh -c "mcedit /usr/local/etc/php/conf.d/docker-php-ext-imagick.ini"
```

Включено:
```bash
extension=imagick.so
```

Выключено:
```bash
; extension=imagick.so
```

После правки перезапускаем контейнеры:

```bash
docker compose restart php cron
```

Проверить список активных расширений:

```bash
docker compose exec --user=bitrix php sh -c "php -m"
```

### Imagick Engine для изображений

Чтобы работать с изображениями через `Imagick`, а не `GD`, убеждаемся, что расширение `imagick` включено, и копируем изображение-заглушку:

```bash
docker compose exec --user=bitrix php sh -c "\cp /usr/local/etc/image.png /opt/www/bitrix/images/"
```

В файл `/opt/www/bitrix/.settings.php` добавляем блок:

```php
        'services' => [
                'value' => [
                        'main.imageEngine' => [
                                'className' => '\Bitrix\Main\File\Image\Imagick',
                                'constructorParams' => [
                                        null,
                                        [
                                                'allowAnimatedImages' => true,
                                                'maxSize' => [
                                                        7000,
                                                        7000
                                                ],
                                                'jpegLoadSize' => [
                                                        2000,
                                                        2000
                                                ],
                                                'substImage' => $_SERVER['DOCUMENT_ROOT'].'/bitrix/images/image.png',
                                        ]
                                ],
                        ],
                ],
                'readonly' => true,
        ],
```

Для отключения удаляем этот блок.

### Веб-антивирус

Веб-антивирус входит в модуль `Проактивная защита (security)`. Для детектирования вирусов, внедренных до старта буферизации вывода, добавляем ini-файл:

```bash
docker compose exec --user=root php sh -c "echo 'auto_prepend_file = /opt/www/bitrix/modules/security/tools/start.php' > /usr/local/etc/php/conf.d/security.ini"
docker compose restart php cron
```

Включаем его на странице `Веб-антивирус` (`/bitrix/admin/security_antivirus.php?lang=ru`).

Для отключения - обратный порядок: выключаем на странице модуля, затем

```bash
docker compose exec --user=root php sh -c "rm -f /usr/local/etc/php/conf.d/security.ini"
docker compose restart php cron
```

Документация: https://dev.1c-bitrix.ru/user_help/settings/security/security_antivirus.php

### Роутинг

Новая система роутинга требует, чтобы обработка 404 шла на файл `/bitrix/routing_index.php`.

До первого запуска правим файл `confs/nginx/conf.d/default.conf` на хосте, у запущенного сайта - `/etc/nginx/conf.d/default.conf` внутри контейнера `nginx`.

Включение роутинга - комментируем строки с `urlrewrite.php`, раскомментируем строки с `routing_index.php`:

```bash
#fastcgi_param SCRIPT_FILENAME $document_root/bitrix/urlrewrite.php;
fastcgi_param SCRIPT_FILENAME $document_root/bitrix/routing_index.php;
```

Отключение - обратные действия:

```bash
fastcgi_param SCRIPT_FILENAME $document_root/bitrix/urlrewrite.php;
#fastcgi_param SCRIPT_FILENAME $document_root/bitrix/routing_index.php;
```

Проверяем конфигурацию и перезапускаем веб-сервер:

```bash
docker compose exec --user=bitrix nginx sh -c "nginx -t"
docker compose restart nginx
```

Документация: https://dev.1c-bitrix.ru/learning/course/index.php?COURSE_ID=43&CHAPTER_ID=013764&LESSON_PATH=3913.3516.5062.13764

<a id="nginx"></a>
## Nginx

Конфигурация подключается с хоста из каталога `confs/nginx/`:

```bash
confs/nginx/nginx.conf          - основной файл
confs/nginx/conf.d/default.conf - серверы сайта (HTTP :80 и служебный :8893)
confs/nginx/inc/                - подключаемые блоки: static, block, bx_temp, proxy, dav, intranet, clouds, errors, modules, composite
confs/nginx/maps/               - карты (resize_cache)
confs/nginx/errors/             - страницы ошибок 403/404/500/502/503/504
confs/nginx/blank/              - заглушка для служебного сервера
```

После правки любого файла проверяем конфигурацию и перезапускаем контейнер:

```bash
docker compose exec --user=bitrix nginx sh -c "nginx -t"
docker compose restart nginx
```

### Модули для Nginx

Модули подключаются в файле `confs/nginx/inc/modules.conf`. По умолчанию в образе `bitrix24/nginx` подключены:
- `brotli`
- `headers-more`
- `zip`

Доступны также `acme`, `geoip`, `geoip2`, `image-filter`, `lua`, `ndk`, `njs`, `perl`, `xslt`.

Подключение модуля - строка вида:

```bash
load_module "/usr/lib/nginx/modules/ngx_http_perl_module.so";
```

Отключение - та же строка с `#` в начале. После правки - `nginx -t` и перезапуск контейнера.

<a id="sitecheckerandperfmon"></a>
## Проверка системы и тест производительности

`Проверка системы` (`/bitrix/admin/site_checker.php?lang=ru`) - всесторонняя проверка соответствия параметров системы техническим требованиям продукта.

Документация:
- https://dev.1c-bitrix.ru/learning/course/index.php?COURSE_ID=35&LESSON_ID=14020&LESSON_PATH=3906.4493.4506.2024.14020
- https://dev.1c-bitrix.ru/user_help/settings/utilities/site_checker.php

`Панель производительности` (`/bitrix/admin/perfmon_panel.php?lang=ru`) - тест конфигурации, показывает "узкие" места системы.

Документация: https://dev.1c-bitrix.ru/learning/course/index.php?COURSE_ID=35&CHAPTER_ID=03376&LESSON_PATH=3906.6663.4904.3376

<a id="softwareversions"></a>
## Версии ПО

Текущие версии по умолчанию:

```bash
Memcached 1.6.x
Redis 8.2.x
Percona Server 8.0.x
PHP 8.2.x
Nginx 1.30.x
Sphinx 2.2.11
```

Альтернативные версии переключаются комментированием строк в `docker-compose.yml` **до первого запуска** проекта.

### Redis

Доступны версии `7.2.x`, `7.4.x`, `8.0.x`, `8.4.x`, `8.6.x`, `8.8.x`. Пример перехода на `8.4.x`:

```yaml
#image: redis:7.2.14-alpine
#image: redis:7.4.9-alpine
#image: redis:8.0.6-alpine
#image: redis:8.2.7-alpine
image: redis:8.4.4-alpine
#image: redis:8.6.4-alpine
#image: redis:8.8.0-alpine
```

### MySQL (Percona Server)

Доступна альтернативная версия `8.4.x`:

```yaml
#image: quay.io/bitrix24/percona-server:8.0.46-v1-rhel
image: quay.io/bitrix24/percona-server:8.4.10-v1-rhel
```

> [!CAUTION]
> Внимание! Менять версию базы у уже наполненного тома `dev_mysql_lib_data` нельзя без предварительного дампа: формат файлов данных между мажорными версиями несовместим. Правильный путь - снять дамп (пункт 2.3), удалить том, поднять новую версию и импортировать дамп заново.

### PHP и Cron

Доступны версии `8.3.x`, `8.4.x`, `8.5.x`. Правки нужны **в обоих** сервисах - `php` и `cron`. Пример перехода на `8.3.x`:

```yaml
#image: quay.io/bitrix24/php:8.2.32-fpm-v1-alpine
image: quay.io/bitrix24/php:8.3.32-fpm-v1-alpine
#image: quay.io/bitrix24/php:8.4.23-fpm-v1-alpine
#image: quay.io/bitrix24/php:8.5.8-fpm-v1-alpine
```

и в секции `volumes` того же сервиса:

```yaml
#- ./confs/php82/etc/:/usr/local/etc/
- ./confs/php83/etc/:/usr/local/etc/
#- ./confs/php84/etc/:/usr/local/etc/
#- ./confs/php85/etc/:/usr/local/etc/
```

После правки:

```bash
docker compose up -d
```

Не забудьте про часовой пояс в файле `confs/php8X/etc/php/conf.d/timezone.ini` выбранной версии.

<a id="customization"></a>
## Кастомизация

Свои сервисы можно запускать двумя способами:

1. отдельным файлом `docker-compose-my.yml`, тогда команды пишутся с указанием обоих файлов:

```bash
docker compose -f docker-compose.yml -f docker-compose-my.yml ps
```

2. добавлением сервиса в существующий `docker-compose.yml`.

Пример: добавляем [Valkey](https://hub.docker.com/r/valkey/valkey/).

В раздел `volumes`:

```yaml
  valkey_data:
    driver: local
```

В раздел `services`:

```yaml
  valkey:
    image: valkey/valkey:7.2.12-alpine
    container_name: dev_valkey
    restart: unless-stopped
    command: valkey-server
    env_file:
      - .env
#    ports:
#      - "6379:6379"
    volumes:
      - valkey_data:/data
    init: true
    cgroup: private
    security_opt:
      - no-new-privileges
    cap_drop:
      - ALL
    cap_add:
      - CHOWN
      - SETUID
      - SETGID
    tmpfs:
      - /tmp:noexec,nodev,nosuid
      - /var/tmp:noexec,nodev,nosuid
      - /dev/shm:noexec,nodev,nosuid
    networks:
      dev:
        aliases:
          - valkey
```

Запускаем и проверяем:

```bash
docker compose up -d
docker compose exec --user=root valkey sh -c "valkey-cli -h 127.0.0.1 -p 6379 ping"
```

<a id="dockerimages"></a>
## Сборка или скачивание Docker образов

Цель - подготовить максимально совместимые образы для продуктов 1С-Битрикс.

### Базовые образы

Там, где возможно, используются официальные образы с [DockerHub](https://hub.docker.com/):

```bash
docker pull redis:8.2.7-alpine
docker pull memcached:1.6.45-alpine
```

### Битрикс образы

> [!CAUTION]
> Внимание! Сборку образов необходимо проводить в `Docker Desktop` - в сборщике участвуют две архитектуры: `x86_64` (для `amd64`) и `aarch64` (для `arm64`).

Что собирается поверх официальных образов:
- база данных: `percona/percona-server:8.0.46` / `8.4.10` + конфигурация -> `bitrix24/percona-server:8.0.46-v1-rhel` / `8.4.10-v1-rhel`;
- веб-сервер: `nginx:1.30.4-alpine-slim` + модули -> `bitrix24/nginx:1.30.4-v1-alpine`;
- PHP: `php:8.2.32-fpm-alpine3.23` (и аналоги 8.3/8.4/8.5) + расширения и утилиты -> `bitrix24/php:8.2.32-fpm-v1-alpine` и т.д.;
- поиск: пакет `sphinx` из репозитория `Alpine Linux` -> `bitrix24/sphinx:2.2.11-v2-alpine`.

Предварительное скачивание базовых образов:

```bash
docker pull percona/percona-server:8.0.46
docker pull percona/percona-server:8.4.10
docker pull nginx:1.30.4-alpine-slim
docker pull php:8.2.32-fpm-alpine3.23
docker pull php:8.3.32-fpm-alpine3.23
docker pull php:8.4.23-fpm-alpine3.23
docker pull php:8.5.8-fpm-alpine3.23
docker pull alpine:3.23
```

Команды сборки:

- `bitrix24/sphinx`:
```bash
cd env-docker/sources/bxsphinx2211/v2/
docker buildx build --platform linux/arm64,linux/amd64 --provenance=false -f Dockerfile -t bitrix24/sphinx:2.2.11-v2-alpine --no-cache .
```

- `bitrix24/php` (пример для `8.2.x`, для других версий - каталоги `bxphp8332`, `bxphp8423`, `bxphp858`):
```bash
cd env-docker/sources/bxphp8232/
docker buildx build --platform linux/arm64,linux/amd64 --provenance=false -f Dockerfile -t bitrix24/php:8.2.32-fpm-v1-alpine --no-cache .
```

- `bitrix24/nginx`:
```bash
cd env-docker/sources/bxnginx1304/
docker buildx build --platform linux/arm64,linux/amd64 --provenance=false -f Dockerfile -t bitrix24/nginx:1.30.4-v1-alpine --no-cache .
```

- `bitrix24/percona-server`:
```bash
cd env-docker/sources/bxpercona8046/
docker buildx build --platform linux/arm64,linux/amd64 --provenance=false -f Dockerfile -t bitrix24/percona-server:8.0.46-v1-rhel --no-cache .
```

```bash
cd env-docker/sources/bxpercona8410/
docker buildx build --platform linux/arm64,linux/amd64 --provenance=false -f Dockerfile -t bitrix24/percona-server:8.4.10-v1-rhel --no-cache .
```

В тегах образов `bitrix24` часть `v1` означает общую отметку версии (`v`) и номер сборки (`1`).

### Модули для Nginx

> [!CAUTION]
> Внимание! Информация о сборке модулей приведена для ознакомления. Повторять эти шаги для развертывания сайта не требуется.

В образе `bitrix24/nginx` доступны модули: `acme`, `brotli`, `geoip`, `geoip2`, `headers-more`, `image-filter`, `lua`, `ndk`, `njs`, `perl`, `xslt`, `zip`.

Модули собираются на базе `nginx:1.30.4-alpine-slim`. Потребуется `Dockerfile` от соответствующей версии с [GitHub](https://github.com/nginx/docker-nginx), пример модификаций - в каталоге `sources/bxnginx1304modules/v1/`.

Сборка образа со сборщиком модулей:

```bash
cd env-docker/sources/bxnginx1304modules/v1/
docker buildx build --platform linux/arm64,linux/amd64 --provenance=false -f Dockerfile -t bitrix24/nginx_modules:1.30.4-v1-alpine --no-cache .
```

Затем запускаются два контейнера - по одному на архитектуру:

```bash
docker run --platform=linux/amd64 -d --name=nginxmodules1304testingamd64 -it bitrix24/nginx_modules:1.30.4-v1-alpine
docker run --platform=linux/arm64 -d --name=nginxmodules1304testingarm64 -it bitrix24/nginx_modules:1.30.4-v1-alpine
```

Собранные модули лежат в каталоге `/root/packages/` каждого контейнера, забираются архивом:

```bash
apk add mc zip
cd /root/packages/
zip -r nginxmodules_amd64.zip *
exit
```

После контейнеры останавливаются и удаляются:

```bash
docker container stop nginxmodules1304testingamd64 && docker container rm nginxmodules1304testingamd64
docker container stop nginxmodules1304testingarm64 && docker container rm nginxmodules1304testingarm64
```

Содержимое архивов размещается в репозитории [bitrix-tools/nginx-modules](https://github.com/bitrix-tools/nginx-modules) и используется при сборке образа `bitrix24/nginx` (см. `sources/bxnginx1304/Dockerfile`).

<a id="fstkos"></a>
## Особенности ОС, сертифицированных ФСТЭК

К ним относятся `Astra Linux Special Edition`, `РЕД ОС`, `Альт СП` и т.д. Обычно они содержат дополнительные средства защиты информации.

### Альт 10 СП

Установка `Docker` и `Docker Compose` из репозиториев ОС:

```bash
apt-get install docker-engine containerd docker-buildx docker-cli runc docker-compose-v2
```

Запуск сервиса и его автозагрузка:

```bash
systemctl restart docker.service && systemctl enable docker.service && systemctl status docker.service --no-pager
```

Проверка:

```bash
docker version
docker compose version
```

Команды compose пишутся через тире: `docker-compose ***`.

Дальнейшее развертывание - по шагам этой инструкции.

### Astra Linux Special Edition 1.8

Убеждаемся, что активированы главный (repository-main) и расширенный (repository-extended) репозитории, и устанавливаем:

```bash
apt install docker.io containerd runc docker-compose-v2 docker-buildx
```

Сервис запускается и добавляется в автозагрузку автоматически. Проверка:

```bash
docker version
docker compose version
```

До запуска проекта нужно поменять порт `nginx` (в ОС действуют ограничения на привилегированные порты). В файле `confs/nginx/conf.d/default.conf` меняем `listen 80` на `listen 8080`, после чего указываем новый порт Traefik в labels сервиса `nginx`:

```yaml
      - "traefik.http.services.${COMPOSE_PROJECT_NAME:-dev}.loadbalancer.server.port=8080"
```

Если сборка запускается без Traefik, вместо этого правим блок `ports`: вместо `- "8588:80"` указываем `- "8588:8080"`.

Дальнейшее развертывание - по шагам этой инструкции.

<a id="part6"></a>
# Часть 6. Диагностика типовых проблем

**Контейнер не поднимается или постоянно перезапускается.**

```bash
docker compose ps
docker compose logs --tail=100 [сервис]
```

Чаще всего причина - синтаксическая ошибка в конфигурации, подключенной с хоста (`confs/...`), или занятый порт на хосте.

**Контейнеры не стартуют: `network proxy declared as external, but could not be found`.** Внешней сети Traefik нет на хосте. Создаем ее и подключаем к ней Traefik:

```bash
docker network create proxy
```

**Traefik отдает 404 page not found.** Роутер не сматчился. Проверяем по порядку:

```bash
docker compose config | grep -A15 "labels:"        # во что развернулся SITE_DOMAIN
docker inspect dev_nginx --format "{{json .NetworkSettings.Networks}}"   # есть ли сеть proxy
docker logs --tail=50 traefik                       # что говорит сам прокси
```

Частые причины: пустой или неверный `SITE_DOMAIN` в `.env`, контейнер не в сети Traefik, у Traefik не включен docker provider, запрос пришел на другой домен.

**Цикл редиректов (`ERR_TOO_MANY_REDIRECTS`) или ссылки на http за HTTPS.** Сайт не видит, что исходный запрос был по https. Проверяем, что подключены `confs/nginx/maps/proxy_scheme.conf` и `confs/nginx/inc/proxy_params.conf` (см. [Публикация через Traefik](#traefik)), и что Traefik передает заголовок `X-Forwarded-Proto`:

```bash
docker compose exec nginx sh -c "wget -q -O /dev/null -H 'X-Forwarded-Proto: https' http://127.0.0.1/ && echo request OK"
docker compose exec --user=bitrix nginx sh -c "nginx -t"
```

Дополнительно проверяем, что в настройках главного модуля `URL сайта` указан без `http://` и без порта.

**Сертификат не выпускается.** Смотрим лог Traefik. Обычно причина внешняя: домен из `SITE_DOMAIN` не указывает на IP хоста, закрыт порт `80` (нужен для HTTP-челленджа) или сработал лимит Let's Encrypt.

**В логах и в панели сайта виден IP прокси вместо IP посетителя.** Не подключен `confs/nginx/inc/real_ip.conf` либо адрес прокси не попадает в перечисленные в нем частные сети - добавьте нужную подсеть.

**Порт занят: `bind: address already in use`.** Возникает, если раскомментирован блок `ports` у `nginx`. Меняем левую часть (`"8588:80"`) на свободный порт хоста и перезапускаем: `docker compose up -d`.

**Сайт не видит изменений в файлах / файлы не создаются.** Проверяем права на каталоге `www/` (на Linux-хосте владелец должен быть `979:979`):

```bash
docker compose exec --user=bitrix php sh -c "touch /opt/www/.write_test && rm /opt/www/.write_test && echo write OK"
sudo chown -R 979:979 ./www
```

**Сайт работает медленно на Windows.** Bind mount в каталог на диске `C:` медленный на большом числе файлов - перенесите проект в файловую систему WSL2, см. [Работа с файлами сайта из папки](#worksitefolder).

**Сайт отдает 502 Bad Gateway.** Не отвечает PHP-FPM:

```bash
docker compose ps php
docker compose logs --tail=100 php
```

Проверяем, что контейнер `php` работает и что в `confs/nginx/conf.d/default.conf` в `fastcgi_pass` указано `php:9000`.

**Сайт отдает 403 Forbidden или пустую страницу.** Проверяем, что файлы распакованы в корень тома и принадлежат `bitrix`:

```bash
docker compose exec --user=bitrix php sh -c "ls -la /opt/www | head"
docker compose exec --user=root php sh -c "chown -R bitrix:bitrix /opt/www"
```

**Ошибка подключения к базе на сайте.** Проверяем параметры в `/opt/www/bitrix/.settings.php` и `/opt/www/bitrix/php_interface/dbconn.php` (хост должен быть `mysql`) и доступность базы:

```bash
docker compose exec mysql mysql -u root -p -e "SHOW DATABASES;"
```

**Импорт дампа падает с `MySQL server has gone away` / `Packet too large`.** Увеличиваем `max_allowed_packet` в `confs/mysql/etc/my.cnf.d/my.cnf` (см. пункт 5.6), перезапускаем `mysql`, удаляем и создаем базу заново, импортируем повторно.

**"Кракозябры" вместо русских букв.** Кодировка базы не совпадает с исходной. Пересоздаем базу с нужными `CHARACTER SET`/`COLLATE` (пункт 8.1) и импортируем дамп заново с `--default-character-set=utf8mb4`.

**Сайт пытается стучаться на старый сервер (долгие ответы, ошибки соединения).** Не вычищены настройки старого окружения - возвращаемся к [Шагу 10](#step10), проверяем блоки `cache`, `session`, `pull` в `.settings.php` и `.settings_extra.php`.

**Агенты не выполняются.** Проверяем `define("BX_CRONTAB_SUPPORT", true);` в `dbconn.php`, состояние контейнера `cron` и его лог:

```bash
docker compose logs --tail=50 cron
```

**Нет места на диске.** Смотрим, что занимает место, и чистим неиспользуемые образы и остановленные контейнеры:

```bash
docker system df
docker system prune
```

> [!CAUTION]
> Внимание! Не используйте `docker system prune -a --volumes` - эта команда удаляет тома, а вместе с ними сайт и базу.

**Нужно посмотреть, сколько ресурсов потребляют контейнеры:**

```bash
docker stats
```

------------------------------------------------

[1С-Битрикс: Разработчикам](https://dev.1c-bitrix.ru)
