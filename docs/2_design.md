# РОЗДІЛ 2. ОБҐРУНТУВАННЯ АРХІТЕКТУРНИХ РІШЕНЬ ТА ПРОЄКТУВАННЯ СИСТЕМИ

Вхідний контракт вимог: [srs.md](./srs.md). Архітектурні рішення:
[adr/](./adr/). API-контракти: [api/openapi.yaml](./api/openapi.yaml).

## 2.1. Обґрунтування архітектурного стилю та патернів проєктування

### 2.1.1. Глобальний архітектурний стиль

Для B2C-підсистеми обрано сервісну архітектуру (service-based
architecture) з подієвою взаємодією між сервісами та Clean
Architecture всередині кожного сервісу. Система складається з
чотирьох окремо розгортуваних NestJS-сервісів — Core API, Trading
Service, Pricing Worker, Realtime Gateway — які використовують
спільну базу даних PostgreSQL 16 + PostGIS ([C4 Container](#diagram-level2)).
Термін «мікросервісна архітектура» до системи не застосовується:
сервіси не мають власних сховищ, тому незалежне розгортання
обмежене спільною схемою БД, а міграції схеми виконуються
централізовано. Цю межу прийнято свідомо з трьох причин.

1. **Коректність конкурентного бронювання.** Гарантія нуля подвійних
   продажів ([SRS-NFR-CONS-01](./srs.md#srs-nfr-cons-01)) забезпечується
   ACID-транзакцією `SELECT … FOR UPDATE SKIP LOCKED`
   ([ADR-001](./adr/0001-choice-of-technology.md)). Розподіл лотів і
   замовлень між окремими БД вимагав би саги з компенсаціями та
   приводив би до eventual consistency саме там, де вимога — сувора
   узгодженість.
2. **Єдине джерело істини для геопошуку.** Пошук за радіусом
   (`ST_DWithin`, GiST) і транзакції працюють з одними й тими самими
   рядками `lots`, тому розбіжність індексу, виявлена в аналогів
   (вузьке місце B1.3), виключена.
3. **Обмеження розгортання.** Один VPS і Docker Compose
   ([SRS-NFR-INF-01](./srs.md#srs-nfr-inf-01)) не мають ресурсу для
   кластера окремих СУБД; операційна вартість (FinOps) обмежена
   одним вузлом PostgreSQL і одним вузлом Redis.

Поділ на сервіси виконано за профілем навантаження, а не за
доменом:

| Сервіс | Профіль навантаження | Масштабування | Вимоги |
|---|---|---|---|
| Core API | читання з геопошуком, піки під час розпродажу | горизонтальне, stateless | [SRS-NFR-PERF-01](./srs.md#srs-nfr-perf-01) |
| Trading Service | короткі запис-транзакції під конкуренцією | горизонтальне; обмежене пулом з'єднань БД | [SRS-NFR-PERF-02](./srs.md#srs-nfr-perf-02) |
| Pricing Worker | пакетні обчислення раз на 60 с | один активний екземпляр (BullMQ repeatable job) | [SRS-NFR-PERF-04](./srs.md#srs-nfr-perf-04) |
| Realtime Gateway | довгоживучі WebSocket-з'єднання | горизонтальне через Redis-адаптер | [SRS-NFR-SCAL-01](./srs.md#srs-nfr-scal-01) |

Сервіси не викликають один одного синхронно. Зовнішній трафік
надходить лише через API Gateway; міжсервісна взаємодія асинхронна:
Pricing Worker ставить задачі `auto-reserve` у чергу BullMQ, а зміни
стану поширюються подіями Redis pub/sub. Відмова одного сервісу не
блокує запити інших: недоступність Realtime Gateway погіршує лише
доставку змін ціни, але не резервування. Єдиний синхронний виклик
зовнішньої системи на шляху користувача — створення платежу LiqPay —
винесено за межі транзакції БД і обмежено тайм-аутом 3 с
(алгоритм A2, 2.3.4).

### 2.1.2. Патерни мікрорівня

| Патерн | Компонент ([C4 Level 3](#diagram-level3)) | Проблема, яку вирішує |
|---|---|---|
| Ports & Adapters, Dependency Injection | інтерфейси `LotUnitRepository`, `PaymentGatewayClient`, `EventPublisher`; адаптери в інфраструктурному модулі | доменна логіка не залежить від TypeORM, Redis і LiqPay; use case тестується з підміною портів без СУБД |
| Repository | `LotUnitRepository`, `OrderRepository`, `LimitOrderRepository` | SQL, зокрема `FOR UPDATE SKIP LOCKED`, зосереджено в одному адаптері; контролери не мають доступу до БД |
| Unit of Work | `UnitOfWork` | резерв, замовлення, журнал статусів і outbox-подія фіксуються однією транзакцією |
| Transactional Outbox | таблиця `outbox_events`, outbox relay у Pricing Worker | публікація події після COMMIT без розподіленої транзакції між PostgreSQL і Redis |
| Idempotency Key | `IdempotencyInterceptor`, таблиця `idempotency_keys`, `jobId` BullMQ | повторні запити мобільного клієнта та повторна доставка задач не створюють дублікатів |
| Pipeline (Chain of Responsibility) | `TelemetryInterceptor` → `JwtAuthGuard` → `IdempotencyInterceptor` → `ValidationPipe` | наскрізні перевірки виконуються до входу в контролер у фіксованому порядку |
| Circuit Breaker + Timeout | адаптер `PaymentGatewayClient` (бібліотека opossum) | після 5 послідовних помилок LiqPay виклики припиняються на 30 с; резервування не очікує недоступного провайдера |

Патерни, які розглядалися й відхилені: CQRS з окремими моделями
читання — пошук лотів виконується одним запитом до тієї самої таблиці,
окрема read-модель додала б синхронізацію без вигоди за латентністю;
Singleton як ручна реалізація — область життя з'єднань і пулів
задається DI-контейнером NestJS; Saga — замінена однією
ACID-транзакцією (див. 2.1.1).

### 2.1.3. Архітектурні ризики та компенсаційні механізми

| Ризик | Наслідок | Компенсація |
|---|---|---|
| PostgreSQL — єдина точка відмови | недоступність запису для всіх сервісів | RPO ≤ 24 год, RTO ≤ 1 год ([SRS-NFR-REL-02](./srs.md#srs-nfr-rel-02)); сервіси повертають 503 з `Retry-After` |
| Redis — єдина точка відмови | втрата черг і подій | AOF `appendfsync everysec`; outbox у PostgreSQL дозволяє повторно опублікувати події; ціни відновлюються наступним циклом A1 |
| Вичерпання пулу з'єднань на «гарячому» лоті | зростання p95 резервування | короткі транзакції (2 запити + вставки); ліміт 10 резервувань/хв на користувача; PgBouncer при навантаженні понад 300 RPS |
| Затримка поширення змін (eventual consistency між БД і клієнтами) | клієнт бачить застарілу ціну до 2 с | ціна фіксується на сервері в момент резервування; клієнтська ціна інформаційна ([SRS-NFR-PERF-03](./srs.md#srs-nfr-perf-03)) |
| Каскадна відмова через LiqPay | блокування потоків Trading Service | тайм-аут 3 с і Circuit Breaker; резерв зберігається до спливу TTL |

### 2.1.4. Реєстр архітектурних рішень (Architecture Decision Records)

| ADR | Рішення | Статус |
|---|---|---|
| [ADR-001](./adr/0001-choice-of-technology.md) | `SELECT FOR UPDATE SKIP LOCKED` для конкурентного бронювання | Approved |

## 2.2. Контекстне та контейнерне моделювання системи (C4 Model)

Діаграми побудовано в нотації C4 (Simon Brown) засобами Mermaid
`flowchart`: тип елемента вказано стереотипом у підписі ([Person],
[Software System], [External System], [Container: …],
[Component: …]), кольори відповідають палітрі C4, кожен зв'язок
підписано дією та протоколом у квадратних дужках. Вбудований
синтаксис `C4Context`/`C4Container` у Mermaid не розводить підписи
зв'язків і дає накладання тексту, тому для читабельності обрано
`flowchart` з автоматичною розкладкою.

<a id="diagram-level1"></a>

### 2.2.1. C4 Level 1 — System Context

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

### 2.2.2. C4 Level 2 — Container Diagram

```mermaid
%%{init: {"flowchart": {"nodeSpacing": 30, "rankSpacing": 55, "wrappingWidth": 170, "padding": 8}}}%%
flowchart TB
  consumer["<b>Покупець</b><br/>[Person]"]
  admin["<b>Адміністратор</b><br/>[Person]"]
  b2b["<b>B2B-система (Тема 2)</b><br/>[External System]"]
  pay["<b>LiqPay (sandbox)</b><br/>[External System]"]
  push["<b>Expo Push / FCM</b><br/>[External System]"]

  subgraph b2c["B2C-підсистема [Software System]"]
    mobile["<b>Mobile App</b><br/>[Container: React Native]<br/>Мапа, лоти, ордери"]
    gw["<b>API Gateway</b><br/>[Container: Traefik v3]<br/>TLS 1.3, rate limiting"]
    subgraph backend["Сервіси бекенду"]
      core["<b>Core API</b><br/>[Container: NestJS]<br/>Auth, каталог, геопошук, B2B"]
      trading["<b>Trading Service</b><br/>[Container: NestJS]<br/>Резервування, оплата, ордери"]
      worker["<b>Pricing Worker</b><br/>[Container: NestJS, BullMQ]<br/>Price Decay, зіставлення, outbox"]
    end
    rt["<b>Realtime Gateway</b><br/>[Container: Socket.IO]<br/>WebSocket-кімнати"]
    db[("<b>PostgreSQL 16 + PostGIS</b><br/>[Container: SQL DB]")]
    redis[("<b>Redis 7</b><br/>[Container: Pub/Sub, черги]")]
    obs["<b>Observability</b><br/>[Container: Prometheus, Grafana]"]
  end

  consumer -- "Використовує" --> mobile
  mobile -- "API, підписка<br/>[HTTPS, WSS]" --> gw
  admin -- "Адмін-API, Grafana<br/>[HTTPS]" --> gw
  b2b -- "Потік лотів<br/>[HTTPS + HMAC]" --> gw
  pay -- "Callback оплати<br/>[HTTPS POST]" --> gw

  gw -- "/auth /lots /integration<br/>[REST / HTTP]" --> core
  gw -- "/reservations /orders<br/>[REST / HTTP]" --> trading
  gw -- "/ws<br/>[WebSocket]" --> rt
  gw -- "/grafana<br/>[HTTP]" --> obs

  backend -- "Транзакції, SKIP LOCKED, outbox<br/>[SQL / TCP 5432]" --> db
  backend -- "Черги BullMQ, кеш, події<br/>[RESP / TCP 6379]" --> redis
  rt -- "Підписка на події<br/>[RESP / TCP 6379]" --> redis
  backend -. "Метрики<br/>[HTTP /metrics]" .-> obs
  rt -. "Метрики<br/>[HTTP /metrics]" .-> obs

  trading -- "Платіж<br/>[HTTPS]" --> pay
  worker -- "Webhook продажу<br/>[HTTPS]" --> b2b
  worker -- "Push про ордер<br/>[HTTPS]" --> push

  classDef person fill:#08427b,stroke:#052e56,color:#ffffff
  classDef container fill:#438dd5,stroke:#2e6295,color:#ffffff
  classDef external fill:#8a8a8a,stroke:#5e5e5e,color:#ffffff
  class consumer,admin person
  class mobile,gw,core,trading,rt,worker,db,redis,obs container
  class b2b,pay,push external
  style b2c fill:#ffffff,stroke:#444444,stroke-dasharray:6 4,color:#222222
  style backend fill:#eef4fb,stroke:#7a9cc6,color:#222222
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

Core API, Trading Service і Pricing Worker на діаграмі згруповано в
«Сервіси бекенду», щоб спільні зв'язки зі сховищами не дублювалися:
усі три сервіси звертаються до PostgreSQL (SQL / TCP 5432) і Redis
(RESP / TCP 6379) — Core API для каталогу, геопошуку й кешу, Trading
Service для транзакцій SKIP LOCKED і задач auto-reserve, Pricing Worker
для пакетного оновлення цін, outbox і черг BullMQ; метрики всіх
сервісів збирає Observability через HTTP `/metrics`.
Протоколи: зовнішній трафік — лише через API Gateway (REST / HTTPS,
WSS, TLS 1.3); внутрішній трафік у мережі Docker — REST / HTTP і
WebSocket; доступ до сховищ — SQL / TCP 5432 та RESP / TCP 6379.
Контейнер Trading Service [C4 Container](#diagram-level2) реалізує
вимогу атомарного резервування [SRS-FUN-09](./srs.md#srs-fun-09) та
базується на рішенні [ADR-001](./adr/0001-choice-of-technology.md)
щодо `SELECT FOR UPDATE SKIP LOCKED`.

<a id="diagram-level3"></a>

## 2.3. Компонентне моделювання та опис серверного ядра

### 2.3.1. C4 Level 3 — Component Diagram: Trading Service

Обрано Trading Service як контейнер з найбільшим інженерним ризиком:
він реалізує конкурентне резервування, оплату та виконання лімітних
ордерів.

```mermaid
%%{init: {"flowchart": {"nodeSpacing": 25, "rankSpacing": 50, "wrappingWidth": 150, "padding": 8}}}%%
flowchart TB
  gw["<b>API Gateway</b><br/>[Container: Traefik]"]
  redisIn[("<b>Redis 7</b><br/>[Container: BullMQ]")]

  subgraph trading["Trading Service [Container: NestJS]"]
    subgraph pipeline["Конвеєр запиту (cross-cutting)"]
      telemetry["<b>TelemetryInterceptor</b><br/>[Component]<br/>correlation-id, метрики"]
      guard["<b>JwtAuthGuard</b><br/>[Component]<br/>JWT RS256, RBAC"]
      idem["<b>IdempotencyInterceptor</b><br/>[Component]<br/>Idempotency-Key"]
      pipe["<b>ValidationPipe</b><br/>[Component]<br/>Валідація DTO"]
    end

    subgraph inbound["Вхідний шар"]
      resCtl["<b>ReservationController</b><br/>[Component: REST]"]
      loCtl["<b>LimitOrderController</b><br/>[Component: REST]"]
      payCtl["<b>PaymentCallbackController</b><br/>[Component: REST]"]
      autoRes["<b>AutoReserveConsumer</b><br/>[Component: BullMQ]"]
    end

    subgraph app["Шар застосунку та домену"]
      reserveUC["<b>ReserveLotUnitUseCase</b><br/>[Component]<br/>Атомарне резервування"]
      loUC["<b>LimitOrderService</b><br/>[Component]<br/>Ордери"]
      payUC["<b>ConfirmPaymentUseCase</b><br/>[Component]<br/>Підтвердження оплати"]
      domain["<b>Domain Model</b><br/>[Component]<br/>Інваріанти"]
      filter["<b>GlobalExceptionFilter</b><br/>[Component]<br/>RFC 9457"]
    end

    subgraph outbound["Шар інфраструктури (порти та адаптери)"]
      uow["<b>UnitOfWork</b><br/>[Port + TypeORM]"]
      unitRepo["<b>LotUnitRepository</b><br/>[Port + SQL]<br/>SKIP LOCKED"]
      orderRepo["<b>OrderRepository</b><br/>[Port + TypeORM]<br/>+ outbox"]
      loRepo["<b>LimitOrderRepository</b><br/>[Port + TypeORM]"]
      payClient["<b>PaymentGatewayClient</b><br/>[Port + HTTPS]"]
      events["<b>EventPublisher</b><br/>[Port + Redis]"]
    end
  end

  db[("<b>PostgreSQL 16</b><br/>[Container: SQL DB]")]
  redisOut[("<b>Redis 7</b><br/>[Container: Pub/Sub]")]
  pay["<b>LiqPay (sandbox)</b><br/>[External System]"]

  gw -- "[REST / HTTP]" --> telemetry
  telemetry --> guard
  guard --> idem
  idem --> pipe
  pipe -- "DTO" --> resCtl
  pipe -- "DTO" --> loCtl
  gw -- "Callback<br/>[REST / HTTP]" --> payCtl
  redisIn -- "auto-reserve<br/>[BullMQ]" --> autoRes

  resCtl -- "[DI]" --> reserveUC
  autoRes -- "[DI]" --> reserveUC
  loCtl -- "[DI]" --> loUC
  payCtl -- "[DI]" --> payUC

  reserveUC --> domain
  reserveUC -. "DomainException" .-> filter
  reserveUC -- "BEGIN/COMMIT" --> uow
  reserveUC -- "lockFirstAvailable()" --> unitRepo
  reserveUC -- "save() + outbox" --> orderRepo
  reserveUC -- "publish()" --> events
  loUC -- "save() / cancel()" --> loRepo
  payUC -- "markPaid()" --> orderRepo
  payUC -- "verifySignature()" --> payClient

  unitRepo -- "[SQL :5432]" --> db
  orderRepo -- "[SQL :5432]" --> db
  loRepo -- "[SQL :5432]" --> db
  events -- "PUBLISH<br/>[RESP :6379]" --> redisOut
  payClient -- "[REST / HTTPS]" --> pay

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

### 2.3.2. Опис компонентів

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

### 2.3.3. Опис серверного ядра: шлях запиту через рівні

Дані проходять три рівні Trading Service строго в одному напрямку;
залежності спрямовані до домену (Dependency Rule Clean Architecture).

1. **Inbound Layer (Controllers, Middlewares).** NestJS-конвеєр
   виконує `TelemetryInterceptor` (correlation-id, гістограма
   `http_request_duration_seconds`) → `JwtAuthGuard` (автентифікація,
   роль) → `IdempotencyInterceptor` (повернення збереженої відповіді)
   → `ValidationPipe` (DTO class-validator) → контролер. Контролер не
   містить бізнес-правил і не має доступу до репозиторіїв: він
   перетворює DTO на команду (`ReserveLotUnitCommand`) з `user_id`,
   взятим з JWT, і викликає use case через інтерфейс. Вхідна точка
   `AutoReserveConsumer` (BullMQ) викликає той самий use case, тому
   ручна й автоматична купівля мають один шлях виконання.
2. **Core Layer (Application Services, Domain Model).** Use case
   (`ReserveLotUnitUseCase`, `ConfirmPaymentUseCase`,
   `LimitOrderService`) керує транзакцією через порт `UnitOfWork` і
   звертається лише до портів (TypeScript-інтерфейсів). Інваріанти
   домену зосереджено в сутностях: `Order.markPaid()` дозволяє перехід
   лише з `pending_payment`; `Reservation` не створюється для лота з
   `expires_at ≤ now`; `LimitOrder.trigger()` дозволений лише зі стану
   `active`. Порушення інваріанта — доменний виняток, який
   `GlobalExceptionFilter` перетворює на RFC 9457 (409, 410).
3. **Outbound Layer (Repositories, External Clients).** Адаптери
   реалізують порти: `LotUnitRepository` (сирий параметризований SQL
   для `FOR UPDATE SKIP LOCKED`), `OrderRepository` і
   `LimitOrderRepository` (TypeORM; кожен запит до ресурсу користувача
   містить умову `user_id = $n`), `PaymentGatewayClient` (HTTPS до
   LiqPay, тайм-аут 3 с, Circuit Breaker), `EventPublisher` (Redis).
   Доменний рівень не імпортує TypeORM, ioredis чи HTTP-клієнти; заміна
   адаптера не змінює use case.

<a id="algorithms"></a>

### 2.3.4. Моделювання логіки критичних вузлів

Позначення: `:=` — присвоювання; `=`, `≠`, `<`, `≤`, `≥` — лише
порівняння в умовах; логічні зв'язки — AND, OR, NOT; `X[i]` — i-й
елемент колекції X (нумерація з 1); `|X|` — потужність множини;
`∅` — порожня множина. Блоки схем: овал — термінатор, паралелограм —
введення/виведення, прямокутник — процес, ромб — рішення, прямокутник
з подвійними бічними сторонами — зумовлений процес (підпрограма,
транзакція або зовнішній виклик).

#### 2.3.4.1. Алгоритм A1 — циклічний перерахунок ціни (Price Decay)

Контейнер: Pricing Worker; вимоги
[SRS-FUN-07](./srs.md#srs-fun-07), [SRS-FUN-08](./srs.md#srs-fun-08),
[SRS-FUN-19](./srs.md#srs-fun-19), [SRS-NFR-PERF-04](./srs.md#srs-nfr-perf-04).

1. `t_now := now()`; `B := 500`; `last_id := NULL`; `attempt := 0`.
2. `Lots := SELECT … FROM lots JOIN venues WHERE status = 'active' AND (last_id = NULL OR id > last_id) ORDER BY id LIMIT B` (keyset-пагінація).
3. Якщо `Lots = ∅` — кінець циклу.
4. `N := |Lots|`; `i := 1`; `Updates := ∅`; `Expired := ∅`.
5. Поки `i ≤ N`:
   1. `T := Lots[i].expires_at − Lots[i].published_at`; `t_rem := Lots[i].expires_at − t_now`.
   2. Якщо `t_rem ≤ 0 OR T ≤ 0`: `Expired := Expired ∪ {Lots[i].id}`.
   3. Інакше: `P_min := MAX(Lots[i].min_price_minor, ROUND(Lots[i].base_price_minor · Lots[i].min_price_ratio))`; `r := MIN(1, MAX(0, t_rem / T))`; `P := ROUND(P_min + (Lots[i].base_price_minor − P_min) · r ^ Lots[i].decay_k)`; якщо `P ≠ Lots[i].current_price_minor`: `Updates := Updates ∪ {(Lots[i].id, P)}`.
   4. `i := i + 1`.
6. Транзакція: пакетний `UPDATE` цін для `Updates`, `status := 'expired'` для `Expired`, запис подій в `outbox_events`.
7. Якщо помилка СУБД `AND attempt < 3`: `attempt := attempt + 1`, очікування `2^attempt · 100` мс, повтор кроку 6; якщо помилка після 3 спроб — задача переходить у dead-letter queue, кінець.
8. Виклик A3 `MatchLimitOrders(Updates)`; `last_id := Lots[N].id`; `attempt := 0`; перехід до кроку 2.

Edge cases: порожня вибірка (крок 3); `T ≤ 0` через некоректні дані
B2B-системи (лот знімається з продажу замість ділення на нуль);
обмеження `r` відрізком [0, 1]; оновлюються лише змінені ціни;
кількість повторів транзакції обмежена трьома.

```mermaid
%%{init: {"flowchart": {"wrappingWidth": 300, "nodeSpacing": 40, "rankSpacing": 45}}}%%
flowchart TD
  S(["Початок: задача price-decay (BullMQ, кожні 60 с)"])
  I1[/"Ввід: t_now := now(); B := 500; last_id := NULL; attempt := 0"/]
  P1["Lots := SELECT id, base_price_minor, min_price_minor, published_at, expires_at, current_price_minor, decay_k, min_price_ratio FROM lots JOIN venues WHERE status = 'active' AND (last_id = NULL OR id > last_id) ORDER BY id LIMIT B"]
  D0{"Lots = ∅ ?"}
  P2["N := |Lots|; i := 1; Updates := ∅; Expired := ∅"]
  D1{"i ≤ N ?"}
  P3["T := Lots[i].expires_at − Lots[i].published_at; t_rem := Lots[i].expires_at − t_now"]
  D2{"t_rem ≤ 0 OR T ≤ 0 ?"}
  P4["Expired := Expired ∪ {Lots[i].id}"]
  P5["P_min := MAX(Lots[i].min_price_minor, ROUND(Lots[i].base_price_minor · Lots[i].min_price_ratio)); r := t_rem / T; r := MIN(1, MAX(0, r)); P := P_min + (Lots[i].base_price_minor − P_min) · r ^ Lots[i].decay_k; P := ROUND(P)"]
  D3{"P ≠ Lots[i].current_price_minor ?"}
  P6["Updates := Updates ∪ {(Lots[i].id, P)}"]
  P7["i := i + 1"]
  P8[["Транзакція: UPDATE lots SET current_price_minor := P, price_updated_at := t_now для Updates; UPDATE lots SET status := 'expired' для Expired; INSERT outbox_events"]]
  D4{"Помилка СУБД AND attempt < 3 ?"}
  P9["attempt := attempt + 1; очікування 2^attempt · 100 мс"]
  D5{"Помилка СУБД ?"}
  O1[/"Вивід: задача → dead-letter queue; метрика pricing_cycle_failed"/]
  P10[["Виклик A3: MatchLimitOrders(Updates)"]]
  P11["last_id := Lots[N].id; attempt := 0"]
  E(["Кінець: метрика pricing_cycle_duration_seconds"])

  S --> I1 --> P1 --> D0
  D0 -- "Так" --> E
  D0 -- "Ні" --> P2 --> D1
  D1 -- "Так" --> P3 --> D2
  D2 -- "Так" --> P4 --> P7
  D2 -- "Ні" --> P5 --> D3
  D3 -- "Так" --> P6 --> P7
  D3 -- "Ні" --> P7
  P7 --> D1
  D1 -- "Ні" --> P8 --> D4
  D4 -- "Так" --> P9 --> P8
  D4 -- "Ні" --> D5
  D5 -- "Так" --> O1 --> E
  D5 -- "Ні" --> P10 --> P11 --> P1
```

#### 2.3.4.2. Алгоритм A2 — атомарне резервування одиниці лота

Контейнер: Trading Service (`ReserveLotUnitUseCase`, [C4 Level 3](#diagram-level3));
вимоги [SRS-FUN-09](./srs.md#srs-fun-09), [SRS-FUN-10](./srs.md#srs-fun-10),
[SRS-NFR-CONS-01](./srs.md#srs-nfr-cons-01); рішення
[ADR-001](./adr/0001-choice-of-technology.md).

1. Ввід: `user_id`, `lot_id`, `idem_key`, `body_hash`.
2. `K := SELECT * FROM idempotency_keys WHERE key = idem_key`.
3. Якщо `K ≠ NULL AND K.user_id = user_id AND K.request_hash = body_hash` — вивід збереженої відповіді, кінець.
4. Якщо `K ≠ NULL AND (K.user_id ≠ user_id OR K.request_hash ≠ body_hash)` — вивід 409 `idempotency-conflict`, кінець.
5. `attempt := 0`; `retry_sold_out := 0`.
6. `BEGIN`; `L := SELECT status, expires_at, current_price_minor, currency FROM lots WHERE id = lot_id`.
7. Якщо `L = NULL` — `ROLLBACK`, вивід 404. Якщо `L.status ≠ 'active' OR t_now ≥ L.expires_at` — `ROLLBACK`, вивід 410.
8. `U := SELECT id FROM lot_units WHERE lot_id = lot_id AND status = 'available' ORDER BY id LIMIT 1 FOR UPDATE SKIP LOCKED`.
9. Якщо `U = NULL`: якщо `retry_sold_out < 1` — `ROLLBACK`, `retry_sold_out := retry_sold_out + 1`, очікування 200 мс, перехід до кроку 6 (захист від хибного «розпродано», ADR-001); інакше — `ROLLBACK`, вивід 409 `sold-out`.
10. `status := 'reserved'`, `reserved_by := user_id`, `reserved_until := t_now + 10 хв` для `U`; `INSERT orders` з `status := 'pending_payment'`, `price_minor := L.current_price_minor`; `INSERT order_status_history`; `INSERT outbox_events ('lot.stock_updated')`; `INSERT idempotency_keys`; `COMMIT`.
11. Якщо помилка СУБД (deadlock `OR` timeout) `AND attempt < 2` — `ROLLBACK`, `attempt := attempt + 1`, перехід до кроку 6; якщо помилка після повторів — вивід 503 з `Retry-After`.
12. Створення платежу LiqPay (тайм-аут 3 с, поза транзакцією): якщо відповідь успішна — `checkout_url := R.checkout_url`, `INSERT payments (status := 'created')`; інакше `checkout_url := NULL`, `INSERT outbox_events ('payment.create_requested')` — фонова задача повторює створення платежу з exponential backoff, а клієнт лише читає стан через `GET /api/v1/orders/{order_id}` (EF-01B, 2.6.2); резерв зберігається до спливу TTL.
13. Вивід 201 `Reservation`.

Edge cases: повторний запит після мережевого збою (кроки 2–4);
неіснуючий або прострочений лот (крок 7); хибна відповідь
«розпродано» через ROLLBACK конкурентної транзакції (крок 9);
deadlock і тайм-аут СУБД (крок 11); недоступність платіжного
провайдера (крок 12) — не скасовує резерв.

```mermaid
%%{init: {"flowchart": {"wrappingWidth": 300, "nodeSpacing": 40, "rankSpacing": 45}}}%%
flowchart TD
  S(["Початок: POST /api/v1/reservations"])
  I1[/"Ввід: user_id, lot_id, idem_key, body_hash"/]
  P0["K := SELECT * FROM idempotency_keys WHERE key = idem_key"]
  D0{"K ≠ NULL AND K.user_id = user_id AND K.request_hash = body_hash ?"}
  O0[/"Вивід: K.response_status, K.response_body"/]
  D0a{"K ≠ NULL AND (K.user_id ≠ user_id OR K.request_hash ≠ body_hash) ?"}
  O0a[/"Вивід: 409 idempotency-conflict"/]
  P1["attempt := 0; retry_sold_out := 0"]
  P2["BEGIN; L := SELECT status, expires_at, current_price_minor, currency FROM lots WHERE id = lot_id"]
  D1{"L = NULL ?"}
  O1[/"ROLLBACK; Вивід: 404 not-found"/]
  D2{"L.status ≠ 'active' OR t_now ≥ L.expires_at ?"}
  O2[/"ROLLBACK; Вивід: 410 gone"/]
  P3["U := SELECT id FROM lot_units WHERE lot_id = lot_id AND status = 'available' ORDER BY id LIMIT 1 FOR UPDATE SKIP LOCKED"]
  D3{"U = NULL ?"}
  D4{"retry_sold_out < 1 ?"}
  P4["ROLLBACK; retry_sold_out := retry_sold_out + 1; очікування 200 мс"]
  O3[/"ROLLBACK; Вивід: 409 sold-out"/]
  P5["UPDATE lot_units SET status := 'reserved', reserved_by := user_id, reserved_until := t_now + 10 хв WHERE id = U.id; O := INSERT orders(status := 'pending_payment', price_minor := L.current_price_minor); INSERT order_status_history; INSERT outbox_events('lot.stock_updated'); INSERT idempotency_keys; COMMIT"]
  D5{"Помилка СУБД (deadlock OR timeout) AND attempt < 2 ?"}
  P6["ROLLBACK; attempt := attempt + 1"]
  D6{"Помилка СУБД ?"}
  O4[/"Вивід: 503 Retry-After"/]
  P7[["LiqPay: створення платежу (тайм-аут 3 с)"]]
  D7{"Відповідь провайдера = success ?"}
  P8["checkout_url := R.checkout_url; INSERT payments(status := 'created')"]
  P9["checkout_url := NULL"]
  O5[/"Вивід: 201 Reservation(order_id := O.id, price_minor, reserved_until, checkout_url)"/]
  E(["Кінець"])

  S --> I1 --> P0 --> D0
  D0 -- "Так" --> O0 --> E
  D0 -- "Ні" --> D0a
  D0a -- "Так" --> O0a --> E
  D0a -- "Ні" --> P1 --> P2 --> D1
  D1 -- "Так" --> O1 --> E
  D1 -- "Ні" --> D2
  D2 -- "Так" --> O2 --> E
  D2 -- "Ні" --> P3 --> D3
  D3 -- "Так" --> D4
  D4 -- "Так" --> P4 --> P2
  D4 -- "Ні" --> O3 --> E
  D3 -- "Ні" --> P5 --> D5
  D5 -- "Так" --> P6 --> P2
  D5 -- "Ні" --> D6
  D6 -- "Так" --> O4 --> E
  D6 -- "Ні" --> P7 --> D7
  D7 -- "Так" --> P8 --> O5
  D7 -- "Ні" --> P9 --> O5
  O5 --> E
```

#### 2.3.4.3. Алгоритм A3 — виконання лімітних ордерів

Контейнери: Pricing Worker (зіставлення) і Trading Service
(`AutoReserveConsumer`); вимоги [SRS-FUN-13](./srs.md#srs-fun-13),
[SRS-FUN-14](./srs.md#srs-fun-14).

1. Ввід: `Updates` — множина пар `(lot_id, P)` з A1; `t_now := now()`.
2. Якщо `Updates = ∅` — перехід до кроку 5.
3. `M := |Updates|`; для `j := 1 … M`:
   1. `A := SELECT COUNT(*) FROM lot_units WHERE lot_id = Updates[j].lot_id AND status = 'available'`; якщо `A = 0` — наступне `j`.
   2. `Orders := SELECT id, user_id FROM limit_orders WHERE lot_id = Updates[j].lot_id AND status = 'active' AND threshold_price_minor ≥ Updates[j].P AND valid_until > t_now ORDER BY created_at LIMIT A` (FIFO за часом створення, не більше доступних одиниць).
   3. Для `m := 1 … |Orders|`: `UPDATE limit_orders SET status := 'triggered' WHERE id = Orders[m].id AND status = 'active'`; якщо оновлено рівно 1 рядок — задача BullMQ `auto-reserve` з `jobId := Orders[m].id`, `idem_key := Orders[m].id`, `attempts := 3`.
4. Кінець циклу за `j`.
5. `UPDATE limit_orders SET status := 'expired' WHERE status = 'active' AND valid_until ≤ t_now`.
6. Обробник задачі (`AutoReserveConsumer`): `R := ReserveLotUnit(user_id, lot_id, idem_key)` (A2); якщо `R.status = 201` — `status := 'executed'`, `order_id := R.order_id`, push-сповіщення; якщо `R.status = 409 OR R.status = 410` — `status := 'failed'`; інакше, якщо `job.attemptsMade < 3` — повтор з exponential backoff, інакше `status := 'failed'` і dead-letter queue.

Edge cases: порожня множина оновлень (крок 2); відсутність вільних
одиниць (крок 3.1); конкурентне скасування ордера користувачем
(умова `status = 'active'` в `UPDATE` кроку 3.3 — ордер не
запускається двічі); повторна доставка задачі BullMQ (`jobId` та
`idem_key` збігаються з id ордера — A2 поверне збережену відповідь); прострочені
ордери (крок 5). A3 викликається після запису кожної пачки з
`B := 500` лотів алгоритму A1, тому затримка між зниженням ціни до
порога і постановкою задачі `auto-reserve` обмежена часом обробки
однієї пачки, що узгоджується з [SRS-FUN-14](./srs.md#srs-fun-14)
(≤ 5 с).

```mermaid
%%{init: {"flowchart": {"wrappingWidth": 300, "nodeSpacing": 40, "rankSpacing": 45}}}%%
flowchart TD
  S(["Початок: MatchLimitOrders(Updates)"])
  I1[/"Ввід: Updates — множина пар (lot_id, P) з A1; t_now := now()"/]
  D0{"Updates = ∅ ?"}
  P1["M := |Updates|; j := 1"]
  D1{"j ≤ M ?"}
  P2["A := SELECT COUNT(*) FROM lot_units WHERE lot_id = Updates[j].lot_id AND status = 'available'"]
  D2{"A = 0 ?"}
  P3["Orders := SELECT id, user_id FROM limit_orders WHERE lot_id = Updates[j].lot_id AND status = 'active' AND threshold_price_minor ≥ Updates[j].P AND valid_until > t_now ORDER BY created_at LIMIT A"]
  P4["K := |Orders|; m := 1"]
  D3{"m ≤ K ?"}
  P5["UPDATE limit_orders SET status := 'triggered' WHERE id = Orders[m].id AND status = 'active'"]
  D4{"Оновлено рядків = 1 ?"}
  P6[["BullMQ: add('auto-reserve', jobId := Orders[m].id, idem_key := Orders[m].id, attempts := 3)"]]
  P7["m := m + 1"]
  P8["j := j + 1"]
  P9["UPDATE limit_orders SET status := 'expired' WHERE status = 'active' AND valid_until ≤ t_now"]
  E(["Кінець"])

  S --> I1 --> D0
  D0 -- "Так" --> P9
  D0 -- "Ні" --> P1 --> D1
  D1 -- "Так" --> P2 --> D2
  D2 -- "Так" --> P8
  D2 -- "Ні" --> P3 --> P4 --> D3
  D3 -- "Так" --> P5 --> D4
  D4 -- "Так" --> P6 --> P7
  D4 -- "Ні" --> P7
  P7 --> D3
  D3 -- "Ні" --> P8
  P8 --> D1
  D1 -- "Ні" --> P9 --> E

  subgraph Consumer["AutoReserveConsumer (Trading Service), для кожної задачі"]
    C0(["Початок: обробка задачі auto-reserve"])
    C1[/"Ввід: job(limit_order_id, user_id, lot_id)"/]
    C2[["R := ReserveLotUnit(user_id, lot_id, idem_key := limit_order_id) — алгоритм A2"]]
    D5{"R.status = 201 ?"}
    C3["UPDATE limit_orders SET status := 'executed', order_id := R.order_id; push-сповіщення"]
    D6{"R.status = 409 OR R.status = 410 ?"}
    C4["UPDATE limit_orders SET status := 'failed'"]
    D7{"job.attemptsMade < 3 ?"}
    C5["throw → повтор BullMQ з exponential backoff"]
    C6["UPDATE limit_orders SET status := 'failed'; job → dead-letter queue"]
    CE(["Кінець задачі"])
    C0 --> C1 --> C2 --> D5
    D5 -- "Так" --> C3
    D5 -- "Ні" --> D6
    D6 -- "Так" --> C4
    D6 -- "Ні" --> D7
    D7 -- "Так" --> C5
    D7 -- "Ні" --> C6
    C3 --> CE
    C4 --> CE
    C5 --> CE
    C6 --> CE
  end
  P6 -.-> C0
```

<a id="appsec"></a>

## 2.4. Контур безпеки програмного рішення (Application Security)

### 2.4.1. Автентифікація та керування сесіями

| Механізм | Специфікація | Компонент |
|---|---|---|
| Хешування паролів | Argon2id, `m = 19 456 KiB`, `t = 2`, `p = 1`, сіль 16 байт (параметри OWASP Password Storage Cheat Sheet); бібліотека `argon2` (node-argon2) | Core API, `users.password_hash` |
| Access-токен | JWT RS256, `exp` = 15 хв; перевірка `iss`, `aud`, `exp`, `nbf`; ключі в форматі JWKS з ротацією кожні 90 діб, старий ключ приймається ще 15 хв | `JwtAuthGuard` |
| Refresh-токен | непрозорий випадковий рядок 256 біт; у БД — лише SHA-256 хеш; строк 30 діб; ротація на кожне оновлення; повторне використання відкликаного токена відкликає всю сім'ю `family_id` | Core API, `refresh_tokens` |
| Зберігання токенів на клієнті | Expo SecureStore (iOS Keychain, Android Keystore); токени не зберігаються в AsyncStorage | Mobile App |
| Захист від перебору паролів | 5 невдалих входів за 15 хв на пару (email, IP) → затримка 15 хв; відповідь не розкриває, чи існує email | Core API, лічильник у Redis |

### 2.4.2. Авторизація та контроль доступу до об'єктів

- RBAC: ролі `consumer`, `admin` з claim `role`; адміністративні
  маршрути захищені декоратором `@Roles('admin')` і `RolesGuard`.
- Контроль доступу на рівні об'єкта (OWASP API1:2023 BOLA): власність
  перевіряється в запиті до БД (`WHERE id = $1 AND user_id = $2`),
  а не після вибірки; для чужого ресурсу повертається 404 (2.5.4).
- `user_id` ніколи не приймається з тіла чи query: джерелом є лише
  claim `sub` перевіреного JWT.
- Callback LiqPay автентифікується підписом
  `base64(sha1(private_key + data + private_key))`, порівняння —
  `crypto.timingSafeEqual`; B2B-потік — HMAC-SHA256 з вікном 300 с і
  захистом від повторного відтворення (2.5.3, DF-02).

### 2.4.3. Валідація вхідних даних і захист від ін'єкцій

- Глобальний `ValidationPipe` з `whitelist: true`,
  `forbidNonWhitelisted: true`, `transform: true`: невідомі поля
  відхиляються з кодом 400, що блокує Mass Assignment (OWASP API3:2023);
  DTO не містять полів `status`, `price_minor`, `user_id`, `role`.
- Усі SQL-запити параметризовані (`$1…$n`); конкатенація рядків у
  SQL заборонена правилом ESLint і перевіряється SonarQube.
- Ліміти розміру: тіло запиту 16 KiB (64 KiB для B2B), рядки DTO
  мають `maxLength`, масиви — `maxItems`.

### 2.4.4. Обмеження ресурсів і захист від зловживань

- Rate limiting на двох рівнях: Traefik `rateLimit` (на IP) і
  `@nestjs/throttler` зі сховищем Redis (на `user_id`): 100 запитів/хв,
  10 резервувань/хв, 600 запитів/хв для B2B; відповідь 429 з
  `Retry-After` (OWASP API4:2023).
- Бізнес-ліміти (OWASP API6:2023): один активний лімітний ордер на пару
  (користувач, лот); резерв діє 10 хв; одна одиниця — одне чинне
  замовлення.

### 2.4.5. Журнал аудиту та виявлення аномалій

- `order_status_history`: кожен перехід статусу замовлення з
  `actor_user_id`, `correlation_id`, часом.
- Події безпеки в структурованому JSON-журналі (pino): невдалий вхід,
  відкликання сім'ї refresh-токенів, невалідний підпис LiqPay або B2B,
  відповідь 429, 401 на WebSocket-handshake. Поля `password`,
  `authorization`, `refresh_token`, `signature`, `data` маскуються
  (`pino.redact`).
- Алерти Grafana: понад 20 невалідних підписів за 5 хв; понад 50
  відповідей 401 з однієї IP-адреси за хвилину.

### 2.4.6. Захист середовища та ланцюга постачання

| Загроза | Механізм |
|---|---|
| Витік секретів з `.env` | секрети (приватні ключі JWT, ключі LiqPay, HMAC B2B) — Docker secrets і GitHub Actions secrets; `.env` у `.gitignore`; сканування комітів gitleaks у CI |
| Вразливі залежності | `npm audit --audit-level=high`, Trivy (образи і `package-lock.json`), Dependabot; збірка блокується при Critical/High ([SRS-NFR-SEC-03](./srs.md#srs-nfr-sec-03)) |
| Небезпечна конфігурація (API8:2023) | `helmet` (HSTS, `X-Content-Type-Options`, `Referrer-Policy`); вимкнений `X-Powered-By`; Swagger UI доступний лише ролі `admin`; контейнери запускаються від непривілейованого користувача |
| Транспорт | TLS 1.3 на API Gateway; внутрішня мережа Docker ізольована, порти PostgreSQL і Redis не публікуються назовні; Redis з паролем і `rename-command` для `FLUSHALL`, `CONFIG` |
| Ненадійне споживання сторонніх API (API10:2023) | відповіді LiqPay і Expo валідуються за схемою; тайм-аути 3 с; Circuit Breaker |

### 2.4.7. Покриття OWASP API Security Top 10 (2023)

| Ризик | Захист | Підрозділ |
|---|---|---|
| API1 Broken Object Level Authorization | власність у SQL-умові, 404 для чужих ресурсів | 2.4.2 |
| API2 Broken Authentication | Argon2id, JWT RS256, ротація refresh-токенів, ліміт входів | 2.4.1 |
| API3 Broken Object Property Level Authorization | whitelist DTO, відсутність службових полів у DTO | 2.4.3 |
| API4 Unrestricted Resource Consumption | rate limiting, Max Body Size, пагінація ≤ 100 | 2.4.4, 2.5.4 |
| API5 Broken Function Level Authorization | `RolesGuard` для адміністративних маршрутів | 2.4.2 |
| API6 Unrestricted Access to Sensitive Business Flows | ліміт резервувань, TTL резерву, один активний ордер | 2.4.4 |
| API7 Server Side Request Forgery | система не завантажує URL, надані користувачем | — |
| API8 Security Misconfiguration | helmet, закриті порти БД, секрети поза репозиторієм | 2.4.6 |
| API9 Improper Inventory Management | єдиний контракт `openapi.yaml`, версія в шляху `/v1` | 2.5 |
| API10 Unsafe Consumption of APIs | перевірка підписів, тайм-аути, Circuit Breaker | 2.4.2, 2.4.6 |

<a id="api-contracts"></a>

## 2.5. Специфікація міжсервісних контрактів та топологія API-маршрутів

### 2.5.1. Контракт OpenAPI 3.0.3

Контракти спроєктовано за підходом API-First у форматі OpenAPI 3.0.3.
Деталізовану специфікацію ендпоінтів, схем даних, авторизації та
помилок зафіксовано у файлі [openapi.yaml](./api/openapi.yaml).

| Операція | Взаємодія контейнерів ([C4 Level 2](#diagram-level2)) | Авторизація | Успіх | Документовані помилки | Вимоги |
|---|---|---|---|---|---|
| `GET /api/v1/lots` | Mobile App → API Gateway → Core API | JWT Bearer | 200 | 400, 401, 429, 500 | [SRS-FUN-05](./srs.md#srs-fun-05), [SRS-NFR-PERF-01](./srs.md#srs-nfr-perf-01) |
| `POST /integration/v1/lots` | B2B-система → API Gateway → Core API | HMAC-SHA256 (`X-Signature`) | 200 / 201 | 400, 401, 404, 409, 413, 429, 500 | [SRS-FUN-01](./srs.md#srs-fun-01), [SRS-FUN-02](./srs.md#srs-fun-02) |
| `POST /api/v1/reservations` | Mobile App → API Gateway → Trading Service | JWT Bearer + `Idempotency-Key` | 201 | 400, 401, 404, 409, 410, 413, 429, 503 | [SRS-FUN-09](./srs.md#srs-fun-09), [SRS-NFR-PERF-02](./srs.md#srs-nfr-perf-02) |
| `GET /api/v1/orders/{order_id}` | Mobile App → API Gateway → Trading Service | JWT Bearer | 200 | 401, 404, 500 | [SRS-FUN-11](./srs.md#srs-fun-11) |
| `POST /api/v1/limit-orders` | Mobile App → API Gateway → Trading Service | JWT Bearer + `Idempotency-Key` | 201 | 400, 401, 404, 409, 410, 413, 429, 500 | [SRS-FUN-13](./srs.md#srs-fun-13) |
| `DELETE /api/v1/limit-orders/{limit_order_id}` | Mobile App → API Gateway → Trading Service | JWT Bearer | 204 | 401, 404, 409, 500 | [SRS-FUN-13](./srs.md#srs-fun-13) |
| `POST /api/v1/payments/liqpay/callback` | LiqPay → API Gateway → Trading Service | підпис LiqPay | 200 | 400, 413, 500 | [SRS-FUN-11](./srs.md#srs-fun-11), [SRS-FUN-12](./srs.md#srs-fun-12) |

Наскрізні правила контракту: помилки повертаються у форматі RFC 9457
(`application/problem+json`, схема `Problem` / `ValidationProblem`);
відповідь 429 містить заголовки `Retry-After`, `RateLimit-Limit`,
`RateLimit-Remaining` ([SRS-FUN-16](./srs.md#srs-fun-16)); операції,
що змінюють стан від імені користувача, вимагають заголовка
`Idempotency-Key`; кожна операція приймає `X-Correlation-Id`
([SRS-NFR-OBS-01](./srs.md#srs-nfr-obs-01)). Внутрішня взаємодія
Pricing Worker → Trading Service виконується не через REST, а через
задачі BullMQ `auto-reserve` (Redis, RESP), тому до HTTP-контракту не
входить.

### 2.5.2. Карта інтерфейсних маршрутів системи

Таблиця 1:1 відповідає [openapi.yaml](./api/openapi.yaml); типи
наведено в нотації OpenAPI (`uuid` — `string, format: uuid`;
`int64` — `integer, format: int64`).

| ID | Метод | URI | Вхідні параметри / Payload | Авторизація | Успіх / помилки | Контейнер-обробник ([C4 Level 2](#diagram-level2)) |
|---|---|---|---|---|---|---|
| API-REQ-01 | GET | `/api/v1/lots` | query: `lat` double [−90; 90], `lon` double [−180; 180], `radius_km` double [0,5; 10] = 3, `limit` int32 [1; 100] = 20, `cursor` string ≤ 128 | JWT Bearer | 200 `LotPage`; 400, 401, 429, 500 | Core API |
| API-REQ-02 | POST | `/integration/v1/lots` | header `X-Signature`, `X-Request-Timestamp` int64; body `LotFeedItem` (9 полів, `quantity` int32 [1; 50], суми int64 ≥ 1) | HMAC-SHA256 | 200 / 201 `LotUpsertResult`; 400, 401, 404, 409, 413, 429, 500 | Core API |
| API-REQ-03 | POST | `/api/v1/reservations` | header `Idempotency-Key` uuid; body `{lot_id: uuid}`, `additionalProperties: false` | JWT Bearer | 201 `Reservation` + `Location`; 400, 401, 404, 409, 410, 413, 429, 503 | Trading Service |
| API-REQ-04 | GET | `/api/v1/orders/{order_id}` | path `order_id` uuid | JWT Bearer + перевірка власника | 200 `Order`; 401, 404, 500 | Trading Service |
| API-REQ-05 | POST | `/api/v1/limit-orders` | header `Idempotency-Key` uuid; body `{lot_id: uuid, threshold_price_minor: int64 ≥ 1, valid_until?: date-time}` | JWT Bearer | 201 `LimitOrder` + `Location`; 400, 401, 404, 409, 410, 413, 429, 500 | Trading Service |
| API-REQ-06 | DELETE | `/api/v1/limit-orders/{limit_order_id}` | path `limit_order_id` uuid | JWT Bearer + перевірка власника | 204; 401, 404, 409, 500 | Trading Service |
| API-REQ-07 | POST | `/api/v1/payments/liqpay/callback` | `application/x-www-form-urlencoded`: `data` base64 ≤ 8192, `signature` ≤ 64 | підпис LiqPay | 200; 400, 413, 500 | Trading Service |

Асинхронні канали, що не є HTTP-маршрутами й специфікуються в
Розділі 3 (AsyncAPI):

| ID | Канал | Повідомлення | Виробник → споживач |
|---|---|---|---|
| API-WS-01 | WSS `/ws`, кімната `venue:{venue_id}` | `lot.price_updated`, `lot.stock_updated` | Realtime Gateway → Mobile App |
| API-Q-01 | черга BullMQ `auto-reserve` | `{limit_order_id, user_id, lot_id}` | Pricing Worker → Trading Service |
| API-Q-02 | Redis pub/sub `lot-events` | події з `outbox_events` | Pricing Worker (outbox relay) → Realtime Gateway |

### 2.5.3. Наскрізні потоки даних (Data Flows)

**DF-01. Резервування одиниці лота (API-REQ-03).**

1. Mobile App надсилає `POST /api/v1/reservations` по HTTPS (TLS 1.3)
   з заголовками `Authorization: Bearer <JWT>`, `Idempotency-Key`,
   `X-Correlation-Id`; тіло JSON ≤ 16 KiB.
2. API Gateway (Traefik) завершує TLS, відхиляє тіло понад 16 KiB
   кодом 413 (middleware `buffering.maxRequestBodyBytes`), застосовує
   rate limit 10 запитів/хв на користувача для `/api/v1/reservations`
   і проксіює запит у Trading Service по HTTP у внутрішній мережі
   Docker.
3. Trading Service: `TelemetryInterceptor` (генерує `X-Correlation-Id`,
   якщо його немає) → `JwtAuthGuard` (RS256,
   `exp`, `iss`, `aud`) → `IdempotencyInterceptor` → `ValidationPipe`
   (`whitelist`, `forbidNonWhitelisted`) → `ReservationController`.
4. `ReserveLotUnitUseCase` виконує транзакцію A2 у PostgreSQL (SQL/TCP
   5432) і записує подію в `outbox_events`.
5. Після COMMIT — виклик LiqPay API по HTTPS (тайм-аут 3 с, Circuit
   Breaker); відповідь `201` з `checkout_url`.

**DF-02. Прийом потоку лотів від B2B-системи (API-REQ-02).**

1. B2B-система обчислює HMAC-SHA256 від `X-Request-Timestamp + "." + body`
   і надсилає `POST /integration/v1/lots` (тіло ≤ 64 KiB).
2. API Gateway застосовує ліміт 600 запитів/хв на B2B-клієнта і
   проксіює запит у Core API.
3. Core API перевіряє, що `|now − X-Request-Timestamp| ≤ 300` с,
   порівнює підпис зі сталим часом і записує хеш підпису в Redis
   (`SET NX EX 600`); повторний підпис → 401.
4. Upsert за `(venue_external_id, external_lot_id)` і генерація
   `quantity` записів `lot_units` в одній транзакції; подія
   `lot.created` у Redis.

**DF-03. Поширення зміни ціни (API-WS-01).**

1. Pricing Worker (A1) пакетно оновлює `lots.current_price_minor` і
   вставляє події в `outbox_events` в одній транзакції.
2. Outbox relay читає непубліковані події (`published_at IS NULL`,
   частковий індекс) і публікує їх у Redis pub/sub `lot-events`.
3. Realtime Gateway (Redis-адаптер Socket.IO) розсилає подію в кімнату
   `venue:{venue_id}`; повідомлення ≤ 1 KiB.
4. Mobile App, автентифікований на handshake за JWT
   ([SRS-FUN-20](./srs.md#srs-fun-20)), оновлює ціну; цільова затримка
   p95 ≤ 2 с ([SRS-NFR-PERF-03](./srs.md#srs-nfr-perf-03)).

### 2.5.4. Мережеві обмеження та інваріанти

| Інваріант | Значення | Де забезпечується |
|---|---|---|
| Max Body Size | 16 KiB `/api/v1/*`; 64 KiB `/integration/v1/*`; 16 KiB callback LiqPay | Traefik `buffering`, відповідь 413 |
| Пагінація | курсорна, `limit` ≤ 100, `next_cursor` у відповіді; списки без пагінації відсутні | API-REQ-01 |
| CORS | заборонено за замовчуванням; дозволено лише origin адмін-інтерфейсу (Swagger UI, Grafana) для методів GET, POST, DELETE; мобільний клієнт CORS не використовує | Traefik `headers.accessControlAllowOriginList` |
| Rate Limiting | 100 запитів/хв на користувача; 10/хв для резервувань; 600/хв для B2B | Traefik `rateLimit` + `@nestjs/throttler` (сховище Redis) |
| Конфіденційні дані в URL | відсутні: токени лише в заголовку `Authorization`, ідентифікатор користувача — з JWT `sub`; координати округлюються до 3 знаків, query-рядок не журналюється | Mobile App, Traefik `accessLog.fields.headers` / `RequestPath` без query |
| Доступ до чужих об'єктів | 404 замість 403 для ресурсів іншого користувача | репозиторії: `WHERE id = $1 AND user_id = $2` |
| Захист від ін'єкцій | параметризовані запити (`$1`, `$2`) у всіх репозиторіях; валідація UUID до звернення до БД | `ValidationPipe`, адаптери репозиторіїв |

<a id="use-cases"></a>

## 2.6. Моделювання користувацького досвіду та транзакційних сценаріїв (Use Cases)

### 2.6.1. Діаграма прецедентів

Діаграму прецедентів побудовано в CASE-інструменті StarUML за
нотацією UML 2.5 (файл моделі [use-case-model.mdj](./uml/use-case-model.mdj),
експорт — [use-case-diagram.png](./uml/use-case-diagram.png)).

![Діаграма прецедентів B2C-підсистеми](./uml/use-case-diagram.png)

Межа системи (Subject): «B2C-підсистема маркетплейсу».

| Актор | Тип | Розташування | Прецеденти |
|---|---|---|---|
| Користувач (абстрактний) | первинний | ліворуч | UC-08 Автентифікуватися, UC-09 Видалити акаунт |
| Покупець | первинний, узагальнення від «Користувач» | ліворуч | UC-01, UC-03, UC-04, UC-05, UC-06 |
| Адміністратор платформи | первинний, узагальнення від «Користувач» | ліворуч | UC-10 Налаштувати параметри Price Decay |
| B2B-система (Тема 2) | первинний (система-ініціатор) | ліворуч | UC-11 Передати потік лотів |
| Планувальник (час) | первинний | ліворуч | UC-07 Перерахувати ціни лотів (включає UC-02 Виконати лімітний ордер), UC-14 Звільнити резерв після TTL |
| LiqPay | вторинний | праворуч | UC-01, UC-12 |
| Expo Push / FCM | вторинний | праворуч | UC-02 |

| ID | Прецедент | Зв'язки |
|---|---|---|
| UC-01 | Зарезервувати та оплатити одиницю лота | «include» UC-13 Перевірити ключ ідемпотентності; «extend» UC-12 Повернути кошти; «extend» UC-14 Звільнити резерв після TTL |
| UC-02 | Виконати лімітний ордер | «include» UC-01 (етап резервування) |
| UC-03 | Знайти лоти поблизу | — |
| UC-04 | Створити лімітний ордер | «include» UC-13 |
| UC-05 | Скасувати лімітний ордер | — |
| UC-06 | Отримувати оновлення цін | — |
| UC-07 | Перерахувати ціни лотів | «include» UC-02 |
| UC-08 | Автентифікуватися | — |
| UC-09 | Видалити акаунт | — |
| UC-10 | Налаштувати параметри Price Decay | — |
| UC-11 | Передати потік лотів | — |
| UC-12 | Повернути кошти | «extend» → UC-01 (точка розширення «callback після спливу TTL») |
| UC-13 | Перевірити ключ ідемпотентності | включається в UC-01, UC-04 |
| UC-14 | Звільнити резерв після TTL | «extend» → UC-01 (точка розширення «оплату не підтверджено за 10 хв») |

### 2.6.2. UC-01: Зарезервувати та оплатити одиницю лота

| Поле | Значення |
|---|---|
| ID та назва | UC-01: Зарезервувати та оплатити одиницю лота |
| Первинний актор | Покупець |
| Вторинні актори | LiqPay (hosted checkout, callback) |
| Короткий опис | Покупець фіксує поточну ціну лота, атомарно отримує одну одиницю й оплачує її; система гарантує відсутність подвійного продажу ([SRS-FUN-09](./srs.md#srs-fun-09), [SRS-FUN-11](./srs.md#srs-fun-11)) |

**Preconditions.**

| Тип | Передумова |
|---|---|
| Продуктова | Покупець автентифікований: access JWT (RS256) дійсний, роль `consumer`, `users.deleted_at IS NULL` |
| Продуктова | Лот має `status = 'active'`, `expires_at > now()`, існує щонайменше одна `lot_units.status = 'available'` |
| Продуктова | На екрані лота відображено `current_price_minor`, отриманий з `GET /api/v1/lots` або події `lot.price_updated` |
| Інфраструктурна | Trading Service має вільне з'єднання в пулі PostgreSQL; health-check `/health` повертає 200 |
| Інфраструктурна | Ключі LiqPay (public/private) завантажено з Docker secrets; Circuit Breaker адаптера `PaymentGatewayClient` у стані closed |
| Інфраструктурна | Лічильник rate limit користувача < 10 резервувань за поточну хвилину ([SRS-FUN-16](./srs.md#srs-fun-16)) |

**Main Success Scenario (Happy Path).**

| Крок | Дія актора | Реакція та відповідь системи |
|---|---|---|
| 1 | Покупець ініціює резервування лота | Mobile App генерує `Idempotency-Key` (UUID v4) і надсилає `POST /api/v1/reservations` з `{lot_id}`; API Gateway перевіряє TLS 1.3 і rate limit, передає запит у Trading Service |
| 2 | — | `JwtAuthGuard` перевіряє підпис RS256 і строк дії; `user_id := sub`; `ValidationPipe` відхиляє зайві поля; `IdempotencyInterceptor` не знаходить ключа в `idempotency_keys` |
| 3 | — | `BEGIN`; `SELECT … FROM lots WHERE id = $1` — лот активний; `SELECT id FROM lot_units … FOR UPDATE SKIP LOCKED LIMIT 1` повертає одиницю |
| 4 | — | `UPDATE lot_units SET status = 'reserved', reserved_by, reserved_until = now() + 10 хв`; `INSERT orders (status 'pending_payment', price_minor := current_price_minor)`; `INSERT order_status_history`; `INSERT outbox_events ('lot.stock_updated')`; `INSERT idempotency_keys`; `COMMIT` |
| 5 | — | `PaymentGatewayClient` викликає LiqPay API (тайм-аут 3 с), отримує `checkout_url`; `INSERT payments (status 'created')`; відповідь `201 Reservation` з `order_id`, `price_minor`, `reserved_until`, `checkout_url` |
| 6 | Покупець переходить на hosted checkout і вводить дані картки на стороні LiqPay | Дані картки не надходять у систему ([SRS-NFR-COMP-02](./srs.md#srs-nfr-comp-02)) |
| 7 | — | LiqPay надсилає `POST /api/v1/payments/liqpay/callback`; `ConfirmPaymentUseCase` перевіряє підпис `base64(sha1(private_key + data + private_key))` порівнянням зі сталим часом, декодує `data` |
| 8 | — | `BEGIN`; `SELECT … FROM orders WHERE id = $1 FOR UPDATE`; перехід `pending_payment → paid`; `UPDATE lot_units SET status = 'sold'`; `UPDATE payments SET status = 'success', provider_payment_id`; `INSERT order_status_history`; `INSERT outbox_events ('order.paid')`; `COMMIT`; відповідь LiqPay `200` |
| 9 | Покупець повертається в застосунок | Mobile App виконує `GET /api/v1/orders/{order_id}` і отримує `status = 'paid'`; Pricing Worker публікує `order.paid` у B2B-систему через webhook ([SRS-FUN-12](./srs.md#srs-fun-12)) |

**Postconditions.**

| Тип | Постумова |
|---|---|
| Дані | `orders.status = 'paid'`, `lot_units.status = 'sold'`, `payments.status = 'success'` з `provider_payment_id` |
| Аудит | В `order_status_history` два записи: `NULL → pending_payment`, `pending_payment → paid` з `correlation_id` |
| Інтеграція | Події `lot.stock_updated` і `order.paid` в `outbox_events` мають `published_at IS NOT NULL`; B2B-система отримала підтвердження продажу |
| Ідемпотентність | Відповідь кроку 5 збережено в `idempotency_keys` на 24 год |

**Alternative Flows.**

| ID | Тип | Старт із кроку | Умова-тригер | Обробка | Повернення на крок |
|---|---|---|---|---|---|
| AF-01A | 1 (вибір користувача) | 3 | Вільних одиниць немає після одного повтору (`409 sold-out`) | Mobile App пропонує створити лімітний ордер на цей лот; Покупець задає поріг → `POST /api/v1/limit-orders` (UC-04) | завершення UC-01; продовження в UC-02 |
| AF-01B | 2 (бізнес-валідація) | 3 | `lots.status ≠ 'active' OR now() ≥ expires_at` | `ROLLBACK`; `410 Gone` (RFC 9457 `.../errors/lot-expired`); Mobile App видаляє лот зі списку | 1 (вибір іншого лота) |
| AF-01C | 2 (бізнес-валідація) | 2 | Ключ `Idempotency-Key` існує з тим самим тілом (повтор після обриву мережі) | Повертається збережена відповідь `201` без нового резерву | 6 |
| AF-01D | 2 (бізнес-валідація) | 1 | Перевищено 10 резервувань/хв | API Gateway / Throttler повертає `429` з `Retry-After`; Mobile App блокує кнопку на вказаний час | 1 |
| AF-01E | 1 (вибір користувача) | 6 | Покупець закриває checkout без оплати | Стан не змінюється; `GET /api/v1/orders/{order_id}` повертає `checkout_url` до спливу `reserved_until` | 6 |

**Exception Flows.**

EF-01A — Відмова СУБД під час резервування (крок 3–4).

| Крок | Дії для UI / Користувача | Технічні дії сервера та БД |
|---|---|---|
| 1 | Індикатор очікування | Помилка з'єднання або deadlock → `ROLLBACK`; блокування `FOR UPDATE` знімається автоматично; повтор до 2 разів (алгоритм A2) |
| 2 | «Сервіс тимчасово недоступний, повторіть через N с» | Після вичерпання повторів — `503` з `Retry-After: 5`; жодна одиниця не змінює статус; подія `reservation_failed` у логах з `correlation_id` |
| 3 | Повторне натискання | Запит із тим самим `Idempotency-Key`; ключ не збережено (транзакція відкочена) → сценарій стартує з кроку 2 |

EF-01B — Недоступність LiqPay під час створення платежу (крок 5).

| Крок | Дії для UI / Користувача | Технічні дії сервера та БД |
|---|---|---|
| 1 | «Одиницю зарезервовано на 10 хв, платіжна сторінка готується» | Тайм-аут 3 с або відкритий Circuit Breaker; резерв уже зафіксовано (COMMIT кроку 4); відповідь `201` з `checkout_url = null` |
| 2 | Автоматичне оновлення кожні 5 с через `GET /api/v1/orders/{order_id}` (лише читання) | Trading Service записує outbox-подію `payment.create_requested`; фонова задача повторює створення платежу з exponential backoff, коли Circuit Breaker переходить у стан half-open; після успіху `checkout_url` з'являється у відповіді `GET` |
| 3 | «Платіж не створено, резерв скасовано» | Якщо до `reserved_until` платіж не створено — UC-14: `lot_units.status := 'available'`, `orders.status := 'expired'`, запис в `order_status_history` |

EF-01C — Успішна оплата після спливу резерву (крок 7–8).

| Крок | Дії для UI / Користувача | Технічні дії сервера та БД |
|---|---|---|
| 1 | — | Callback `success` надходить, коли `orders.status = 'expired'` (одиницю вже звільнено й, можливо, продано іншому покупцю) |
| 2 | — | Перехід `expired → paid` заборонено доменною моделлю; `UPDATE payments SET status = 'success'`; `INSERT outbox_events ('payment.refund_requested')`; відповідь LiqPay `200` |
| 3 | Push і екран замовлення: «Оплата надійшла після завершення резерву, кошти повертаються» | Pricing Worker викликає LiqPay refund API (до 3 спроб, exponential backoff); `payments.status := 'reversed'`, `orders.status := 'refunded'` (UC-12); при невдачі — алерт адміністратору в Grafana |

EF-01D — Відмова СУБД під час обробки callback (крок 8).

| Крок | Дії для UI / Користувача | Технічні дії сервера та БД |
|---|---|---|
| 1 | Статус «очікує підтвердження оплати» | `ROLLBACK`; Trading Service повертає LiqPay `500` |
| 2 | — | LiqPay повторює callback; обробка ідемпотентна за `provider_payment_id`; після відновлення БД сценарій продовжується з кроку 8 |
| 3 | — | Якщо до `reserved_until` callback не оброблено — після відновлення виконується EF-01C |

### 2.6.3. UC-02: Виконати лімітний ордер

| Поле | Значення |
|---|---|
| ID та назва | UC-02: Виконати лімітний ордер |
| Первинний актор | Планувальник (задача Pricing Worker кожні 60 с) |
| Вторинні актори | Expo Push / FCM |
| Короткий опис | Коли ціна лота знижується до порога, система автоматично резервує одиницю для власника ордера й надсилає сповіщення з пропозицією оплатити протягом 10 хв ([SRS-FUN-14](./srs.md#srs-fun-14)). Автоматичне списання коштів не виконується: для нього потрібна токенізація картки, що виходить за межі PCI DSS-обсягу системи |

**Preconditions.**

| Тип | Передумова |
|---|---|
| Продуктова | Існує `limit_orders.status = 'active'` з `valid_until > now()`; власник ордера не видалений |
| Продуктова | Лот активний, має вільні одиниці; `threshold_price_minor ≥` нової ціни |
| Інфраструктурна | Repeatable job `price-decay` зареєстровано в BullMQ; Redis доступний (AOF увімкнено) |
| Інфраструктурна | `AutoReserveConsumer` у Trading Service підписаний на чергу `auto-reserve` |
| Інфраструктурна | Для власника ордера збережено `users.expo_push_token` |

**Main Success Scenario (Happy Path).**

| Крок | Дія актора | Реакція та відповідь системи |
|---|---|---|
| 1 | Планувальник запускає цикл | Pricing Worker виконує A1 для пачки з 500 лотів; формує `Updates` |
| 2 | — | Після COMMIT пачки викликається A3: `SELECT … FROM limit_orders WHERE lot_id = $1 AND status = 'active' AND threshold_price_minor ≥ $2 AND valid_until > now() ORDER BY created_at LIMIT A` |
| 3 | — | `UPDATE limit_orders SET status = 'triggered' WHERE id = $1 AND status = 'active'` (оновлено 1 рядок); задача `auto-reserve` з `jobId := limit_order_id` |
| 4 | — | `AutoReserveConsumer` виконує `ReserveLotUnitUseCase` (кроки 2–5 UC-01) з `idem_key := limit_order_id`; отримує `201` |
| 5 | — | `UPDATE limit_orders SET status = 'executed', order_id = $1`; `INSERT outbox_events ('limit_order.executed')` |
| 6 | — | Pricing Worker (outbox relay) надсилає push через Expo Push API: «Лот зарезервовано за X грн, оплатіть до HH:MM» |
| 7 | Покупець відкриває сповіщення | Mobile App відкриває замовлення; далі — кроки 6–9 UC-01 |

**Postconditions.**

| Тип | Постумова |
|---|---|
| Дані | `limit_orders.status = 'executed'`, `order_id` вказує на замовлення `pending_payment`; одиниця лота `reserved` |
| Інтеграція | Подію `limit_order.executed` опубліковано; push надіслано (статус відповіді Expo збережено в логах) |

**Alternative Flows.**

| ID | Тип | Старт із кроку | Умова-тригер | Обробка | Повернення на крок |
|---|---|---|---|---|---|
| AF-02A | 1 (вибір користувача) | 3 | Покупець скасовує ордер одночасно з запуском (`DELETE /api/v1/limit-orders/{id}`) | Умовний `UPDATE … AND status = 'active'` оновлює 0 рядків для одного з двох запитів; якщо першим спрацював Worker — `DELETE` повертає `409` | завершення UC-02 |
| AF-02B | 2 (бізнес-валідація) | 4 | Резервування повертає `409 sold-out` або `410` | `status := 'failed'`; push «Лот розпродано до досягнення вашої ціни» | завершення UC-02 |
| AF-02C | 2 (бізнес-валідація) | 2 | `valid_until ≤ now()` | Ордер не вибирається; наприкінці A3 `status := 'expired'` | завершення UC-02 |

**Exception Flows.**

EF-02A — Падіння Trading Service під час обробки задачі (крок 4).

| Крок | Дії для UI / Користувача | Технічні дії сервера та БД |
|---|---|---|
| 1 | — | Задача не підтверджена (stalled job); BullMQ повертає її в чергу після `stalledInterval` 30 с |
| 2 | — | Повтор із тим самим `idem_key`: якщо резерв уже створено, A2 повертає збережену відповідь, дубль не виникає |
| 3 | Push «Не вдалося зарезервувати лот» | Після 3 невдалих спроб — `status := 'failed'`, задача в dead-letter queue, алерт у Grafana |

EF-02B — Недоступність Expo Push (крок 6).

| Крок | Дії для UI / Користувача | Технічні дії сервера та БД |
|---|---|---|
| 1 | Замовлення видно в розділі «Мої замовлення» | Outbox-подія залишається з `published_at IS NULL`; relay повторює відправку з exponential backoff до 5 спроб |
| 2 | — | Резерв не залежить від доставки push; після `reserved_until` діє UC-14 |

<a id="data-model"></a>

## 2.7. Проєктування та документування моделей даних

Шар збереження даних реалізовано в одному вузлі PostgreSQL 16 з
розширенням PostGIS (контейнер «PostgreSQL 16 + PostGIS»,
[C4 Container](#diagram-level2)). Схема відповідає третій нормальній
формі; єдиний свідомо збережений похідний атрибут —
`lots.current_price_minor` (кеш функції Price Decay, оновлюється
раз на 60 с алгоритмом A1, див. 2.3.4). Доменна сутність PriceSnapshot
з підрозділу 1.4.2 реалізована полями `current_price_minor` і
`price_updated_at` лота: історія цін не зберігається, оскільки ціна
детерміновано відновлюється з параметрів лота (10 000 лотів × 1 440
циклів на добу дали б 14,4 млн рядків на добу).

Конвенції: первинні ключі — `uuid` (v7, монотонні для B-Tree), крім
журнальних таблиць (`bigint GENERATED ALWAYS AS IDENTITY`); суми —
`bigint` у копійках (`*_minor`) + `char(3)` ISO 4217; мітки часу —
`timestamptz` (UTC); статуси — типи `ENUM`; `varchar_N` на діаграмі
означає `varchar(N)`, `numeric_P_S` — `numeric(P,S)`.

### 2.7.1. ER-діаграма (Mermaid, Crow's Foot)

```mermaid
erDiagram
  USERS ||--o{ REFRESH_TOKENS : "має"
  USERS ||--o{ ORDERS : "оформлює"
  USERS ||--o{ LIMIT_ORDERS : "створює"
  USERS ||--o{ IDEMPOTENCY_KEYS : "надсилає"
  VENUES ||--o{ LOTS : "публікує"
  LOTS ||--|{ LOT_UNITS : "складається з"
  LOTS ||--o{ LIMIT_ORDERS : "цільовий лот"
  LOT_UNITS ||--o| ORDERS : "продається як"
  ORDERS ||--o| PAYMENTS : "оплачується"
  ORDERS ||--|{ ORDER_STATUS_HISTORY : "журнал статусів"
  LIMIT_ORDERS |o--o| ORDERS : "виконується в"

  USERS {
    uuid id PK
    citext email UK "unique, RFC 5321 до 254 символів"
    text password_hash "Argon2id"
    varchar_64 display_name
    varchar_255 expo_push_token "NULL, якщо сповіщення вимкнено"
    user_role role "consumer | admin"
    integer impact_points "CHECK >= 0"
    timestamptz created_at
    timestamptz updated_at
    timestamptz deleted_at "soft delete + анонімізація"
  }
  REFRESH_TOKENS {
    uuid id PK
    uuid user_id FK
    bytea token_hash UK "SHA-256"
    uuid family_id "ланцюжок ротації"
    timestamptz expires_at
    timestamptz revoked_at
    timestamptz created_at
  }
  VENUES {
    uuid id PK
    varchar_64 external_id UK "ID закладу в B2B-системі (venue_external_id)"
    varchar_120 name
    geography location "Point, SRID 4326"
    time opens_at
    time closes_at
    numeric_4_2 decay_k "CHECK > 0"
    numeric_3_2 min_price_ratio "CHECK 0..1"
    timestamptz created_at
    timestamptz updated_at
    timestamptz deleted_at
  }
  LOTS {
    uuid id PK
    uuid venue_id FK
    varchar_64 external_lot_id "UK (venue_id, external_lot_id)"
    varchar_160 title
    bigint base_price_minor "копійки, CHECK > 0"
    bigint min_price_minor "CHECK <= base_price_minor"
    bigint current_price_minor "кеш Price Decay"
    char_3 currency "ISO 4217, UAH"
    smallint quantity "CHECK 1..50"
    timestamptz published_at
    timestamptz expires_at "CHECK > published_at"
    lot_status status "active | sold_out | expired | withdrawn"
    timestamptz price_updated_at
    integer version "оптимістичне блокування"
    timestamptz created_at
    timestamptz updated_at
  }
  LOT_UNITS {
    uuid id PK
    uuid lot_id FK
    unit_status status "available | reserved | sold"
    uuid reserved_by FK "users.id, NULL якщо available"
    timestamptz reserved_until
    timestamptz updated_at
  }
  ORDERS {
    uuid id PK
    uuid user_id FK
    uuid lot_unit_id FK "частковий UK WHERE status IN (pending_payment, paid)"
    bigint price_minor "ціна фіксації"
    char_3 currency
    order_status status "pending_payment | paid | expired | cancelled | refunded"
    timestamptz created_at
    timestamptz updated_at
  }
  PAYMENTS {
    uuid id PK
    uuid order_id FK "UK"
    varchar_64 provider_payment_id UK "LiqPay payment_id, NULL до callback"
    payment_status status "created | success | failure | reversed"
    bigint amount_minor
    char_3 currency
    jsonb callback_payload "останній верифікований callback"
    timestamptz created_at
    timestamptz updated_at
  }
  ORDER_STATUS_HISTORY {
    bigint id PK
    uuid order_id FK
    order_status from_status
    order_status to_status
    uuid actor_user_id FK "NULL для системних переходів"
    varchar_64 correlation_id
    timestamptz changed_at
  }
  LIMIT_ORDERS {
    uuid id PK
    uuid user_id FK
    uuid lot_id FK
    bigint threshold_price_minor "CHECK > 0"
    limit_order_status status "active | triggered | executed | cancelled | expired | failed"
    uuid order_id FK "NULL до виконання"
    timestamptz valid_until
    timestamptz created_at
    timestamptz updated_at
  }
  IDEMPOTENCY_KEYS {
    uuid key PK "заголовок Idempotency-Key"
    uuid user_id FK
    varchar_64 request_hash "SHA-256 тіла запиту"
    smallint response_status
    jsonb response_body
    timestamptz expires_at "TTL 24 год"
  }
  OUTBOX_EVENTS {
    bigint id PK
    uuid aggregate_id
    varchar_64 event_type "lot.stock_updated, order.paid"
    jsonb payload
    smallint attempts
    timestamptz created_at
    timestamptz published_at "NULL до публікації"
  }
```

### 2.7.2. Специфікація таблиць

**ENUM-типи:** `user_role` (consumer, admin); `lot_status` (active,
sold_out, expired, withdrawn); `unit_status` (available, reserved,
sold); `order_status` (pending_payment, paid, expired, cancelled,
refunded); `payment_status` (created, success, failure, reversed);
`limit_order_status` (active, triggered, executed, cancelled,
expired, failed).

| Таблиця | Призначення | Ключові обмеження | Вимоги |
|---|---|---|---|
| `users` | обліковий запис покупця або адміністратора | `email` CITEXT UNIQUE; `impact_points >= 0`; `deleted_at` + анонімізація персональних полів протягом 30 діб | [SRS-FUN-03](./srs.md#srs-fun-03), [SRS-FUN-17](./srs.md#srs-fun-17), [SRS-FUN-18](./srs.md#srs-fun-18) |
| `refresh_tokens` | ротація refresh-токенів | `token_hash` UNIQUE; `family_id` для відкликання ланцюжка при повторному використанні | [SRS-FUN-04](./srs.md#srs-fun-04) |
| `venues` | заклад і параметри Price Decay | `external_id` UNIQUE; `decay_k > 0`; `0 <= min_price_ratio <= 1` | [SRS-FUN-05](./srs.md#srs-fun-05), [SRS-FUN-19](./srs.md#srs-fun-19) |
| `lots` | лот закладу, кеш поточної ціни | UNIQUE (`venue_id`, `external_lot_id`); `min_price_minor <= base_price_minor`; `expires_at > published_at`; `quantity` 1..50; `version` | [SRS-FUN-02](./srs.md#srs-fun-02), [SRS-FUN-07](./srs.md#srs-fun-07), [SRS-FUN-08](./srs.md#srs-fun-08) |
| `lot_units` | одиниця лота як об'єкт блокування | `reserved_by` / `reserved_until` NOT NULL тоді й лише тоді, коли `status = 'reserved'` (CHECK) | [SRS-FUN-09](./srs.md#srs-fun-09), [SRS-FUN-10](./srs.md#srs-fun-10), [ADR-001](./adr/0001-choice-of-technology.md) |
| `orders` | замовлення з ціною фіксації | частковий UNIQUE (`lot_unit_id`) WHERE `status IN ('pending_payment', 'paid')`: одна одиниця має не більше одного чинного замовлення, а після спливу резерву її можна продати повторно | [SRS-FUN-09](./srs.md#srs-fun-09), [SRS-FUN-11](./srs.md#srs-fun-11) |
| `payments` | платіж LiqPay | `order_id` UNIQUE; `provider_payment_id` UNIQUE, NULL до callback | [SRS-FUN-11](./srs.md#srs-fun-11), [SRS-NFR-COMP-02](./srs.md#srs-nfr-comp-02) |
| `order_status_history` | журнал аудиту переходів статусів | append-only; `correlation_id` | [SRS-NFR-SEC-05](./srs.md#srs-nfr-sec-05) |
| `limit_orders` | лімітний ордер | `threshold_price_minor > 0`; частковий UNIQUE (`user_id`, `lot_id`) WHERE `status = 'active'` | [SRS-FUN-13](./srs.md#srs-fun-13), [SRS-FUN-14](./srs.md#srs-fun-14) |
| `idempotency_keys` | збережені відповіді на повторні запити | PK `key`; TTL 24 год (очищення фоновою задачею) | [SRS-FUN-11](./srs.md#srs-fun-11) |
| `outbox_events` | transactional outbox | `published_at` NULL до відправки; `attempts` для backoff | [SRS-FUN-12](./srs.md#srs-fun-12), [SRS-NFR-REL-03](./srs.md#srs-nfr-rel-03) |

### 2.7.3. План індексування

| Індекс | Тип | Запит, який прискорює | Вимога |
|---|---|---|---|
| `venues (location)` | GiST | `ST_DWithin` у `GET /api/v1/lots` | [SRS-NFR-PERF-01](./srs.md#srs-nfr-perf-01) |
| `lots (venue_id) WHERE status = 'active'` | B-Tree, частковий | лоти закладів у радіусі | [SRS-NFR-PERF-01](./srs.md#srs-nfr-perf-01) |
| `lots (id) WHERE status = 'active'` | B-Tree, частковий | keyset-пагінація циклу A1 | [SRS-NFR-PERF-04](./srs.md#srs-nfr-perf-04) |
| `lot_units (lot_id, id) WHERE status = 'available'` | B-Tree, частковий | `FOR UPDATE SKIP LOCKED` в A2, `COUNT` в A3 | [SRS-NFR-PERF-02](./srs.md#srs-nfr-perf-02), [ADR-001](./adr/0001-choice-of-technology.md) |
| `lot_units (reserved_until) WHERE status = 'reserved'` | B-Tree, частковий | звільнення резервів після TTL | [SRS-FUN-10](./srs.md#srs-fun-10) |
| `limit_orders (lot_id, threshold_price_minor) WHERE status = 'active'` | B-Tree, частковий | пошук ордерів для виконання в A3 | [SRS-FUN-14](./srs.md#srs-fun-14) |
| `orders (user_id, created_at DESC)` | B-Tree | історія замовлень користувача | — |
| `outbox_events (id) WHERE published_at IS NULL` | B-Tree, частковий | outbox relay Pricing Worker | [SRS-NFR-REL-03](./srs.md#srs-nfr-rel-03) |
| `idempotency_keys (expires_at)` | B-Tree | очищення застарілих ключів | — |
| `refresh_tokens (user_id) WHERE revoked_at IS NULL` | B-Tree, частковий | відкликання сесій | [SRS-FUN-04](./srs.md#srs-fun-04) |

<a id="integration-audit"></a>

## 2.8. Інтеграційний аудит та висновок про готовність архітектури до реалізації

Проведено наскрізну звірку схеми даних (2.7), контракту
[openapi.yaml](./api/openapi.yaml) (2.5), алгоритмів A1–A3 (2.3.4), сценаріїв Use Case (2.6) та
[srs.md](./srs.md) за трьома осями: API ↔ БД, логіка ↔ API/БД,
вимоги ↔ артефакти.

### 2.8.1. Виявлені розбіжності та їх усунення

| № | Тип | Розбіжність | Усунення |
|---|---|---|---|
| 1 | API ↔ БД | API передає `venue_external_id`, у таблиці `venues` поле називалося `merchant_external_id` без обмеження унікальності | поле перейменовано на `venues.external_id` з `UNIQUE` |
| 2 | вимоги ↔ БД | SRS-FUN-02 задавала ключ ідемпотентності `(merchant_id, external_lot_id)`, а схема — `UNIQUE (venue_id, external_lot_id)` | SRS-FUN-02 узгоджено: ключ `(venue_external_id, external_lot_id)` |
| 3 | вимоги ↔ API | SRS-NFR-PERF-01 посилалася на `GET /lots/nearby`; за рекомендацією API-рев'ю шлях змінено на ресурс-колекцію | SRS-NFR-PERF-01 посилається на `GET /api/v1/lots` |
| 4 | БД (3NF) | `reserved_until` зберігалося і в `lot_units`, і в `orders` (аномалія оновлення при продовженні або звільненні резерву) | поле видалено з `orders`; `Order.reserved_until` в API береться з `lot_units` |
| 5 | логіка ↔ БД | мінімальна ціна мала два джерела: `lots.min_price_minor` (потік B2B) і `venues.min_price_ratio` (адміністратор, SRS-FUN-19); A1 використовував лише перше | A1: `P_min := MAX(min_price_minor, ROUND(base_price_minor · min_price_ratio))` |
| 6 | логіка ↔ БД | A2 створює запис `payments` до callback, а `provider_payment_id` був задекларований як обов'язковий | `provider_payment_id` допускає NULL до callback (UNIQUE ігнорує NULL у PostgreSQL) |
| 7 | Use Case ↔ БД | EF-01C і UC-14 звільняють одиницю після спливу резерву, а `orders.lot_unit_id UNIQUE` забороняв би нове замовлення на ту саму одиницю | частковий UNIQUE (`lot_unit_id`) WHERE `status IN ('pending_payment', 'paid')` |
| 8 | Use Case ↔ БД | UC-02 надсилає push власнику ордера, а схема не зберігала push-токен | додано `users.expo_push_token` |
| 9 | Use Case ↔ API | чернетка EF-01B повторно створювала платіж у `GET /api/v1/orders/{order_id}` (побічний ефект безпечного методу) | повтор створення платежу винесено в outbox-задачу `payment.create_requested`; `GET` лише читає |
| 10 | API ↔ безпека | `GET /api/v1/orders/{order_id}` і `DELETE /api/v1/limit-orders/{limit_order_id}` повертали 403 для чужих ресурсів, що дозволяє перебір ідентифікаторів | 403 замінено на 404 з однаковим тілом відповіді |

### 2.8.2. Перевірені інваріанти (без розбіжностей)

- Ідентифікатори: `uuid` у БД ↔ `string, format: uuid` в API.
- Гроші: `bigint` у копійках ↔ `integer, format: int64` (`*_minor`) + `currency` ISO 4217 в обох шарах; дробові типи не використовуються.
- Час: `timestamptz` ↔ `string, format: date-time` (RFC 3339, UTC).
- Статуси: множини значень ENUM `lot_status`, `order_status`, `limit_order_status` збігаються зі схемами `LotStatus`, `OrderStatus`, `LimitOrderStatus` і зі значеннями, які присвоюють A1–A3 (`expired`, `reserved`, `pending_payment`, `triggered`, `executed`, `failed`).
- Обмеження: `quantity` 1..50 (`smallint` + CHECK ↔ `minimum: 1, maximum: 50`); `radius_km` 0,5..10 (API ↔ SRS-FUN-05).
- Колекції та індекси в алгоритмах (`Lots[i]`, `Updates[j]`, `Orders[m]`) відповідають таблицям `lots`, `limit_orders` та повертаються запитами, підкріпленими індексами з 2.7.3.
- Ідемпотентність: `Idempotency-Key` (API) ↔ `idempotency_keys` (БД) ↔ кроки 2–4 A2 ↔ `jobId` A3.

### 2.8.3. Production Readiness Report

| Критерій | Статус |
|---|---|
| Кожна операція API має таблицю-джерело та типи без втрати точності | виконано |
| Кожен статус, який присвоюють алгоритми, існує в ENUM БД і в схемі API | виконано |
| Кожен алгоритм має обмеження повторів, обробку порожніх даних і помилок СУБД або зовнішнього сервісу | виконано |
| Кожна змінювальна операція ідемпотентна (Idempotency-Key, upsert за природним ключем або `jobId`) | виконано |
| Критичні бізнес-сценарії (SRS-FUN-01, 02, 05, 07–14, 16) мають відображення в API, БД та алгоритмах | виконано |
| SRS-FUN-03, 04, 06, 15, 17–20 мають відображення в схемі БД; їхні API-контракти не входять у критичний сценарій ЛР | відкладено до Розділу 3 |
| Критичні блокери (showstoppers) | відсутні |

Висновок: специфікації шару даних, міжсервісних контрактів і логіки
критичних вузлів узгоджені між собою та з [srs.md](./srs.md) і
[ADR-001](./adr/0001-choice-of-technology.md) і готові до реалізації
в Розділі 3. Некритичні пункти, що переносяться в реалізацію:
алгоритм звільнення резервів після TTL (SRS-FUN-10) реалізується
окремою задачею Pricing Worker за індексом
`lot_units (reserved_until)`; контракти автентифікації (SRS-FUN-03,
04), кластеризації (SRS-FUN-06), адміністративного API (SRS-FUN-19),
видалення акаунта (SRS-FUN-17) та WebSocket-подій (SRS-FUN-15, 20)
буде специфіковано разом з їхньою реалізацією.

## Висновки до розділу 2

Обґрунтовано сервісну архітектуру з чотирьох NestJS-сервісів зі
спільною базою даних PostgreSQL 16 + PostGIS, асинхронною взаємодією
через BullMQ і Redis pub/sub та Clean Architecture всередині
сервісів. Відмова від мікросервісів з окремими сховищами зумовлена
вимогою суворої узгодженості резервування (0 подвійних продажів), яку
забезпечує одна ACID-транзакція `SELECT … FOR UPDATE SKIP LOCKED`
(ADR-001), та обмеженням розгортання на одному вузлі. Масштабування
виконується горизонтально для stateless-сервісів Core API, Trading
Service і Realtime Gateway; архітектурні ризики (єдина точка відмови
СУБД і Redis, каскадна відмова платіжного провайдера, затримка
поширення змін) компенсовано RPO/RTO, AOF, transactional outbox,
тайм-аутами та Circuit Breaker.

Спроєктовано всі рівні системи: моделі C4 Level 1–3; серверне ядро з
портами й адаптерами та трьома алгоритмами критичних вузлів (Price
Decay, атомарне резервування, виконання лімітних ордерів); контур
безпеки з покриттям OWASP API Security Top 10 (2023); контракт
OpenAPI 3.0.3 із 7 операцій і топологією маршрутів з мережевими
обмеженнями; сценарії Use Case з альтернативними та виключними
потоками; схему даних з 11 таблиць у третій нормальній формі з
планом індексування. Наскрізний аудит усунув 10 розбіжностей між
вимогами, даними, контрактами, сценаріями та алгоритмами; критичних
блокерів не виявлено.

Специфікації Розділу 2 є вхідним контрактом для Розділу 3, де буде
виконано реалізацію сервісів, автоматизоване тестування вимог SRS
інструментами k6, Jest і Testcontainers, налаштування CI/CD та
вимірювання нефункціональних характеристик.
