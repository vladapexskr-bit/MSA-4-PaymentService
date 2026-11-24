# Task 5. Набор интеграционных и E2E тест-кейсов

Для обеспечения надежности платёжной системы используется многоуровневое тестирование:
- **Интеграционные тесты:** проверяют взаимодействие Orchestrator Service с БД (PostgreSQL), кэшем (Redis), внешними сервисами (через WireMock) и доменными сервисами.
- **E2E (End-to-End) тесты:** проверяют полный жизненный цикл саги от запроса клиента до финального статуса в БД, включая компенсационные сценарии. Исполняются с помощью Camunda Test Engine и Testcontainers.
- **Архитектурные тесты:** проверяют соблюдение границ доменов с помощью ArchUnit.

## Матрица тест-кейсов

| Название | Тип | Компоненты | Предусловия |
| :--- | :--- | :--- | :--- |
| `Arch_Layer_Boundaries_Check` | Архитектурный | Payment Service, Orchestrator Service | В кодовой базе настроены правила ArchUnit. Payment Service не должен содержать зависимостей от классов Camunda (движок изолирован в Orchestrator Service). |
| `Int_Payment_Debit_Success` | Интеграционный | Orchestrator, Payment Service, PostgreSQL | В БД создан платеж. Настроен WireMock для успешного ответа от платёжного шлюза. Вызов команды `DEBIT_FUNDS` из Camunda. Проверка: статус в БД изменен на `DEBITED`, событие записано в outbox. |
| `Int_Fraud_Timeout_Redis_Cache` | Интеграционный | Orchestrator, FraudCheck, Redis | Redis запущен через Testcontainers. Настроен WireMock с задержкой ответа (таймаут). Вызов `FRAUD_CHECK`. Проверка: срабатывает cut-off, результат по умолчанию `ALLOW` кэшируется в Redis, сага идет дальше. |
| `E2E_Happy_Path_Payment_Completed` | E2E | Orchestrator, Payment, Fraud, Notification, БД | Валидный запрос на оплату. FraudCheck возвращает `ALLOW`. Проверка: Camunda проходит все шаги, деньги списаны и переведены, уведомление отправлено, финальный статус в БД — `COMPLETED`. |
| `E2E_Fraud_Deny_Compensation` | E2E | Orchestrator, Payment, Fraud, Notification, БД | FraudCheck возвращает `DENY`. Проверка: перевод контрагенту не выполняется, запускается компенсация (`REFUND_FUNDS` -> `CANCEL_PAYMENT`), отправляются уведомления клиенту и в безопасность, статус — `CANCELLED`. |
| `E2E_Manual_Review_Approve` | E2E | Orchestrator, Payment, Fraud, Notification | FraudCheck возвращает `MANUAL_REVIEW`. Сага встает на паузу в User Task. Эмулируется внешнее событие `OPERATOR_APPROVE`. Проверка: транзакция продолжается и завершается статусом `COMPLETED`. |
| `E2E_Manual_Review_Reject` | E2E | Orchestrator, Payment, Fraud, Notification | FraudCheck возвращает `MANUAL_REVIEW`. Эмулируется событие `OPERATOR_REJECT`. Проверка: сага уходит в ветку компенсации, деньги возвращаются, статус `CANCELLED`. |
| `E2E_Manual_Review_20min_CutOff` | E2E | Orchestrator, Payment, Camunda Timer | FraudCheck возвращает `MANUAL_REVIEW`. В Camunda Test Engine эмулируется срабатывание таймера 20 минут (без решения оператора). Проверка: транзакция по умолчанию считается разрешенной и переводится контрагенту (статус `COMPLETED`). |
| `E2E_Transfer_Failed_Refund` | E2E | Orchestrator, Payment, WireMock, БД | FraudCheck вернул `ALLOW`. WireMock настроен на возврат ошибки `500` при запросе к платёжному шлюзу на этапе `TRANSFER_TO_COUNTERPARTY`. Проверка: срабатывает Boundary Error Event, запускается `REFUND_FUNDS`, финальный статус `CANCELLED`. |
| `E2E_Refund_Retry_Mechanism` | E2E | Orchestrator, Payment, WireMock | Транзакция отклонена (`DENY`). WireMock настроен так, чтобы первый вызов `REFUND_FUNDS` вернул ошибку, а второй — успех. Проверка: Camunda выполняет retry компенсационного шага, возврат в итоге проходит, статус `CANCELLED`. |
| `Int_Payment_Idempotency_Check` | Интеграционный | Payment Service, PostgreSQL | Отправка двух одинаковых запросов на создание платежа с одним `idempotencyKey` (параллельно или последовательно). Проверка: в БД создана только одна запись, второй запрос возвращает существующий `paymentId`, дублирование списания исключено. |