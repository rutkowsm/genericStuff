```mermaid

sequenceDiagram
  title AutoSys Morning ETL: Lotus Notes → Linux → Magellan (Oracle)

  actor AutoSys as AutoSys (scheduler)
  participant LN as Lotus Notes Modules
  participant LNFS as LN Server (CSV location)
  participant Xfer as Xfer (file mover)
  participant LFS as Linux Disk (landing)

  box Magellan DB (Oracle)
    participant PLSQL as PL/SQL Loader (procedures)
    participant LND as Schema: STG/LND (raw)
    participant UNI as Schema: Unified/Final
    participant ERR as Error Log (rejects)
  end
  participant Alert as Alerting

  %% 1) Extract CSVs from Lotus Notes
  AutoSys->>LN: Run extract modules
  LN->>LNFS: Write .csv extracts

  %% 2) Move files LN → Linux
  AutoSys->>Xfer: Start transfer batch
  Xfer->>LNFS: Fetch .csv files
  Xfer->>LFS: Place .csv files

  %% 3) Load files to Magellan (LND)
  AutoSys->>PLSQL: Invoke load_from_files()
  PLSQL->>LFS: Discover files
  alt Files present?
    LFS-->>PLSQL: File list
  else No files
    LFS-->>PLSQL: (none)
    PLSQL-->>AutoSys: Missing input files
    AutoSys->>Alert: ALERT: No files present → abort
    AutoSys-->>AutoSys: Abort job
    deactivate PLSQL
    return
  end

  loop For each file in list
    PLSQL->>LND: Load rows from file
    alt Load succeeded?
      LND-->>PLSQL: Insert/Merge OK
    else Load failed
      LND-->>PLSQL: Error
      PLSQL-->>AutoSys: Load failure
      AutoSys->>Alert: ALERT: File load failed → abort
      AutoSys-->>AutoSys: Abort job
      deactivate PLSQL
      break
    end
  end

  %% 4) Cleanse & unify to final schema
  PLSQL->>UNI: Transform / cleanse / unify
  alt Row-level validation errors occur
    PLSQL->>ERR: Insert rejected records
    note right of ERR: Job continues despite rejects
  else No rejects
    UNI-->>PLSQL: Commit OK
  end

  %% 5) Finish
  PLSQL-->>AutoSys: All files processed, final schema loaded
  AutoSys->>Alert: Success notification (optional)
