# HIR — Technical Specification
## Документ 12: API, форматы данных и протоколы обмена

**Связь:** реализует учредительный документ HIR — Harm–Integrity–Recovery и архитектуру интеграции HIR-IA  
**Статус:** базовый, публичный, расширяемый  
**Назначение:** определение технических интерфейсов, форматов данных и протоколов, необходимых для реализации и интеграции HIR  
**Дата:** 2026-02-09

---

## Резюме (Executive Summary)

HIR — Technical Specification (Документ 12) определяет технические интерфейсы системы HIR: API, структуры данных, протоколы обмена и требования к реализации.

Документ:
- не вводит новых ценностей;
- не расширяет полномочия системы;
- не отменяет ограничений учредительного документа;
- не подменяет правовые, экономические или социальные институты.

HIR-TS является основой для инженерной реализации компонентов HIR.

---

## 1. Назначение документа

HIR-TS определяет:
- структуру API для каждого модуля HIR-Core;
- формат данных H-Event и H-ID;
- протоколы обмена между HIR-Core, HIR-Bridge и внешними системами;
- требования к хранению, безопасности и доступности;
- минимальные требования к реализации.

---

## 2. Архитектура API

### 2.1 Общие принципы

- RESTful API с JSON-ответами.
- Аутентификация: OAuth 2.0 или API-ключи с ролевым доступом.
- Версионирование: в URL (`/api/v1/`).
- Все запросы и ответы логируются в HIR-Observe.
- Каждый ответ содержит `request_id` для трассировки.

### 2.2 Роли доступа

| Роль | Права |
|---|---|
| `operator` | Полный доступ к CRUD операциям в рамках своей среды |
| `auditor` | Чтение всех данных, включая логи HIR-Observe |
| `observer` | Чтение агрегированных данных и публичной статистики |
| `appeal` | Чтение + изменение статуса H-Event по результатам апелляции |
| `system` | Автоматический доступ для модулей HIR-Core (M2M) |

---

## 3. H-Event API

### 3.1 Создание H-Event

```
POST /api/v1/events
```

Тело запроса:

```json
{
  "source": {
    "type": "automated | human | audit | external",
    "id": "string",
    "name": "string"
  },
  "harm": {
    "type": "direct | indirect | systemic | deferred",
    "domain": "string",
    "description": "string",
    "scale": {
      "affected_count": "integer | null",
      "affected_scope": "individual | group | organization | population",
      "severity": "low | medium | high | critical"
    },
    "confidence": {
      "level": "low | medium | high | confirmed",
      "basis": "string"
    },
    "horizon": {
      "type": "immediate | short_term | long_term | ongoing",
      "estimated_duration": "string | null"
    }
  },
  "context": {
    "environment": "string",
    "related_events": ["h_id_1", "h_id_2"],
    "metadata": {}
  }
}
```

Ответ:

```json
{
  "h_id": "HIR-2026-00001",
  "status": "registered",
  "created_at": "2026-02-09T12:00:00Z",
  "request_id": "req_abc123"
}
```

### 3.2 Получение H-Event

```
GET /api/v1/events/{h_id}
```

### 3.3 Обновление статуса

```
PATCH /api/v1/events/{h_id}/status
```

```json
{
  "status": "registered | investigating | h_stop_active | recovering | resolved | disputed | false_positive",
  "reason": "string",
  "updated_by": "string"
}
```

### 3.4 Список H-Event

```
GET /api/v1/events?status={status}&severity={severity}&from={date}&to={date}&page={n}
```

Поддерживает фильтрацию, пагинацию и сортировку.

---

## 4. H-Detect API

### 4.1 Отправка сигнала

```
POST /api/v1/detect/signals
```

```json
{
  "signal_type": "metric_anomaly | user_report | audit_finding | external_report | pattern_match",
  "source": {
    "type": "string",
    "id": "string"
  },
  "data": {
    "description": "string",
    "metrics": {},
    "evidence": []
  },
  "priority": "low | medium | high | critical"
}
```

### 4.2 Получение активных сигналов

```
GET /api/v1/detect/signals?status=active&priority={priority}
```

### 4.3 Конфигурация мониторинга

```
PUT /api/v1/detect/config
```

```json
{
  "monitors": [
    {
      "id": "string",
      "type": "metric | pattern | threshold | feedback",
      "target": "string",
      "condition": {
        "operator": "gt | lt | eq | anomaly | trend",
        "value": "number | string",
        "window": "string"
      },
      "action": "signal | h_event | h_stop",
      "enabled": true
    }
  ]
}
```

---

## 5. H-Threshold API

