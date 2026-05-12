## Individual Work — Fidel Akilah

### Order & Notification Module

Modul ini bertanggung jawab atas dua tanggung jawab terkait di BidMart:

1. **Order lifecycle** — saat sebuah lelang berakhir dan pemenangnya ditentukan,
   modul ini membuat record `Order` yang menautkan buyer, seller, listing, dan
   harga final, lalu memajukan order tersebut melalui status `PENDING_PAYMENT
   → PAID → SHIPPED → DELIVERED` (atau `CANCELLED`).
2. **Notifications** — modul ini juga mengirim notifikasi email / push kepada
   user untuk peristiwa penting (outbid, lelang dimenangkan, order dikirim,
   user di-banned, dll.) melalui satu Email/Push Provider eksternal.

Di arsitektur saat ini kedua tanggung jawab tersebut hidup di dalam satu Spring
Boot module dengan database PostgreSQL bersama, dan menerima event dari modul
Auction melalui Spring `ApplicationEventPublisher`. Di arsitektur masa depan
modul ini menjadi `Order & Notif Service` independen yang berlangganan ke
event `auction.winner_determined`, `bid.placed`, dan `user.banned` melalui
message broker.

---

### Container Diagram — Order & Notification (zoom dari Container Diagram grup)

Diagram ini memperbesar bagian "📦 Order & Notif Module" dari Container Diagram
keseluruhan grup. Inbound: REST dari Frontend (lihat history order, baca
notifikasi), event internal dari Auction Module, dan event dari Auth Module
ketika ada user di-banned. Outbound: tabel `orders` + `notifications` di
PostgreSQL bersama, dan API Email/Push Provider untuk pengiriman aktual.

```mermaid
graph TB
  Buyer["👤 Buyer"]
  Seller["👤 Seller"]
  FE["🌐 Next.js Frontend"]

  subgraph monolith ["Spring Boot Monolith (:8080) — fokus pada Order & Notif"]
    direction TB

    subgraph other ["Modul lain (sumber event)"]
      AUCTION["🔨 Auction Module<br/><i>publishes BidPlacedEvent, AuctionEndedEvent</i>"]
      AUTH["🔐 Auth Module<br/><i>publishes UserBannedEvent</i>"]
      WALLET["💰 Wallet Module<br/><i>publishes PaymentSettledEvent</i>"]
    end

    subgraph orderNotif ["📦 Order &amp; Notif Module"]
      direction TB
      ORDER_API["🚚 Order API<br/><i>/api/orders/**</i>"]
      NOTIF_API["🔔 Notification API<br/><i>/api/notifications/**</i>"]
      ORDER_CORE["⚙️ Order Lifecycle<br/><i>create, transition, cancel</i>"]
      NOTIF_CORE["⚙️ Notification Pipeline<br/><i>compose, queue, dispatch</i>"]
      EVT["📨 Event Listener<br/><i>@EventListener</i>"]
    end
  end

  DB[("🗄️ PostgreSQL<br/><i>orders, notifications</i>")]
  EMAIL["📧 Email / Push Provider<br/><i>SMTP / FCM / APNs</i>"]

  Buyer --> FE
  Seller --> FE
  FE -->|"REST"| ORDER_API
  FE -->|"REST (poll / WS)"| NOTIF_API

  AUCTION -.->|"Spring ApplicationEvent"| EVT
  AUTH -.->|"Spring ApplicationEvent"| EVT
  WALLET -.->|"Spring ApplicationEvent"| EVT

  EVT --> ORDER_CORE
  EVT --> NOTIF_CORE
  ORDER_API --> ORDER_CORE
  NOTIF_API --> NOTIF_CORE
  ORDER_CORE -->|"create / update"| DB
  NOTIF_CORE -->|"persist + status"| DB
  NOTIF_CORE -->|"SMTP / API"| EMAIL

  style orderNotif fill:#831843,color:#FCE7F3,stroke:#EC4899
  style other fill:#1E1B4B,color:#C7D2FE,stroke:#6366F1
  style ORDER_API fill:#EC4899,color:#fff,stroke:#DB2777
  style NOTIF_API fill:#EC4899,color:#fff,stroke:#DB2777
  style ORDER_CORE fill:#9333EA,color:#fff,stroke:#7E22CE
  style NOTIF_CORE fill:#9333EA,color:#fff,stroke:#7E22CE
  style EVT fill:#F59E0B,color:#fff,stroke:#D97706
  style DB fill:#3ECF8E,color:#fff,stroke:#22C55E
  style EMAIL fill:#7C3AED,color:#fff,stroke:#6D28D9
```

