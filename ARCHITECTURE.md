# Production Architecture Design
## Credit Card RAG Application - Next.js + Prisma + PostgreSQL + Chroma

---

## 1. System Architecture Overview

```mermaid
graph TB
    subgraph Client["Client Layer"]
        Browser[Desktop Browsers<br/>2,500 Agents]
    end

    subgraph CDN["Edge Layer"]
        CF[CloudFlare CDN<br/>Static Assets + DDoS Protection]
        LB[Load Balancer<br/>SSL/TLS Termination]
    end

    subgraph App["Application Layer"]
        NextJS[Next.js 14 App Router<br/>React UI + API Routes]

        subgraph Services["Services"]
            Search[Search Service]
            Process[Process Service]
            Analytics[Analytics Service]
        end
    end

    subgraph Cache["Cache Layer"]
        Redis[(Redis<br/>Query Cache + Sessions<br/>TTL: 1hr)]
    end

    subgraph Data["Data Layer"]
        Postgres[(PostgreSQL<br/>Prisma ORM<br/>Process Docs + Users)]
        Chroma[(ChromaDB<br/>Vector Embeddings<br/>1,247 Processes)]
    end

    subgraph External["External APIs"]
        OpenAI[OpenAI API<br/>Embeddings<br/>text-embedding-3-small]
        Claude[Anthropic Claude<br/>Response Generation<br/>claude-3-5-sonnet]
    end

    Browser --> CF --> LB --> NextJS
    NextJS --> Services
    Services --> Redis
    Services --> Postgres
    Services --> Chroma
    Services --> OpenAI
    Services --> Claude

    style Browser fill:#e1f5ff
    style NextJS fill:#4CAF50
    style Redis fill:#ff9800
    style Postgres fill:#2196F3
    style Chroma fill:#9c27b0
    style OpenAI fill:#00bcd4
    style Claude fill:#ff5722
```

---

## 2. Search Request Dataflow

```mermaid
sequenceDiagram
    actor Agent as Support Agent
    participant UI as Next.js UI
    participant API as API Route
    participant Cache as Redis Cache
    participant RAG as RAG Service
    participant Vec as ChromaDB
    participant DB as PostgreSQL
    participant LLM as Claude API

    Agent->>UI: Enter query: "lost card abroad"
    UI->>API: POST /api/search

    API->>Cache: Check cache (query key)
    alt Cache Hit
        Cache-->>API: Return cached results (50ms)
        API-->>UI: Response (80ms total)
        UI-->>Agent: Display results ✓
    else Cache Miss
        Cache-->>API: Not found

        API->>RAG: Perform RAG search

        RAG->>Vec: Vector similarity search
        Note over Vec: 1. Generate query embedding<br/>2. Search top-k similar docs
        Vec-->>RAG: Top 5 process IDs (80ms)

        RAG->>DB: Fetch process details
        DB-->>RAG: Full process documents (30ms)

        RAG->>LLM: Generate response with context
        LLM-->>RAG: Formatted answer (120ms)

        RAG-->>API: Search results
        API->>Cache: Store results (TTL: 1hr)
        API-->>UI: Response (280ms total)
        UI-->>Agent: Display results ✓
    end

    API->>DB: Log search history
    DB-->>API: Logged
```

**Target Performance:** 95% of requests < 300ms

---

## 3. RAG Pipeline Architecture

