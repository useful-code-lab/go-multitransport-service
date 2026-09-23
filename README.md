# Go Multi-Transport Service

Практический пример production-style сервиса на **Go**, который показывает, как одну бизнес-логику предоставить сразу через несколько способов взаимодействия: **REST, gRPC, NATS RPC и RabbitMQ RPC**.

Проект можно использовать не только как демонстрацию технологий, но и как **готовую основу для изучения микросервисной архитектуры или создания собственного Go-сервиса**.

## Зачем нужен этот проект?

Если вы изучаете Go и микросервисную архитектуру, часто приходится отдельно разбираться с REST, gRPC, брокерами сообщений, PostgreSQL, Docker, логированием и архитектурой приложения.

Этот проект объединяет эти задачи в одном работающем примере.

Изучая репозиторий, вы можете увидеть:

* как организовать Go-приложение по принципам **Clean Architecture**;
* как отделить бизнес-логику от транспорта;
* как предоставить один UseCase через REST, gRPC и message brokers;
* как организовать **RPC через NATS и RabbitMQ**;
* как работать с PostgreSQL и миграциями;
* как генерировать Go-код из `.proto`;
* как организовать структурированное JSON-логирование;
* как запускать окружение через Docker Compose;
* как корректно завершать работу приложения и его соединений;
* как тестировать разные варианты взаимодействия с одним сервисом.

Главная идея проекта — показать, что **бизнес-логика не должна зависеть от способа доставки запроса**.

Например, операция перевода может быть вызвана через HTTP:

```text
Client → REST → UseCase → PostgreSQL
```

или через gRPC:

```text
Client → gRPC → UseCase → PostgreSQL
```

или через NATS:

```text
Client → NATS → RPC Handler → UseCase → PostgreSQL
```

или через RabbitMQ:

```text
Client → RabbitMQ → RPC Handler → UseCase → PostgreSQL
```

При этом сам UseCase остается единым.

---

## Что можно получить от проекта

### 1. Понять Clean Architecture на работающем примере

Проект показывает практическое разделение приложения на уровни:

```text
Transport / Controller
        ↓
     UseCase
        ↓
    Repository
        ↓
    PostgreSQL
```

Транспортный слой отвечает за получение и преобразование запросов, UseCase — за бизнес-логику, Repository — за работу с данными.

Благодаря этому REST можно заменить на gRPC или message broker, не переписывая основную бизнес-логику.

---

### 2. Изучить несколько способов взаимодействия с одним сервисом

В проекте реализованы четыре транспорта:

| Транспорт    | Для чего можно использовать                     |
| ------------ | ----------------------------------------------- |
| REST / HTTP  | Внешние API, frontend, интеграции               |
| gRPC         | Быстрое взаимодействие между сервисами          |
| NATS RPC     | Легковесное взаимодействие через message broker |
| RabbitMQ RPC | Request-Reply взаимодействие через AMQP         |

Это позволяет на одном проекте сравнить разные подходы к коммуникации между компонентами системы.

---

### 3. Получить пример RPC через message brokers

Помимо классического HTTP API проект показывает более интересный сценарий:

```text
Request → Broker → Consumer → UseCase → Response
```

Для NATS и RabbitMQ реализовано взаимодействие в стиле **Request-Reply / RPC**.

Это полезно для понимания того, как сервисы могут взаимодействовать без прямого HTTP-вызова.

---

### 4. Увидеть PostgreSQL в составе реального приложения

Проект использует PostgreSQL для хранения истории операций.

В репозитории есть:

* подключение к PostgreSQL;
* Repository-слой;
* SQL-миграции;
* сохранение истории;
* получение данных;
* обработка ошибок БД.

Посмотреть записи можно непосредственно из контейнера:

```bash
docker exec -it core-platform-db \
  psql -U user -d core_db \
  -c "SELECT * FROM history;"
```

---

### 5. Быстро поднять полноценное окружение

Не требуется вручную устанавливать PostgreSQL, NATS и RabbitMQ.

Инфраструктура запускается через Docker Compose:

```bash
make compose-up
```

В результате приложение получает готовое окружение с:

* PostgreSQL;
* NATS;
* RabbitMQ.

Остановить инфраструктуру:

```bash
make compose-down
```

---

### 6. Посмотреть практическую организацию логирования

В проекте используется `zerolog`.

Логи:

* структурированные;
* представлены в JSON;
* выводятся в консоль;
* сохраняются в файлы;
* разделяются по датам.

Например:

