# 08. Ключевые сценарии

## 8.1. Массовый выпуск карт из файла

Самый показательный сценарий: объединяет загрузку файла, валидацию, согласование,
сагу с четырьмя внешними системами и построчные статусы.

```mermaid
sequenceDiagram
    autonumber
    participant U as Сотрудник<br/>организации
    participant BFF as bff-client
    participant FILE as file-svc
    participant W as fileproc-worker
    participant CARD as cards-svc
    participant WF as workflow-svc
    participant A as Администратор
    participant HUB as integration-hub
    participant EXT as Factory/КартПерс/<br/>WAY4/CRM

    U->>BFF: загрузка реестра сотрудников
    BFF->>FILE: POST /files (multipart)
    FILE->>FILE: антивирус, тип, размер
    FILE->>FILE: сохранение в MinIO
    FILE-->>BFF: fileId, importJobId
    FILE->>W: RMQ file.parse
    W->>W: разбор по FileTemplate + валидация
    alt есть ошибки
        W-->>FILE: ImportJob=Failed + протокол
        BFF-->>U: протокол ошибок с номерами строк
    else валидно
        W-->>FILE: ImportJob=Valid, N строк
        U->>BFF: создать заявку из importJobId
        BFF->>CARD: POST /card-requests
        CARD->>CARD: дедупликация по employee-svc,<br/>проверка договора и продуктов
        CARD-->>U: заявка Validated, предпросмотр
        U->>BFF: отправить на согласование
        BFF->>CARD: POST /card-requests/{id}/submit
        CARD->>WF: запуск процесса (тип, сумма, организация)
        WF->>WF: подбор маршрута, задачи согласующим
        WF-->>A: задача в очереди согласования
        alt возврат на доработку
            A->>WF: return + комментарий
            WF-->>CARD: workflow.returned
            CARD-->>U: статус Rework, комментарий
        else согласовано
            A->>WF: approve
            WF-->>CARD: workflow.approved
            CARD->>CARD: статус Approved → Processing
            loop по каждой строке (параллельно, батчами)
                CARD->>HUB: RMQ команды саги
                HUB->>EXT: CRM → Factory → КартПерс → WAY4
                EXT-->>HUB: результат
                HUB-->>CARD: integration.callback
                CARD->>CARD: обновление статуса строки
            end
            CARD-->>U: построчные статусы в реальном времени
        end
    end
```

**Отказ на шаге саги:** строка переходит в `Retrying`, повтор по политике;
после исчерпания — `Failed` с кодом ошибки, задача администратору, ручное
решение (повтор или компенсация). Успешные строки не откатываются — заявка
завершается в статусе `PartiallyCompleted`, что видно клиенту построчно.

## 8.2. Начисление зарплаты

```mermaid
sequenceDiagram
    autonumber
    participant U as Сотрудник<br/>организации
    participant PAY as payment-svc
    participant REF as reference-svc
    participant WF as workflow-svc
    participant HUB as integration-hub
    participant DBO as ДБО ЮРЛ

    U->>PAY: заявка из файла или формы<br/>(Idempotency-Key)
    PAY->>PAY: сверка суммы с суммой строк
    PAY->>PAY: проверка счетов сотрудников
    PAY->>REF: рабочий ли день исполнения?
    REF-->>PAY: да / нет + ближайший рабочий
    PAY-->>U: Validated, итоги (сумма, строк, комиссия)
    U->>PAY: submit
    PAY->>WF: запуск согласования
    WF-->>PAY: approved
    PAY->>PAY: Approved, формирование PaymentOrder
    PAY->>HUB: RMQ integration.dbo.payment
    HUB->>DBO: платёжное поручение (идемпотентный ключ)
    DBO-->>HUB: принято, внешний номер
    HUB-->>PAY: callback: Sent
    DBO-->>HUB: статусы исполнения построчно
    HUB-->>PAY: callback: Executed / Failed по строкам
    PAY-->>U: статусы по каждому сотруднику
```

**Критично:** после отправки в ДБО ЮРЛ автоматического отката нет. Все проверки
(рабочий день, лимиты, корректность счетов, отсутствие дубля по
`Idempotency-Key`) выполняются до отправки. Повторная отправка возможна только
после явного подтверждённого отказа от ДБО ЮРЛ.

## 8.3. Казначейский суточный цикл

```mermaid
stateDiagram-v2
    [*] --> WaitingRegistry: 00:00 создание цикла на дату
    WaitingRegistry --> RegistryReceived: письмо ДБО ЮРЛ разобрано
    WaitingRegistry --> Alert: 08:00, письма нет
    Alert --> RegistryReceived: письмо поступило
    Alert --> ManualDecision: решение администратора
    RegistryReceived --> MorningInProgress: запрос остатков в EQ
    MorningInProgress --> MorningCompleted: пополнение через WAY4 выполнено
    MorningInProgress --> MorningFailed: ошибка WAY4/EQ
    MorningFailed --> MorningInProgress: повтор
    MorningCompleted --> EveningInProgress: вечерний триггер
    EveningInProgress --> EveningCompleted: остаток списан
    EveningInProgress --> Discrepancy: расхождение сумм
    Discrepancy --> ManualDecision
    ManualDecision --> Closed: разбор завершён
    EveningCompleted --> Closed
    Closed --> [*]
```

Незакрытый цикл блокирует создание цикла следующего дня — это защита от
наложения операций и потери контроля над остатками.

## 8.4. Бесшовный переход из ДБО ЮРЛ

Диаграмма — в [05-integrations.md](05-integrations.md), раздел 5.3.

## 8.5. Расчётные листы

```mermaid
sequenceDiagram
    autonumber
    participant B as Расчётчик банка
    participant FILE as file-svc
    participant SLIP as payslip-svc
    participant PAY as payment-svc
    participant S3 as MinIO
    participant LK as ЛК

    B->>FILE: загрузка данных по шаблону
    FILE-->>SLIP: file.import.completed (importJobId)
    SLIP->>SLIP: создание PayslipBatch
    SLIP->>PAY: формирование платёжных поручений
    SLIP->>SLIP: RMQ payslip.render (по сотруднику)
    SLIP->>S3: HTML расчётных листов
    B->>SLIP: publish
    SLIP->>SLIP: листы доступны по API
    LK->>SLIP: GET /public/payslips?personId=
    SLIP-->>LK: список + HTML
    Note over SLIP: каждый запрос ЛК — в аудит
```

## 8.6. Выдача карт

1. Карты изготовлены → `cards.card.issued`.
2. Администратор создаёт точку выдачи (адрес офиса) или визит (дата, время, адрес).
3. `cards.handover.scheduled` → `notification-svc` → уведомление сотруднику
   организации в интерфейсе (и по email, если включено).
4. Перед визитом формируется `PrintJob`: пакет документов на подпись
   (`document-svc` по шаблонам) → печать.
5. Факт выдачи фиксируется, статус карты → `Handed`, событие в аудит.

## 8.7. Массовая смена продуктов в договорах

1. Администратор задаёт фильтр (организация, продукт-источник, продукт-приёмник, дата).
2. Предпросмотр: сколько договоров и сотрудников затронуто, какие конфликты.
3. Подтверждение → фоновое применение батчами, идемпотентно по `jobId`.
4. Отчёт: применено, пропущено (с причинами), ошибки.
5. Событие `org.product.replaced` → потребители обновляют проекции.

Операция не изменяет уже исполненные заявки и не переоформляет выпущенные карты —
только состав продуктов в договоре, с закрытием прежней связи датой.
