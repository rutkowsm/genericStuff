```mermaid
sequenceDiagram
  title EMIS → Data Gateway (DG) → Artemis → Magellan: JSON ingest and reconciliation

  participant EMIS as EMIS (API provider)
  participant DG as Data Gateway (DG)
  participant Artemis as Oracle DB (Artemis)
  actor Magellan as Oracle DB (Magellan)

  %% 1. DG collects JSONs from EMIS API
  DG->>EMIS: API request for JSON data
  EMIS-->>DG: JSON response

  %% 2. DG filters out-of-scope data
  DG->>DG: Filter out-of-scope data

  %% 3. DG pushes JSON to Artemis
  DG->>Artemis: Push JSON payloads

  %% 4. Magellan fetches JSON from Artemis via DB Link
  Note over Magellan,Artemis: Magellan connects using a dedicated user via DB Link
  Magellan->>Artemis: Fetch JSON payloads
  alt Data available?
    Artemis-->>Magellan: JSON payload list
  else No data
    Artemis-->>Magellan: Empty result / not found
    Magellan-->>Magellan: Abort process
  end

  %% 5. Magellan parses JSON → relational model (oldest → newest)
  loop For each JSON (oldest → newest)
    Magellan->>Magellan: Parse JSON → relational rows
    alt JSON.last_update_date ≤ DB.latest_update
      Magellan->>Magellan: Ignore entire JSON (stale)
    else Fresh data
      Magellan->>Magellan: Insert/merge rows into relational tables
    end
  end

  %% 6. Magellan compares history and updates if needed
  Magellan->>Magellan: Compare historical entries
  Magellan->>Magellan: Update records if differences detected
  %% 6. Magellan compares history and updates if needed
  Magellan->>Magellan: Compare historical entries
  Magellan->>Magellan: Update records if differences detected

