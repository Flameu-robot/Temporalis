## Тип архитектуры: Модульный монолит

temporalis/
├── core/                    # Модуль 1: Ядро
│   ├── SagaDefinition
│   ├── SagaInstance
│   ├── StepDefinition
│   ├── SagaStatus / StepStatus
│   ├── ExpressionResolver
│   └── StateMachine
│
├── engine/                  # Модуль 2: Движок (логика выполнения)
│   ├── SagaEngine
│   ├── StepExecutor
│   ├── CompensationExecutor
│   ├── RetryPolicy
│   └── CrashRecoveryScheduler
│
├── persistence/             # Модуль 3: Хранилище
│   ├── entities/
│   │   ├── SagaEntity
│   │   ├── SagaStepEntity
│   │   └── SagaEventEntity
│   ├── repositories/
│   └── migrations/ (Flyway)
│
├── api/                     # Модуль 4: REST API
│   ├── controllers/
│   ├── dto/
│   └── validation/
│
├── callback/                # Модуль 5: Async callbacks
│   └── CallbackHandler
│
├── ui/                      # Модуль 6: Frontend (встроен в JAR)
│   └── static/              # Собранный React
│
└── observability/           # Модуль 7: Метрики и логи
    ├── MetricsConfig
    └── StructuredLogg

## Технологический стек

### Модуль 1: Core (Ядро)

> Чистая бизнес-логика, без фреймворков

- **Java 25** 
- Никаких фреймворков — только чистые Java классы и интерфейсы
- **Jackson 3.0** — сериализация/десериализация SagaDefinition (JSON)
- **Jakarta Bean Validation 3.1** — валидация моделей (`@NotNull`, `@NotBlank`)

### Модуль 2: Engine (Движок)

> Логика выполнения саг, retry, компенсации

- **Java 25**
- **Spring Framework 7.0** (входит в Spring Boot 4)
- **Spring ApplicationEventPublisher** — внутренняя шина событий между модулями
- **Spring @Scheduled** — планировщик Crash Recovery
- **Spring RestClient** (Spring 6.1+) — HTTP вызовы к внешним сервисам
- **Resilience4j 3.x** — retry с exponential backoff, таймауты на уровне шага
  
### Модуль 3: Persistence (Хранилище)

> JPA сущности, репозитории, миграции

- **Java 25**
- **Spring Data JPA 4.0** — репозитории, CRUD
- **Hibernate 7.0** — JPA провайдер (совместим со Spring Boot 4)
- **PostgreSQL 18** — основная БД
- **PostgreSQL JDBC Driver 42.7.x** — JDBC драйвер
- **Flyway 11.x** — миграции схемы БД
- **HikariCP 6.x** — connection pool (встроен в Spring Boot 4)

### Модуль 4: API (REST)

> HTTP контроллеры, DTO, валидация запросов

- **Java 25**
- **Spring Boot 4.0** — автоконфигурация, embedded Tomcat
- **Spring Web MVC 7.0** — REST контроллеры
- **Jakarta Bean Validation 3.1** — валидация входящих DTO (`@Valid`)
- **Jackson 3.0** — сериализация JSON ответов
- **SpringDoc OpenAPI 3.x** — автогенерация Swagger UI (`/swagger-ui.html`)

### Модуль 5: Callback (Async режим)

> Приём webhook от внешних сервисов

- **Java 25**
- **Spring Web MVC 7.0** — endpoint для приёма callbacks
- **Spring ApplicationEventPublisher** — публикация события о полученном callback
- **Idempotency** — проверка через таблицу `saga_events` (без доп. зависимостей)

### Модуль 6: UI (Frontend)

> React SPA встроенный в JAR

- **React 19** — UI фреймворк
- **TypeScript 5.8** — типизация
- **Vite 6.x** — сборщик (быстрая dev сборка)
- **ReactFlow 12.x** — визуальный граф шагов саги
- **TanStack Query 5.x** — fetching/кэширование данных с API
- **Tailwind CSS 4.x** — стилизация
- **Maven Frontend Plugin 1.15.x** — сборка React внутри Maven lifecycle, результат копируется в `src/main/resources/static`

### Модуль 7: Observability (Метрики и логи)

> Prometheus метрики, structured logging, health checks

- **Java 25**
- **Spring Boot Actuator 4.0** — `/actuator/health`, `/actuator/prometheus`
- **Micrometer 1.15.x** — метрики (counter, histogram, timer)
- **Micrometer Prometheus Registry 1.15.x** — экспорт в формате Prometheus
- **Logback 1.5.x** — логирование (встроен в Spring Boot)
- **Logstash Logback Encoder 8.x** — JSON формат логов
- **MDC (Mapped Diagnostic Context)** — `sagaId` в каждой строке лога (встроен в Logback)

## Полный стек одной таблицей

| Слой               | Инструмент                       | Версия     |
| ------------------ | -------------------------------- | ---------- |
| Язык               | Java                             | **25**     |
| Фреймворк          | Spring Boot                      | **4.0**    |
| Web                | Spring Web MVC                   | **7.0**    |
| БД                 | PostgreSQL                       | **18**     |
| ORM                | Spring Data JPA                  | **4.0**    |
| JPA провайдер      | Hibernate                        | **7.0**    |
| JDBC драйвер       | PostgreSQL JDBC                  | **42.7.x** |
| Connection Pool    | HikariCP                         | **6.x**    |
| Миграции           | Flyway                           | **11.x**   |
| HTTP клиент        | Spring RestClient                | **7.0**    |
| Retry / Timeout    | Resilience4j                     | **3.x**    |
| Планировщик        | Spring @Scheduled                | **7.0**    |
| События            | Spring ApplicationEventPublisher | **7.0**    |
| Валидация          | Jakarta Bean Validation          | **3.1**    |
| Сериализация       | Jackson                          | **3.0**    |
| Real-time          | SSE (Spring SseEmitter)          | **7.0**    |
| Метрики            | Micrometer                       | **1.15.x** |
| Метрики экспорт    | Micrometer Prometheus Registry   | **1.15.x** |
| Health / Info      | Spring Boot Actuator             | **4.0**    |
| Логи формат        | Logstash Logback Encoder         | **8.x**    |
| Логи движок        | Logback                          | **1.5.x**  |
| API документация   | SpringDoc OpenAPI                | **3.x**    |
| Frontend фреймворк | React                            | **19**     |
| Frontend язык      | TypeScript                       | **5.8**    |
| Frontend сборщик   | Vite                             | **6.x**    |
| Граф шагов         | ReactFlow                        | **12.x**   |
| Data fetching      | TanStack Query                   | **5.x**    |
| Стилизация         | Tailwind CSS                     | **4.x**    |
| Frontend в JAR     | Maven Frontend Plugin            | **1.15.x** |
| Контейнеризация    | Docker + Docker Compose          | **27.x**   |
| CI/CD              | GitHub Actions                   | —          |
