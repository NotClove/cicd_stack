# CI/CD Stack (Jenkins + SonarQube + Nexus)

Готовый docker-compose стенд, который поднимает Jenkins с предустановленными плагинами и инструментами, SonarQube (c PostgreSQL) и Nexus Repository.

## Состав

| Сервис | Версия | Порт | Особенности |
|-------|--------|------|-------------|
| Jenkins | `jenkins/jenkins:2.528.2-lts-jdk21` | `8080`, агент `50000` | Плагины из `plugins.txt`, Maven+Gradle+Node+Python+Go+sonar-scanner, Wizard отключён, пользователь `admin/admin123`. Docker socket смонтирован, директория `../test` примонтирована в `/var/jenkins_home/test`. |
| SonarQube | `25.11.0.114957-community` | `9000` | Работает с PostgreSQL 16 (контейнер `postgres`). Стандартный логин `admin/admin`. |
| Nexus Repository 3 | `3.86.0` | `8081` (для Docker registry можно раскомментировать `8082`) | `NEXUS_SECURITY_RANDOMPASSWORD=false`, поэтому логин `admin/admin123`. Docker hosted репозиторий нужно создать руками. |

> DNS имена внутри compose: `jenkins`, `sonarqube`, `nexus`, `postgres`. Вне контейнеров используйте `localhost`.

## Требования

- Docker Engine + docker compose plugin (или Docker Desktop 4.30+).

- ~8 GB RAM и 4 vCPU.

- Для push Docker-образов в Nexus по HTTP добавьте в `/etc/docker/daemon.json` (или Docker Desktop → *Settings → Docker Engine*) секцию(при условии что создан docker registry(hosted) и он смотрит на отдельный порт 8082):

```json
{
  "insecure-registries": [
    "localhost:8082",
    "host.docker.internal:8082"
  ]
}
```

После правки перезапустите Docker.

## Развёртывание

```bash
git clone https://github.com/NotClove/cicd_stack.git && cd cicd_stack
docker compose up -d
# Проверяем статусы
docker compose ps
```

Первый старт займёт 2–3 мин (Nexus и SonarQube поднимаются дольше остальных). Логи:

```bash
docker compose logs -f jenkins
docker compose logs -f sonarqube
docker compose logs -f nexus
```

Остановка/удаление:

```bash
docker compose down        # остановить
docker compose down -v     # остановить и удалить volume'ы
```

## Доступ и учётки

| Сервис | URL | Логин | Пароль |
|--------|-----|-------|--------|
| Jenkins | http://localhost:8080 | `admin` | `admin123` |
| SonarQube | http://localhost:9000 | `admin` | `admin` |
| Nexus | http://localhost:8081 | `admin` | `admin123` |
| Docker Registry (Nexus) | http://localhost:8082 *(раскомментируйте порт в compose и настройте insecure registry)* | `admin` | `admin123` |

### Jenkins

- **Init‑скрипт** `jenkins-init.groovy` создаёт админа и включает стратегию `FullControlOnceLoggedInAuthorizationStrategy`.
- **Плагины** берутся из `plugins.txt` (BlueOcean, docker-workflow, nodejs, sonar, matrix-auth, etc.).
- **Инструменты**: Maven 3.9.9, Gradle 8.10.2, Node.js 22.11.0, Python 3.11, Go 1.23.4,  sonar-scanner 5.0.1.
- **Docker**: CLI установлен, сокет `/var/run/docker.sock` смонтирован, т.е. все сборки происходят на host docker daemon.
- __Каталог `/var/jenkins_home/test`__ — это примонтированная директория `../test` из репозитория (примерные пайплайны для Java/Python/Go).

> Чтобы Jenkins мог проверять Quality Gate, в *Manage Jenkins → Configure System → SonarQube servers* добавьте сервер `http://sonarqube:9000` и `Secret text` токен (создаётся в SonarQube → My Account → Tokens). Аналогично создайте credentials для Nexus (`username/password`) и Sonar (`secret text`) через *Manage Credentials*.

### SonarQube

1. Открой http://localhost:9000, залогинься `admin/admin`, задай новый пароль (если нужно).
2. Создай токен для Jenkins: *My Account → Security → Generate Tokens*. Скопируй и добавь в Jenkins (см. выше).
3. Quality Gate и Quality Profile по умолчанию — Sonar way. Можно импортировать свои профили.

### Nexus Repository

1. http://localhost:8081 → `admin/admin123`.
2. Создай Docker hosted репозиторий (например, `docker-hosted`, HTTP port 8082). При желании раскомментируй порт в compose (`# - "8082:8082"`).
3. В Jenkins добавь credentials `nexus-credentials` (`usernamePassword`) и используй в пайплайнах.
4. Если пушишь по HTTP, не забудь настроить `insecure-registries` (см. Требования).

Проверка логина:

```bash
docker login localhost:8082 -u admin -p admin123
```

## Обновление конфигурации

- **Плагины**: редактируй `plugins.txt` и пересобирай Jenkins образ: `docker compose build jenkins && docker compose up -d jenkins`.
- __Init‑скрипт__: любые правки в `jenkins-init.groovy` применяются только при пустом `jenkins_home` volume. Чтобы переинициализировать, удали volume: `docker compose down -v && docker compose up -d`.
- __Версии инструментов__ задаются в `Dockerfile.jenkins` через ARG (MAVEN_VERSION, GRADLE_VERSION, NODE_VERSION, GO_VERSION).

## Troubleshooting

- **Docker push в Nexus отдаёт 401** — включи в Nexus realm *Docker Bearer Token Realm* (Administration → Security → Realms) и перезапусти логин.
- **Push по HTTP ругается на SSL** — добавь registry в `insecure-registries` (см. выше).