```mermaid
graph LR
    subgraph Input
        Query["User Query<br/>lost card abroad"]
    end

    subgraph Preprocessing
        Clean["Clean & Normalize<br/>lowercase, trim"]
        Expand["Expand Abbreviations<br/>CC to Credit Card"]
    end

    subgraph Embedding
        Embed["OpenAI Embedding API<br/>text-embedding-3-small<br/>1536 dimensions"]
    end

    subgraph Retrieval
        Chroma[("ChromaDB<br/>Vector Search")]
        Filter["Filter by Category<br/>Apply metadata filters"]
        Rerank["Re-rank Results<br/>Business rules"]
    end

    subgraph Augmentation
        Fetch["Fetch from PostgreSQL<br/>Full process details"]
        Context["Build Context<br/>Steps + Forms + Codes"]
    end

    subgraph Generation
        LLM["Claude Sonnet<br/>Temperature: 0.3"]
        Format["Format Response<br/>Step-by-step instructions"]
    end

    subgraph Output
        Result["Structured Response<br/>+ Metadata"]
    end

    Query --> Clean --> Expand --> Embed
    Embed --> Chroma --> Filter --> Rerank
    Rerank --> Fetch --> Context --> LLM --> Format --> Result

    style Query fill:#e3f2fd
    style Embed fill:#fff3e0
    style Chroma fill:#f3e5f5
    style LLM fill:#fce4ec
    style Result fill:#e8f5e9
```

---

## 4. Database Schema (Prisma)

```mermaid
erDiagram
    Process ||--o{ SearchHistory : "searched_in"
    Process ||--o{ ProcessAnalytics : "has_analytics"
    User ||--o{ SearchHistory : "performs"
    User ||--o{ Session : "has"
    User ||--o{ AuditLog : "generates"

    Process {
        string id PK
        string processId UK "CC-PROC-001"
        string title
        string category
        string subcategory
        json steps "Array of process steps"
        string[] forms "CC-DSP-447"
        json timeline
        string[] keywords
        datetime createdAt
        datetime updatedAt
        boolean isActive
    }

    SearchHistory {
        string id PK
        string userId FK
        string processId FK
        string query
        float relevanceScore
        int responseTimeMs
        boolean wasSuccessful
        int feedbackRating "1-5 stars"
        datetime createdAt
    }

    User {
        string id PK
        string employeeId UK
        string email UK
        string name
        enum role "AGENT|SUPERVISOR|ADMIN"
        string location
        datetime lastLoginAt
    }

    Session {
        string id PK
        string userId FK
        string token UK
        datetime expiresAt
        string ipAddress
    }

    ProcessAnalytics {
        string id PK
        string processId FK
        date date
        int viewCount
        int searchCount
        int avgResponseTimeMs
        float avgRelevanceScore
    }

    AuditLog {
        string id PK
        string userId FK
        string action "SEARCH|VIEW|EXPORT"
        string resource
        json metadata
        datetime createdAt
    }
```

---

## 5. Caching Strategy

```mermaid
graph TD
    Request[Incoming Search Request]

    Request --> L1{Browser Cache<br/>Static Assets}
    L1 -->|Hit| Return1[Return Immediately<br/>~10ms]
    L1 -->|Miss| L2{CDN Cache<br/>API Responses}

    L2 -->|Hit| Return2[Return from Edge<br/>~50ms]
    L2 -->|Miss| L3{Redis Cache<br/>Search Results}

    L3 -->|Hit 80%| Return3[Return Cached<br/>~80ms]
    L3 -->|Miss 20%| L4[RAG Pipeline<br/>Full Processing]

    L4 --> Return4[Fresh Result<br/>~280ms]
    Return4 --> Store[Store in Redis<br/>TTL: 1 hour]

    Store --> Return5[Return to User]

    style Return1 fill:#4CAF50
    style Return2 fill:#8BC34A
    style Return3 fill:#CDDC39
    style Return4 fill:#FFC107
```

**Cache Tiers:**
1. **Browser**: Static assets (indefinite)
2. **CDN**: API responses for common queries (5 min)
3. **Redis**: Search results + process details (1 hour)
4. **Application**: Connection pools, Prisma client

---

## 6. Deployment Architecture

