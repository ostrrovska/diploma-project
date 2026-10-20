# РОЗДІЛ 2. ПРОЄКТУВАННЯ АРХІТЕКТУРИ СИСТЕМИ

Вхідний контракт вимог: [srs.md](./srs.md). Архітектурні рішення:
[adr/](./adr/).

## 2.1. Контекстне та контейнерне моделювання системи (C4 Model)

Діаграми побудовано в нотації C4 (Simon Brown) засобами Mermaid
`flowchart`: тип елемента вказано стереотипом у підписі ([Person],
[Software System], [External System], [Container: …],
[Component: …]), кольори відповідають палітрі C4, кожен зв'язок
підписано дією та протоколом у квадратних дужках. Вбудований
синтаксис `C4Context`/`C4Container` у Mermaid не розводить підписи
зв'язків і дає накладання тексту, тому для читабельності обрано
`flowchart` з автоматичною розкладкою.

<a id="diagram-level1"></a>

### 2.1.1. C4 Level 1 — System Context

```mermaid
%%{init: {"flowchart": {"nodeSpacing": 70, "rankSpacing": 80, "wrappingWidth": 240, "padding": 12}}}%%
flowchart TB
  consumer["<b>Покупець</b><br/>[Person]<br/>Шукає лоти поруч, купує,<br/>створює лімітні ордери"]
  admin["<b>Адміністратор платформи</b><br/>[Person]<br/>Модерує заклади, налаштовує<br/>параметри Price Decay"]

  b2c["<b>B2C-підсистема маркетплейсу</b><br/>[Software System]<br/>Динамічне ціноутворення, лімітні ордери,<br/>конкурентне бронювання, геопошук"]

  b2b["<b>B2B-система закладів (Тема 2)</b><br/>[External System]<br/>Облік залишків і публікація лотів"]
  pay["<b>LiqPay (sandbox)</b><br/>[External System]<br/>Hosted checkout і callback оплати"]
  push["<b>Expo Push Service / FCM</b><br/>[External System]<br/>Доставка push-сповіщень"]
  osm["<b>OpenStreetMap tiles</b><br/>[External System]<br/>Картографічна підкладка"]

  consumer -- "Пошук, купівля, ордери<br/>[HTTPS REST, WSS]" --> b2c
  admin -- "Адміністрування, моніторинг<br/>[HTTPS]" --> b2c
  consumer -- "Фрагменти мапи (tiles)<br/>[HTTPS]" --> osm
  b2b -- "Потік лотів<br/>[REST / HTTPS + HMAC-SHA256]" --> b2c
  b2c -- "Підтвердження продажу<br/>[Webhook REST / HTTPS]" --> b2b
  b2c -- "Створення платежу<br/>[REST / HTTPS]" --> pay
  pay -- "Callback статусу оплати<br/>[HTTPS POST + підпис]" --> b2c
  b2c -- "Сповіщення про виконання ордера<br/>[REST / HTTPS]" --> push

  classDef person fill:#08427b,stroke:#052e56,color:#ffffff
  classDef system fill:#1168bd,stroke:#0b4884,color:#ffffff
  classDef external fill:#8a8a8a,stroke:#5e5e5e,color:#ffffff
  class consumer,admin person
  class b2c system
  class b2b,pay,push,osm external
```

**Опис вузлів і зв'язків (Level 1):**

