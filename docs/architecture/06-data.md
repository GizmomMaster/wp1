# 06. Модель данных и хранилища

## 6.1. Стратегия хранения

| Хранилище | Что хранит |
|---|---|
| **PostgreSQL** | Все транзакционные данные. Логическая БД на сервис; физически на этапе 1 — один кластер, отдельная база и роль на сервис, кросс-БД запросы запрещены на уровне прав |
| **Redis** | Кеш справочников и прав, идемпотентность, распределённые локи, счётчики rate limit, `jti` SSO-токенов |
| **MinIO** | Файлы: реестры, сканы договоров, сгенерированные документы, расчётные листы, протоколы ошибок |
| **Kafka** | Событийный лог (7 дней, аудит — 30) |

**[ДОПУЩЕНИЕ]** Один кластер Postgres с отдельными базами на сервис. При ~500
организациях этого достаточно; при росте разделяются по кластерам без изменения кода.

## 6.2. Общие соглашения

- Первичные ключи — `uuid` (UUIDv7 для сортируемости по времени вставки).
- Все таблицы содержат `created_at`, `created_by`, `updated_at`, `updated_by` (`timestamptz`, UTC).
- Мягкое удаление (`deleted_at`) — только там, где ТЗ требует архива; денежные
  документы не удаляются никогда.
- Оптимистическая блокировка — колонка `version integer`.
- Денежные суммы — `numeric(19,4)` + `currency char(3)`. Тип `money` не используется.
- Мультитенантность — обязательная колонка `organization_id` во всех клиентских
  таблицах + Row Level Security (см. [07-security.md](07-security.md)).
- Именование — `snake_case`, таблицы во множественном числе.
- Миграции — единый инструмент на все сервисы **[ДОПУЩЕНИЕ]**: EF Core Migrations;
  для сложных изменений на больших таблицах — ручные скрипты с `CONCURRENTLY`.

## 6.3. Ключевые схемы

### org-svc

```sql
CREATE TABLE organizations (
    id                uuid PRIMARY KEY,
    parent_id         uuid REFERENCES organizations(id),   -- холдинг
    full_name         text NOT NULL,
    short_name        text NOT NULL,
    inn               varchar(12) NOT NULL,
    kpp               varchar(9),
    ogrn              varchar(15),
    legal_address     jsonb NOT NULL,
    actual_address    jsonb,
    region_id         uuid,            -- справочник reference-svc
    status            text NOT NULL,   -- Draft|Active|Suspended|Terminated
    version           integer NOT NULL DEFAULT 1,
    created_at        timestamptz NOT NULL DEFAULT now(),
    created_by        uuid NOT NULL,
    updated_at        timestamptz,
    updated_by        uuid
);
CREATE UNIQUE INDEX ux_org_inn_kpp_active ON organizations (inn, coalesce(kpp,''))
    WHERE status <> 'Terminated';
CREATE INDEX ix_org_parent ON organizations (parent_id) WHERE parent_id IS NOT NULL;

CREATE TABLE contracts (
    id               uuid PRIMARY KEY,
    organization_id  uuid NOT NULL REFERENCES organizations(id),
    number           text NOT NULL,
    signed_at        date NOT NULL,
    valid_from       date NOT NULL,
    valid_to         date,
    status           text NOT NULL,     -- Draft|Active|Suspended|Terminated
    legal_data       jsonb NOT NULL,    -- реквизиты, подписанты
    version          integer NOT NULL DEFAULT 1
);
CREATE UNIQUE INDEX ux_contract_number ON contracts (organization_id, number);

CREATE TABLE contract_products (
    id            uuid PRIMARY KEY,
    contract_id   uuid NOT NULL REFERENCES contracts(id),
    product_code  text NOT NULL,
    tariff_code   text,
    valid_from    date NOT NULL,
    valid_to      date,                 -- закрывается при смене продукта
    replaced_by   uuid REFERENCES contract_products(id)
);
-- один активный экземпляр продукта в договоре на дату
CREATE UNIQUE INDEX ux_contract_product_active
    ON contract_products (contract_id, product_code) WHERE valid_to IS NULL;

CREATE TABLE contract_scans (
    id           uuid PRIMARY KEY,
    contract_id  uuid NOT NULL REFERENCES contracts(id),
    file_id      uuid NOT NULL,          -- ссылка в file-svc
    doc_type     text NOT NULL,
    uploaded_at  timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE organization_integration_settings (
    id               uuid PRIMARY KEY,
    organization_id  uuid NOT NULL REFERENCES organizations(id),
    contract_id      uuid REFERENCES contracts(id),
    system_code      text NOT NULL,      -- WAY4|EQ|DBO
    settings         jsonb NOT NULL,     -- схемы, счета, продуктовые коды
    valid_from       timestamptz NOT NULL DEFAULT now(),
    valid_to         timestamptz         -- версионирование настроек
);
```

