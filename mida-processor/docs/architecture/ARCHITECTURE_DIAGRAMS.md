# Architecture Diagrams

This document describes the current architecture of `mida-processor` based on the implementation in `src/`.

The service is a long-running Python worker that:

- loads region- and environment-specific configuration at startup
- polls Manhattan Active for eligible open orders
- retrieves facilities, customers, rules, scenarios, and SLA settings from MIDA MSSQL
- evaluates dynamic business rules with `asteval`
- builds update payloads for Manhattan
- updates and releases orders in batches
- writes operational outcomes to legacy MSSQL logs
- pushes successful updates to Redis so downstream components can continue the flow

## 1. Architecture Summary

At a high level, `mida-processor` is organized into six layers:

- `main.py`: infinite execution loop and scheduling
- `runtime.py`: singleton-like service locator for config and shared clients
- `services/`: orchestration by facility, customer, and order batch
- `core/repositories/`: data access to MIDA MSSQL and Manhattan search APIs
- `core/configuration_rules.py`: dynamic rule, scenario, and SLA evaluation engine
- `core/adapters/` and `core/tracking/`: outbound integrations and operational tracking

The main dependencies are:

- **MIDA MSSQL**
  - source of facilities, customers, rules, scenarios, SLA scenarios
  - sink for legacy processing logs in `dbo.mida_log`
- **Manhattan Active API**
  - source of open orders via search
  - target for bulk order update
  - target for release pipeline triggering
- **Redis**
  - queue/set used to publish successful updates for downstream release validation
  - hash used to store API call details with retention
- **Filesystem configuration**
  - YAML regional configuration
  - constants
  - Manhattan JSON templates

## 2. Repository Structure

```text
src/
├── main.py
├── runtime.py
├── business_objects/
│   ├── customer.py
│   ├── facility.py
│   ├── order.py
│   ├── rule.py
│   ├── scenario.py
│   └── sla.py
├── services/
│   ├── facility_service.py
│   ├── customer_service.py
│   └── order_service.py
├── core/
│   ├── configuration_rules.py
│   ├── adapters/
│   │   ├── legacy_sql_logger.py
│   │   ├── manhattan_api_client.py
│   │   ├── mssql_client.py
│   │   └── redis_client.py
│   ├── repositories/
│   │   ├── customer_repository.py
│   │   ├── facility_repository.py
│   │   ├── order_repository.py
│   │   ├── rule_repository.py
│   │   ├── scenario_repository.py
│   │   └── sla_repository.py
│   └── tracking/
│       ├── event_tracker.py
│       └── loki_sink.py
├── helpers/
│   ├── config_loader.py
│   ├── config_schema.py
│   ├── data_transforms.py
│   ├── datetime_helpers.py
│   └── version_helpers.py
└── support_functions/
    └── order_utils.py
```

## 3. Component Diagram

```mermaid
flowchart LR
    subgraph Entry["Entry Point"]
        MAIN["main.py\nmain_loop()"]
    end

    subgraph RuntimeLayer["Runtime and Shared Dependencies"]
        RT["Runtime"]
        CFG["Config loader\nYAML + templates + env vars"]
        MA["ManhattanAPIClient"]
        SQL["MSSQLClient"]
        ET["EventTracker"]
        RED["RedisClient"]
    end

    subgraph Services["Service Layer"]
        FS["facility_service"]
        CS["customer_service"]
        OS["order_service"]
    end

    subgraph Rules["Rule Engine"]
        CR["ConfigurationRules"]
        AE["asteval Interpreter"]
        SU["support_functions/order_utils"]
    end

    subgraph Repositories["Repositories"]
        FR["facility_repository"]
        CUR["customer_repository"]
        OR["order_repository"]
        RR["rule_repository"]
        SR["scenario_repository"]
        SLAR["sla_repository"]
    end

    subgraph Domain["Business Objects"]
        FAC["Facility"]
        CUS["Customer"]
        ORD["Order"]
        RUL["Rule"]
        SCN["Scenario"]
        SLA["SLA"]
    end

    subgraph External["External Systems"]
        MSSQL[("MIDA MSSQL")]
        MAN[("Manhattan Active APIs")]
        REDIS[("Redis / Sentinel")]
        CFGFILES[("YAML + JSON templates + .env")]
    end

    MAIN --> FS
    MAIN --> RT

    RT --> CFG
    RT --> MA
    RT --> SQL
    RT --> ET
    ET --> RED

    FS --> FR
    FS --> CUR
    FS --> OR
    FS --> CS

    CS --> RR
    CS --> SLAR
    CS --> OS

    RR --> SR
    OS --> CR
    OS --> MA
    OS --> ET

    FR --> FAC
    CUR --> CUS
    OR --> ORD
    RR --> RUL
    SR --> SCN
    SLAR --> SLA

    CR --> AE
    CR --> SU
    CR --> ET

    SQL --> MSSQL
    FR --> MSSQL
    CUR --> MSSQL
    RR --> MSSQL
    SR --> MSSQL
    SLAR --> MSSQL

    OR --> MAN
    MA --> MAN
    RED --> REDIS
    CFG --> CFGFILES
```

