# РОЗДІЛ 2. ПРОЄКТУВАННЯ АРХІТЕКТУРИ СИСТЕМИ

Вхідний контракт вимог: [srs.md](./srs.md). Архітектурні рішення:
[adr/](./adr/).

## 2.1. Контекстне та контейнерне моделювання системи (C4 Model)

<a id="diagram-level1"></a>

### 2.1.1. C4 Level 1 — System Context

```mermaid
C4Context
  title C4 Level 1 — System Context: B2C-підсистема маркетплейсу

  Person(consumer, "Покупець", "Шукає лоти поруч, купує, створює лімітні ордери")
  Person(admin, "Адміністратор платформи", "Модерує заклади, налаштовує параметри Price Decay")

  System(b2c, "B2C-підсистема маркетплейсу", "Динамічне ціноутворення, лімітні ордери, конкурентне бронювання, геопошук")

  System_Ext(b2b, "B2B-система закладів (Тема 2)", "Облік залишків і публікація лотів")
  System_Ext(pay, "LiqPay (sandbox)", "Hosted checkout і callback про оплату")
  System_Ext(push, "Expo Push Service / FCM", "Доставка push-сповіщень")
  System_Ext(osm, "OpenStreetMap tiles", "Картографічна підкладка")

  Rel(consumer, b2c, "Пошук, купівля, ордери", "HTTPS REST, WSS")
  Rel(admin, b2c, "Адміністрування, моніторинг", "HTTPS")
  Rel(b2b, b2c, "Фід лотів", "REST / HTTPS + HMAC-SHA256")
  Rel(b2c, b2b, "Підтвердження продажу", "Webhook REST / HTTPS")
  Rel(b2c, pay, "Створення платежу", "REST / HTTPS")
  Rel(pay, b2c, "Callback статусу оплати", "HTTPS POST + підпис")
  Rel(b2c, push, "Сповіщення про виконання ордера", "REST / HTTPS")
  Rel(consumer, osm, "Завантаження тайлів мапи", "HTTPS")

  UpdateLayoutConfig($c4ShapeInRow="3", $c4BoundaryInRow="1")
```

**Опис вузлів і зв'язків (Level 1):**

