## Individual Work - Tsaniya Fini Ardiyanti

### Component Diagram - Auction & Bidding Module

Diagram ini menunjukkan struktur internal Bidding Module, bagaimana 
request bid masuk melalui Controller, diproses BiddingService yang 
menangani concurrency dan anti-sniping, lalu berinteraksi dengan 
WalletService secara sinkronus untuk hold dana dan mempublish event 
ke modul lain secara asinkronus.

```mermaid
flowchart TB
    Client["👤 Client / API Gateway"]

    subgraph bidding ["Auction & Bidding Module"]
        BC["BiddingController\n@RestController"]
        BS["BiddingService\n@Service\nCore bidding logic"]
        AR["AuctionRepository\n@Repository"]
        BR["BidRepository\n@Repository"]
    end

    DB[("PostgreSQL\nShared Database")]
    WS["WalletService\nExternal Module"]
    EP["ApplicationEventPublisher\nSpring Events"]

    Client -->|"POST /bid\nPOST /create\nGET /auctions"| BC
    BC --> BS
    BC --> AR
    BS --> AR
    BS --> BR
    AR -->|"JPA / Hibernate"| DB
    BR -->|"JPA / Hibernate"| DB
    BS -->|"holdBalance()\nreleaseHeldBalance()\nSync call"| WS
    BS -->|"BidPlacedEvent\nAuctionEndedEvent"| EP

    style bidding fill:#1E1B4B,color:#C7D2FE,stroke:#6366F1
    style BC fill:#0284C7,color:#fff,stroke:#0369A1
    style BS fill:#7C3AED,color:#fff,stroke:#6D28D9
    style AR fill:#059669,color:#fff,stroke:#047857
    style BR fill:#059669,color:#fff,stroke:#047857
    style DB fill:#059669,color:#fff,stroke:#047857
    style WS fill:#D97706,color:#fff,stroke:#B45309
    style EP fill:#DB2777,color:#fff,stroke:#BE185D
```

### Code Diagram - Auction & Bidding Module

Class diagram dipetakan langsung dari source code, mencakup relasi 
antar entity dan dependency ke WalletService.

```mermaid
classDiagram
    direction TB

    class BiddingController {
        +createAuction(request) ResponseEntity
        +placeBid(userId, auctionId, amount) String
        +getAllAuctions() ResponseEntity
        +getAuction(id) ResponseEntity
    }

    class BiddingService {
        +createAuction(request) Auction
        +placeBid(userId, auctionId, amount) String
        +determineWinner(auction) String
        +closeExpiredAuctions() void
    }

    class AuctionRepository {
        <<interface>>
        +findByStatusIn(statuses) List~Auction~
    }

    class BidRepository {
        <<interface>>
    }

    class Auction {
        -Long id
        -Long listingId
        -Double reservePrice
        -LocalDateTime endTime
        -AuctionStatus status
        -Double startingPrice
        -String sellerId
        -Long version
    }

    class Bid {
        -Long id
        -String buyerId
        -Double amount
        -LocalDateTime timestamp
    }

    class AuctionStatus {
        <<enumeration>>
        DRAFT
        ACTIVE
        EXTENDED
        WON
        UNSOLD
        CLOSED
    }

    BiddingController --> BiddingService : uses
    BiddingController --> AuctionRepository : uses
    BiddingService --> AuctionRepository : uses
    BiddingService --> BidRepository : uses
    BiddingService --> WalletService : sync call
    Auction "1" *-- "*" Bid : contains
    Auction --> AuctionStatus : has
```

### Korelasi Diagram

Component Diagram menunjukkan alur request dari Client masuk ke 
BiddingController, diproses BiddingService yang menangani validasi bid 
dan optimistic locking untuk mencegah race condition saat bidding war. 
Class Diagram memperbesar bagian data model `Auction` sebagai aggregate 
root yang menyimpan daftar `Bid`, dengan `AuctionStatus` yang merepresentasikan 
state machine lifecycle lelang dari DRAFT hingga WON atau UNSOLD.

Poin kritis ada di BiddingService yang memanggil WalletService secara 
sinkronus untuk hold dana, inilah yang menjadi bottleneck R2 di Risk 
Storming dan akan digantikan dengan event `bid.placed` di arsitektur 
masa depan.