### employee-svc

```sql
CREATE TABLE employees (
    id                uuid PRIMARY KEY,
    organization_id   uuid NOT NULL,
    person_ref        text,               -- идентификатор в CRM
    last_name         text NOT NULL,
    first_name        text NOT NULL,
    middle_name       text,
    birth_date        date NOT NULL,
    gender            char(1),
    inn               varchar(12),
    snils             varchar(14),
    doc_type          text NOT NULL,
    doc_series        varchar(10),
    doc_number        varchar(20),
    doc_issued_at     date,
    doc_issued_by     text,
    phone             text,
    email             text,
    personnel_number  text,               -- табельный номер
    status            text NOT NULL,      -- Active|Dismissed|Archived
    dismissed_at      date,
    archived_at       timestamptz,
    deleted_at        timestamptz,
    version           integer NOT NULL DEFAULT 1
);
CREATE INDEX ix_emp_org_status ON employees (organization_id, status)
    WHERE deleted_at IS NULL;
CREATE UNIQUE INDEX ux_emp_org_inn ON employees (organization_id, inn)
    WHERE inn IS NOT NULL AND deleted_at IS NULL;
-- поиск по ФИО без учёта регистра и ё/е
CREATE INDEX ix_emp_fio ON employees
    USING gin (to_tsvector('russian', last_name||' '||first_name||' '||coalesce(middle_name,'')));

CREATE TABLE employee_accounts (
    id           uuid PRIMARY KEY,
    employee_id  uuid NOT NULL REFERENCES employees(id),
    account_no   varchar(20) NOT NULL,
    card_mask    varchar(19),              -- только маска, PAN не хранится
    currency     char(3) NOT NULL DEFAULT 'RUB',
    status       text NOT NULL,
    opened_at    date,
    closed_at    date
);

CREATE TABLE employee_history (
    id           bigserial PRIMARY KEY,
    employee_id  uuid NOT NULL,
    changed_at   timestamptz NOT NULL DEFAULT now(),
    changed_by   uuid NOT NULL,
    field        text NOT NULL,
    old_value    text,
    new_value    text,
    reason       text
);
```

### payment-svc

```sql
CREATE TABLE payment_requests (
    id                uuid PRIMARY KEY,
    organization_id   uuid NOT NULL,
    contract_id       uuid NOT NULL,
    payer_org_id      uuid NOT NULL,     -- может быть родитель или дочерняя
    request_type      text NOT NULL,     -- Salary|Bonus|Other|Cashback
    channel           text NOT NULL,     -- DBO|WAY4
    period            text,              -- 2026-09
    total_amount      numeric(19,4) NOT NULL,
    currency          char(3) NOT NULL DEFAULT 'RUB',
    item_count        integer NOT NULL,
    status            text NOT NULL,
    workflow_id       uuid,
    import_job_id     uuid,
    idempotency_key   text NOT NULL,
    execute_on        date,
    created_at        timestamptz NOT NULL DEFAULT now(),
    created_by        uuid NOT NULL,
    version           integer NOT NULL DEFAULT 1
);
CREATE UNIQUE INDEX ux_payreq_idem ON payment_requests (organization_id, idempotency_key);
CREATE INDEX ix_payreq_org_status ON payment_requests (organization_id, status, created_at DESC);

CREATE TABLE payment_items (
    id                 uuid NOT NULL,
    request_id         uuid NOT NULL REFERENCES payment_requests(id),
    row_number         integer NOT NULL,
    employee_id        uuid,
    account_no         varchar(20) NOT NULL,
    amount             numeric(19,4) NOT NULL,
    purpose            text,
    status             text NOT NULL,    -- Pending|Sent|Executed|Failed
    external_ref       text,
    error_code         text,
    error_message      text,
    processed_at       timestamptz,
    created_at         timestamptz NOT NULL DEFAULT now(),
    -- ключ партиционирования обязан входить в PK и уникальные индексы
    PRIMARY KEY (id, created_at),
    CONSTRAINT ck_amount_positive CHECK (amount > 0)
) PARTITION BY RANGE (created_at);      -- помесячные партиции
CREATE UNIQUE INDEX ux_payitem_row ON payment_items (request_id, row_number, created_at);
CREATE INDEX ix_payitem_status ON payment_items (status, created_at)
    WHERE status IN ('Pending','Sent');


CREATE TABLE payment_orders (
    id            uuid PRIMARY KEY,
    request_id    uuid NOT NULL REFERENCES payment_requests(id),
    external_id   text,                  -- номер в ДБО ЮРЛ
    sent_at       timestamptz,
    status        text NOT NULL,
    status_at     timestamptz,
    raw_response  jsonb
);
```

