# Платформа Печать аттестатов — API-сервис

Backend-сервис на **Java 21 / Spring Boot 3**, реализующий предметную область курсовой
работы: учёт учащихся, образовательных программ, итоговых отметок и жизненного цикла
печати аттестатов (DRAFT → VALIDATED → GENERATED → ISSUED / CANCELLED).

## Использование API-сервиса

Этот раздел описывает, как установить, развернуть, подключиться к сервису
и какими техническими возможностями он обладает.

### Технические возможности

- **REST API** на Spring Web с документацией в формате OpenAPI 3 (Swagger UI).
- **Аутентификация и авторизация** — JWT (access + refresh токены), 5 ролей
  с разграничением доступа на уровне эндпоинтов (`@PreAuthorize`).
- **Хранение данных** — реляционная модель на JPA/Hibernate; поддерживаются
  два режима: PostgreSQL 16 (боевой) и встроенная H2 в памяти (для разработки
  и тестов, без внешних зависимостей).
- **Формирование документов** — генерация PDF-аттестатов через OpenPDF с
  проверкой полноты и корректности данных перед печатью (жизненный цикл
  DRAFT → VALIDATED → GENERATED → ISSUED / CANCELLED).
- **Аудит** — журналирование значимых действий пользователей (`/api/audit`).
- **Идемпотентная инициализация** — при первом старте автоматически создаётся
  учётная запись администратора, повторные запуски её не дублируют.
- **Контейнеризация** — готовые `Dockerfile` и `docker-compose.yml` для
  запуска сервиса вместе с базой данных одной командой.

### Требования к окружению

| Компонент | Версия |
|---|---|
| JDK | 21 |
| Maven | 3.9+ |
| PostgreSQL (для прод-режима) | 16 |
| Docker / Docker Compose (опционально) | актуальная версия |

### Установка

1. Скачайте и распакуйте архив с исходным кодом сервиса.
2. Убедитесь, что установлены JDK 21 и Maven (`java -version`, `mvn -version`).
3. Перейдите в корень проекта:
   ```bash
   cd certificate-platform
   ```
4. Соберите проект:
   ```bash
   mvn clean package -DskipTests
   ```
   В папке `target/` появится исполняемый файл `certificate-platform.jar`.

### Варианты развёртывания (deployment)

**1. Локально с H2 (без внешних зависимостей) — для разработки и демонстрации:**
```bash
mvn spring-boot:run -Dspring-boot.run.profiles=local
```

**2. Локально с PostgreSQL — приближено к боевому окружению:**
```bash
# создать БД certificate_platform в PostgreSQL, затем:
export DB_URL=jdbc:postgresql://localhost:5432/certificate_platform
export DB_USERNAME=postgres
export DB_PASSWORD=postgres
export JWT_SECRET=<секрет не короче 32 символов>

java -jar target/certificate-platform.jar
```

**3. Docker Compose — поднимает сервис и PostgreSQL одной командой:**
```bash
docker compose up --build
```
Данные PostgreSQL и файлы сгенерированных аттестатов сохраняются в
именованных Docker-томах (`pgdata`, `certificates`) и переживают перезапуск
контейнеров.

Сервис во всех вариантах поднимается на порту **8080** (переопределяется
переменной `SERVER_PORT`).

### Конфигурация через переменные окружения

| Переменная | Назначение | Значение по умолчанию |
|---|---|---|
| `DB_URL` | JDBC-строка подключения к PostgreSQL | `jdbc:postgresql://localhost:5432/certificate_platform` |
| `DB_USERNAME` / `DB_PASSWORD` | Учётные данные БД | `postgres` / `postgres` |
| `JWT_SECRET` | Ключ подписи JWT (не короче 32 символов) | тестовое значение, **обязательно смените в проде** |
| `JWT_ACCESS_TTL` | Время жизни access-токена, мин | `30` |
| `JWT_REFRESH_TTL` | Время жизни refresh-токена, мин | `10080` (7 дней) |
| `SERVER_PORT` | Порт HTTP-сервера | `8080` |
| `CERTIFICATES_DIR` | Каталог хранения сгенерированных PDF | `./storage/certificates` |

### Подключение к сервису

После старта сервис доступен по адресу `http://localhost:8080` (или на IP/
домене вашего сервера при удалённом развёртывании).

1. **Проверка доступности:**
   ```bash
   curl http://localhost:8080/swagger-ui.html
   ```
2. **Получение токена доступа:**
   ```bash
   curl -X POST http://localhost:8080/api/auth/login \
     -H "Content-Type: application/json" \
     -d '{"username":"admin","password":"admin12345"}'
   ```
   В ответе — `accessToken` и `refreshToken`.
3. **Обращение к защищённым эндпоинтам** — передавайте токен в заголовке:
   ```
   Authorization: Bearer <accessToken>
   ```
4. **Обновление токена** без повторного логина:
   ```
   POST /api/auth/refresh    { "refreshToken": "..." }
   ```
