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
docker compose build --pull app mock-source
docker compose run --rm --no-deps app composer install --no-interaction --prefer-dist
docker compose run --rm --no-deps app php artisan key:generate
docker compose up -d --wait --wait-timeout 180 mysql
docker compose run --rm app php artisan migrate
docker compose up -d --wait --wait-timeout 120 app mock-source
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
docker compose logs --tail=100 -f app mysql mock-source
# При необходимости посмотреть журнал Laravel:
docker compose exec app tail -n 100 storage/logs/laravel.log
# Обычная остановка с сохранением базы:
docker compose down
```

`docker compose down` удаляет контейнеры и сеть, но сохраняет именованный том `mysql_data`. Следующий запуск использует ту же базу. Добавление `--volumes` к `down` удаляет том и все данные MySQL — это отдельная разрушительная операция, для обычной остановки она не нужна.

## Как это устроено

- `Dockerfile` собирает PHP с MySQL-драйвером и ZIP, копирует фиксированную версию Composer и задаёт команду запуска Laravel. Исходники подключаются из текущей папки, изменения доступны контейнеру сразу.
- `compose.yaml` описывает три сервиса: `app` исполняет Laravel, `mysql` хранит данные, `mock-source` раздаёт статические учебные страницы. HTTP опубликован только на локальном интерфейсе; порт базы на хост не публикуется.
- `mysql_data` хранит MySQL отдельно от жизненного цикла контейнера; это Docker-том, а не папка исходников.
- `.env.example` — образец, `.env` — ваша локальная конфигурация. Laravel читает `DB_*`; Compose передаёт эти же параметры серверу MySQL. `DB_HOST=mysql` — имя сервиса в сети Docker, `DB_PORT=3306` — его внутренний порт.
- Сессии и кеш хранятся в файлах, `QUEUE_CONNECTION=sync`; отдельных работников и дополнительных сервисов нет.

Параметры `MYSQL_DATABASE`, `MYSQL_USER` и пароли применяются при первичной инициализации пустого тома. Изменение `.env` само по себе не меняет пользователей или пароли в уже созданной базе. `vendor/`, `node_modules/`, локальные настройки IDE и сгенерированные кеши исключены из Git.

## Локальный mock-источник (Task 02)

`mock-source/public/ads.html` содержит ровно 20 вымышленных native-объявлений с постоянными ID и данными. Источник предназначен для будущего упражнения со скрапером; скрапер ещё не реализован. Данные охватывают страны `PL`, `DE`, `FR`, `GB`, `US` и категории `education`, `home-living`, `travel`, `wellness`. Три SVG-иллюстрации и три вымышленных лендинга хранятся рядом с HTML. Страницы используют только локальные ресурсы, без JavaScript, внешних запросов и динамической генерации.

- Адрес в браузере: [http://localhost:8001/ads.html](http://localhost:8001/ads.html).
- Адрес из контейнера `app`: [http://mock-source:8000/ads.html](http://mock-source:8000/ads.html).

Запуск только источника из корня проекта после создания `.env` по инструкции выше:

```sh
docker compose config --quiet
docker compose build mock-source
docker compose up -d --no-deps --wait --wait-timeout 120 mock-source
docker compose ps mock-source
curl --fail --silent --show-error --output /dev/null --write-out 'HTTP %{http_code}\n' http://localhost:8001/ads.html
# Из уже запущенного Laravel-контейнера:
docker compose exec app php -r 'echo file_get_contents("http://mock-source:8000/ads.html");'
docker compose logs --tail=100 -f mock-source
```

У `mock-source` явно заданы `build: .` и тот же образ `adscope-lab-app:php8.5.11`, что у приложения. Поэтому сборка работает и без ранее созданного локального образа. Сервис запускает встроенный сервер PHP на внутреннем порту 8000 и монтирует только `mock-source/public/` в каталог раздачи `/var/www/html` с доступом только для чтения. Сервис не зависит от Laravel, `vendor/` или MySQL.

Если порт 8001 занят, сохраните чужой сервис и задайте свободный порт в локальном `.env`, например `MOCK_SOURCE_PORT=8002`. Затем повторите `docker compose up -d --no-deps --wait mock-source` и используйте выбранный порт в браузере и host-проверке. Для этого изменения не нужно менять `APP_PORT` или `APP_URL`: Laravel остаётся на своём порту 8000. Внутренний адрес источника всегда `http://mock-source:8000/ads.html`.

Порт хоста 8001 перенаправляется в порт 8000 контейнера источника. Каждый контейнер имеет собственное сетевое окружение, поэтому `app` и `mock-source` могут оба слушать 8000 внутри себя. Compose позволяет контейнерам обращаться друг к другу по имени сервиса `mock-source`; `localhost` внутри `app` обозначает сам `app`, а не компьютер и не источник.

Поля карточки для будущего парсинга:

| Селектор или атрибут | Значение |
| --- | --- |
| `article.ad-card` | Одна карточка объявления |
| `data-external-id` | Уникальный постоянный ID, например `mock-ad-001` |
| `data-country` | Двухбуквенный код страны |
| `data-category` | Slug категории |
| `data-format` | Всегда `native` |
| `.ad-title` | Заголовок |
| `.ad-description` | Короткое описание |
| `.ad-image` → `src` | URL SVG-иллюстрации |
| `.ad-landing-link` → `href` | URL лендинга |

URL в HTML заданы относительно корня источника (`/images/...`, `/landing/...`). При будущем парсинге их нужно разрешать относительно URL страницы источника: так одни и те же файлы работают и в браузере, и по внутреннему адресу Compose.
