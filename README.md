# AdScope Lab

Минимальная локальная учебная основа: Laravel в корне проекта, PHP и MySQL в Docker. Сохранена стандартная welcome-страница. Сервер `artisan serve` предназначен для локальной разработки.

## Требования и версии

Нужны Docker с запущенным daemon и Docker Compose v2+ с поддержкой `--wait`, доступ к интернету при первой сборке и свободный порт 8000. PHP, Composer, Node.js и MySQL на хосте для этого этапа не нужны: welcome-страница содержит встроенные стили и работает без сборки Vite.

Версии проверены 14 октября 2026 года:

- PHP **8.5.11** — [текущий стабильный релиз](https://www.php.net/downloads.php). Образ `php:8.5.11-cli-trixie`; `require.php` и `config.platform.php` в `composer.json` соответствуют этому runtime.
- Laravel **13.36.0**, официальный шаблон **13.11.0** — [релизы framework](https://github.com/laravel/framework/releases), [релизы шаблона](https://github.com/laravel/laravel/releases). Laravel 13 [поддерживает PHP 8.3–8.5](https://laravel.com/docs/13.x/releases).
- MySQL **26.7.1** — [стабильное обновление Docker-образа](https://dev.mysql.com/doc/relnotes/mysql/26.7/en/news-26-7-1.html), образ `container-registry.oracle.com/mysql/community-server:26.7.1` из [официального реестра Oracle](https://dev.mysql.com/doc/refman/26.7/en/docker-mysql-getting-started.html). Laravel [поддерживает MySQL 5.7+](https://laravel.com/docs/13.x/database). MySQL 26.10 пока Early Access и не используется.
- Composer **2.10.3** — [стабильный релиз](https://getcomposer.org/download/), образ `composer:2.10.3`.
- PHPUnit **13.4.2** — [PHPUnit 13 поддерживает PHP 8.4+](https://phpunit.de/supported-versions.html).

Точные версии PHP-зависимостей закреплены в `composer.lock`: сохраняйте его в Git и используйте `composer install` для повторяемой установки.

Образ MySQL 26.7.1 Oracle опубликован для `linux/amd64`. На компьютерах ARM64 (включая Apple Silicon) Docker должен поддерживать эмуляцию x86_64; Compose явно задаёт эту платформу. PHP-образ работает с родной архитектурой.

## Первая настройка

Все команды выполняются из корня `adscope-lab`. Если `.env` уже существует, сохраните его: копирование нужно только при первом запуске.

```sh
cp -n .env.example .env
docker compose config --quiet
docker compose build --pull app
docker compose run --rm --no-deps app composer install --no-interaction --prefer-dist
docker compose run --rm --no-deps app php artisan key:generate
docker compose up -d --wait --wait-timeout 180 mysql
docker compose run --rm app php artisan migrate
docker compose up -d --wait --wait-timeout 120 app
```

Не генерируйте ключ повторно при обычном запуске. `.env` содержит локальную конфигурацию и ключ приложения, исключён из Git. `.env.example` содержит только открытые учебные значения; эти пароли предназначены исключительно для локальной разработки.

Открыть: **http://localhost:8000**. Для локального переопределения порта измените вместе `APP_PORT` и `APP_URL` в `.env`, затем повторите `docker compose up -d --wait`. HTTP-примеры ниже используют порт по умолчанию; при переопределении подставьте свой порт. `APP_PORT` — порт хоста, внутри контейнера приложение слушает 8000.

До создания приложения backend URL в проекте отсутствовали. Порт по умолчанию — 8000, `APP_URL=http://localhost:8000`. Шаблонное `DB_CONNECTION=sqlite` заменено на `mysql`, а пример `DB_HOST=127.0.0.1` — на `mysql`, чтобы обращаться к сервису базы в сети Compose.

## Обычный запуск и миграции

```sh
docker compose up -d --wait --wait-timeout 180
docker compose exec app php artisan migrate
```

Compose запускает приложение после успешного SQL-запроса healthcheck MySQL. Миграции выполняются явно, а не автоматически при каждом старте.

## Проверка

```sh
docker compose config --quiet
docker compose ps
curl --fail --silent --show-error --output /dev/null --write-out 'HTTP %{http_code}\n' http://localhost:8000/
docker compose exec app php artisan migrate:status
docker compose exec app php artisan test
docker compose exec app composer validate --strict
docker compose exec app composer check-platform-reqs
```

Ожидаются HTTP 200, статус MySQL `healthy`, выполненные стандартные миграции и успешные стандартные тесты Laravel. Тесты используют отдельную SQLite `:memory:` согласно `phpunit.xml` и не изменяют рабочую MySQL.

При начальной проверке прошли сборка образа, готовность обоих сервисов, все три стандартные миграции, HTTP 200 с содержимым welcome-страницы, два стандартных теста (два утверждения), `composer validate --strict` и `composer check-platform-reqs`. Проверено наличие требуемых расширений PHP. `composer update` создал lock-файл без сообщений о известных уязвимостях. Настройки IDE сохранены; Git-репозиторий не создавался.

## Логи и остановка

```sh
docker compose logs --tail=100 -f app mysql
# При необходимости посмотреть журнал Laravel:
docker compose exec app tail -n 100 storage/logs/laravel.log
# Обычная остановка с сохранением базы:
docker compose down
```

`docker compose down` удаляет контейнеры и сеть, но сохраняет именованный том `mysql_data`. Следующий запуск использует ту же базу. Добавление `--volumes` к `down` удаляет том и все данные MySQL — это отдельная разрушительная операция, для обычной остановки она не нужна.

## Как это устроено

- `Dockerfile` собирает PHP с MySQL-драйвером и ZIP, копирует фиксированную версию Composer и задаёт команду запуска Laravel. Исходники подключаются из текущей папки, изменения доступны контейнеру сразу.
- `compose.yaml` описывает два сервиса: `app` исполняет Laravel, `mysql` хранит данные. HTTP опубликован только на локальном интерфейсе; порт базы на хост не публикуется.
- `mysql_data` хранит MySQL отдельно от жизненного цикла контейнера; это Docker-том, а не папка исходников.
- `.env.example` — образец, `.env` — ваша локальная конфигурация. Laravel читает `DB_*`; Compose передаёт эти же параметры серверу MySQL. `DB_HOST=mysql` — имя сервиса в сети Docker, `DB_PORT=3306` — его внутренний порт.
- Сессии и кеш хранятся в файлах, `QUEUE_CONNECTION=sync`; отдельных работников и дополнительных сервисов нет.

Параметры `MYSQL_DATABASE`, `MYSQL_USER` и пароли применяются при первичной инициализации пустого тома. Изменение `.env` само по себе не меняет пользователей или пароли в уже созданной базе. `vendor/`, `node_modules/`, локальные настройки IDE и сгенерированные кеши исключены из Git.