5. Для интерактивного изучения и тестирования всех эндпоинтов используйте
   Swagger UI (`/swagger-ui.html`) — там же можно авторизоваться и выполнять
   запросы прямо из браузера. Для сценарного тестирования — файл
   `requests.http` (открывается в VS Code с расширением REST Client или в
   IntelliJ HTTP Client).

⚠️ Перед выводом в промышленную эксплуатацию обязательно смените пароль
администратора по умолчанию (`admin / admin12345`) и задайте собственный
`JWT_SECRET`.

## Стек технологий

- Java 21, Spring Boot 3.3.4
- Spring Web, Spring Data JPA (Hibernate), Spring Security + JWT (jjwt)
- PostgreSQL (прод), H2 (профиль `local` для быстрого старта)
- OpenPDF — формирование PDF аттестатов
- MapStruct + Lombok
- springdoc-openapi (Swagger UI)

## Структура проекта

```
src/main/java/ru/education/certificateplatform/
  entity/       — JPA-сущности (Student, EducationProgram, Subject,
                  CurriculumSubject, FinalGrade, Certificate,
                  CertificateTemplate, User, Role, AuditLog, ValidationResult)
  repository/   — Spring Data JPA репозитории
  service/      — бизнес-логика (Auth, Student, Program, Grade,
                  Validation, Pdf, Certificate, Template, Audit)
  controller/   — REST-контроллеры
  dto/          — запросы/ответы (records)
  mapper/       — MapStruct мапперы entity <-> DTO
  security/     — JWT-провайдер, фильтр, UserDetailsService
  config/       — SecurityConfig, OpenApiConfig, DataInitializer
  exception/    — доменные исключения + GlobalExceptionHandler
```

## Сборка и запуск

Требуется Maven и JDK 21.

### Быстрый старт (без PostgreSQL, встроенная H2)

```bash
mvn spring-boot:run -Dspring-boot.run.profiles=local
```

### С PostgreSQL

1. Создайте базу данных `certificate_platform`.
2. Задайте переменные окружения (или отредактируйте `application.yml`):
   `DB_URL`, `DB_USERNAME`, `DB_PASSWORD`, `JWT_SECRET`.
3. Соберите и запустите:

```bash
mvn clean package -DskipTests
java -jar target/certificate-platform.jar
```

Сервис поднимется на `http://localhost:8080`.
Swagger UI: `http://localhost:8080/swagger-ui.html`.

При первом запуске автоматически создаётся пользователь-администратор:
`admin / admin12345` (роль `ADMIN`) — смените пароль после первого входа.

## Аутентификация

```
POST /api/auth/login       { "username": "admin", "password": "admin12345" }
POST /api/auth/refresh     { "refreshToken": "..." }
POST /api/auth/register    (только ADMIN) { "username", "password", "role" }
```

Полученный `accessToken` передаётся в заголовке `Authorization: Bearer <token>`.

## Роли

- `ADMIN` — полный доступ, управление пользователями, программами, шаблонами
- `OPERATOR` — ведение учащихся и итоговых отметок, создание аттестатов
- `VALIDATOR` — проверка данных перед печатью (`/validate`)
- `PRINTER` — формирование PDF и выдача аттестатов
- `MANAGER` — выдача/отмена аттестатов, просмотр журнала аудита

## Основные эндпоинты

```
/api/students            CRUD учащихся
/api/programs             CRUD образовательных программ и учебных планов
/api/grades/import        импорт итоговых отметок
/api/grades/student/{id}  отметки учащегося
/api/certificates          создание/список аттестатов
/api/certificates/{id}/validate    проверка данных перед печатью
/api/certificates/{id}/generate    формирование PDF
/api/certificates/{id}/issue       выдача аттестата
/api/certificates/{id}/cancel      отмена
/api/certificates/{id}/pdf         скачивание PDF
/api/templates             шаблоны аттестатов
/api/audit                 журнал аудита действий
```

## Жизненный цикл аттестата

```
DRAFT --validate--> VALIDATED --generate--> GENERATED --issue--> ISSUED
  \_________________________cancel____________________/
```

Формирование PDF (`/generate`) всегда повторно проверяет данные учащегося и
откажет (409, `VALIDATION_ERROR`), если не заполнены обязательные сведения или
отсутствуют/некорректны итоговые отметки по обязательным предметам программы.

Дополнительные эндпоинты: `/api/subjects` (каталог предметов) и `/api/users`
(управление учётными записями, только ADMIN).

## Запуск через Docker Compose

Поднимает сервис вместе с PostgreSQL:

```bash
docker compose up --build
```

Готовые примеры запросов — в файле `requests.http` (REST Client для VS Code
или IntelliJ HTTP Client): вход, создание программы, учащегося, импорт
отметок, полный цикл аттестата и скачивание PDF.

## Тесты

```bash
mvn test
```
