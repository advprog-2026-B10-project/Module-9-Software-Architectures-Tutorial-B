# BidMart — Kelompok B10

> Platform lelang kompetitif real-time yang mempertemukan pembeli dan penjual
> dalam sesi bidding yang terstruktur, aman, dan responsif.

---

## Individual Work

- [Ahmad Wasis Shofiyulloh - Auth Module](Auth.md)
- [Fidel Akilah](#)
- [Josiah Naphta Simorangkir](#)
- [Petrus Wermasaubun](Wallet.md)
- [Tsaniya Fini Ardiyanti - Bidding Module](Bidding.md)

---

## Arsitektur Saat Ini

### System Context (C4 Level 1)

BidMart melayani tiga jenis pengguna: Buyer yang aktif mencari barang dan
memasang bid, Seller yang membuka sesi lelang, dan Admin yang menjaga kesehatan
platform. Untuk memproses transaksi keuangan dan pengiriman notifikasi, BidMart
bergantung pada dua sistem eksternal, Payment Gateway dan Email/Push Provider.

```mermaid
flowchart TD
    Buyer["👤 Buyer<br/>Cari barang, pasang bid, kelola saldo"]
    Seller["👤 Seller<br/>Buat listing lelang, pantau bid"]
    Admin["👤 Admin<br/>Moderasi user & listing, urus sengketa"]
    BidMart["🔨 BidMart Platform<br/>Real-time Auction & Marketplace"]
    Payment["💳 Payment Gateway<br/>Proses top-up & penarikan dana"]
    Email["📧 Email / Push Provider<br/>OTP 2FA & notifikasi outbid/menang"]

    Buyer -->|"REST API / WebSocket"| BidMart
    Seller -->|"REST API"| BidMart
    Admin -->|"REST API"| BidMart
    BidMart -->|"API Call"| Payment
    BidMart -->|"SMTP / API"| Email

    style BidMart fill:#4F46E5,color:#fff,stroke:#3730A3
    style Buyer fill:#059669,color:#fff,stroke:#047857
    style Seller fill:#0284C7,color:#fff,stroke:#0369A1
    style Admin fill:#D97706,color:#fff,stroke:#B45309
    style Payment fill:#DC2626,color:#fff,stroke:#B91C1C
    style Email fill:#7C3AED,color:#fff,stroke:#6D28D9
```

| Actor | Role | Description |
| --- | --- | --- |
| Buyer | Person | Menelusuri katalog, melakukan bidding, mengelola saldo |
| Seller | Person | Membuat listing lelang (harga awal, reserve price, durasi) |
| Admin | Person | Mengawasi platform, banned user, atur permission dinamis |
| BidMart Platform | Software System | Platform lelang real-time yang sedang dibangun |
| Payment Gateway | External System | Proses keluar-masuk uang ke dompet digital |
| Email/Push Provider | External System | Kirim kode 2FA dan notifikasi outbid/menang |

---

### Container Diagram (C4 Level 2)

Seluruh logika bisnis saat ini hidup dalam satu Spring Boot monolith yang
berjalan di port 8080. Frontend Next.js berkomunikasi dengan backend melalui
REST dan WebSocket. Di dalam monolith, lima modul berkomunikasi secara
sinkronus dan berbagi satu database PostgreSQL.

```mermaid
graph TB
    Buyer["👤 Buyer"]
    Seller["👤 Seller"]
    FE["🌐 Next.js Frontend<br/><i>(Styled with Tailwind CDN)</i>"]

    subgraph monolith ["Spring Boot Monolith (:8080)"]
        direction TB
        AUTH["🔐 Auth Module<br/><i>Login, 2FA, JWT, Roles</i>"]
        CATALOG["📦 Catalog Module<br/><i>Listings, Categories, Search</i>"]
        AUCTION["🔨 Auction Module<br/><i>Bidding logic, Anti-sniping</i>"]
        WALLET["💰 Wallet Module<br/><i>Balance, Hold/Release Funds</i>"]
        ORDER["🚚 Order & Notif Module<br/><i>Create order, Alerts</i>"]
    end

    DB[("🗄️ PostgreSQL<br/><i>Single shared DB</i>")]
    Payment["💳 Payment Gateway"]

    Buyer --> FE
    Seller --> FE
    FE -->|"REST / WS"| monolith

    AUTH -.->|"Check status"| AUCTION
    AUCTION -->|"Sync API (Hold Funds)"| WALLET
    AUCTION -.->|"Direct update price"| CATALOG
    AUCTION -.->|"Trigger alert"| ORDER
    WALLET -->|"Process Top-up"| Payment

    AUTH --> DB
    CATALOG --> DB
    AUCTION --> DB
    WALLET --> DB
    ORDER --> DB

    style monolith fill:#1E1B4B,color:#C7D2FE,stroke:#6366F1
    style AUTH fill:#6366F1,color:#fff,stroke:#4F46E5
    style CATALOG fill:#06B6D4,color:#fff,stroke:#0891B2
    style AUCTION fill:#F59E0B,color:#fff,stroke:#D97706
    style WALLET fill:#10B981,color:#fff,stroke:#059669
    style ORDER fill:#EC4899,color:#fff,stroke:#DB2777
    style DB fill:#3ECF8E,color:#fff,stroke:#22C55E
    style FE fill:#0EA5E9,color:#fff,stroke:#0284C7

```

Ketergantungan sinkronus antar modul menjadi perhatian utama. Auction Module
harus menunggu respons dari Wallet Module sebelum bid dapat dikonfirmasi,
artinya kelambatan di Wallet langsung berdampak pada pengalaman bidding.

| From | To | Jenis Coupling | Alasan |
| --- | --- | --- | --- |
| Auction → Wallet | Sinkronus | Wajib hold dana seketika sebelum bid diterima |
| Auction → Catalog | Sinkronus | Update harga tertinggi di katalog saat bid masuk |
| Auction → Order/Notif | Sinkronus | Langsung kirim notifikasi outbid/menang |
| Auth → Auction | Sync/Event | Kalau admin banned user, bid aktifnya harus langsung di-handle |

---

### Deployment Diagram

Backend di-deploy sebagai Docker container di EC2, terhubung ke PostgreSQL
terkelola. Frontend Next.js di-deploy secara independen. GitHub Actions
menangani CI/CD, setiap push ke main branch memicu testing otomatis
sebelum deployment.

```mermaid
graph TB
    subgraph dev ["Developer Machine"]
        IDE["💻 IDE"]
        DockerLocal["🐳 Docker Compose<br/><i>PostgreSQL :5432</i>"]
        Boot["☕ gradlew bootRun<br/><i>:8080</i>"]
        NextDev["🌐 npm run dev<br/><i>:3000</i>"]
    end

    subgraph github ["GitHub"]
        Repo["📦 Repository<br/><i>advprog-2026-B10/bidmart</i>"]
        CI["🔄 CI Pipeline<br/><i>ci.yml - Test, Checkstyle</i>"]
    end

    subgraph aws ["AWS EC2 (Production / Staging)"]
        EC2["🖥️ EC2 Instance"]
        subgraph containers ["Docker Compose on EC2"]
            AppContainer["🔨 bidmart-backend<br/><i>Spring Boot :8080</i>"]
        end
    end

    subgraph managed ["Managed Services"]
        ProdDB[("🗄️ PostgreSQL DB<br/><i>(RDS/Supabase)</i>")]
        TailwindCDN["⚡ Tailwind CSS CDN<br/><i>Style delivery</i>"]
    end

    IDE -->|"git push"| Repo
    Repo -->|"trigger"| CI
    CI -->|"auto-deploy"| EC2
    AppContainer -->|"JDBC"| ProdDB
    NextDev -.->|"fetch CSS"| TailwindCDN
    Boot -->|"JDBC"| DockerLocal

    style aws fill:#FF9900,color:#fff,stroke:#E68A00
    style github fill:#24292F,color:#fff,stroke:#1B1F23
    style managed fill:#3ECF8E,color:#fff,stroke:#22C55E
    style dev fill:#1E293B,color:#E2E8F0,stroke:#475569

```

| Layer | Tech Stack | Detail |
| --- | --- | --- |
| Backend Runtime | Spring Boot (Java) | Di-build via Gradle, running di Docker |
| Frontend | Next.js | Tailwind CSS via CDN |
| CI/CD | GitHub Actions | Testing otomatis via ci.yml |
| Database | PostgreSQL | Single instance untuk semua modul |

---

## Arsitektur Masa Depan

Setelah migrasi, BidMart bertransisi ke arsitektur microservices berbasis
Event-Driven Architecture (EDA). Setiap modul menjadi service independen
dengan database-nya sendiri. Komunikasi antar service dilakukan secara
asinkronus melalui message broker (RabbitMQ/Kafka). API Gateway menangani
routing, autentikasi, dan rate limiting di lapisan terdepan.

```mermaid
graph TB
    User["👤 Buyer / Seller"]
    FE["🌐 Next.js Frontend"]
    APIGW["🚪 API Gateway<br/><i>Rate limiting, Auth routing</i>"]

    subgraph broker ["Message Broker (RabbitMQ / Kafka)"]
        direction LR
        E1["📨 bid.placed"]
        E2["📨 auction.winner_determined"]
        E3["📨 user.banned"]
        E4["📨 catalog.price_updated"]
    end

    subgraph services ["Microservices"]
        direction TB
        AUTH_SVC["🔐 Auth Service<br/><i>Own DB</i>"]
        CATALOG_SVC["📦 Catalog Service<br/><i>Own DB</i>"]
        AUCTION_SVC["🔨 Auction Service<br/><i>Own DB (Redis for fast bids)</i>"]
        WALLET_SVC["💰 Wallet Service<br/><i>Own DB</i>"]
        ORDER_SVC["🚚 Order & Notif Service<br/><i>Own DB</i>"]
    end

    User --> FE
    FE -->|HTTPS / WS| APIGW
    APIGW --> AUTH_SVC
    APIGW --> CATALOG_SVC
    APIGW --> AUCTION_SVC
    APIGW --> WALLET_SVC
    APIGW --> ORDER_SVC

    AUCTION_SVC <-->|"Sync gRPC/REST<br/>(Fast Fund Check)"| WALLET_SVC

    AUCTION_SVC -->|"publish"| E1
    AUCTION_SVC -->|"publish"| E2
    AUTH_SVC -->|"publish"| E3
    
    E1 -->|"subscribe"| CATALOG_SVC
    E1 -->|"subscribe"| ORDER_SVC
    E2 -->|"subscribe"| WALLET_SVC
    E2 -->|"subscribe"| ORDER_SVC
    E3 -->|"subscribe"| AUCTION_SVC

    style broker fill:#7C3AED,color:#fff,stroke:#6D28D9
    style services fill:#1E293B,color:#E2E8F0,stroke:#475569
    style APIGW fill:#F97316,color:#fff,stroke:#EA580C

```

### Key Events

| Event | Publisher | Subscribers | Tujuan |
| --- | --- | --- | --- |
| `bid.placed` | Auction Service | Catalog, Order/Notif | Update harga katalog async & notif outbid |
| `auction.winner_determined` | Auction Service | Wallet, Order/Notif | Konversi hold dana jadi payment, buat order |
| `user.banned` | Auth Service | Auction | Gugurkan bid aktif & lepas dana tertahan |

---

## Risk Storming

Sesi Risk Storming dilakukan dengan skenario: BidMart sedang mengadakan
lelang barang populer yang diikuti ribuan user secara bersamaan.

### Peta Risiko

| Risk | Risiko | Dampak | Likelihood | Score | Area |
| --- | --- | --- | --- | --- | --- |
| R1 | Database locking saat bidding war | High (3) | High (3) | 🔴 9 | Performance |
| R2 | Wallet sync bottleneck memblokir bid | High (3) | High (3) | 🔴 9 | Availability |
| R3 | Single point of failure (monolith crash) | High (3) | Medium (2) | 🔴 6 | Availability |
| R4 | Anti-sniping thread exhaustion | Medium (2) | Medium (2) | 🟡 4 | Performance |
| R5 | Catalog read terganggu tulis auction | Medium (2) | Medium (2) | 🟡 4 | Performance |

R1 dan R2 adalah risiko paling kritis. Ratusan bid yang masuk dalam hitungan
detik akan menyebabkan ratusan transaksi berebut mengunci baris yang sama di
database. Di saat yang sama, setiap bid harus menunggu Wallet Module selesai
mem-hold dana secara sinkronus, bottleneck di satu titik ini memperlambat
seluruh proses lelang.

R3 menjadi risiko eksistensial karena kegagalan di Auction Module akibat
lonjakan traffic bisa menyebabkan OOM yang mematikan seluruh aplikasi,
termasuk halaman login dan halaman katalog yang sama sekali tidak berkaitan.

### Justifikasi Modifikasi Arsitektur

Pemisahan setiap modul menjadi service independen dengan database-nya sendiri
secara langsung memutus rantai kegagalan yang teridentifikasi. Ketika Auction
Service membutuhkan lebih banyak kapasitas saat bidding war, ia bisa di-scale-out
secara independen tanpa membebani Auth atau Catalog Service. Database yang
terpisah berarti query berat di Auction tidak lagi bisa menghambat pembacaan
katalog (menjawab R1, R3, dan R5 sekaligus).

Penggantian pemanggilan sinkronus dengan event asinkronus melalui message
broker menjadi kunci untuk mengatasi R2 dan R4. Dalam arsitektur baru, Auction
Service hanya perlu melakukan satu pengecekan saldo cepat via gRPC ke Wallet
Service, kemudian langsung mengembalikan respons sukses ke user. Pembaruan
harga di Catalog dan pengiriman notifikasi outbid terjadi di background melalui
event `bid.placed`, response time bidding menjadi jauh lebih cepat dan tidak
terpengaruh oleh beban di service lain.

Message broker juga berperan sebagai buffer saat lonjakan traffic terjadi.
Event yang belum sempat diproses akan mengantri di broker dan dikonsumsi
secara bertahap sesuai kapasitas masing-masing consumer, bukan langsung
membebani database seperti yang terjadi di arsitektur monolitik saat ini.

---
