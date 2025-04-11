# Корпоративный стандарт API

## 1. Введение

### Цель документа
Обеспечение единого подхода к проектированию, разработке и эксплуатации API, гарантирующего качество, согласованность, безопасность и масштабируемость API-решений компании.

### Область применения
Распространяется на все API компании, включая:
- REST API (внутренние и публичные)
- GraphQL API
- gRPC API
- Webhook-интерфейсы

### Аудитория
- Разработчики
- Архитекторы
- DevOps-инженеры
- Тестировщики
- Технические PM

## 2. Общие принципы

### Единый стиль
- **REST API**: Приоритетный стиль для большинства сервисов, особенно публичных API
- **GraphQL**: Применяется для гибких запросов с разнородными данными, где важно минимизировать количество запросов
- **gRPC**: Используется для высоконагруженного внутреннего взаимодействия между сервисами
- **Гибридный подход**: Допустим при четком обосновании преимуществ

### Стабильность и обратная совместимость
- Изменения должны быть обратно совместимыми в пределах версии
- Любые несовместимые изменения требуют увеличения мажорной версии API
- Поддержка предыдущей мажорной версии минимум 12 месяцев после выпуска новой
- Уведомление о депрекации за 6 месяцев до отключения

### Безопасность
- HTTPS обязателен для всех API
- Аутентификация через OAuth 2.0 для пользовательских API
- JWT для внутренних сервисов с коротким сроком жизни токенов
- API Keys для партнерских/публичных API с ротацией ключей
- Регулярный аудит безопасности
- Токены в производственной среде хранятся в куках с флагом `httpOnly=true`
- Токены в dev-среде хранятся в куках с флагом `httpOnly=false` для обеспечения работы инструментов автоматизации

### Производительность
- Пагинация обязательна для коллекций более 100 элементов
- Кэширование по протоколу HTTP с корректными заголовками
- Rate limiting для защиты от перегрузок и злоупотреблений
- Сжатие (gzip, brotli) для ответов

### Документирование
- OpenAPI/Swagger для REST API
- GraphQL Schema с интроспекцией для GraphQL API
- Protocol Buffers для gRPC
- Автоматическая генерация документации из кода
- Примеры использования для каждого эндпоинта

### Автоматизация соблюдения стандарта
- Обязательное использование инструмента Spectral (Stoplight.io) для статического анализа и контроля API-контрактов
- Интеграция проверок в CI/CD-пайплайн с блокировкой мержа при нарушении стандарта
- Автоматическая валидация спецификаций перед деплоем
- Автоматизированные тесты на соответствие API-контракту
- Регулярные автоматические аудиты существующих API

## 3. Проектирование API

### 3.1. Формат запросов и ответов

#### HTTP-методы
- `GET`: Чтение ресурсов без изменения состояния
- `POST`: Создание новых ресурсов или выполнение операций
- `PUT`: Полная замена существующего ресурса
- `PATCH`: Частичное обновление ресурса
- `DELETE`: Удаление ресурса

#### URL-структура
- Ресурсно-ориентированный дизайн: `/v{majorVersion}/{resource}/{id}/{subresource}`
- Использование множественного числа для коллекций: `/users/{id}/orders`
- Только нижний регистр, дефисы для разделения слов: `/order-items`
- Предпочтение иерархической структуре: `/tenants/{tenantId}/users/{userId}`
- Глаголы допустимы только для специальных действий: `/users/{id}/reset-password`

#### Статус-коды
- `200 OK`: Успешный запрос
- `201 Created`: Успешное создание ресурса
- `202 Accepted`: Асинхронная обработка принята
- `204 No Content`: Успешная операция без тела ответа
- `400 Bad Request`: Ошибка в запросе клиента
- `401 Unauthorized`: Отсутствие аутентификации
- `403 Forbidden`: Недостаточные права
- `404 Not Found`: Ресурс не найден
- `409 Conflict`: Конфликт состояний
- `422 Unprocessable Entity`: Валидационные ошибки
- `429 Too Many Requests`: Превышение лимита запросов
- `500 Internal Server Error`: Внутренняя ошибка сервера
- `503 Service Unavailable`: Сервис временно недоступен

#### Формат данных
- JSON как основной формат данных
- UTF-8 кодировка обязательна
- camelCase для имен полей в JSON
- ISO 8601 для дат и времени (RFC 3339): `YYYY-MM-DDThh:mm:ss.sssZ`
- Консистентное именование полей во всех API
- Сложные типы (enum, unions) документируются явно