## 4. Startup and Configuration Loading

`Runtime.config()` lazily loads a validated `AppConfig` through `helpers.config_loader.Config`.

Configuration loading works as follows:

- load `.env`
- resolve `CONFIG_DIR` and `TEMPLATES_DIR`
- load `configurations/constants.yaml`
- load `configurations/regions_<region>_<env>.yaml`
- merge both dictionaries
- inject JSON templates such as `manhattan_templates/search.json`
- substitute `${ENV_VAR}` placeholders recursively
- validate the final structure with `AppConfig` from `helpers.config_schema`

If configuration files are missing or environment variables are unresolved, the process aborts immediately with `os.abort()`.

```mermaid
flowchart TD
    A["Runtime.config()"] --> B["Config.__new__ / __init__"]
    B --> C["Load .env"]
    C --> D["Locate CONFIG_DIR and TEMPLATES_DIR"]
    D --> E["Load constants.yaml"]
    E --> F["Load regions_<region>_<env>.yaml"]
    F --> G["Merge configuration dictionaries"]
    G --> H["Load Manhattan JSON templates"]
    H --> I["Substitute ${ENV_VAR} placeholders"]
    I --> J["Validate with AppConfig"]
    J --> K["Cache validated config in Runtime"]
```

## 5. Execution Loop

`main.py` runs an infinite loop. Each cycle:

- computes the next scheduled run
- spawns a worker thread targeting `process_facilities()`
- waits up to `process.max_execution_seconds`
- logs a warning if the worker is still alive after the timeout
- sleeps until the next `process.interval_seconds` boundary

Important detail:

- the warning does not kill the worker thread
- a long-running thread may continue in the background after the main loop logs the timeout

```mermaid
flowchart TD
    A["main.py"] --> B["Read version from pyproject.toml"]
    B --> C["while True"]
    C --> D["Compute next run time"]
    D --> E["Create thread(process_facilities)"]
    E --> F["Start thread"]
    F --> G["Join with max_execution_seconds"]
    G --> H{"Thread still alive?"}
    H -- "Yes" --> I["Log timeout warning"]
    H -- "No" --> J["Continue"]
    I --> J
    J --> K["Log next scheduled run"]
    K --> L{"Sleep duration > 0?"}
    L -- "Yes" --> M["Sleep until next cycle"]
    L -- "No" --> C
    M --> C
```

## 6. Runtime and Shared Clients

`Runtime` is the central dependency hub. It caches:

- `AppConfig`
- `ManhattanAPIClient`
- `MSSQLClient`
- `EventTracker`

Creation behavior:

- Manhattan client creation failure aborts the process
- MSSQL client creation failure aborts the process
- Event tracker creation failure is logged, but the process continues

```mermaid
classDiagram
    class Runtime {
        +config() AppConfig
        +manhattan_api_client() ManhattanAPIClient
        +mida_mssql_client() MSSQLClient
        +event_tracker() EventTracker
        -_config
        -_manhattan_api_client
        -_mida_mssql_client
        -_event_tracker
    }

    class AppConfig {
        +region
        +env
        +database
        +manhattan
        +redis
        +process
        +limits
        +business
        +orders
        +proxies
        +templates
        +force_facilities
    }

    class ManhattanAPIClient
    class MSSQLClient
    class EventTracker

    Runtime --> AppConfig
    Runtime --> ManhattanAPIClient
    Runtime --> MSSQLClient
    Runtime --> EventTracker
```

## 7. Facility-to-Order Processing Pipeline

The main orchestration path is:

1. enumerate target facilities
2. enumerate enabled customers per facility
3. stop early if too many orders are already in `PREPROCESSED`
4. search Manhattan for open orders
5. group orders by customer
6. load rules and SLA scenarios for each customer
7. evaluate each order
8. build Manhattan update payloads
9. bulk update orders
10. release successfully updated orders
11. persist audit logs and clear in-memory tracking buffers

