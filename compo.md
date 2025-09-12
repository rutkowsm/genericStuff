```mermaid
flowchart LR
  %% ===== LAYERS =====
  subgraph ext[External / Producers & Consumers]
    EMIS[EMIS System\n(API Producer)]
    Admin[Ops/Admin UI]
    BIUser[BI / Analysts]
  end

  subgraph integ[Integration Layer]
    DG[Data Gateway\n(API Client + Ingestion Service)]
    MQ[Event Bus / Queue]
  end

  subgraph proc[Processing & Orchestration]
    PySched[Python Schedulers\n(Cron/Workers)]
    ETL[ETL Tool\n(Informatica Cloud)]
    BM[BlueMoon Parser\n(JSON → Relational)]
  end

  subgraph data[Data Stores]
    Art1[(Oracle - Art1\nRaw JSON Staging)]
    BMDB[(Oracle - BlueMoon\nParsed Relational)]
    DW[(Synapse / DWH\nStar Schemas)]
  end

  subgraph analytics[Analytics & Serving]
    PBI[Power BI / Dashboards]
    API[Read API / Data Service]
  end

  subgraph sec[Cross-Cutting]
    IAM[Identity & Access Mgmt\n(SSO/OIDC)]
    Vault[Secrets Manager]
    Mon[Monitoring & Logs\n(Prom/Grafana/ELK)]
    Alert[Alerting]
  end

  %% ===== FLOWS =====
  EMIS -- REST/JSON --> DG
  DG -- Retry/Bulk --> Art1
  DG -- Async Events --> MQ

  MQ --> PySched
  PySched --> BM
  BM -- Parse/Validate --> BMDB

  ETL -- Extract --> BMDB
  ETL -- Load/Transform --> DW

  API -- SQL/Views --> DW
  PBI <-- Datasets/DirectQuery --> DW
  BIUser --> PBI
  Admin --> DG

  %% ===== CONTROLS & TELEMETRY =====
  DG -. auth .- IAM
  API -. auth .- IAM
  ETL -. secrets .- Vault
  PySched -. secrets .- Vault
  DG -. metrics/logs .-> Mon
  BM -. metrics/logs .-> Mon
  ETL -. metrics/logs .-> Mon
  API -. metrics/logs .-> Mon
  Mon --> Alert

  %% ===== NOTES / PROTOCOLS =====
  classDef store fill:#f6f8fa,stroke:#999,stroke-width:1px;
  class Art1,BMDB,DW store;