#### Ошибки
```json
{
  "error": {
    "code": "RESOURCE_NOT_FOUND",
    "message": "Requested resource not found",
    "details": [
      {
        "field": "userId",
        "reason": "User with ID 12345 does not exist"
      }
    ],
    "requestId": "f4f96b24-10cc-4c61-8860-c5aad90b0539"
  }
}
```
- Стандартизированный формат для всех ошибок
- Уникальные коды ошибок для автоматизированной обработки
- Человекочитаемые сообщения ошибок
- Детали ошибок по конкретным полям, где применимо
- Идентификатор запроса для трассировки

### 3.2. Взаимодействие с API

#### Аутентификация и авторизация
- OAuth 2.0 для пользовательских интерфейсов
- Поддержка потоков Authorization Code с PKCE и Client Credentials
- JWT для внутренних сервисов с коротким TTL (максимум 1 час)
- Централизованное управление токенами через Identity Provider
- API Key + Secret для партнерских интеграций
- Все токены передаются в заголовке `Authorization` или в куках с соответствующими настройками безопасности

#### Лимиты запросов
- Каждый эндпоинт должен иметь определенные лимиты запросов
- Возврат `429 Too Many Requests` при превышении лимита
- Заголовки с информацией о лимитах:
  - `X-RateLimit-Limit`: Общий лимит
  - `X-RateLimit-Remaining`: Оставшееся число запросов
  - `X-RateLimit-Reset`: Время сброса лимита (Unix timestamp)

#### Заголовки
- `Content-Type`: Определяет формат тела запроса
- `Accept`: Определяет ожидаемый формат ответа
- `Authorization`: Токены аутентификации
- `User-Agent`: Информация о клиенте
- `Idempotency-Key`: Гарантия идемпотентности операций
- `X-Request-ID`: Идентификатор запроса для трассировки
- `X-Correlation-ID`: Идентификатор для связанных запросов

#### Пагинация и сортировка
- Курсорная пагинация для больших наборов данных:
  ```
  /users?cursor=eyJpZCI6MTIzNDV9&limit=100
  ```
- Поддержка offset/limit для простых случаев:
  ```
  /users?offset=100&limit=50
  ```
- Явные параметры сортировки:
  ```
  /users?sort=lastName:asc,createdAt:desc
  ```
- Метаинформация в ответе:
  ```json
  {
    "data": [...],
    "pagination": {
      "totalCount": 1320,
      "nextCursor": "eyJpZCI6MjAwfQ==",
      "hasMore": true
    }
  }
  ```

### 3.3. Версионирование

#### Схема версионирования
- Версия в URL-пути для максимальной совместимости: `/v1/users`
- Семантическое версионирование (MAJOR.MINOR.PATCH):
  - MAJOR: Несовместимые изменения API
  - MINOR: Совместимые дополнения функциональности
  - PATCH: Исправления ошибок (не влияют на контракт API)
- В URL используется только MAJOR версия

#### Политика поддержки
- Поддержка минимум двух последних мажорных версий
- Предупреждение о депрекации через заголовок `Warning` или специальное поле в ответе
- Период параллельной работы версий: 12 месяцев от релиза новой мажорной версии
- Документированный процесс миграции между версиями

## 4. Безопасность

### Обязательное HTTPS
- Все API должны работать только по HTTPS
- TLS 1.2 минимум, рекомендуется TLS 1.3
- Регулярное обновление сертификатов (автоматизированное)
- HSTS обязателен для публичных API
- Тестирование безопасности конфигурации TLS (TLS Scanner)

### Защита от атак
- WAF для публичных API
- Защита от SQL-инъекций через параметризованные запросы
- Защита от XSS через валидацию и санитизацию входных данных
- Защита от CSRF с использованием токенов для аутентифицированных эндпоинтов
- Защита от DDoS:
  - Rate limiting
  - Распределенное развертывание за CDN
  - Автоматическое блокирование подозрительного трафика

### Валидация входных данных
- Проверка типов и форматов данных
- Проверка бизнес-ограничений
- Проактивное отклонение запросов с недопустимыми данными (код 400)
- Использование схем валидации (JSON Schema, Protobuf)
- Защита от десериализационных атак

### Роли и доступы
- RBAC (Role-Based Access Control) для управления доступом:
  - Роли на уровне системы (Admin, Operator, User)
  - Роли на уровне ресурсов (Owner, Editor, Viewer)
- Принцип наименьших привилегий
- Проверка разрешений на уровне каждого запроса
- Централизованное управление политиками доступа
- Логирование всех критичных действий

## 5. Документирование

### Обязательное использование OpenAPI/Swagger
- OpenAPI 3.0+ для REST API
- Автоматическая генерация документации из кода
- Поддержка актуальности документации на уровне CI/CD
- Интеграция документации с внутренним порталом для разработчиков