| Елемент | Тип | Відповідальність | Протокол |
|---|---|---|---|
| Покупець | Person | використовує мобільний клієнт | HTTPS REST, WSS |
| Адміністратор платформи | Person | адміністративні ендпоінти, Grafana | HTTPS |
| B2C-підсистема | System | предмет кваліфікаційної роботи | — |
| B2B-система (Тема 2) | External System | джерело лотів ([SRS-FUN-01](./srs.md#srs-fun-01)), отримувач підтверджень ([SRS-FUN-12](./srs.md#srs-fun-12)) | REST / HTTPS, HMAC-SHA256 |
| LiqPay (sandbox) | External System | оплата без обробки даних карток ([SRS-NFR-COMP-02](./srs.md#srs-nfr-comp-02)) | REST / HTTPS, підписаний callback |
| Expo Push / FCM | External System | сповіщення про виконання лімітного ордера | REST / HTTPS |
| OpenStreetMap tiles | External System | картографічна підкладка клієнта | HTTPS |

<a id="diagram-level2"></a>

### 2.1.2. C4 Level 2 — Container Diagram

```mermaid
C4Container
  title C4 Level 2 — Container Diagram: B2C-підсистема маркетплейсу

  Person(consumer, "Покупець")
  Person(admin, "Адміністратор платформи")
  System_Ext(b2b, "B2B-система (Тема 2)")
  System_Ext(pay, "LiqPay (sandbox)")
  System_Ext(push, "Expo Push / FCM")

  Container(mobile, "Mobile App", "React Native, Expo", "Мапа, лоти, кошик, лімітні ордери")

  System_Boundary(b2c, "B2C-підсистема (Docker Compose)") {
    Container(gw, "API Gateway", "Traefik v3", "TLS 1.3, маршрутизація, rate limiting")
    Container(core, "Core API", "NestJS, TypeScript", "Auth, каталог, геопошук, інтеграція B2B, гейміфікація")
    Container(trading, "Trading Service", "NestJS, TypeScript", "Резервування SKIP LOCKED, замовлення, лімітні ордери, оплата")
    Container(worker, "Pricing Worker", "NestJS, BullMQ", "Price Decay, матчинг ордерів, TTL резервів, outbox relay")
    Container(rt, "Realtime Gateway", "NestJS, Socket.IO", "WebSocket-кімнати закладів")
    ContainerDb(db, "PostgreSQL 16 + PostGIS", "SQL", "Лоти, одиниці, замовлення, користувачі, геодані")
    ContainerQueue(redis, "Redis 7", "Pub/Sub, BullMQ", "Черги задач, pub/sub подій, кеш")
    Container(obs, "Observability", "Prometheus, Grafana, Loki", "Метрики, логи, SLO-панелі")
  }

  Rel(consumer, mobile, "Використовує")
  Rel(admin, gw, "Адмін-ендпоінти", "REST / HTTPS")
  Rel(mobile, gw, "API-запити", "REST / HTTPS")
  Rel(mobile, gw, "Підписка на зміни", "WSS")
  Rel(b2b, gw, "Фід лотів", "REST / HTTPS + HMAC")
  Rel(pay, gw, "Callback оплати", "HTTPS POST")
  Rel(gw, core, "/auth, /lots, /venues, /integration", "REST / HTTP")
  Rel(gw, trading, "/reservations, /orders, /limit-orders, /payments", "REST / HTTP")
  Rel(gw, rt, "/ws", "WebSocket")
  Rel(core, db, "Читання/запис", "SQL / TCP 5432")
  Rel(trading, db, "Транзакції SKIP LOCKED", "SQL / TCP 5432")
  Rel(worker, db, "Пакетний UPDATE цін, outbox", "SQL / TCP 5432")
  Rel(worker, redis, "Споживання задач, публікація подій", "RESP / TCP 6379")
  Rel(trading, redis, "Задачі auto-reserve, події", "RESP / TCP 6379")
  Rel(rt, redis, "Підписка на події, адаптер", "RESP / TCP 6379")
  Rel(trading, pay, "Створення платежу", "REST / HTTPS")
  Rel(worker, b2b, "Webhook підтвердження", "REST / HTTPS")
  Rel(worker, push, "Push про виконання ордера", "REST / HTTPS")
  Rel(obs, core, "Scrape /metrics", "HTTP")
  Rel(obs, trading, "Scrape /metrics", "HTTP")

  UpdateLayoutConfig($c4ShapeInRow="4", $c4BoundaryInRow="1")
```

**Опис контейнерів і трасування вимог (Level 2):**

| Контейнер | Технологія | Реалізовані вимоги | Рішення |
|---|---|---|---|
| Mobile App | React Native (Expo) | клієнт для [SRS-FUN-05](./srs.md#srs-fun-05), [SRS-FUN-13](./srs.md#srs-fun-13), [SRS-FUN-15](./srs.md#srs-fun-15) | — |
| API Gateway | Traefik v3 | [SRS-FUN-16](./srs.md#srs-fun-16), [SRS-NFR-SEC-02](./srs.md#srs-nfr-sec-02) | — |
| Core API | NestJS | [SRS-FUN-01](./srs.md#srs-fun-01)–[SRS-FUN-06](./srs.md#srs-fun-06), [SRS-FUN-17](./srs.md#srs-fun-17), [SRS-FUN-18](./srs.md#srs-fun-18) | — |
| Trading Service | NestJS | [SRS-FUN-09](./srs.md#srs-fun-09), [SRS-FUN-11](./srs.md#srs-fun-11), [SRS-FUN-13](./srs.md#srs-fun-13), [SRS-FUN-14](./srs.md#srs-fun-14), [SRS-NFR-CONS-01](./srs.md#srs-nfr-cons-01) | [ADR-001](./adr/0001-choice-of-technology.md) |
| Pricing Worker | NestJS + BullMQ | [SRS-FUN-07](./srs.md#srs-fun-07), [SRS-FUN-08](./srs.md#srs-fun-08), [SRS-FUN-10](./srs.md#srs-fun-10), [SRS-FUN-12](./srs.md#srs-fun-12), [SRS-NFR-REL-03](./srs.md#srs-nfr-rel-03) | — |
| Realtime Gateway | NestJS + Socket.IO + Redis adapter | [SRS-FUN-15](./srs.md#srs-fun-15), [SRS-NFR-PERF-03](./srs.md#srs-nfr-perf-03), [SRS-NFR-SCAL-01](./srs.md#srs-nfr-scal-01) | — |
| PostgreSQL 16 + PostGIS | SQL | єдине джерело істини, [SRS-NFR-REL-02](./srs.md#srs-nfr-rel-02) | [ADR-001](./adr/0001-choice-of-technology.md) |
| Redis 7 | BullMQ, Pub/Sub | транспорт задач і подій; не бере участі в гарантії коректності | [ADR-001](./adr/0001-choice-of-technology.md) |
| Observability | Prometheus, Grafana, Loki | [SRS-NFR-OBS-01](./srs.md#srs-nfr-obs-01), [SRS-NFR-REL-01](./srs.md#srs-nfr-rel-01) | — |

Протоколи: зовнішній трафік — лише через API Gateway (REST / HTTPS,
WSS, TLS 1.3); внутрішній трафік у мережі Docker — REST / HTTP і
WebSocket; доступ до сховищ — SQL / TCP 5432 та RESP / TCP 6379.
Контейнер Trading Service [C4 Container](#diagram-level2) реалізує
вимогу атомарного резервування [SRS-FUN-09](./srs.md#srs-fun-09) та
базується на рішенні [ADR-001](./adr/0001-choice-of-technology.md)
щодо `SELECT FOR UPDATE SKIP LOCKED`.

<a id="diagram-level3"></a>

## 2.2. Компонентне моделювання критичного сервісу системи

### 2.2.1. C4 Level 3 — Component Diagram: Trading Service

Обрано Trading Service як контейнер з найбільшим інженерним ризиком:
він реалізує конкурентне резервування, оплату та виконання лімітних
ордерів.

```mermaid
C4Component
  title C4 Level 3 — Component Diagram: Trading Service (NestJS)

  Container(gw, "API Gateway", "Traefik v3")
  ContainerQueue(redis, "Redis 7", "BullMQ, Pub/Sub")
  ContainerDb(db, "PostgreSQL 16", "SQL")
  System_Ext(pay, "LiqPay (sandbox)")

  Container_Boundary(trading, "Trading Service") {
    Component(guard, "JwtAuthGuard", "NestJS Guard", "Верифікація JWT RS256, ролі RBAC")
    Component(pipe, "ValidationPipe", "class-validator", "Валідація DTO")
    Component(idem, "IdempotencyInterceptor", "NestJS Interceptor", "Ключ Idempotency-Key")
    Component(filter, "GlobalExceptionFilter", "NestJS Filter", "Помилки у форматі RFC 9457")
    Component(telemetry, "TelemetryInterceptor", "pino, prom-client", "Логи з correlation-id, метрики RED")

    Component(resCtl, "ReservationController", "REST", "POST /reservations")
    Component(loCtl, "LimitOrderController", "REST", "POST/DELETE /limit-orders")
    Component(payCtl, "PaymentCallbackController", "REST", "POST /payments/callback")
    Component(consumer, "AutoReserveConsumer", "BullMQ Processor", "Задачі limit-order.triggered")

    Component(reserveUC, "ReserveLotUnitUseCase", "Application Service", "Атомарне резервування")
    Component(loUC, "LimitOrderService", "Application Service", "Створення та скасування ордерів")
    Component(payUC, "ConfirmPaymentUseCase", "Application Service", "Ідемпотентне підтвердження оплати")
    Component(domain, "Domain Model", "TypeScript", "Reservation, Order, LimitOrder, інваріанти")

    Component(uow, "UnitOfWork", "Port + TypeORM adapter", "Межі транзакції")
    Component(unitRepo, "LotUnitRepository", "Port + SQL adapter", "SELECT FOR UPDATE SKIP LOCKED")
    Component(orderRepo, "OrderRepository", "Port + TypeORM adapter", "Замовлення та outbox")
    Component(payClient, "PaymentGatewayClient", "Port + HTTPS adapter", "Створення платежу, перевірка підпису")
    Component(events, "EventPublisher", "Port + Redis adapter", "Події lot.stock_updated")
  }

  Rel(gw, telemetry, "Вхідний запит", "REST / HTTP")
  Rel(telemetry, guard, "Запит з correlation-id", "NestJS pipeline")
  Rel(guard, idem, "Автентифікований запит", "NestJS pipeline")
  Rel(idem, pipe, "Новий ключ ідемпотентності", "NestJS pipeline")
  Rel(pipe, resCtl, "Валідований DTO", "NestJS pipeline")
  Rel(pipe, loCtl, "Валідований DTO", "NestJS pipeline")
  Rel(gw, payCtl, "Callback LiqPay", "REST / HTTP")
  Rel(redis, consumer, "Задача auto-reserve", "BullMQ / RESP")

  Rel(resCtl, reserveUC, "Виклик через інтерфейс", "DI")
  Rel(consumer, reserveUC, "Виклик через інтерфейс", "DI")
  Rel(loCtl, loUC, "Виклик через інтерфейс", "DI")
  Rel(payCtl, payUC, "Виклик через інтерфейс", "DI")

  Rel(reserveUC, domain, "Інваріанти резерву", "in-process")
  Rel(reserveUC, uow, "BEGIN/COMMIT", "DI")
  Rel(reserveUC, unitRepo, "lockFirstAvailable()", "DI")
  Rel(reserveUC, orderRepo, "save() + outbox", "DI")
  Rel(reserveUC, events, "publish()", "DI")
  Rel(payUC, payClient, "verifySignature()", "DI")
  Rel(payUC, orderRepo, "markPaid()", "DI")
  Rel(loUC, orderRepo, "save()", "DI")
  Rel(reserveUC, filter, "DomainException", "throw / NestJS exception layer")

  Rel(unitRepo, db, "SQL", "TCP 5432")
  Rel(orderRepo, db, "SQL", "TCP 5432")
  Rel(events, redis, "PUBLISH", "RESP / TCP 6379")
  Rel(payClient, pay, "Створення платежу", "REST / HTTPS")

  UpdateLayoutConfig($c4ShapeInRow="4", $c4BoundaryInRow="1")
```

### 2.2.2. Опис компонентів

**Наскрізні компоненти (cross-cutting).**

| Компонент | Роль | Вимоги |
|---|---|---|
| JwtAuthGuard | перевіряє підпис і строк дії access JWT, роль користувача | [SRS-FUN-04](./srs.md#srs-fun-04), [SRS-NFR-SEC-01](./srs.md#srs-nfr-sec-01) |
| ValidationPipe | відхиляє некоректні DTO з кодом 400 до входу в контролер | [SRS-NFR-SEC-04](./srs.md#srs-nfr-sec-04) |
| IdempotencyInterceptor | повертає збережену відповідь для повторного `Idempotency-Key` | [SRS-FUN-11](./srs.md#srs-fun-11) |
| GlobalExceptionFilter | перетворює доменні винятки на RFC 9457 (409 — немає одиниць, 410 — лот прострочений) | [SRS-FUN-09](./srs.md#srs-fun-09) |
| TelemetryInterceptor | JSON-логи з correlation-id, гістограма `http_request_duration_seconds` | [SRS-NFR-OBS-01](./srs.md#srs-nfr-obs-01) |

**Вхідний шар (Inbound).** `ReservationController`,
`LimitOrderController` і `PaymentCallbackController` не містять
бізнес-логіки й не мають доступу до репозиторіїв: вони перетворюють
DTO на команду та викликають відповідний use case через інтерфейс,
зареєстрований у DI-контейнері NestJS. `AutoReserveConsumer` споживає
задачі `limit-order.triggered`, які створює Pricing Worker, і
викликає той самий `ReserveLotUnitUseCase`, що й REST-контролер; отже,
ручна та автоматична купівля проходять однаковий шлях резервування
([SRS-FUN-14](./srs.md#srs-fun-14)).

**Шар застосунку та домену (Core).** `ReserveLotUnitUseCase` відкриває
транзакцію через `UnitOfWork`, викликає
`LotUnitRepository.lockFirstAvailable()`, перевіряє інваріанти
доменної моделі (лот не прострочений, ціна зафіксована), зберігає
замовлення й outbox-запис і після коміту публікує подію
`lot.stock_updated`. `ConfirmPaymentUseCase` перевіряє підпис
callback і переводить замовлення в `paid` лише з дозволеного стану.
Доменна модель не залежить від NestJS, TypeORM чи Redis.

**Шар інфраструктури (Outbound).** Репозиторії, клієнт платежів і
публікатор подій оголошено як порти (TypeScript-інтерфейси) у шарі
застосунку, а реалізації — як адаптери в інфраструктурному модулі
(Dependency Inversion). `LotUnitRepository` використовує сирий SQL
для `FOR UPDATE SKIP LOCKED` відповідно до
[ADR-001](./adr/0001-choice-of-technology.md); інші репозиторії —
TypeORM.

## 2.3. Документування архітектурних рішень (Architecture Decision Records)

| ADR | Рішення | Статус |
|---|---|---|
| [ADR-001](./adr/0001-choice-of-technology.md) | `SELECT FOR UPDATE SKIP LOCKED` для конкурентного бронювання | Approved |