```mermaid
flowchart TD
    A["process_facilities()"] --> B{"force_facilities configured?"}
    B -- "Yes" --> C["Validate forced facility/customer/order tuple"]
    B -- "No" --> D["Load facilities from MSSQL"]
    C --> E["Process forced facility"]
    D --> F["Process each facility"]

    F --> G["Load enabled customers from MSSQL"]
    E --> G
    G --> H{"Customers found?"}
    H -- "No" --> I["Skip facility"]
    H -- "Yes" --> J["Search count of PREPROCESSED orders in Manhattan"]
    J --> K{"Count > max_preprocessed?"}
    K -- "Yes" --> I
    K -- "No" --> L["Search open orders in Manhattan"]
    L --> M{"Orders found?"}
    M -- "No" --> I
    M -- "Yes" --> N["Group orders by customer_code"]
    N --> O["process_customer(customer, orders)"]
    O --> P["Load enabled rules from MSSQL"]
    P --> Q["Load enabled SLA scenarios from MSSQL"]
    Q --> R["process_orders(orders, rules, sla_scenarios)"]
    R --> S["ConfigurationRules.evaluate_order(order)"]
    S --> T["Build payloads"]
    T --> U["Bulk update orders in Manhattan"]
    U --> V["Track UPD_OK / UPD_ERR"]
    V --> W["Release successful orders in Manhattan"]
    W --> X["save_logs() and clear_logs()"]
```

## 8. Forced-Processing Branch

For UAT and automated testing, `force_facilities` can override the standard discovery flow.

This branch:

- validates facility codes against MSSQL
- optionally validates explicit customers
- optionally fetches explicit order IDs from Manhattan
- logs security warnings
- still reuses the same downstream customer and order processing pipeline

This setting is explicitly marked in the code as unsafe for production use.

## 9. Repository and Data Access View

Repositories split responsibilities by source:

- **MSSQL-backed repositories**
  - `facility_repository`
  - `customer_repository`
  - `rule_repository`
  - `scenario_repository`
  - `sla_repository`
- **Manhattan-backed repository**
  - `order_repository`

`order_repository` builds Manhattan queries for:

- count of `PREPROCESSED` but unreleased orders
- retrieval of eligible open orders
- retrieval of one explicit order for forced execution

Important Manhattan search filters currently embedded in the repository:

- `MaximumStatus=0000`
- `Extended.DhlEventCode=null` for open orders
- `Extended.DhlEventCode='PREPROCESSED'` for backlog count
- `OrderProcessTypeId=null`
- exclusion of `PipelineId='Add iLPN to Order'`
- `CreatedTimestamp >= today - default_days_back`

```mermaid
flowchart LR
    subgraph MSSQLSide["MSSQL-backed repositories"]
        FR["facility_repository\nfacilities"]
        CUR["customer_repository\ncustomers"]
        RR["rule_repository\nrules"]
        SR["scenario_repository\nscenarios"]
        SLAR["sla_repository\nsla_scenarios"]
    end

    subgraph ManhattanSide["Manhattan-backed repository"]
        OR["order_repository\norder search"]
    end

    subgraph Storage["Storage / APIs"]
        DB[("MIDA MSSQL")]
        API[("Manhattan Active")]
    end

    FR --> DB
    CUR --> DB
    RR --> DB
    SR --> DB
    SLAR --> DB
    OR --> API
```

## 10. Rule Evaluation Engine

`ConfigurationRules` is the core business engine.

It evaluates:

- rule conditions
- scenario conditions under each matched rule
- SLA conditions after payload enrichment
- line-level modifications for `OrderLine`

Implementation characteristics:

- rules and scenarios are data-driven and loaded from MSSQL
- conditions are normalized with `condition_fix()`
- expressions are evaluated with `asteval.Interpreter`
- public functions from `support_functions/order_utils.py` are dynamically exposed to the interpreter
- evaluation context includes:
  - `o`: merged order header/body fields
  - `l`: current line when evaluating line-level modifications
  - Python built-ins: `any`, `all`, `len`, `min`, `max`

```mermaid
flowchart TD
    A["Order enters evaluate_order()"] --> B["Extract CurrencyCode from OrderLineFinancial"]
    B --> C["Set Extended.DhlCurrencyCode when found"]
    C --> D["_evaluate_rules(order)"]
    D --> E["Merge order.other_fields + order.payload"]
    E --> F["Create asteval interpreter with o, built-ins, support functions"]
    F --> G["Evaluate rules in priority order"]
    G --> H{"Rule condition is true?"}
    H -- "No" --> G
    H -- "Yes" --> I["Evaluate rule scenarios in priority order"]
    I --> J{"Scenario condition is true?"}
    J -- "No" --> I
    J -- "Yes" --> K["Merge scenario payload into order.payload"]
    K --> L["Track MATCH_OK"]
    L --> M["_evaluate_sla(order)"]
    M --> N{"SLA condition is true?"}
    N -- "No" --> O["_update_rows(order)"]
    N -- "Yes" --> P["Calculate target date from start_date, offset, weekdays, time"]
    P --> Q["Set PickupStartDateTime / PickupEndDateTime / DeliveryEndDateTime"]
    Q --> O
    O --> R["Build OrderLine / OriginalOrderLine payload rows"]
    R --> S["Return enriched order"]
```