### cards-svc

Структура зеркальна `payment-svc`: `card_requests` / `card_request_items` с
построчными статусами и полями саги (`saga_step`, `saga_state`, `attempt_count`,
`next_attempt_at`), плюс `cards`, `card_reissue_offers`, `card_handovers`.

### audit-svc

```sql
CREATE TABLE audit_events (
    id               bigserial,
    occurred_at      timestamptz NOT NULL,
    user_id          uuid,
    user_login       text,
    organization_id  uuid,
    service          text NOT NULL,
    action           text NOT NULL,      -- payment.request.approve
    target_type      text,
    target_id        text,
    result           text NOT NULL,      -- Success|Denied|Error
    ip               inet,
    user_agent       text,
    correlation_id   uuid,
    payload_before   jsonb,
    payload_after    jsonb,
    PRIMARY KEY (id, occurred_at)
) PARTITION BY RANGE (occurred_at);      -- помесячные партиции
CREATE INDEX ix_audit_user   ON audit_events (user_id, occurred_at DESC);
CREATE INDEX ix_audit_target ON audit_events (target_type, target_id, occurred_at DESC);
CREATE INDEX ix_audit_org    ON audit_events (organization_id, occurred_at DESC);
```

Права на таблицу: `INSERT` и `SELECT`, без `UPDATE`/`DELETE` для сервисной роли.

## 6.4. Партиционирование и ретеншн

| Таблица | Партиционирование | Хранение |
|---|---|---|
| `audit_events` | по месяцам | 3 года онлайн, далее выгрузка в MinIO **[ДОПУЩЕНИЕ]** |
| `payment_items` | по месяцам | 5 лет (срок хранения платёжных документов) |
| `card_request_items` | по месяцам | 5 лет |
| `treasury_operations` | по месяцам | 5 лет |
| `integration_calls` | по неделям | 90 дней |
| `notifications` | по месяцам | 1 год |
| `payslips` | по периоду расчёта | по политике заказчика, с архивацией |

Партиции создаются заранее фоновой задачей (`pg_partman` либо собственная задача
в `scheduler-svc`), отсечение старых — с предварительной выгрузкой в MinIO.

> Сроки хранения указаны как разумные значения по умолчанию и **должны быть
> подтверждены** комплаенсом банка — они регулируются законодательством
> (в частности, требованиями к хранению первичных документов и ПДн).

## 6.5. Redis

| Ключ | Назначение | TTL |
|---|---|---|
| `perm:{userId}` | Эффективные права пользователя | 60 с |
| `dict:{code}:{version}` | Справочник | 1 ч, инвалидация событием |
| `calendar:{year}` | Календарь рабочих дней | 24 ч |
| `idem:{scope}:{key}` | Результат идемпотентной операции | 24 ч |
| `lock:treasury:{contractId}:{date}` | Лок суточного цикла | 30 мин |
| `lock:outbox:{service}` | Лидер-лок диспетчера outbox | 30 с, продление |
| `sso:jti:{jti}` | Одноразовость SSO-токена | 5 мин |
| `rl:{clientId}:{window}` | Rate limit | окно |

Режим — Redis Sentinel или Cluster **[ДОПУЩЕНИЕ]**. Redis не является источником
истины: потеря кеша приводит только к росту латентности.

## 6.6. MinIO

| Бакет | Содержимое | Политика |
|---|---|---|
| `orion-imports` | Загруженные реестры | Версионирование, ретеншн 3 года |
| `orion-contracts` | Сканы договоров | Версионирование, WORM (object lock) на срок действия договора |
| `orion-documents` | Сгенерированные документы, задания на печать | 1 год |
| `orion-payslips` | HTML расчётных листов | По политике заказчика |
| `orion-reports` | Протоколы импорта, выгрузки | 90 дней |
| `orion-archive` | Выгруженные партиции БД | Долгосрочно |

Правила: шифрование на стороне сервера (SSE-S3), доступ приложений — только
через presigned URL с TTL 5 минут, прямой доступ пользователей к MinIO закрыт,
ключ объекта содержит `organizationId` для изоляции политиками бакета.