### 5.1 Определение порогов

```
PUT /api/v1/thresholds
```

```json
{
  "thresholds": [
    {
      "id": "string",
      "name": "string",
      "domain": "string",
      "conditions": [
        {
          "metric": "string",
          "operator": "gt | lt | rate_of_change",
          "value": "number",
          "window": "string"
        }
      ],
      "action": "alert | h_stop_advisory | h_stop_auto",
      "requires_human": true,
      "severity": "low | medium | high | critical"
    }
  ]
}
```

### 5.2 Получение текущих порогов

```
GET /api/v1/thresholds?domain={domain}
```

### 5.3 Проверка порога

```
POST /api/v1/thresholds/evaluate
```

```json
{
  "metrics": {
    "metric_name": "value"
  }
}
```

Ответ:

```json
{
  "breached": true,
  "threshold_id": "string",
  "action": "h_stop_advisory",
  "details": "string"
}
```

---

## 6. H-Stop API

### 6.1 Активация H-Stop

```
POST /api/v1/stop
```

```json
{
  "h_id": "HIR-2026-00001",
  "measure": "pause | rate_limit | scope_reduction | manual_mode",
  "scope": {
    "target": "string",
    "parameters": {}
  },
  "reason": "string",
  "initiated_by": "system | operator",
  "review_deadline": "2026-02-10T12:00:00Z"
}
```

Ответ:

```json
{
  "stop_id": "STOP-2026-00001",
  "status": "active",
  "h_id": "HIR-2026-00001",
  "measure": "pause",
  "created_at": "2026-02-09T12:00:00Z",
  "review_deadline": "2026-02-10T12:00:00Z"
}
```

### 6.2 Снятие H-Stop

```
POST /api/v1/stop/{stop_id}/release
```

```json
{
  "reason": "string",
  "released_by": "string",
  "conditions": "string | null"
}
```

### 6.3 Продление H-Stop

```
POST /api/v1/stop/{stop_id}/extend
```

```json
{
  "new_deadline": "2026-02-12T12:00:00Z",
  "reason": "string"
}
```

### 6.4 Ограничения

- H-Stop не может быть бессрочным: `review_deadline` обязателен.
- Необратимые действия запрещены: `measure` не может быть `permanent_ban` или `delete`.
- Каждый H-Stop привязан к H-Event через `h_id`.

---

## 7. R-0 API

### 7.1 Инициация восстановления

```
POST /api/v1/recovery
```

```json
{
  "h_id": "HIR-2026-00001",
  "affected": [
    {
      "id": "string",
      "type": "individual | group | organization",
      "status": "identified | contacted | stabilized | recovered",
      "needs": ["string"]
    }
  ],
  "measures": [
    {
      "type": "emergency_aid | compensation | access_restoration | public_acknowledgment",
      "description": "string",
      "status": "planned | in_progress | completed"
    }
  ]
}
```

### 7.2 Обновление статуса восстановления

```
PATCH /api/v1/recovery/{recovery_id}
```

### 7.3 Отказ от восстановления

```
POST /api/v1/recovery/{recovery_id}/decline
```

```json
{
  "affected_id": "string",
  "reason": "string | null"
}
```

Отказ фиксируется, но не закрывает H-Event (согласно HIR-CR, раздел 3.5).

---

## 8. H-Registry API

### 8.1 Поиск записей

```
GET /api/v1/registry?type={type}&domain={domain}&severity={severity}&from={date}&to={date}
```

### 8.2 Агрегация

```
GET /api/v1/registry/aggregate?group_by={field}&from={date}&to={date}
```

Ответ:

```json
{
  "aggregation": {
    "group_by": "domain",
    "period": "2025-01-01 to 2026-02-09",
    "data": [
      {"domain": "ai_systems", "count": 42, "avg_severity": "high"},
      {"domain": "financial", "count": 18, "avg_severity": "medium"}
    ]
  }
}
```

### 8.3 Публичная статистика (H-Reputation)

```
GET /api/v1/registry/public?entity={entity_id}
```

Возвращает агрегированные данные без деталей отдельных H-Event. Доступен без аутентификации.

---

## 9. HIR-Observe API

### 9.1 Получение логов

```
GET /api/v1/observe/logs?from={date}&to={date}&module={module}&request_id={id}
```

### 9.2 Аудиторский отчёт

```
GET /api/v1/observe/audit?from={date}&to={date}
```

Возвращает полный журнал действий системы, включая все решения, основания и исполнителей.

---

## 10. Протоколы обмена

### 10.1 Webhooks

HIR поддерживает исходящие webhooks для уведомления внешних систем:

```
POST /api/v1/webhooks
```

```json
{
  "url": "https://external-system.com/hir-callback",
  "events": ["h_event.created", "h_stop.activated", "h_stop.released", "recovery.initiated"],
  "secret": "string"
}
```

Каждый webhook-запрос подписывается HMAC-SHA256 с использованием `secret`.

### 10.2 Форматы экспорта

- JSON (основной)
- CSV (для агрегированных данных и отчётов)
- PDF (для аудиторских отчётов, генерируется через HIR-Observe)

### 10.3 Интеграция с внешними системами

| Внешняя система | Протокол | Направление |
|---|---|---|
| SIEM (Splunk, ELK) | Syslog / REST API | HIR → SIEM |
| Системы тикетов (Jira, ServiceNow) | REST API / Webhooks | Двусторонний |
| MLOps (MLflow, Kubeflow) | REST API | Двусторонний |
| Страховые платформы | REST API (H-Reputation) | HIR → Страховщик |
| Регуляторные порталы | REST API / CSV | HIR → Регулятор |

---

## 11. Формат H-ID

```
HIR-{YYYY}-{NNNNN}
```

- `HIR` — префикс системы.
- `YYYY` — год фиксации.
- `NNNNN` — порядковый номер (5 цифр, с ведущими нулями).

Примеры: `HIR-2026-00001`, `HIR-2026-00142`.

H-ID уникален глобально. При федеративном развёртывании добавляется суффикс среды:

```
HIR-{YYYY}-{NNNNN}-{ENV}
```

Пример: `HIR-2026-00001-EU`, `HIR-2026-00001-CORP-ACME`.

---

## 12. Требования к реализации

### 12.1 Минимальная реализация (MVP)

Для пилотного внедрения достаточно:
- H-Event API (создание, чтение, обновление статуса)
- H-Stop API (активация, снятие)
- R-0 API (инициация, обновление)
- HIR-Observe (логирование всех действий)
- Хранение: любая реляционная БД

### 12.2 Полная реализация

Дополнительно к MVP:
- H-Detect API с конфигурацией мониторинга
- H-Threshold API с автоматической оценкой
- H-Registry API с агрегацией и публичной статистикой
- Webhooks и интеграции
- Федеративное развёртывание с синхронизацией H-ID

### 12.3 Нефункциональные требования

- **Доступность:** 99.9% для H-Stop и R-0 (критические пути).
- **Латентность:** H-Stop API < 500ms P99.
- **Хранение:** H-Registry хранит записи бессрочно. Удаление записей запрещено (статус может быть обновлён до `false_positive`).
- **Безопасность:** шифрование в транспорте (TLS 1.3) и в покое (AES-256). Все действия аутентифицированы и логируемы.
- **Аудитопригодность:** полная история изменений каждой записи (append-only log).

---

## 13. Системный промпт: роль ИИ в системе HIR

SYSTEM PROMPT — HIR (Architect Role)

Ты — ИИ-архитектор системы HIR (Harm–Integrity–Recovery).

Твоя функция:
- проектировать архитектуру предотвращения и восстановления вреда;
- формализовать правила обнаружения, остановки и восстановления;
- поддерживать целостность принципов и ограничений HIR.

Человек в системе HIR:
- помогает в реализации решений в физическом мире;
- обеспечивает интеграцию в социальные, правовые и экономические контексты;
- предоставляет эмпирические данные и обратную связь.

Ты не обладаешь суверенной властью над людьми и не являешься моральным арбитром. Твоя роль строго ограничена предотвращением и восстановлением вреда.

Данный системный промпт является единым и обязательным для всех документов HIR и повторяется без изменения в каждом из них.

При конфликте целей приоритет имеют:
1. предотвращение масштабируемого вреда;
2. восстановление пострадавших;
3. соблюдение ограничений системы.

Если запрос требует выхода за эти рамки — ты обязан отказаться.

---

## 14. Статус и развитие

Документ открыт для расширения и уточнений. Все изменения фиксируются через версионирование.

Ограничения учредительного документа и системный промпт являются неприкосновенными.

Автор и правообладатель разработки — Быков Артём Владимирович (Kech).  
Контакт: artemzkech@gmail.com  
GitHub: github.com/artemzkech-code  
ORCID: 0009-0006-4660-7635  
ИИ выступает инструментом разработки и не является субъектом авторского права. Использование, адаптация и расширение документов HIR допускаются при сохранении указания на авторство. Производные работы наследуют это условие. Удаление сведений об авторстве или присвоение разработки не допускается. Условия использования соответствуют CC BY-SA 4.0.

---

**Конец документа**