## 11. Rule, Scenario, and SLA Object Model

```mermaid
classDiagram
    class Order {
        +order_id
        +facility_code
        +customer_code
        +other_fields
        +payload
        +do_release
        +get_currency()
        +move_notes_to_instructions()
    }

    class Rule {
        +rule_id
        +name
        +condition
        +do_release
        +scenarios
    }

    class Scenario {
        +scenario_id
        +name
        +condition
        +payload
    }

    class SLA {
        +sla_id
        +name
        +condition
        +date_offset
        +weekdays
        +new_time
        +start_date
        +calculate_date(date)
    }

    class ConfigurationRules {
        +evaluate_order(order)
        -_evaluate_rules(order)
        -_evaluate_scenarios(scenarios, order)
        -_evaluate_sla(order)
        -_update_rows(order)
    }

    ConfigurationRules --> Order
    ConfigurationRules --> Rule
    ConfigurationRules --> Scenario
    ConfigurationRules --> SLA
    Rule --> Scenario
```

## 12. Payload Construction and Transformation

After a successful evaluation, `order_service` transforms orders into Manhattan payloads.

Key transformations currently implemented:

- move selected notes into order instructions
- rename mapped fields according to `config.orders.mapped_fields`
- force these identifiers into the payload:
  - `OriginFacilityId`
  - `BusinessUnitId`
  - `OriginalOrderId`
- force `Extended.DhlEventCode = PREPROCESSED`
- remove unnecessary line payload content if only identity fields are present
- recursively strip milliseconds from UTC timestamps

Line construction inside `_update_rows()` uses:

- `PK`
- `OriginalOrderLineId`
- synthetic `Unique_Identifier = "<PK>__<OriginalOrderLineId>"`
- optional `Priority`
- optional `OriginalOrderLineNotes`
- optional `OriginalOrderLineInstruction`

## 13. Order Update and Release Pipeline

Once payloads are built:

- payloads are chunked using `limits.orders_chunk_update`
- each chunk is sent to Manhattan `originalOrder/bulkImport`
- per-order success and error information is parsed from Manhattan messages
- successful order IDs are accumulated
- orders with `MaximumStatus >= 1000` are excluded from release during the migration period
- releasable IDs are chunked using `limits.orders_chunk_release`
- each chunk is sent asynchronously to Manhattan `order/criteria/evaluatePipelineAndPriority`

```mermaid
sequenceDiagram
    participant FS as facility_service
    participant CS as customer_service
    participant OS as order_service
    participant CR as ConfigurationRules
    participant MA as ManhattanAPIClient
    participant ET as EventTracker
    participant RED as Redis
    participant DB as MSSQL

    FS->>CS: process_customer(customer, orders)
    CS->>OS: process_orders(orders, rules, sla_scenarios)

    loop each order
        OS->>CR: evaluate_order(order)
        CR->>ET: track_evaluate_result(...)
        CR-->>OS: enriched order or None
    end

    OS->>OS: prepare payloads

    loop update chunks
        OS->>MA: update_orders(facility, customer, payloads)
        MA-->>OS: success IDs + error messages
        OS->>ET: track_update_result(UPD_OK / UPD_ERR)
        ET->>RED: send successful IDs to update set
    end

    loop release chunks
        OS->>MA: release_orders(facility, customer, ids)
        Note over MA: asynchronous background thread
    end

    FS->>ET: save_logs()
    ET->>DB: bulk insert into dbo.mida_log
    FS->>ET: clear_logs()
```

## 14. Manhattan API Adapter Behavior

`ManhattanAPIClient` owns authentication and all outbound HTTP POST calls.

Capabilities:

- password grant authentication on startup
- refresh-token renewal before expiry
- proxy support for HTTP and HTTPS
- order search
- bulk order update
- release pipeline triggering

Operational behavior:

- authentication failures abort the process
- request timeouts return `{success: False}`
- responses are parsed to extract business-key-based order errors
- customer authorization errors during search are handled by removing unauthorized customer codes and retrying
- release calls are fire-and-forget background threads