| From | To | Interaksi | Protokol |
| --- | --- | --- | --- |
| Frontend (Buyer/Seller) | `OrderController` | Lihat history order, detail order, tandai diterima | REST/JSON |
| Frontend | `NotificationController` | List unread notif, mark as read, kelola preferensi | REST/JSON |
| Auction Module | `AuctionEventListener` | `BidPlacedEvent`, `AuctionEndedEvent` (Spring event) | In-process |
| Auth Module | `AuctionEventListener` | `UserBannedEvent` — batalkan order pending milik user | In-process |
| Wallet Module | `AuctionEventListener` | `PaymentSettledEvent` — pindah order ke `PAID` | In-process |
| `NotificationDispatcher` | Email/Push Provider | Kirim email verifikasi, outbid, won, order shipped | SMTP / HTTPS |

---

### Code Diagram 1 — Component Diagram (Controllers, Services, Repositories)

Component diagram memperlihatkan struktur internal modul: controller membuka
endpoint REST, service membungkus logika bisnis, dispatcher mengisolasi
panggilan ke provider eksternal, dan event listener menjembatani modul ini
dengan modul lain di monolith.

```mermaid
graph TB
  FE["🌐 Frontend"]

  subgraph controllers ["Controllers"]
    OC["🚚 OrderController<br/>@RestController<br/>GET /api/orders, GET /api/orders/{id},<br/>POST /api/orders/{id}/mark-shipped,<br/>POST /api/orders/{id}/mark-delivered"]
    NC["🔔 NotificationController<br/>@RestController<br/>GET /api/notifications,<br/>POST /api/notifications/{id}/read,<br/>PATCH /api/notifications/preferences"]
  end

  subgraph services ["Services"]
    OS["⚙️ OrderService<br/>createOrder(WinnerInfo),<br/>transition(orderId, newStatus),<br/>cancelPendingOrdersOf(userId)"]
    NS["⚙️ NotificationService<br/>notifyOutbid(prevBidder, auction),<br/>notifyWinner(buyer, auction),<br/>notifyShipped(order),<br/>queueAndDispatch(notification)"]
    TS["📝 TemplateService<br/>render(templateId, model) → String"]
  end

  subgraph dispatch ["Dispatchers (Anti-Corruption Layer)"]
    ED["📧 EmailDispatcher<br/>send(to, subject, body)"]
    PD["📲 PushDispatcher<br/>send(deviceToken, title, body)"]
  end

  subgraph events ["Event Listeners"]
    AEL["📨 AuctionEventListener<br/>@EventListener BidPlacedEvent,<br/>AuctionEndedEvent"]
    AUL["📨 AuthEventListener<br/>@EventListener UserBannedEvent"]
    WEL["📨 WalletEventListener<br/>@EventListener PaymentSettledEvent"]
  end

  subgraph repos ["Repositories (JPA)"]
    OR["🗄️ OrderRepository<br/>findByBuyerId, findByStatusIn"]
    NR["🗄️ NotificationRepository<br/>findByUserIdAndStatus, countUnread"]
    PR["🗄️ NotificationPreferenceRepository<br/>findByUserId"]
  end

  DB[("🗄️ PostgreSQL")]
  EmailAPI["📧 Email Provider"]
  PushAPI["📲 FCM / APNs"]

  FE -->|"REST / Bearer JWT"| OC
  FE -->|"REST / Bearer JWT"| NC

  OC --> OS
  NC --> NS
  AEL --> OS
  AEL --> NS
  AUL --> OS
  WEL --> OS
  WEL --> NS

  OS --> OR
  NS --> NR
  NS --> PR
  NS --> TS
  NS --> ED
  NS --> PD

  OR --> DB
  NR --> DB
  PR --> DB
  ED --> EmailAPI
  PD --> PushAPI

  style controllers fill:#312E81,color:#C7D2FE,stroke:#6366F1
  style services fill:#374151,color:#F8FAFC,stroke:#4B5563
  style dispatch fill:#831843,color:#FCE7F3,stroke:#EC4899
  style events fill:#78350F,color:#FED7AA,stroke:#F59E0B
  style repos fill:#0EA5E9,color:#fff,stroke:#0284C7
```

Pemisahan `EmailDispatcher` dan `PushDispatcher` sebagai komponen tersendiri
penting untuk menjaga `NotificationService` tetap testable: di unit-test
keduanya dapat diganti dengan mock tanpa pernah memanggil provider asli.
Ketika nanti modul ini dipindah ke service tersendiri, dispatcher inilah
yang berubah implementasi (misal jadi adapter Kafka/Outbox pattern) tanpa
menyentuh logika bisnis di service.

---

### Code Diagram 2 — Class Diagram (Domain Model)

Class diagram menggambarkan entitas inti modul: `Order` sebagai aggregate root
yang punya state machine status, dan `Notification` sebagai event log yang
dipasangkan dengan `NotificationPreference` per user untuk menentukan channel
apa saja yang aktif.