| Елемент | Тип | Відповідальність | Протокол |
|---|---|---|---|
| Покупець | Person | використовує мобільний клієнт | HTTPS REST, WSS |
| Адміністратор платформи | Person | адміністративні методи REST API, Grafana | HTTPS |
| B2C-підсистема | System | предмет кваліфікаційної роботи | — |
| B2B-система (Тема 2) | External System | джерело лотів ([SRS-FUN-01](./srs.md#srs-fun-01)), отримувач підтверджень ([SRS-FUN-12](./srs.md#srs-fun-12)) | REST / HTTPS, HMAC-SHA256 |
| LiqPay (sandbox) | External System | оплата без обробки даних карток ([SRS-NFR-COMP-02](./srs.md#srs-nfr-comp-02)) | REST / HTTPS, підписаний callback |
| Expo Push / FCM | External System | сповіщення про виконання лімітного ордера | REST / HTTPS |
| OpenStreetMap tiles | External System | картографічна підкладка клієнта | HTTPS |

<a id="diagram-level2"></a>

### 2.1.2. C4 Level 2 — Container Diagram

```mermaid
%%{init: {"flowchart": {"nodeSpacing": 70, "rankSpacing": 80, "wrappingWidth": 240, "padding": 12}}}%%
flowchart TB
  consumer["<b>Покупець</b><br/>[Person]"]
  admin["<b>Адміністратор платформи</b><br/>[Person]"]
  b2b["<b>B2B-система (Тема 2)</b><br/>[External System]"]
  pay["<b>LiqPay (sandbox)</b><br/>[External System]"]
  push["<b>Expo Push / FCM</b><br/>[External System]"]

  subgraph b2c["B2C-підсистема [Software System]"]
    mobile["<b>Mobile App</b><br/>[Container: React Native, Expo]<br/>Мапа, лоти, кошик, лімітні ордери"]
    gw["<b>API Gateway</b><br/>[Container: Traefik v3]<br/>TLS 1.3, маршрутизація, rate limiting"]
    core["<b>Core API</b><br/>[Container: NestJS]<br/>Auth, каталог, геопошук,<br/>інтеграція B2B, гейміфікація"]
    trading["<b>Trading Service</b><br/>[Container: NestJS]<br/>Резервування SKIP LOCKED,<br/>замовлення, лімітні ордери, оплата"]
    rt["<b>Realtime Gateway</b><br/>[Container: NestJS, Socket.IO]<br/>WebSocket-кімнати закладів"]
    worker["<b>Pricing Worker</b><br/>[Container: NestJS, BullMQ]<br/>Price Decay, зіставлення ордерів<br/>із ціною, TTL резервів, outbox relay"]
    db[("<b>PostgreSQL 16 + PostGIS</b><br/>[Container: SQL Database]<br/>Лоти, одиниці, замовлення,<br/>користувачі, геодані")]
    redis[("<b>Redis 7</b><br/>[Container: Pub/Sub, BullMQ]<br/>Черги задач, події, кеш")]
    obs["<b>Observability</b><br/>[Container: Prometheus, Grafana, Loki]<br/>Метрики, логи, SLO-панелі"]
  end

  consumer -- "Використовує" --> mobile
  mobile -- "API-запити, підписка на зміни<br/>[REST / HTTPS, WSS]" --> gw
  admin -- "Адмін-API (Swagger UI), Grafana<br/>[HTTPS]" --> gw
  b2b -- "Потік лотів<br/>[REST / HTTPS + HMAC]" --> gw
  pay -- "Callback оплати<br/>[HTTPS POST]" --> gw

  gw -- "/auth, /lots, /venues, /integration<br/>[REST / HTTP]" --> core
  gw -- "/reservations, /orders,<br/>/limit-orders, /payments<br/>[REST / HTTP]" --> trading
  gw -- "/ws<br/>[WebSocket]" --> rt
  gw -- "/grafana<br/>[HTTP]" --> obs

  core -- "Читання / запис<br/>[SQL / TCP 5432]" --> db
  core -- "Кеш геопошуку, подія lot.created<br/>[RESP / TCP 6379]" --> redis
  trading -- "Транзакції SKIP LOCKED<br/>[SQL / TCP 5432]" --> db
  trading -- "Задачі auto-reserve, події<br/>[RESP / TCP 6379]" --> redis
  worker -- "Пакетний UPDATE цін, outbox<br/>[SQL / TCP 5432]" --> db
  worker -- "Споживання задач, публікація подій<br/>[RESP / TCP 6379]" --> redis
  rt -- "Підписка на події, адаптер<br/>[RESP / TCP 6379]" --> redis

  trading -- "Створення платежу<br/>[REST / HTTPS]" --> pay
  worker -- "Webhook підтвердження<br/>[REST / HTTPS]" --> b2b
  worker -- "Push про виконання ордера<br/>[REST / HTTPS]" --> push

  core -. "Метрики (pull)<br/>[HTTP /metrics]" .-> obs
  trading -. "Метрики (pull)<br/>[HTTP /metrics]" .-> obs
  worker -. "Метрики (pull)<br/>[HTTP /metrics]" .-> obs
  rt -. "Метрики (pull)<br/>[HTTP /metrics]" .-> obs

  classDef person fill:#08427b,stroke:#052e56,color:#ffffff
  classDef container fill:#438dd5,stroke:#2e6295,color:#ffffff
  classDef external fill:#8a8a8a,stroke:#5e5e5e,color:#ffffff
  class consumer,admin person
  class mobile,gw,core,trading,rt,worker,db,redis,obs container
  class b2b,pay,push external
  style b2c fill:#ffffff,stroke:#444444,stroke-dasharray:6 4,color:#222222
```

**Опис контейнерів і трасування вимог (Level 2):**

| Контейнер | Технологія | Реалізовані вимоги | Рішення |
|---|---|---|---|
| Mobile App | React Native (Expo) | клієнт для [SRS-FUN-05](./srs.md#srs-fun-05), [SRS-FUN-13](./srs.md#srs-fun-13), [SRS-FUN-15](./srs.md#srs-fun-15) | — |
| API Gateway | Traefik v3 | [SRS-FUN-16](./srs.md#srs-fun-16), [SRS-NFR-SEC-02](./srs.md#srs-nfr-sec-02) | — |
| Core API | NestJS | [SRS-FUN-01](./srs.md#srs-fun-01)–[SRS-FUN-06](./srs.md#srs-fun-06), [SRS-FUN-17](./srs.md#srs-fun-17)–[SRS-FUN-19](./srs.md#srs-fun-19) | — |
| Trading Service | NestJS | [SRS-FUN-09](./srs.md#srs-fun-09), [SRS-FUN-11](./srs.md#srs-fun-11), [SRS-FUN-13](./srs.md#srs-fun-13), [SRS-FUN-14](./srs.md#srs-fun-14), [SRS-NFR-CONS-01](./srs.md#srs-nfr-cons-01) | [ADR-001](./adr/0001-choice-of-technology.md) |
| Pricing Worker | NestJS + BullMQ | [SRS-FUN-07](./srs.md#srs-fun-07), [SRS-FUN-08](./srs.md#srs-fun-08), [SRS-FUN-10](./srs.md#srs-fun-10), [SRS-FUN-12](./srs.md#srs-fun-12), [SRS-NFR-REL-03](./srs.md#srs-nfr-rel-03) | — |
| Realtime Gateway | NestJS + Socket.IO + Redis adapter | [SRS-FUN-15](./srs.md#srs-fun-15), [SRS-FUN-20](./srs.md#srs-fun-20), [SRS-NFR-PERF-03](./srs.md#srs-nfr-perf-03), [SRS-NFR-SCAL-01](./srs.md#srs-nfr-scal-01) | — |
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
%%{init: {"flowchart": {"nodeSpacing": 70, "rankSpacing": 80, "wrappingWidth": 240, "padding": 12}}}%%
flowchart TB
  gw["<b>API Gateway</b><br/>[Container: Traefik v3]"]
  redisIn[("<b>Redis 7</b><br/>[Container: BullMQ]")]

  subgraph trading["Trading Service [Container: NestJS]"]
    subgraph pipeline["Конвеєр обробки запиту (cross-cutting)"]
      telemetry["<b>TelemetryInterceptor</b><br/>[Component: pino, prom-client]<br/>Логи з correlation-id, метрики RED"]
      guard["<b>JwtAuthGuard</b><br/>[Component: NestJS Guard]<br/>Верифікація JWT RS256, RBAC"]
      idem["<b>IdempotencyInterceptor</b><br/>[Component: NestJS Interceptor]<br/>Ключ Idempotency-Key"]
      pipe["<b>ValidationPipe</b><br/>[Component: class-validator]<br/>Валідація DTO"]
    end

    subgraph inbound["Вхідний шар"]
      resCtl["<b>ReservationController</b><br/>[Component: REST]<br/>POST /reservations"]
      loCtl["<b>LimitOrderController</b><br/>[Component: REST]<br/>POST / DELETE /limit-orders"]
      payCtl["<b>PaymentCallbackController</b><br/>[Component: REST]<br/>POST /payments/callback"]
      autoRes["<b>AutoReserveConsumer</b><br/>[Component: BullMQ Processor]<br/>Задачі limit-order.triggered"]
    end

    subgraph app["Шар застосунку та домену"]
      reserveUC["<b>ReserveLotUnitUseCase</b><br/>[Component: Application Service]<br/>Атомарне резервування"]
      loUC["<b>LimitOrderService</b><br/>[Component: Application Service]<br/>Створення та скасування ордерів"]
      payUC["<b>ConfirmPaymentUseCase</b><br/>[Component: Application Service]<br/>Ідемпотентне підтвердження оплати"]
      domain["<b>Domain Model</b><br/>[Component: TypeScript]<br/>Reservation, Order, LimitOrder"]
      filter["<b>GlobalExceptionFilter</b><br/>[Component: NestJS Filter]<br/>Помилки у форматі RFC 9457"]
    end

    subgraph outbound["Шар інфраструктури (порти та адаптери)"]
      uow["<b>UnitOfWork</b><br/>[Component: Port + TypeORM]<br/>Межі транзакції"]
      unitRepo["<b>LotUnitRepository</b><br/>[Component: Port + SQL]<br/>SELECT FOR UPDATE SKIP LOCKED"]
      orderRepo["<b>OrderRepository</b><br/>[Component: Port + TypeORM]<br/>Замовлення та outbox"]
      loRepo["<b>LimitOrderRepository</b><br/>[Component: Port + TypeORM]<br/>Лімітні ордери"]
      payClient["<b>PaymentGatewayClient</b><br/>[Component: Port + HTTPS]<br/>Платіж, перевірка підпису"]
      events["<b>EventPublisher</b><br/>[Component: Port + Redis]<br/>Події lot.stock_updated"]
    end
  end

  db[("<b>PostgreSQL 16</b><br/>[Container: SQL Database]")]
  redisOut[("<b>Redis 7</b><br/>[Container: Pub/Sub]")]
  pay["<b>LiqPay (sandbox)</b><br/>[External System]"]

  gw -- "Вхідний запит<br/>[REST / HTTP]" --> telemetry
  telemetry -- "Запит з correlation-id" --> guard
  guard -- "Автентифікований запит" --> idem
  idem -- "Новий ключ ідемпотентності" --> pipe
  pipe -- "Валідований DTO" --> resCtl
  pipe -- "Валідований DTO" --> loCtl
  gw -- "Callback LiqPay<br/>[REST / HTTP]" --> payCtl
  redisIn -- "Задача auto-reserve<br/>[BullMQ / RESP]" --> autoRes

  resCtl -- "Інтерфейс [DI]" --> reserveUC
  autoRes -- "Інтерфейс [DI]" --> reserveUC
  loCtl -- "Інтерфейс [DI]" --> loUC
  payCtl -- "Інтерфейс [DI]" --> payUC

  reserveUC -- "Інваріанти резерву" --> domain
  reserveUC -. "DomainException" .-> filter
  reserveUC -- "BEGIN / COMMIT [DI]" --> uow
  reserveUC -- "lockFirstAvailable() [DI]" --> unitRepo
  reserveUC -- "save() + outbox [DI]" --> orderRepo
  reserveUC -- "publish() [DI]" --> events
  loUC -- "save() / cancel() [DI]" --> loRepo
  payUC -- "markPaid() [DI]" --> orderRepo
  payUC -- "verifySignature() [DI]" --> payClient

  unitRepo -- "[SQL / TCP 5432]" --> db
  orderRepo -- "[SQL / TCP 5432]" --> db
  loRepo -- "[SQL / TCP 5432]" --> db
  events -- "PUBLISH<br/>[RESP / TCP 6379]" --> redisOut
  payClient -- "Створення платежу<br/>[REST / HTTPS]" --> pay

  classDef component fill:#85bbf0,stroke:#5d82a8,color:#000000
  classDef container fill:#438dd5,stroke:#2e6295,color:#ffffff
  classDef external fill:#8a8a8a,stroke:#5e5e5e,color:#ffffff
  class telemetry,guard,idem,pipe,resCtl,loCtl,payCtl,autoRes,reserveUC,loUC,payUC,domain,filter,uow,unitRepo,orderRepo,loRepo,payClient,events component
  class gw,redisIn,db,redisOut container
  class pay external
  style trading fill:#ffffff,stroke:#444444,stroke-dasharray:6 4,color:#222222
  style pipeline fill:#f4f8fc,stroke:#9bb7d4,color:#222222
  style inbound fill:#f4f8fc,stroke:#9bb7d4,color:#222222
  style app fill:#f4f8fc,stroke:#9bb7d4,color:#222222
  style outbound fill:#f4f8fc,stroke:#9bb7d4,color:#222222
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
`PaymentCallbackController` не проходить через `JwtAuthGuard`, оскільки
запит надходить від LiqPay: автентичність підтверджується перевіркою
підпису callback у `ConfirmPaymentUseCase`.

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