### Примеры запросов и ответов
- Примеры для каждого эндпоинта и операции
- Включение реальных сценариев использования
- Примеры обработки ошибок
- Постман-коллекции для тестирования и демонстрации
- Тестовая песочница для разработчиков

### Описание ошибок
- Полный каталог возможных ошибок
- Сценарии возникновения каждой ошибки
- Рекомендации по обработке ошибок на стороне клиента
- Связь ошибок с бизнес-процессами

### Чеклист для ревью API
- Соответствие именований стандарту
- Использование правильных HTTP-методов
- Корректные статус-коды для ответов
- Корректная обработка ошибок
- Идемпотентность для соответствующих операций
- Валидация входных данных
- Документация соответствует реализации
- Безопасность и производительность

## 6. Тестирование и мониторинг

### Обязательные тесты
- Unit-тесты для бизнес-логики
- Интеграционные тесты для API в целом
- E2E тесты для критических сценариев
- Нагрузочные тесты для определения лимитов производительности
- Автоматизированные тесты безопасности (SAST, DAST)
- Контрактное тестирование для проверки соответствия спецификации

### Логирование и трейсинг
- Распределенная трассировка запросов (Jaeger/Zipkin)
- Централизованное логирование (ELK Stack)
- Согласованный формат логов для всех сервисов
- Корреляция запросов через уникальные идентификаторы
- Структурированное логирование (JSON)
- Различные уровни детализации логов (DEBUG, INFO, WARN, ERROR)

### Метрики
- Основные метрики для каждого API:
  - Latency (p50, p90, p99)
  - Throughput (RPS)
  - Error Rate
  - Доступность (Uptime)
- Бизнес-метрики для ключевых процессов
- Метрики использования ресурсов (CPU, RAM, I/O)
- Алертинг на аномалии в метриках
- Дашборды для визуализации метрик (Grafana)

## 7. Развертывание и жизненный цикл

### CI/CD для API
- Автоматическая сборка и тестирование при каждом коммите
- Линтинг кода и спецификаций API с помощью Spectral
- Автоматическая валидация спецификаций OpenAPI
- Автоматическая публикация документации
- Голубой/зеленый деплой для нулевого простоя
- Канарейный релиз для новых версий API

### Каналы релизов
- `dev`: Нестабильная версия для разработчиков
- `staging`: Предрелизная версия для тестирования
- `prod`: Стабильная версия для пользователей
- Автоматическое продвижение изменений между окружениями
- Возможность быстрого отката в случае проблем

### Политика депрекации
- Формальное объявление депрекации через документацию и заголовки
- Уведомление пользователей минимум за 6 месяцев до отключения
- График миграции на новую версию
- Инструменты для автоматической миграции клиентского кода
- Мониторинг использования устаревших эндпоинтов

## 8. Примеры

### Пример REST API

#### Эндпоинты

```
GET    /v1/users                # Получить список пользователей
POST   /v1/users                # Создать нового пользователя
GET    /v1/users/{id}           # Получить пользователя по ID
PUT    /v1/users/{id}           # Обновить пользователя (полностью)
PATCH  /v1/users/{id}           # Обновить пользователя (частично)
DELETE /v1/users/{id}           # Удалить пользователя
GET    /v1/users/{id}/orders    # Получить заказы пользователя
POST   /v1/users/{id}/activate  # Действие: активировать пользователя
```

#### Пример запроса

```http
POST /v1/users HTTP/1.1
Host: api.company.com
Content-Type: application/json
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
X-Request-ID: 7b44d281-c6bf-45a4-ac13-16df3a3b30a1

{
  "firstName": "Иван",
  "lastName": "Иванов",
  "email": "ivan@example.com",
  "role": "user",
  "department": "engineering"
}
```

#### Пример успешного ответа

```http
HTTP/1.1 201 Created
Content-Type: application/json
Location: https://api.company.com/v1/users/12345
X-Request-ID: 7b44d281-c6bf-45a4-ac13-16df3a3b30a1

{
  "id": "12345",
  "firstName": "Иван",
  "lastName": "Иванов",
  "email": "ivan@example.com",
  "role": "user",
  "department": "engineering",
  "createdAt": "2025-04-10T15:30:45.123Z",
  "updatedAt": "2025-04-10T15:30:45.123Z"
}
```

### Пример ошибки

```http
HTTP/1.1 400 Bad Request
Content-Type: application/json
X-Request-ID: 7b44d281-c6bf-45a4-ac13-16df3a3b30a1

{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Validation failed",
    "details": [
      {
        "field": "email",
        "reason": "Invalid email format"
      },
      {
        "field": "role",
        "reason": "Must be one of: admin, user, guest"
      }
    ],
    "requestId": "7b44d281-c6bf-45a4-ac13-16df3a3b30a1"
  }
}
```