```mermaid
classDiagram
  direction TB

  class Order {
    -Long id
    -Long auctionId
    -Long listingId
    -String buyerId
    -String sellerId
    -BigDecimal finalPrice
    -OrderStatus status
    -Instant createdAt
    -Instant updatedAt
    -Long version
    +transitionTo(OrderStatus next) void
    +cancel(String reason) void
  }

  class OrderStatus {
    <<enumeration>>
    PENDING_PAYMENT
    PAID
    SHIPPED
    DELIVERED
    CANCELLED
  }

  class Notification {
    -Long id
    -String userId
    -NotificationType type
    -NotificationChannel channel
    -String payloadJson
    -NotificationStatus status
    -int attempts
    -Instant createdAt
    -Instant sentAt
    +markSent() void
    +markFailed(String error) void
  }

  class NotificationType {
    <<enumeration>>
    OUTBID
    AUCTION_WON
    AUCTION_LOST
    ORDER_SHIPPED
    ORDER_DELIVERED
    USER_BANNED
  }

  class NotificationChannel {
    <<enumeration>>
    EMAIL
    PUSH
    IN_APP
  }

  class NotificationStatus {
    <<enumeration>>
    QUEUED
    SENT
    FAILED
    READ
  }

  class NotificationPreference {
    -String userId
    -boolean emailEnabled
    -boolean pushEnabled
    -boolean inAppEnabled
    +isChannelEnabled(NotificationChannel c) boolean
  }

  class NotificationTemplate {
    -NotificationType type
    -String emailSubjectTemplate
    -String emailBodyTemplate
    -String pushTitleTemplate
    -String pushBodyTemplate
  }

  Order --> OrderStatus
  Notification --> NotificationType
  Notification --> NotificationChannel
  Notification --> NotificationStatus
  NotificationPreference "1" --> "*" NotificationChannel : enabled set
  NotificationTemplate --> NotificationType
```

Catatan desain:

- `Order.version` (optimistic locking) penting agar transisi status concurrent
  dari Wallet event dan dari admin tidak saling menimpa.
- `Notification.payloadJson` sengaja disimpan sebagai snapshot (denormalisasi)
  supaya isi email tidak berubah walau data sumber (mis. listing dihapus)
  sudah berubah. Ini juga memudahkan re-send saat dispatcher gagal sementara.
- `NotificationTemplate` membuat copy-writing dapat di-update tanpa redeploy.

---

### Code Diagram 3 — Sequence Diagram (Auction Winner → Create Order → Notify)

Sequence diagram ini menelusuri alur paling kritis: ketika `BiddingService`
menentukan pemenang sebuah lelang, modul ini harus membuat `Order` baru dan
mengirim notifikasi "menang" ke buyer serta notifikasi "terjual" ke seller,
dalam satu transaksi yang aman dari double-emit.

```mermaid
sequenceDiagram
  participant BS as BiddingService<br/>(Auction Module)
  participant EP as ApplicationEventPublisher
  participant AEL as AuctionEventListener
  participant OS as OrderService
  participant NS as NotificationService
  participant OR as OrderRepository
  participant NR as NotificationRepository
  participant PR as NotificationPreferenceRepository
  participant ED as EmailDispatcher
  participant PD as PushDispatcher

  BS->>EP: publish AuctionEndedEvent(auctionId, winnerId, sellerId, finalPrice)
  EP->>AEL: dispatch event (sync, @TransactionalEventListener)

  AEL->>OS: createOrder(winnerInfo)
  OS->>OR: save(Order{status=PENDING_PAYMENT})
  OR-->>OS: Order persisted

  AEL->>NS: notifyWinner(winnerInfo, order)
  NS->>PR: findByUserId(winnerId)
  PR-->>NS: NotificationPreference(emailEnabled=true, pushEnabled=true)
  NS->>NR: save(Notification{type=AUCTION_WON, channel=EMAIL, status=QUEUED})
  NS->>NR: save(Notification{type=AUCTION_WON, channel=PUSH, status=QUEUED})
  NS->>ED: send(winner.email, "You won!", body)
  ED-->>NS: ok
  NS->>NR: update status=SENT
  NS->>PD: send(winner.deviceToken, title, body)
  PD-->>NS: ok
  NS->>NR: update status=SENT

  AEL->>NS: notifySeller(sellerInfo, order)
  Note over NS,NR: similar persist-then-dispatch flow,<br/>kembar untuk seller
```

Poin penting:

- Listener menggunakan `@TransactionalEventListener(phase = AFTER_COMMIT)` agar
  notifikasi hanya dikirim setelah transaksi `Order` ter-commit; ini mencegah
  notifikasi "menang" terkirim ketika order gagal disimpan karena rollback.
- Notification ditulis ke DB **sebelum** dispatcher dipanggil (Outbox-style)
  supaya jika provider down sementara, kita masih bisa re-dispatch lewat
  scheduler tanpa kehilangan kejadiannya.