Endpoints currently used:

- `/oauth/token`
- `/dcorder/api/dcorder/order/search`
- `/dcorder/api/dcorder/originalOrder/bulkImport`
- `/dcorder/api/dcorder/order/criteria/evaluatePipelineAndPriority`

## 15. Tracking, Logging, and Operational Observability

`EventTracker` is the operational bridge between the processing pipeline and the monitoring/audit sinks.

It maintains an in-memory SQL buffer and flushes it at the end of each facility-processing cycle.

Tracked event types:

- `MATCH_OK`
- `MATCH_ERR`
- `UPD_OK`
- `UPD_ERR`
- `GENERIC`

Sinks:

- **MSSQL**
  - bulk insert into `dbo.mida_log`
  - `source` is hardcoded as `PREPROC`
- **Redis set**
  - successful updates are added to the configured update queue
  - downstream consumers can check release completion later
- **Redis hash**
  - API call payloads can be stored with retention

```mermaid
flowchart TD
    A["Rule evaluation or update result"] --> B["EventTracker"]
    B --> C["Append SQL entry in memory"]
    B --> D{"UPD_OK?"}
    D -- "Yes" --> E["Push order metadata to Redis set"]
    D -- "No" --> F["No Redis queue update"]
    C --> G["save_logs() at end of run"]
    G --> H["Chunk entries in groups of 2000"]
    H --> I["Insert into dbo.mida_log"]
```

## 16. Deployment and Runtime Boundaries

The service is built as a containerized Python application.

Runtime boundaries visible in the repository:

- `Dockerfile`
- `entrypoint.sh`
- `.github/workflows/*`
- `configurations/`
- `manhattan_templates/`

Environment-specific behavior is driven by:

- `REGION`
- `ENV`
- `CONFIG_DIR`
- `TEMPLATES_DIR`
- secret values referenced through `${...}` placeholders

Regional files exist for:

- `apac`
- `emea`
- `latam`
- `noram`

Environment variants exist for:

- `dev`
- `test`
- `prod`

## 17. Operational Limits and Safeguards

Current safeguards from configuration and code include:

- polling interval control with `process.interval_seconds`
- soft execution threshold with `process.max_execution_seconds`
- skip facility when `PREPROCESSED` backlog exceeds `limits.max_preprocessed`
- max fetched orders per execution with `limits.max_orders_per_execution`
- API paging with `limits.api_fetch_page_size`
- update chunking with `limits.orders_chunk_update`
- release chunking with `limits.orders_chunk_release`
- explicit warnings when `force_facilities` is enabled

## 18. Architecture Boundaries

```mermaid
flowchart LR
    subgraph Core["Core Processing Logic"]
        SVC["services/*"]
        ENG["core/configuration_rules.py"]
        BO["business_objects/*"]
        SUP["support_functions/order_utils.py"]
    end

    subgraph Infrastructure["Infrastructure and Integration"]
        REP["core/repositories/*"]
        ADP["core/adapters/*"]
        TRK["core/tracking/*"]
        CFG["helpers/config_loader.py"]
    end

    subgraph External["External Platforms"]
        X1[("MIDA MSSQL")]
        X2[("Manhattan Active")]
        X3[("Redis")]
        X4[("Config files")]
    end

    Core --> Infrastructure
    Infrastructure --> External
```

## 19. Current Design Observations

The current design has a few notable characteristics that are useful to keep in mind when extending the system:

- `Runtime` acts as a global dependency container rather than explicit dependency injection
- service functions are procedural, not class-based
- repositories return rich domain objects but depend directly on `Runtime`
- the rule engine mutates `Order` objects in place
- release is asynchronous and not awaited
- tracking is buffered in memory and flushed only at the end of facility processing
- Manhattan search and update logic is tightly coupled to current business filters and payload conventions

## 20. Recommended Usage Notes

This document is most useful together with:

- `src/main.py`
- `src/runtime.py`
- `src/services/*.py`
- `src/core/configuration_rules.py`
- `src/core/adapters/manhattan_api_client.py`
- `src/core/tracking/event_tracker.py`

If the implementation changes, this document should be updated together with:

- configuration keys
- Manhattan endpoints
- repository query semantics
- Redis queue behavior
- rule-evaluation semantics

## 21. Rendering Notes

- GitHub and GitLab render Mermaid diagrams directly in Markdown
- VS Code can render Mermaid with the appropriate extension
- Confluence usually supports Mermaid only through an app or macro
- Jira does not natively render Mermaid in issue descriptions in most setups

