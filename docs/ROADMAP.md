Документ описывает последовательные этапы реализации.
Каждый этап опирается на результаты предыдущего.

---

## Этап 1. Ядро движка

**Цель:** Сага из N шагов проходит последовательно и компенсируется
при ошибке на любом шаге.

### Механизмы:
- [ ] Модель определения саги (SagaDefinition, Step, Action, Compensation)
- [ ] Конечный автомат состояний саги:
      `PENDING → RUNNING → COMPLETED`
      `RUNNING → COMPENSATING → COMPENSATED`
      `RUNNING → FAILED` (некомпенсируемая ошибка)
- [ ] Исполнитель шагов (StepExecutor): синхронный HTTP-вызов к
      внешнему сервису, ожидание ответа
- [ ] Исполнитель компенсаций (CompensationExecutor): обход завершённых
      шагов в обратном порядке, вызов compensating action
- [ ] Передача контекста между шагами: response предыдущего шага
      доступен в body/url последующего через выражения
      `{{steps.<name>.response.<field>}}`
- [ ] Резолвер выражений (ExpressionResolver): парсинг и подстановка
      значений из контекста саги

---

## Этап 2. Персистентность и журнал событий

**Цель:** Состояние саги сохраняется в БД и переживает перезапуск
приложения.

### Механизмы:
- [ ] JPA-сущности: Saga, SagaStep, SagaEvent
- [ ] Append-only журнал событий (каждое изменение состояния = запись):
      `SAGA_STARTED`, `STEP_STARTED`, `STEP_COMPLETED`, `STEP_FAILED`,
      `COMPENSATION_STARTED`, `COMPENSATION_COMPLETED`, `SAGA_COMPLETED`,
      `SAGA_COMPENSATED`, `SAGA_FAILED`
- [ ] Оптимистичная блокировка (version column) для защиты от
      race conditions при параллельных обновлениях
- [ ] Профиль БД: PostgreSQL 
- [ ] Миграции схемы (Flyway)
---

## Этап 3. REST API

**Цель:** Внешние запросы могут определять, запускать и отслеживать саги
через HTTP.

### Механизмы:
- [ ] `POST /api/sagas/definitions` — создать определение саги
- [ ] `GET  /api/sagas/definitions/{name}` — получить определение
- [ ] `POST /api/sagas/{id}/start` — запустить экземпляр саги с payload
- [ ] `GET  /api/sagas/{id}` — статус саги + история шагов
- [ ] `GET  /api/sagas` — список саг с фильтрацией по статусу
- [ ] `POST /api/sagas/{id}/cancel` — принудительная отмена
- [ ] Валидация JSON-схемы определения саги при создании
- [ ] Корреляционный ID (sagaId) в каждом HTTP-запросе к внешним
      сервисам (заголовок `X-Saga-Id`)

---

## Этап 4. Надёжность

**Цель:** Оркестратор корректно обрабатывает сбои: таймауты,
временные ошибки, падение самого оркестратора.

### Механизмы:
- [ ] Retry с exponential backoff на уровне шага
      (конфигурируемо: maxAttempts, initialBackoff, multiplier)
- [ ] Таймауты на HTTP-вызов (конфигурируемо на уровне шага)
- [ ] Crash Recovery: фоновый планировщик сканирует саги в статусе
      RUNNING дольше N минут и возобновляет их с последнего
      незавершённого шага
- [ ] Обработка «зависших» компенсаций: если compensation action
      упал — retry до успеха (компенсация не может быть пропущена)
- [ ] Graceful shutdown: при остановке приложения текущие саги
      не теряются (дожидаемся завершения текущего шага)

---

## Этап 5. Асинхронный режим (Callbacks)

**Цель:** Поддержка долгоживущих шагов, где внешний сервис не
отвечает мгновенно, а присылает callback позже.

### Механизмы:
- [ ] `POST /api/sagas/{id}/callback` — приём webhook от внешнего
      сервиса
- [ ] Режим шага: `sync` (по умолчанию, ждём HTTP-ответ) vs
      `async` (отправляем запрос, ждём callback)
- [ ] Для async-шага: оркестратор передаёт callback URL в теле
      запроса внешнему сервису
- [ ] Обработка race condition: callback пришёл после того, как
      таймаут уже запустил компенсацию
- [ ] Идемпотентность callback-ов (повторный callback с тем же
      stepId игнорируется)

### Критерий готовности:
Async-шаг: оркестратор отправляет запрос → сервис отвечает 202 →
через 30 сек сервис шлёт callback → сага продолжает следующий шаг.

---
## Этап 6. Пользовательский интерфейс

**Цель:** Визуальное отображение состояния саг и управление ими.

### Механизмы:
- [ ] Страница списка саг (таблица: ID, имя, статус, время, шаги)
- [ ] Страница деталей саги:
      - Визуальный граф шагов (зелёный/красный/жёлтый/серый)
      - Event log (хронология событий)
      - Payload и response каждого шага
- [ ] Страница создания/редактирования определения саги
      (JSON-редактор с валидацией)
- [ ] Real-time обновление через SSE (Server-Sent Events)
- [ ] Кнопки управления: Retry, Force Compensate, Cancel
- [ ] Встраивание React SPA в Spring Boot (static resources)

---

## Этап 7. Наблюдаемость

**Цель:** Оркестратор интегрируется в стандартный стек мониторинга.

### Механизмы:
- [ ] Метрики Micrometer:
      - `temporalis.saga.started` (counter)
      - `temporalis.saga.completed` (counter)
      - `temporalis.saga.compensated` (counter)
      - `temporalis.saga.failed` (counter)
      - `temporalis.step.duration` (histogram, tags: sagaName, stepName)
      - `temporalis.step.retry.attempts` (histogram)
- [ ] Prometheus endpoint (`/actuator/prometheus`)
- [ ] Structured logging (JSON-формат, sagaId в каждом log-событии)
- [ ] Health check (`/actuator/health`)
- [ ] Готовый Grafana dashboard (JSON-файл для импорта)

### Критерий готовности:
Подключить Prometheus → импортировать Grafana dashboard → видеть
графики успешности саг и latency шагов.

---

## Этап 8. Дистрибуция и полировка

**Цель:** Проект готов к публикации и использованию другими людьми.

### Механизмы:
- [ ] Docker-образ на Docker Hub
- [ ] Docker Compose файл (temporalis + postgres + пример сервисов)
- [ ] GitHub Actions CI/CD (тесты → сборка → публикация образа)
- [ ] README с quickstart (3 команды от нуля до работающей саги)
- [ ] Примеры определений саг (e-commerce, user onboarding, payment)
- [ ] Документация API (OpenAPI/Swagger)
- [ ] Лицензия, CONTRIBUTING, CHANGELOG