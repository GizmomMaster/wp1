# 04. Асинхронное взаимодействие

## 4.1. Разделение Kafka и RabbitMQ

Оба брокера заданы стеком, но решают разные задачи. Смешивать их роли нельзя —
иначе появляются два способа сделать одно и то же и теряется предсказуемость.

| | **Kafka** | **RabbitMQ** |
|---|---|---|
| Назначение | Доменные события — факты о произошедшем | Команды и задачи — поручения выполнить работу |
| Модель | Publish/subscribe, лог с ретеншном | Work queue, конкурирующие потребители |
| Потребителей на сообщение | Много, независимо | Один |
| Порядок | Гарантирован в партиции | Не гарантирован |
| Повтор истории | Да, реплей с офсета | Нет |
| Типовое сообщение | `payment.request.approved` | `file.parse`, `way4.transfer.execute` |
| Ретеншн | 7 дней (30 для аудита) | до подтверждения + DLQ |

**Правило:** событие в Kafka описывает прошлое и не адресовано конкретному
получателю. Команда в RabbitMQ адресована конкретному обработчику и должна быть
выполнена ровно один раз.

## 4.2. Топики Kafka

Именование: `orion.<домен>.<сущность>.<событие>.v<версия>`.
Ключ партиционирования — `organizationId` (сохраняет порядок событий по организации).
Партиций — 12 **[ДОПУЩЕНИЕ]** (при ~500 организациях достаточно, запас на рост).

| Топик | Продюсер | Основные потребители |
|---|---|---|
| `orion.iam.user.v1` | iam-svc | audit-svc, notification-svc |
| `orion.org.organization.v1` | org-svc | employee-svc, cards-svc, payment-svc, treasury-svc |
| `orion.org.contract.v1` | org-svc | cards-svc, payment-svc, treasury-svc |
| `orion.employee.employee.v1` | employee-svc | cards-svc, payment-svc, payslip-svc |
| `orion.cards.request.v1` | cards-svc | workflow-svc, notification-svc, audit-svc |
| `orion.cards.item.v1` | cards-svc | notification-svc (агрегированно) |
| `orion.payment.request.v1` | payment-svc | workflow-svc, notification-svc, audit-svc |
| `orion.treasury.cycle.v1` | treasury-svc | notification-svc, audit-svc |
| `orion.workflow.instance.v1` | workflow-svc | cards-svc, payment-svc, notification-svc |
| `orion.file.import.v1` | file-svc | cards-svc, payment-svc, payslip-svc |
| `orion.reference.dictionary.v1` | reference-svc | все (инвалидация кеша) |
| `orion.audit.event.v1` | все сервисы | audit-svc |
| `orion.notification.request.v1` | все сервисы | notification-svc |
| `orion.integration.callback.v1` | integration-hub | cards-svc, payment-svc, treasury-svc |

**DLQ:** `<топик>.dlq`. Сообщение уходит в DLQ после исчерпания повторов;
на непустой DLQ настроен алерт.

## 4.3. Формат события (CloudEvents-совместимый)

```json
{
  "specversion": "1.0",
  "id": "018f3c2a-...",
  "type": "orion.payment.request.approved.v1",
  "source": "/orion/payment-svc",
  "time": "2026-09-04T10:15:30.123Z",
  "subject": "payment-request/8f21c3d4",
  "correlationid": "b7e1...",
  "causationid": "a11c...",
  "datacontenttype": "application/json",
  "data": {
    "requestId": "8f21c3d4",
    "organizationId": "1f0e...",
    "contractId": "77aa...",
    "channel": "DBO",
    "totalAmount": "1250000.00",
    "currency": "RUB",
    "itemCount": 420,
    "approvedBy": "user-123",
    "approvedAt": "2026-09-04T10:15:29Z"
  }
}
```

**Правила эволюции:** добавление необязательного поля — минорное изменение без
смены версии; удаление или смена типа поля — новый топик `.v2` с параллельной
публикацией на время миграции.

**В событиях не передаются ПДн** — только идентификаторы. Потребитель, которому
нужны ФИО, запрашивает их у `employee-svc` с проверкой прав.

## 4.4. Очереди RabbitMQ

Exchange `orion.commands` (direct), routing key = имя команды.

