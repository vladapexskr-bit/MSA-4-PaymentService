# Task 2. Таблица переходов State Machine для платежа OrchestrPay

## Контекст

Состояние платежа — это основа Saga-оркестрации: по нему Camunda восстанавливает процесс
после сбоев, а PostgreSQL хранит актуальный статус для идемпотентности и аудита.
Ниже — реестр состояний, событий и переходов, покрывающий все развилки бизнес-процесса:
успех, отказ antifraud, ручную проверку, cut-off и компенсации.

## Реестр состояний

| Состояние | Тип | Смысл |
|-----------|-----|-------|
| `CREATED` | начальное | Платёж создан, деньги не списаны |
| `DEBIT_PENDING` | промежуточное | Запрос на списание отправлен, ждём подтверждения |
| `DEBITED` | промежуточное | Деньги списаны со счёта клиента |
| `FRAUD_CHECK_PENDING` | промежуточное (ожидание) | Ждём ответа FraudCheck/AML/GeoRisk |
| `MANUAL_REVIEW_PENDING` | промежуточное (ожидание) | Ждём решения оператора (не более 20 мин) |
| `FRAUD_APPROVED` | промежуточное | Проверки разрешили проведение |
| `FRAUD_REJECTED` | промежуточное | Проверки отклонили проведение |
| `TRANSFER_PENDING` | промежуточное | Запрос на перевод контрагенту отправлен |
| `TRANSFERRED` | **pivot** | Деньги у контрагента — точка невозврата |
| `REFUNDING` | промежуточное (компенсация) | Возврат средств клиенту в процессе (с ретраями) |
| `REFUNDED` | промежуточное (компенсация) | Средства возвращены клиенту |
| `COMPLETED` | терминальное | Платёж успешно проведён |
| `CANCELLED` | терминальное | Платёж отменён, деньги возвращены клиенту |
| `FAILED` | терминальное | Платёж не проведён, деньги не двигались |

## Реестр событий

| Событие | Источник |
|---------|----------|
| `PAYMENT_INITIATED` | клиент/маркетплейс |
| `DEBIT_REQUESTED` | Payment Service |
| `DEBIT_SUCCESS` | платёжный шлюз/банк |
| `DEBIT_FAILED` | платёжный шлюз/банк |
| `DEBIT_TIMEOUT` | таймаут вызова шлюза (статус неизвестен) |
| `FRAUD_CHECK_REQUESTED` | Payment Service |
| `FRAUD_ALLOW` | FraudCheck Service |
| `FRAUD_DENY` | FraudCheck Service |
| `FRAUD_MANUAL_REVIEW` | FraudCheck Service |
| `CUTOFF_TIMEOUT` | таймер Camunda (FraudCheck не ответил) |
| `OPERATOR_APPROVE` | оператор ручной проверки |
| `OPERATOR_REJECT` | оператор ручной проверки |
| `MANUAL_REVIEW_TIMEOUT` | таймер Camunda (20 минут) |
| `TRANSFER_REQUESTED` | Payment Service |
| `TRANSFER_SUCCESS` | платёжный шлюз |
| `TRANSFER_FAILED` | платёжный шлюз |
| `COMPENSATION_TRIGGERED` | Saga-оркестратор (после отказа) |
| `REFUND_REQUESTED` | Payment Service |
| `REFUND_SUCCESS` | платёжный шлюз |
| `REFUND_FAILED` | платёжный шлюз (ретрай) |
| `PAYMENT_COMPLETED` | фиксация финального статуса |
| `CANCEL_COMPLETED` | фиксация финального статуса |

## Таблица переходов

| Исходное состояние | Переходное состояние | Событие |
|--------------------|----------------------|---------|
| — (нет) | `CREATED` | `PAYMENT_INITIATED` |
| `CREATED` | `DEBIT_PENDING` | `DEBIT_REQUESTED` |
| `DEBIT_PENDING` | `DEBITED` | `DEBIT_SUCCESS` |
| `DEBIT_PENDING` | `FAILED` | `DEBIT_FAILED` |
| `DEBIT_PENDING` | `REFUNDING` | `DEBIT_TIMEOUT` (страховочный возврат) |
| `DEBITED` | `FRAUD_CHECK_PENDING` | `FRAUD_CHECK_REQUESTED` |
| `FRAUD_CHECK_PENDING` | `FRAUD_APPROVED` | `FRAUD_ALLOW` |
| `FRAUD_CHECK_PENDING` | `FRAUD_REJECTED` | `FRAUD_DENY` |
| `FRAUD_CHECK_PENDING` | `MANUAL_REVIEW_PENDING` | `FRAUD_MANUAL_REVIEW` |
| `FRAUD_CHECK_PENDING` | `FRAUD_APPROVED` | `CUTOFF_TIMEOUT` (default-allow) |
| `MANUAL_REVIEW_PENDING` | `FRAUD_APPROVED` | `OPERATOR_APPROVE` |
| `MANUAL_REVIEW_PENDING` | `FRAUD_REJECTED` | `OPERATOR_REJECT` |
| `MANUAL_REVIEW_PENDING` | `FRAUD_APPROVED` | `MANUAL_REVIEW_TIMEOUT` (cut-off, default-allow) |
| `FRAUD_APPROVED` | `TRANSFER_PENDING` | `TRANSFER_REQUESTED` |
| `TRANSFER_PENDING` | `TRANSFERRED` | `TRANSFER_SUCCESS` |
| `TRANSFER_PENDING` | `REFUNDING` | `TRANSFER_FAILED` |
| `TRANSFERRED` | `COMPLETED` | `PAYMENT_COMPLETED` |
| `FRAUD_REJECTED` | `REFUNDING` | `COMPENSATION_TRIGGERED` |
| `REFUNDING` | `REFUNDING` | `REFUND_FAILED` (self-loop, retry) |
| `REFUNDING` | `REFUNDED` | `REFUND_SUCCESS` |
| `REFUNDED` | `CANCELLED` | `CANCEL_COMPLETED` |