```text
logs/
├── 2026-03-17.log
├── 2026-03-18.log
└── 2026-03-19.log
```

Уровень логирования и директория настраиваются через environment variables.

---

## Архитектура

Основная идея проекта — разделить **что делает приложение** и **как к нему обращаются**.

Упрощенная схема:

```text
                    ┌─────────────┐
                    │    REST     │
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │             │
┌─────────────┐     │   UseCase   │     ┌─────────────┐
│    gRPC     │────►│             │────►│ PostgreSQL  │
└─────────────┘     │             │     └─────────────┘
                    └──────▲──────┘
                           │
              ┌────────────┴────────────┐
              │                         │
       ┌──────┴──────┐          ┌───────┴───────┐
       │    NATS     │          │   RabbitMQ    │
       └─────────────┘          └───────────────┘
```

Один UseCase обслуживает разные способы взаимодействия.

Это позволяет добавлять новый транспорт без необходимости переписывать бизнес-логику.

---

## Структура проекта

```text
.
├── api/
│   └── proto/              # Protobuf API
│
├── cmd/
│   └── app/                # Точка запуска приложения
│
├── config/                 # Конфигурация
│
├── internal/
│   ├── controller/         # REST, gRPC, NATS, AMQP
│   ├── usecase/            # Бизнес-логика
│   └── repo/               # Работа с данными
│
├── migrations/             # SQL-миграции
│
├── pkg/                    # Переиспользуемые компоненты
│
├── test/                   # Тестовые клиенты
│
├── Dockerfile
├── docker-compose.yml
├── Makefile
└── README.md
```

---

## Технологии

* **Go 1.21+**
* **Fiber** — HTTP API
* **gRPC / Protobuf** — RPC
* **NATS** — message broker
* **RabbitMQ / AMQP** — message broker
* **PostgreSQL**
* **pgx**
* **zerolog**
* **Docker / Docker Compose**
* **golang-migrate**
* **Makefile**

---

# Быстрый запуск

## 1. Клонировать проект

```bash
git clone <repository-url>
cd go-multitransport-service
```

## 2. Создать конфигурацию

Скопировать пример environment variables:

```bash
cp .env.example .env
```

При необходимости изменить параметры подключения.

## 3. Запустить инфраструктуру

```bash
make compose-up
```

Будут запущены PostgreSQL, NATS и RabbitMQ.

## 4. Запустить приложение

```bash
make run
```

После запуска один сервис становится доступен через несколько транспортов.

---

# REST API

REST — самый простой способ проверить сервис.

## Healthcheck

```bash
curl http://localhost:8080/healthz
```

## Создать перевод

```bash
curl -X POST http://localhost:8080/v1/translation/do \
  -H "Content-Type: application/json" \
  -d '{
    "source":"en",
    "destination":"ru",
    "original":"Hello World"
  }'
```

## Получить историю

```bash
curl http://localhost:8080/v1/translation/history
```

REST удобно использовать для ручного тестирования и интеграции с клиентскими приложениями.

---

# gRPC

gRPC позволяет обращаться к той же бизнес-логике через RPC.

Сервер поддерживает Reflection API.

## Установка grpcurl

```bash
go install github.com/fullstorydev/grpcurl/cmd/grpcurl@latest
```

## Посмотреть доступные методы

```bash
grpcurl -plaintext localhost:8081 list v1.Translation
```

## Выполнить перевод

```bash
grpcurl -plaintext \
  -d '{"source":"en","destination":"ru","original":"Hello gRPC"}' \
  localhost:8081 \
  v1.Translation/Translate
```

## Получить историю

```bash
grpcurl -plaintext \
  localhost:8081 \
  v1.Translation/GetHistory
```

---

# NATS RPC

NATS используется как транспорт для Request-Reply взаимодействия.

Установить CLI:

```bash
go install github.com/nats-io/natscli/nats@latest
```

## Выполнить перевод

```bash
nats request \
  v1.translation.translate \
  '{"source":"en","destination":"ru","original":"Hello NATS"}'
```

## Получить историю

```bash
nats request \
  v1.translation.history \
  '{}'
```

NATS CLI также позволяет увидеть Round Trip Time запроса.

Например:

```text
Received with rtt 7.5ms
```

Это позволяет быстро проверить сам механизм взаимодействия через брокер.

---

# RabbitMQ RPC

Для RabbitMQ реализован Request-Reply сценарий.

Для тестирования используется готовый Go-клиент:

```bash
go run test/amqp_client.go
```

Пример:

