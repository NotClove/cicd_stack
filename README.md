# CI/CD Stack (Jenkins + SonarQube + Nexus)

Комплект инфраструктуры для локального развертывания Jenkins, SonarQube и Nexus Repository с помощью docker-compose. Решение подходит для демонстраций и быстрой проверки CI/CD пайплайнов.

## Состав

| Сервис | Версия | Порт | Особенности |
|-------|--------|------|-------------|
| Jenkins | `jenkins/jenkins:2.528.2-lts-jdk21` | `8080`, агент `50000` | Плагины из `plugins.txt`, предустановлены Maven, Gradle, Node.js, Python, Go, sonar-scanner. Включена авторизация, пользователь `admin/admin123`. Docker socket смонтирован, директория `../test` примонтирована в `/var/jenkins_home/test`. |
| SonarQube | `25.11.0.114957-community` | `9000` | Использует PostgreSQL 16, логин по умолчанию `admin/admin`. |
| Nexus Repository 3 | `3.86.0` | `8081` (при необходимости включается `8082` под Docker registry) | Установлен фиксированный пароль `admin/admin123`. Docker hosted репозиторий настраивается вручную. |

> DNS имена внутри compose: `jenkins`, `sonarqube`, `nexus`, `postgres`. Вне контейнеров используйте `localhost`.

## Требования

- Docker Engine (или Docker Desktop 4.30+) с docker compose plugin.

- Минимум 4 vCPU и 8 GB RAM.

- Для публикации образов в Nexus по HTTP необходимо добавить registry в `/etc/docker/daemon.json` (или Docker Desktop → *Settings → Docker Engine*) и перезапустить Docker:

```json
{
  "insecure-registries": [
    "localhost:8082",
    "host.docker.internal:8082"
  ]
}
```

## Развёртывание

```bash
git clone https://github.com/NotClove/cicd_stack.git && cd cicd_stack
docker compose up -d
docker compose ps
```

Первый старт занимает 2–3 минуты. Для диагностики используйте:

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
| Docker Registry | http://localhost:8082 *(если порт включён и Docker настроен как insecure)* | `admin` | `admin123` |

### Jenkins

- **Init‑скрипт** `jenkins-init.groovy` создаёт админа и включает стратегию `FullControlOnceLoggedInAuthorizationStrategy`.
- **Плагины** берутся из `plugins.txt` (BlueOcean, docker-workflow, nodejs, sonar, matrix-auth, etc.).
- **Инструменты**: Maven 3.9.9, Gradle 8.10.2, Node.js 22.11.0, Python 3.11, Go 1.23.4,  sonar-scanner 5.0.1.
- **Docker**: CLI установлен, сокет `/var/run/docker.sock` смонтирован, т.е. все сборки происходят на host docker daemon.
- __Каталог `/var/jenkins_home/test`__ — это примонтированная директория `../test` из репозитория (примерные пайплайны для Java/Python/Go).

> Для работы `waitForQualityGate` необходимо в *Manage Jenkins → Configure System → SonarQube servers* добавить сервер `http://sonarqube:9000` и указать `Secret text` токен из SonarQube. Учётные данные для Nexus также создаются через *Manage Credentials*.

### SonarQube

1. Откройте http://localhost:9000, войдите `admin/admin`, при необходимости смените пароль.
2. Создайте токен для Jenkins: *My Account → Security → Generate Tokens*.
3. Quality Gate и Quality Profile по умолчанию — Sonar way; при необходимости импортируйте свои профили.

### Nexus Repository

1. Перейдите на http://localhost:8081 (`admin/admin123`).
2. Создайте Docker hosted репозиторий (например, `docker-hosted`, HTTP port 8082) и при необходимости раскомментируйте порт в compose.
3. Добавьте в Jenkins credentials `nexus-credentials` (`Username with password`) и используйте в пайплайнах.
4. Убедитесь, что Docker daemon допускает insecure registry (см. Требования).

Проверка логина:

```bash
docker login localhost:8082 -u admin -p admin123
```

## Обновление конфигурации

- **Плагины Jenkins**: измените `plugins.txt`, затем пересоберите образ `docker compose build jenkins && docker compose up -d jenkins`.
- __Init‑скрипт__: правки в `jenkins-init.groovy` применяются только при пустом volume `jenkins_home`. Для переинициализации выполните `docker compose down -v && docker compose up -d`.
- __Версии инструментов__: управляются через ARG в `Dockerfile.jenkins` (MAVEN_VERSION, GRADLE_VERSION, NODE_VERSION, GO_VERSION).

## Troubleshooting

- **Docker push в Nexus возвращает 401** — активируйте в Nexus realm *Docker Bearer Token Realm* (Administration → Security → Realms).
- **Push по HTTP завершается ошибкой SSL** — убедитесь, что registry добавлен в `insecure-registries`.
