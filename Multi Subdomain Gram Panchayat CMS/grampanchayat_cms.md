
# Grampanchayat CMS — Mermaid Diagrams
**Date:** 2025-10-05

---

## 1) System Architecture (Budget Hybrid)
```mermaid
flowchart LR
  subgraph Edge["Cloudflare (DNS + Proxy + TLS + CDN)"]
    CF[Cloudflare]
  end

  U((Citizen/Admin)) --> CF

  CF -->|SPA assets| Static["Azure Storage Static Website"]
  CF -->|/api/*| API["Node.js API (Azure Functions - Consumption)"]

  API --> PG[(PostgreSQL)]
  API --> Blob[(Azure Blob Storage: Media)]
```

---

## 2) Request Lifecycle (Sequence)
```mermaid
sequenceDiagram
  participant User as User (Browser)
  participant CF as Cloudflare Edge
  participant Static as Azure Static Site
  participant React as React App
  participant API as Node.js API (Azure Functions)
  participant Repo as Tenant Repo (Cache/DB)
  participant PG as PostgreSQL
  participant Blob as Blob Storage

  User->>CF: GET https://alpha.gp.example.in/
  CF->>Static: Fetch index.html + assets
  Static-->>CF: 200 (cached)
  CF-->>User: 200 (index.html, JS/CSS)

  User->>React: Boot
  React->>CF: GET /api/tenant-context (Host: alpha.gp.example.in)
  CF->>API: Forward request
  API->>Repo: resolve host → tenant_id
  Repo-->>API: t_alpha
  API->>PG: SELECT theme/flags for t_alpha
  PG-->>API: config JSON
  API-->>CF: 200 (tenant-context)
  CF-->>React: 200

  React->>CF: GET /api/content?tenant=t_alpha
  CF->>API: Forward request
  API->>PG: SELECT posts, notices WHERE tenant_id=t_alpha
  PG-->>API: rows
  API-->>CF: 200 (JSON)
  CF-->>React: 200

  React->>CF: GET /media/t_alpha/2025/10/banner.jpg
  CF->>Blob: fetch (cache miss)
  Blob-->>CF: 200 (image)
  CF-->>React: 200 (cached at edge)
```

---

## 3) Multi‑Tenant Data Model (ER)
```mermaid
erDiagram
  TENANT ||--o{ TENANT_DOMAIN : has
  TENANT ||--o{ TENANT_UI_CONFIG : versions
  TENANT ||--o{ POST : publishes

  TENANT {
    uuid id PK
    text code UK
    text display_name
    bool is_active
  }
  TENANT_DOMAIN {
    text host_name PK
    uuid tenant_id FK
    text kind
  }
  TENANT_UI_CONFIG {
    uuid tenant_id FK
    int version
    jsonb config
    timestamptz created_at
  }
  POST {
    bigint id PK
    uuid tenant_id FK
    text slug
    text title
    text body
    timestamptz published_at
  }
```

---

## 4) CI/CD Pipeline
```mermaid
flowchart LR
  Dev[(Commit)] --> CI["CI: Build & Test"]
  CI --> BuildWeb["Build React SPA"]
  CI --> BuildAPI["Build Node Functions"]
  BuildWeb --> Upload["Upload to Azure Storage Static Website"]
  BuildAPI --> DeployAPI["Deploy to Azure Functions"]
  DeployAPI --> Migrate["Run DB Migrations (Prisma/Knex)"]
  Upload --> Purge["Cloudflare: Purge index.html only"]
  Migrate --> Done["✅ Deploy Complete"]
```

---

## 5) Security Layers
```mermaid
flowchart TB
  User((User)) --> CF[Cloudflare WAF/Rate Limit\nTLS termination]
  CF --> Origin[Origin Protection]
  Origin --> API[Node.js API]
  Origin --> Static[Static Site]
  API --> PG[(PostgreSQL)]
  API --> KV[(Key Vault / Secrets)]
  API --> Logs[(App Insights / Logs)]

  classDef edge fill:#eef,stroke:#66f,stroke-width:1px
  class CF edge
```