```mermaid
graph TB
    subgraph Internet
        Users[2,500 Support Agents]
    end

    subgraph CloudFlare["CloudFlare Edge"]
        WAF[WAF + DDoS Protection]
        CDN[Global CDN<br/>Static Assets]
    end

    subgraph K8s["Kubernetes Cluster"]
        subgraph AppPods["App Pods (Auto-scale 2-10)"]
            NextJS1[Next.js Pod 1]
            NextJS2[Next.js Pod 2]
            NextJS3[Next.js Pod 3]
        end

        Ingress[Nginx Ingress<br/>SSL Termination]
    end

    subgraph DataServices["Data Services"]
        RedisHA[(Redis Cluster<br/>3 nodes<br/>Master-Replica)]
        PostgresHA[(PostgreSQL<br/>Primary + Standby<br/>Streaming Replication)]
        ChromaService[Chroma Service<br/>Persistent Volume]
    end

    subgraph Monitoring
        Metrics[Prometheus<br/>Metrics Collection]
        Logs[Loki<br/>Log Aggregation]
        Alerts[AlertManager<br/>PagerDuty Integration]
    end

    Users --> WAF --> CDN --> Ingress
    Ingress --> AppPods
    AppPods --> RedisHA
    AppPods --> PostgresHA
    AppPods --> ChromaService
    AppPods --> Metrics
    AppPods --> Logs
    Metrics --> Alerts

    style AppPods fill:#4CAF50
    style DataServices fill:#2196F3
    style Monitoring fill:#FF9800
```

---

## 7. Technology Stack

| Layer | Technology | Purpose | Rationale |
|-------|-----------|---------|-----------|
| **Frontend** | Next.js 14 + React | UI + API Routes | Full-stack framework, excellent DX |
| **Backend** | Next.js API Routes | REST endpoints | Same codebase, TypeScript end-to-end |
| **ORM** | Prisma | Database access | Type-safe, migrations, great DX |
| **Database** | PostgreSQL 15 | Process docs, users | ACID compliance, reliability |
| **Vector DB** | ChromaDB | Embeddings | Open-source, easy setup, good for POC |
| **Cache** | Redis | Query cache | Sub-millisecond latency |
| **Embeddings** | OpenAI text-embedding-3-small | Vector generation | Fast, cheap ($0.02/1M tokens) |
| **LLM** | Claude 3.5 Sonnet | Response generation | Best reasoning, accurate |
| **Container** | Docker + Kubernetes | Orchestration | Scalability, high availability |
| **Monitoring** | Prometheus + Grafana | Metrics & alerting | Industry standard |

---

## 8. Performance Budget

### Response Time Breakdown (300ms target)

```mermaid
pie title Response Time Budget (300ms)
    "Network Latency" : 50
    "Vector Search (Chroma)" : 80
    "Database Query (PostgreSQL)" : 30
    "LLM Generation (Claude)" : 100
    "App Processing" : 40
```

### Optimization Strategies

1. **Caching**: 80% cache hit rate → 80% of queries < 100ms
2. **Connection Pooling**: Reuse DB/Redis connections (50 max)
3. **Query Optimization**: Database indexes on `category`, `processId`, `userId`
4. **Parallel Execution**: Fetch from DB + LLM simultaneously where possible
5. **Pre-warming**: Cache top 100 queries on startup

---

## 9. Security Architecture

```mermaid
graph LR
    subgraph Perimeter
        WAF[CloudFlare WAF<br/>Rate Limiting]
        DDoS[DDoS Protection]
    end

    subgraph Network
        VPC[Private VPC<br/>10.0.0.0/16]
        SG[Security Groups<br/>Least Privilege]
    end

    subgraph Application
        Auth[NextAuth.js<br/>SSO Integration]
        RBAC[Role-Based Access<br/>Agent/Supervisor/Admin]
        Session[Encrypted Sessions<br/>HTTP-only cookies]
    end

    subgraph Data
        Encryption[TLS 1.3 in transit<br/>AES-256 at rest]
        Audit[Audit Logging<br/>All actions logged]
        NoCustomerData[No Customer Data<br/>Process docs only]
    end

    WAF --> VPC --> Auth --> Encryption
    DDoS --> VPC
    VPC --> SG --> RBAC --> Audit
    Auth --> Session --> NoCustomerData

    style Perimeter fill:#f44336
    style Network fill:#ff9800
    style Application fill:#4CAF50
    style Data fill:#2196F3
```