---

### Code Diagram 4 — Sequence Diagram (BidPlaced → Outbid Notification)

Diagram terakhir menelusuri jalur yang paling sering jalan: setiap kali ada
bid baru, peserta sebelumnya (bidder tertinggi yang lama) harus diberi tahu
bahwa dirinya ter-outbid. Ini high-volume dan tidak boleh menahan critical
path bidding.

```mermaid
sequenceDiagram
  participant BS as BiddingService
  participant EP as ApplicationEventPublisher
  participant AEL as AuctionEventListener
  participant NS as NotificationService
  participant TS as TemplateService
  participant NR as NotificationRepository
  participant PR as NotificationPreferenceRepository
  participant ED as EmailDispatcher
  participant PD as PushDispatcher

  BS->>EP: publish BidPlacedEvent(auctionId, newBidderId, prevBidderId, amount)
  EP->>AEL: dispatch (async @Async)

  alt prevBidderId is null (first bid)
    AEL-->>AEL: skip
  else there is a previous top bidder
    AEL->>NS: notifyOutbid(prevBidderId, auctionId, amount)
    NS->>PR: findByUserId(prevBidderId)
    PR-->>NS: NotificationPreference
    NS->>TS: render(OUTBID, {auctionTitle, amount})
    TS-->>NS: rendered body

    Note over NS,NR: persist outbox row first
    NS->>NR: save(Notification{type=OUTBID, status=QUEUED})

    opt emailEnabled
      NS->>ED: send(user.email, subject, body)
      alt success
        NS->>NR: update status=SENT, sentAt=now
      else failure
        NS->>NR: update status=FAILED, attempts++<br/>(scheduler will retry)
      end
    end

    opt pushEnabled
      NS->>PD: send(user.deviceToken, title, body)
      alt success
        NS->>NR: update status=SENT, sentAt=now
      else failure
        NS->>NR: update status=FAILED, attempts++
      end
    end
  end
```

Catatan desain:

- Listener untuk `BidPlacedEvent` dipasangi `@Async` (di-execute oleh thread
  pool terpisah) agar pemanggilan `EmailDispatcher` yang lambat **tidak**
  memperlambat respons HTTP dari `BiddingController` ke pembidder.
- Karena retry dilakukan oleh scheduler yang membaca `Notification` dengan
  `status=FAILED && attempts<MAX`, kegagalan provider tidak menyebabkan
  notifikasi hilang. Inilah cikal-bakal pola **Transactional Outbox** yang
  nanti akan saya pakai saat modul ini dipindah ke message broker di
  arsitektur masa depan.
- Saat migrasi ke arsitektur microservices, `AuctionEventListener` cukup
  diganti menjadi consumer `bid.placed` dari RabbitMQ/Kafka — sisa logika
  (template, persistence, retry) tetap sama. Inilah yang membuat pemisahan
  antara *Event Listener* dan *Notification Pipeline* pada Component Diagram
  bernilai: ia membuat migrasi nantinya bersifat lokal, bukan rewrite.

---

### Korelasi antar diagram

- **Container Diagram** menempatkan modul ini dalam konteks monolith yang ada,
  menunjukkan boundary I/O-nya (Frontend di sisi inbound; database + email/push
  provider di sisi outbound; event Spring dari Auction/Auth/Wallet di sisi
  internal).
- **Component Diagram** memperbesar boundary tersebut menjadi tiga lapisan
  (Controllers → Services → Repositories) plus dua kotak khusus
  (`Dispatchers` dan `Event Listeners`) yang menjadi titik sambung ke dunia
  luar maupun ke modul lain dalam monolith.
- **Class Diagram** menjelaskan model domain di balik service tersebut —
  `Order` sebagai aggregate dengan state machine, dan `Notification` plus
  `NotificationPreference`/`NotificationTemplate` sebagai pola Outbox + per-user
  preference.
- Dua **Sequence Diagram** mendemonstrasikan dua alur paling penting:
  `AuctionEndedEvent` (low-volume, harus reliable) dan `BidPlacedEvent`
  (high-volume, harus cepat). Strategi yang dipakai berbeda
  (`@TransactionalEventListener(AFTER_COMMIT)` synchronous vs `@Async`), dan
  perbedaan itu langsung tercermin di Component Diagram pada arah panah
  antara `AuctionEventListener` dan `NotificationService`.

Saat arsitektur masa depan dijalankan, perubahan utamanya hanyalah mengganti
`ApplicationEventPublisher` + `@EventListener` dengan RabbitMQ/Kafka producer
dan consumer. Struktur internal (Service → Repository → Dispatcher) tidak
berubah, hingga modul ini benar-benar siap dipindah ke service-nya sendiri
tanpa rewrite.
