# SonarQube для Coolify

Готовый, настраиваемый сервис для деплоя [SonarQube Community Build](https://www.sonarsource.com/products/sonarqube/) через [Coolify](https://coolify.io): SonarQube + PostgreSQL 17. Домен с HTTPS привязывается стандартным механизмом Coolify, пароли БД генерируются автоматически, healthcheck'и и персистентные тома из коробки.

## Быстрый старт

Заранее нужна A-запись в DNS: `sonar.ваш-домен.ру → IP сервера` (или одна wildcard-запись `*.ваш-домен.ру → IP` — тогда для новых сервисов DNS вообще не придётся трогать).

1. В Coolify: **Project → + New → Resource → Docker Compose** (Public Repository), укажите `https://github.com/mas-orchestartor/sonar`, ветка `main` → **Load Compose File**.
2. В секции **Domains** в поле сервиса `sonarqube` впишите `https://sonar.ваш-домен.ру`. Поле `postgresql` оставьте пустым — базе домен не нужен. Это делается один раз: домен сохраняется в Coolify и переживает все последующие деплои.
3. **Deploy**. HTTPS-сертификат Let's Encrypt, редирект с HTTP, пароли Postgres — автоматически. Первый запуск занимает 1–3 минуты.

Дальше всё само: redeploy и автодеплой по git push поднимают сервис на том же домене, ничего вписывать повторно не нужно.

### Первый вход

Откройте `https://sonar.ваш-домен.ру` и войдите: логин `admin`, пароль `admin`. SonarQube сразу попросит сменить пароль.

## Настройка

Все параметры — обычные переменные окружения, в Coolify они редактируются на вкладке **Environment Variables** ресурса:

| Переменная | По умолчанию | Что делает |
| --- | --- | --- |
| `SONARQUBE_VERSION` | `community` | Тег образа `sonarqube`. Можно зафиксировать версию (например `25.7.0.110598-community`) или использовать `developer` / `enterprise` при наличии лицензии |
| `POSTGRES_VERSION` | `17-alpine` | Тег образа `postgres` |
| `POSTGRES_DB` | `sonar` | Имя базы данных |
| `SONAR_ES_BOOTSTRAP_CHECKS_DISABLE` | `true` | `true` — работает без настройки хоста; `false` — строгий режим Elasticsearch (см. ниже) |
| `SONAR_WEB_JAVAOPTS` | `-Xmx512m -Xms128m` | Память JVM веб-сервера |
| `SONAR_CE_JAVAOPTS` | `-Xmx512m -Xms128m` | Память JVM Compute Engine (фоновый анализ) |
| `SONAR_SEARCH_JAVAOPTS` | `-Xmx512m -Xms512m` | Память JVM Elasticsearch |
| `SONAR_TELEMETRY_ENABLE` | `false` | Телеметрия SonarSource |

Переменные `SERVICE_URL_SONARQUBE` (из поля Domains), `SERVICE_USER_POSTGRES` и `SERVICE_PASSWORD_POSTGRES` Coolify генерирует автоматически — руками их задавать не нужно.

Любые другие настройки SonarQube тоже можно передать через env: свойство `sonar.foo.bar` превращается в переменную `SONAR_FOO_BAR` — просто добавьте её в Environment Variables в Coolify.

## Требования к серверу

- **RAM**: минимум 3 ГБ свободных (≈2 ГБ SonarQube + ≈0.5 ГБ PostgreSQL), рекомендуется 4 ГБ+. Если памяти больше — увеличьте `SONAR_*_JAVAOPTS`.
- **CPU**: 2+ vCPU.
- **Диск**: SSD, от 10 ГБ под тома (`sonarqube-data` растёт с числом проектов).

### vm.max_map_count (рекомендуется для продакшена)

По умолчанию сервис стартует где угодно за счёт `SONAR_ES_BOOTSTRAP_CHECKS_DISABLE=true`. Для стабильной работы Elasticsearch под нагрузкой настройте хост, на котором крутится Coolify:

```bash
sudo sysctl -w vm.max_map_count=524288
sudo sysctl -w fs.file-max=131072
# сохранить после перезагрузки:
echo -e "vm.max_map_count=524288\nfs.file-max=131072" | sudo tee /etc/sysctl.d/99-sonarqube.conf
```

После этого можно выставить `SONAR_ES_BOOTSTRAP_CHECKS_DISABLE=false`.

## Данные и обновления

Все данные лежат в именованных томах: `sonarqube-data` (индексы, БД H2-кэши), `sonarqube-extensions` (плагины), `sonarqube-logs`, `postgresql-data`. Redeploy и обновление образа их не трогают.

Обновление SonarQube: поменяйте `SONARQUBE_VERSION` (или оставьте `community` и нажмите Redeploy — подтянется свежий Community Build). Миграции БД SonarQube выполняет сам при старте; при мажорных обновлениях сначала откройте `/setup` на сервере, как просит SonarQube. Перед обновлением сделайте бэкап тома `postgresql-data`.

## Запуск локально (без Coolify)

```bash
cp .env.example .env   # отредактируйте пароль
docker compose --env-file .env up -d
# SonarQube будет на http://localhost:9000
```

Для локального доступа добавьте проброс порта в `docker-compose.yaml` (в Coolify это не нужно — трафик идёт через его прокси):

```yaml
services:
  sonarqube:
    ports:
      - "9000:9000"
```

## Как это устроено

- `SERVICE_URL_SONARQUBE_9000` — «магическая» переменная Coolify: связывает домен из поля **Domains** с портом 9000 контейнера и настраивает прокси (Traefik/Caddy) с HTTPS-сертификатом. PostgreSQL наружу не выставлен — он доступен только внутри сети ресурса.
- `SONAR_CORE_SERVERBASEURL` автоматически получает публичный URL из того же домена — корректные ссылки в письмах, вебхуках и бейджах из коробки.
- `SERVICE_USER_POSTGRES` / `SERVICE_PASSWORD_POSTGRES` — автогенерация кредов БД средствами Coolify, одни и те же значения подставляются и в PostgreSQL, и в JDBC-подключение SonarQube.
- SonarQube стартует только после того, как PostgreSQL пройдёт healthcheck; сам SonarQube считается здоровым, когда `/api/system/status` вернёт `UP`.
- Метаданные в шапке `docker-compose.yaml` (`# documentation:`, `# slogan:`, …) — формат шаблонов сервисов Coolify: файл можно предложить в официальный каталог (`templates/compose/` в репозитории coollabsio/coolify).