---

## 10. Scalability & High Availability

### Horizontal Scaling

- **App Layer**: Auto-scale Next.js pods (2-10 based on CPU/memory)
- **Database**: PostgreSQL read replicas for analytics queries
- **Cache**: Redis cluster with 3 nodes (master + 2 replicas)
- **Vector DB**: Chroma with persistent volumes (manual scale)

### Failure Handling

```mermaid
graph TD
    Request[User Request]

    Request --> Primary{Primary Service}
    Primary -->|Success| Response[Return Response]
    Primary -->|Failure| Circuit{Circuit Breaker<br/>Open?}

    Circuit -->|Closed| Retry[Retry 3x<br/>Exponential Backoff]
    Circuit -->|Open| Fallback[Fallback Strategy]

    Retry -->|Success| Response
    Retry -->|Fail| Fallback

    Fallback --> Cache2[Try Stale Cache]
    Cache2 -->|Hit| Response2[Return Stale Data<br/>+ Warning]
    Cache2 -->|Miss| Error[Return Error<br/>+ Retry Guidance]

    style Response fill:#4CAF50
    style Response2 fill:#FFC107
    style Error fill:#f44336
```

**SLA Target:** 99.9% uptime = 43 minutes downtime/month

---

## 11. Data Ingestion Pipeline

```mermaid
graph LR
    subgraph Source
        Excel[Excel Files<br/>1,247 Processes]
        Docs[Policy Documents]
    end

    subgraph ETL
        Parse[Parse & Validate<br/>Zod Schema]
        Transform[Transform to JSON<br/>Normalize Fields]
    end

    subgraph Load
        PG[Load to PostgreSQL<br/>Prisma Client]
        Embed[Generate Embeddings<br/>OpenAI API]
        ChromaLoad[Load to ChromaDB<br/>Batch Insert]
    end

    subgraph Verify
        Check[Verify Counts<br/>Run Test Queries]
    end

    Excel --> Parse --> Transform
    Docs --> Parse
    Transform --> PG --> Embed --> ChromaLoad --> Check

    style Excel fill:#e3f2fd
    style Transform fill:#fff3e0
    style ChromaLoad fill:#f3e5f5
    style Check fill:#e8f5e9
```

**Ingestion Script:** `npm run ingest:documents`

---

## 12. Monitoring & Observability

### Key Metrics

| Metric | Target | Alert Threshold |
|--------|--------|----------------|
| Response Time (p95) | < 300ms | > 500ms |
| Response Time (p99) | < 500ms | > 1000ms |
| Error Rate | < 0.1% | > 1% |
| Cache Hit Rate | > 80% | < 60% |
| Concurrent Users | 2,500 peak | > 3,000 |
| Database Connections | < 40 | > 45 |
| Chroma Latency | < 100ms | > 200ms |

### Health Checks

```typescript
// Liveness: Is service running?
GET /api/health/live

// Readiness: Can service handle traffic?
GET /api/health/ready
```

---

## 13. Disaster Recovery

**Backup Strategy:**
- **PostgreSQL**: Daily automated backups (30-day retention)
- **Redis**: AOF persistence (append-only file)
- **ChromaDB**: Weekly vector DB snapshots
- **Application**: Immutable Docker images

**Recovery Time Objective (RTO):** 1 hour
**Recovery Point Objective (RPO):** 24 hours

---

## Summary

This architecture delivers:
- ✅ **<300ms response time** (95th percentile) via aggressive caching
- ✅ **2,500 concurrent users** via horizontal scaling
- ✅ **99.9% uptime** via redundancy and failover
- ✅ **Type-safe development** via TypeScript + Prisma
- ✅ **Cost efficiency** via open-source stack (Chroma, PostgreSQL)
- ⚠️ **Trade-off**: Chroma lacks enterprise features; plan migration to Pinecone if scaling beyond 5,000 users

**Next Steps:** Review IMPLEMENTATION_GUIDE.md for setup instructions.