### Пример OpenAPI-спецификации

```yaml
openapi: 3.0.3
info:
  title: User Management API
  description: API для управления пользователями
  version: 1.0.0
  contact:
    name: API Support
    email: api-support@company.com
servers:
  - url: https://api.company.com/v1
    description: Production environment
  - url: https://staging-api.company.com/v1
    description: Staging environment
paths:
  /users:
    get:
      summary: Получить список пользователей
      parameters:
        - name: limit
          in: query
          schema:
            type: integer
            default: 20
        - name: cursor
          in: query
          schema:
            type: string
      responses:
        '200':
          description: Успешный запрос
          content:
            application/json:
              schema:
                type: object
                properties:
                  data:
                    type: array
                    items:
                      $ref: '#/components/schemas/User'
                  pagination:
                    $ref: '#/components/schemas/Pagination'
components:
  schemas:
    User:
      type: object
      required:
        - id
        - email
      properties:
        id:
          type: string
        firstName:
          type: string
        lastName:
          type: string
        email:
          type: string
          format: email
        role:
          type: string
          enum: [admin, user, guest]
    Pagination:
      type: object
      properties:
        totalCount:
          type: integer
        nextCursor:
          type: string
        hasMore:
          type: boolean
  securitySchemes:
    BearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
security:
  - BearerAuth: []
```

### Пример правил Spectral для валидации API

```yaml
extends: spectral:oas
rules:
  operation-tags: error
  operation-description: error
  operation-success-response: error
  no-unused-components: error
  openapi-tags: error
  paths-kebab-case: error
  typed-enum: error
  response-contains-example: warn
  info-contact: error
  oas3-schema: error
  security-defined: error
  pagination-style:
    description: API должен использовать курсорную пагинацию или offset/limit
    message: Пагинация должна использовать параметры cursor/limit или offset/limit
    given: $.paths..parameters[?(@.name == "page")]
    severity: error
    then:
      function: falsy
```

## 9. Приложения

### Глоссарий

- **API** - Application Programming Interface
- **REST** - Representational State Transfer
- **GraphQL** - Query language for APIs
- **gRPC** - High-performance RPC framework
- **OAuth** - Open Authorization protocol
- **JWT** - JSON Web Token
- **RBAC** - Role-Based Access Control
- **Webhook** - HTTP callback
- **Idempotent** - Операция, которую можно повторять без дополнительных эффектов
- **Rate Limiting** - Ограничение частоты запросов
- **CI/CD** - Continuous Integration/Continuous Deployment
- **Spectral** - Инструмент для линтинга и проверки API-контрактов

### Чеклист для Code Review API

#### Проектирование
- [ ] Соответствует ресурсной модели REST
- [ ] URL-структура согласуется со стандартом
- [ ] Правильно выбраны HTTP-методы для операций
- [ ] Правильно используются статус-коды HTTP

#### Безопасность
- [ ] Реализованы механизмы аутентификации
- [ ] Проверяются права доступа для каждой операции
- [ ] Валидируются входные данные
- [ ] Защита от инъекций и атак
- [ ] Чувствительные данные не передаются в открытом виде
- [ ] Правильная настройка флагов безопасности для кук в зависимости от окружения

#### Производительность
- [ ] Реализована пагинация для коллекций
- [ ] Настроены заголовки кэширования
- [ ] Применяется сжатие ответов
- [ ] Обеспечивается приемлемая производительность под нагрузкой

#### Документация
- [ ] API полностью документировано в OpenAPI
- [ ] Все параметры и ответы имеют описания
- [ ] Документация содержит примеры запросов и ответов
- [ ] Объяснены все возможные ошибки

#### Устойчивость
- [ ] Идемпотентные операции реализованы корректно
- [ ] Корректная обработка конкурентных запросов
- [ ] Предусмотрена обработка всех исключений
- [ ] Транзакционная целостность обеспечена

#### Версионирование
- [ ] Версия API отражена в пути
- [ ] Обеспечена обратная совместимость в пределах версии
- [ ] Новые поля добавляются не нарушая совместимости

#### Автоматизация
- [ ] Spectral-правила настроены и интегрированы в CI/CD
- [ ] API проходит все проверки Spectral без ошибок
- [ ] Настроены автотесты на соответствие API-контракту

#### Тестирование
- [ ] Написаны unit-тесты
- [ ] Написаны интеграционные тесты
- [ ] Тесты проверяют happy path и edge case сценарии
- [ ] Проведено нагрузочное тестирование