```text
[x] Sent request:
{"source":"en","destination":"ru","original":"Hello Rabbit RPC"}

[.] Got response:
{"source":"en","destination":"ru","original":"Hello Rabbit RPC", ...}
```

Тестовый клиент показывает полный цикл:

```text
Client
  ↓
RabbitMQ
  ↓
RPC Handler
  ↓
UseCase
  ↓
Response
  ↓
Client
```

---

# Миграции базы данных

Для управления схемой используется `golang-migrate`.

Создать новую миграцию:

```bash
make migrate-create name=init_table
```

Применить миграции:

```bash
make migrate-up
```

Откатить последнюю миграцию:

```bash
make migrate-down
```

---

# Генерация gRPC-кода

Protobuf-файлы находятся в:

```text
api/proto/v1/
```

Сгенерировать Go-код:

```bash
make proto
```

Это позволяет изменять API через `.proto` и автоматически получать соответствующие структуры и интерфейсы Go.

---

# Логирование

Для логирования используется `zerolog`.

Логи одновременно выводятся:

```text
Console
   +
Files
```

Настройки:

```env
LOG_LEVEL=info
LOG_FILE_PATH=./logs/
```

Файлы создаются с привязкой к дате:

```text
logs/2026-03-18.log
```

Посмотреть актуальный лог:

```bash
tail -f logs/$(date +%Y-%m-%d).log
```

Ошибки инфраструктурного слоя сохраняют исходную причину через error wrapping:

```go
return fmt.Errorf("HistoryRepo - StoreHistory - Exec: %w", err)
```

При этом внутренние технические детали не возвращаются непосредственно клиенту API.

---

# Graceful Shutdown

При завершении приложения корректно закрываются используемые ресурсы:

* HTTP server;
* gRPC server;
* NATS connection;
* RabbitMQ connection;
* PostgreSQL connection.

Это позволяет избежать некорректного завершения соединений и потери незавершенных операций.

---

# Что можно использовать в своем проекте

Этот репозиторий можно рассматривать как стартовую точку для собственного Go-сервиса.

Из него можно взять отдельные части:

### Для HTTP-сервиса

* структуру Clean Architecture;
* Fiber;
* конфигурацию;
* PostgreSQL Repository;
* миграции;
* логирование.

### Для микросервисов

* gRPC;
* Protobuf;
* NATS RPC;
* RabbitMQ RPC;
* разделение transport и business logic.

### Для инфраструктуры

* Docker Compose;
* PostgreSQL;
* NATS;
* RabbitMQ;
* автоматизацию через Makefile.

### Для обучения

Проект особенно полезен, если вы хотите на одном примере увидеть связь между:

```text
Architecture
     ↓
Business Logic
     ↓
Transport
     ↓
Database / Message Broker
     ↓
Infrastructure
```

---

# Makefile

Основные команды:

| Команда               | Назначение                |
| --------------------- | ------------------------- |
| `make deps`           | Проверить зависимости     |
| `make run`            | Запустить приложение      |
| `make compose-up`     | Запустить инфраструктуру  |
| `make compose-down`   | Остановить инфраструктуру |
| `make migrate-create` | Создать миграцию          |
| `make migrate-up`     | Применить миграции        |
| `make migrate-down`   | Откатить миграцию         |
| `make proto`          | Сгенерировать gRPC-код    |

Полный список команд можно посмотреть через:

```bash
make
```

---

# Идея проекта

Проект показывает один из важных архитектурных принципов:

> **Бизнес-логика не должна зависеть от способа доставки запроса.**

REST, gRPC, NATS и RabbitMQ здесь выступают разными входными точками к одной системе.

Поэтому репозиторий можно использовать как:

* учебный пример Clean Architecture;
* пример реализации нескольких транспортов в Go;
* основу для собственного микросервиса;
* playground для экспериментов с gRPC и message brokers;
* пример Docker-окружения с PostgreSQL, NATS и RabbitMQ;
* основу для дальнейшего расширения сервиса.

---

## Идеи для дальнейшего развития

Проект можно расширить, например:

* добавить реальный внешний API перевода;
* добавить авторизацию;
* добавить Docker image для production;
* добавить unit и integration tests;
* добавить OpenTelemetry;
* добавить Prometheus metrics;
* добавить Swagger/OpenAPI;
* добавить retry и timeout для брокеров;
* добавить обработку Dead Letter Queue в RabbitMQ;
* добавить NATS JetStream;
* добавить CI/CD;
* добавить Kubernetes deployment.

---

## Лицензия

Проект предназначен для изучения, экспериментов и использования в качестве основы для собственных решений.