| Очередь | Потребитель | Назначение |
|---|---|---|
| `file.parse` | fileproc-worker | Парсинг и валидация загруженного файла |
| `document.generate` | document-svc | Генерация документов и заданий на печать |
| `payslip.render` | payslip-svc | Формирование HTML расчётных листов |
| `integration.way4.transfer` | integration-hub | Перевод средств |
| `integration.way4.card` | integration-hub | Операции по картам |
| `integration.eq.balance` | integration-hub | Запрос остатков |
| `integration.dbo.payment` | integration-hub | Отправка платёжного поручения |
| `integration.factory.order` | integration-hub | Заказ выпуска карт |
| `integration.pers.job` | integration-hub | Задание на персонализацию |
| `integration.crm.sync` | integration-hub | Синхронизация физлиц |
| `notification.send` | notification-svc | Доставка уведомления |
| `scheduler.trigger` | целевые сервисы | Запуск по расписанию |

**Политики:**
- `x-dead-letter-exchange` на каждой очереди, DLQ `<queue>.dlq`.
- Повторы с экспоненциальной задержкой через `rabbitmq_delayed_message_exchange`:
  10 с → 1 мин → 5 мин → 30 мин, далее DLQ.
- `prefetch` = 10 для лёгких задач, 1 для денежных операций (строгая последовательность).
- Приоритетные очереди для казначейства: `x-max-priority: 10`.

## 4.5. Transactional Outbox

Ни одно событие не публикуется напрямую из бизнес-транзакции. В каждой схеме БД:

```sql
CREATE TABLE outbox_messages (
    id             bigserial PRIMARY KEY,
    message_id     uuid        NOT NULL UNIQUE,
    aggregate_type text        NOT NULL,
    aggregate_id   text        NOT NULL,
    type           text        NOT NULL,
    payload        jsonb       NOT NULL,
    destination    text        NOT NULL,      -- kafka:<topic> | rmq:<queue>
    partition_key  text,
    created_at     timestamptz NOT NULL DEFAULT now(),
    published_at   timestamptz,
    attempts       int         NOT NULL DEFAULT 0,
    last_error     text
);
CREATE INDEX ix_outbox_pending ON outbox_messages (created_at)
    WHERE published_at IS NULL;
```

Фоновый диспетчер (одна активная реплика через Redis-лок или `FOR UPDATE SKIP LOCKED`)
читает неопубликованные, отправляет в брокер, проставляет `published_at`.
Гарантия — **at-least-once**, поэтому все потребители идемпотентны.

**Inbox на стороне потребителя:**

```sql
CREATE TABLE inbox_messages (
    message_id   uuid PRIMARY KEY,
    consumer     text NOT NULL,
    processed_at timestamptz NOT NULL DEFAULT now()
);
```

Обработка события начинается со вставки в `inbox_messages`; конфликт по
первичному ключу означает дубль — сообщение подтверждается без повторной обработки.

## 4.6. Саги

Оркестрация — внутри сервиса-владельца процесса, отдельного оркестратора нет
(ADR-010). Состояние саги хранится в таблице процесса, шаги идемпотентны,
компенсации явные.

**Сага выпуска карт** (`cards-svc`, на каждую строку заявки):

| Шаг | Действие | Компенсация |
|---|---|---|
| 1 | CRM: найти или создать физлицо | нет (данные остаются в CRM) |
| 2 | Factory: заказ выпуска карты | Factory: отмена заказа |
| 3 | Карт Перс: задание на персонализацию | Карт Перс: отзыв задания |
| 4 | WAY4: открытие счёта и привязка карты | WAY4: блокировка счёта, ручной разбор |
| 5 | Фиксация `Card` в ОРИОН, статус `Done` | — |

Компенсация запускается только при неустранимой ошибке шага (не таймаут).
Таймаут → повтор по политике; после исчерпания — статус `Failed`, задача
администратору, ручное решение: повтор или компенсация.

**Сага выплаты** (`payment-svc`) короче и **не имеет автокомпенсации**: после
успешной отправки в ДБО ЮРЛ откат невозможен. Отсюда требование: проверка
рабочего дня, лимитов и остатков — **до** отправки, а не после.

## 4.7. Порядок и конкурентность

- Ключ партиции `organizationId` гарантирует порядок событий по организации.
- Внутри заявки строки обрабатываются параллельно; последовательность нужна
  только для казначейства (утро → вечер) и обеспечивается статусной проверкой.
- Конкурентные правки агрегата — оптимистическая блокировка (`xmin` / `rowversion`),
  конфликт → HTTP 409 с текущей версией.
- Распределённые блокировки на Redis (Redlock) — только для суточных циклов
  казначейства и одиночных фоновых диспетчеров.
