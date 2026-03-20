# Changelog

All significant changes for this project are documented in this file.

The structure of the entries is maintained according to **[Keep a Changelog](https://keepachangelog.com/en/1.1.0/)** for clear categorization of updates.

Details on our internal versioning scheme (e.g., `YYMM.S.P`) can be found in the **[README.md](README.md#versioning-scheme)** file.

---

## [2602.2.2] - UAT: 2026-02-11 - PROD: unreleased

### ⚠️ Status Summary

- **PROD Deployment** not scheduled.

### Added

- **Configurable order search templates**: added `constants.templates.search` to perform order searches retrieving only the required fields instead of the full order payload.

---

## [2603.2.1] - PROD: 2026-03-19

### ⚠️ Status Summary

- **Hotfix** applied to correct an issue in the order search query.
- Updated condition from `status = 0000` to `status <= 1000`.
- The correct implementation was **already present in both dev and test**, so no backport was required.

---

## [2602.2.2] - UAT: 2026-02-20 - PROD: 2026-02-20

### ⚠️ Status Summary

- Released to UAT and PROD environments.

### Changed

- Updated the query used to retrieve orders to be processed:
  - From: orders in status `CREATED (0000)` with `eventCode IS NULL`.
  - To: orders with status `<= RELEASED (1000)` and `eventCode IS NULL`.
- Skipped the pipeline call (order release) when the order status is already `RELEASED`.

---

## [2601.1.19] - UAT: 2025-12-18 - PROD: 2026-01-13

### ⚠️ Status Summary

#### Customers in PROD as of 2026-01-28

- APAC region fully migrated to production.

#### Customers in PROD as of 2026-01-26

- EMEA region fully migrated to production.

#### Customers in PROD as of 2026-01-22

| Facility code | Customer code | Customer name      |
|---------------|---------------|--------------------|
| ITLIS01FN     | ASSAABLOY     | Assa Abloy         |
| ITLIS01FN     | CID258409     | Twenty-two Devices |
| AE_5000_DFN   | CID004183     | Breville           |
| GB_5081_DFN   | CID004183     | Breville           |
| GB_5081_DFN   | CID213248     | HMD Global         |
| ZA_0218_DFN   | CID009363     | Oriflame           |
| NL_0267       | CID228174     | Butterfly          |
| GB_0629_DFN   | CID248856     | Tru Earth          |
| GB_0629_DFN   | CID263376     | The 4th Utility    |
| GB_0629_DFN   | CID257277     | OHS                |
| DE_0710       | CID251398     | Solera             |
| DE_0710       | CID253349     | MOMI               |
| DE_0710       | CID257752     | RMG                |
| DE_0710       | CID265005     | GrowTechnology     |
| GB_5028_DFN   | CID253934     | Adyen              |
| SE_0002_DFN   | CID257733     | Winshape           |
| CZ_0020       | CID254832     | Noima              |
| CZ_0020       | CID263015     | The Noble Shoe     |
| CZ_0020       | CID263379     | SUPERCELL          |
| CZ_0020       | CID263462     | 5am Fuel           |
| FR_0033_DFN   | CID264937     | Ei Electronics     |
| DE_0700_DFN   | CID265162     | Wyrld              |
| ES_0432       | CID249432     | Miravia            |


#### Customers in PROD as of 2026-01-19

| Facility code | Customer code | Customer name |
|---------------|---------------|---------------|
| GB_5081_DFN   | CID215686     | Deliveroo GB  |
| FR_0430_DFN   | CID215686     | Deliveroo FR  |
| ITLIS01FN     | CID215686     | Deliveroo IT  |
| DE_0305_DFN   | CID228157     | AVM           |

#### Customers in PROD as of 2026-01-13

| Facility code | Customer code | Customer name    |
|---------------|---------------|------------------|
| CZ_0185_DFN   | CID246109     | Wolt             |
| DE_0305_DFN   | CID264883     | Rubinum          |
| DE_0305_DFN   | CID265220     | Heybee           |
| DE_0710       | CID267062     | Fashion Knit     |
| CZ_0020       | CID254751     | Giga Computer    |
| CZ_0020       | CID247598     | Ferm             |
| ES_0432       | CID247598     | Ferm             |
| SE_5008_DFN   | CID247598     | Ferm             |
| FR_0033_DFN   | CID258209     | Maison Ernest    |
| GB_0629_DFN   | CID246781     | Ostrich Pillow   |
| NL_0267       | CID254675     | Bamboi           |
| NL_0267       | CID260675     | Red Light Rising |

### Added

- **Manhattan API Timeout**: Added configurable `timeout` for POST requests in the Manhattan adapter. This prevents the process from hanging indefinitely in case of network latency or MA API unavailability, ensuring the Pod can terminate/restart correctly as per the new reliability logic.
- **Manhattan API**: Quotes have been added to all queries using `OrderId` to properly escape special characters.
- **Dynamic configuration via ConfigMaps**: configuration is no longer embedded in the Docker image at build time. It is now provided through ConfigMaps generated via CI/CD, allowing runtime updates without rebuilding.
- **Region-level configuration overrides**: added support to override values defined in `constants.yaml` by specifying them in region-specific configuration files.
- **Manhattan API Order Search**: added a filter to exclude orders with `PipelineId = "Add iLPN to Order"`, which are outbound-related (currently used only by Swappie) and must never be processed or released by MIDA.
- **`ANY_ITEM_FIT_INTO` Function:** Added a utility function that returns `True` if at least one item within the **order lines** fits within all the provided dimensional constraints:
  - `max_dim`: Iterable of (Height, Width, Length) limits (order-agnostic).
  - `max_vol`: Maximum allowed UnitVolume.
  - `max_weight`: Maximum allowed UnitWeight.
  - `max_girth`: Maximum allowed Girth (calculated as 2 x small + 2 x medium + largest dimension).
  - `tol`: Numeric tolerance for constraint checking.

### Changed

- **SLA Scenarios** are now evaluated on the enriched order instead of the initial raw order retrieved from Manhattan.
- **DateTime management system:** all values recognized as DateTime are truncated by removing milliseconds.
- **OrderLines handling**: when no changes are detected, or when the order has no lines, the `OrderLines` field is no longer included in the JSON payload.
- **Forced execution configuration**: the ability to force execution by facility, customer, or order has been moved from environment variables to `constants.yaml` for improved maintainability and clarity.
- **Context and configuration loading** fully refactored.
  - Introduced a centralized **Runtime** class to provide access to application configuration and shared clients.
  - Added **Pydantic-based validation** for all configuration objects.
- **Manhattan API**: Filter orders with `OrderProcessTypeId = NULL`.
- **Update Payload**: the `OriginalOrderLine` field is now included in the payload only if at least one line contains changes beyond default keys.
- **Order Processing**: orders with a single quote (') in the order ID are now excluded from processing.
- **Logging: refactored**: API_CALL logging to store full request details in Redis and send only a reference hash to Loki, reducing label size and improving log stability.
- **AEval logic for rules and scenarios**: updated evaluation logic to use payload together with order fields.

### Removed

- **Fixed fields:** all fields are now updated in both Order and OriginalOrder.
- **Bulk to order**: we only update the Original Order, that auto propagate to the order.
- **Order update behavior**: the update request now includes only the `Order` object. Previously, both `Order` and `OriginalOrder` were sent in the update. Manhattan correctly propagates changes from `OriginalOrder` to `Order`, making the duplicate update unnecessary.
- **Null Pattern** in the MS SQL client, as it was unused.

---

## [2511.1.25] - UAT: 2025-11-18 - PROD: aborted

### ⚠️ Status Summary

- **PROD deployment aborted** due to a DHL Link issue: the DHL Link APIs cannot handle a standard UTC timestamp in the format `yyyy-mm-ddThh:mm:ss.sss`, so we must force the following fields to `null`: `PickupStartDateTime`, `DeliveryStartDateTime`, `PickupEndDateTime`, `DeliveryEndDateTime`.
- While investigating this issue, a new requirement was raised by the customer: **SLA Scenarios** must now be applied on the already processed payload rather than on the raw order coming from Manhattan. Because of this, the deployment was aborted to fix both problems.
- **Root Cause:** The DHL Link APIs cannot handle a standard UTC timestamp in the format `yyyy-mm-ddThh:mm:ss.sss`.

### Added

- **Order Note Evaluation for Instructions:** Implemented logic to evaluate specific order notes (based on a defined `TypeId` and `CodeId`) and automatically transfer their content to the respective order instructions.
    - Supports both **header-level** and **line-item** instructions.
    - **Note Sequence Logic:** New notes are appended, starting from the current maximum sequence number found on the order notes plus one (`MAX_SEQUENCE + 1`).
- **`ALL_ITEMS_FIT_INTO` Function:** Added a utility function that returns `True` if and only if every item within the **order lines** fits within all the provided dimensional constraints:
    - `max_dim`: Iterable of (Height, Width, Length) limits (order-agnostic).
    - `max_vol`: Maximum allowed UnitVolume.
    - `max_weight`: Maximum allowed UnitWeight.
    - `max_girth`: Maximum allowed Girth (calculated as 2 x small + 2 x medium + largest dimension).
    - `tol`: Numeric tolerance for constraint checking.
- **`CHECK_SINGLE_LINE_ORDER` Function:** Added a utility function that returns `True` if and only if the order has exactly one **order line**.
- **New Service Layer Functions:**
    - `customer_service`.
    - `facility_service`.
- **Support for Loki Integration:** Introduced `LokiSink` to centralize logging and facilitate structured log analysis.
- **Structured API Logging:**
    * Added a custom log level, **`API_CALL` (Level 25)**, dedicated to tracking external API requests.
    * All outgoing `POST` requests made to the Manhattan API are now logged using the `API_CALL` level.
    * Implemented dynamic labels (`facility`, `customer`, `order`) in the Loki context to significantly improve log filtering and tracing capabilities.
- **Asynchronous Order Processing Queue (Redis):** Introduced a dedicated Redis queue to decouple and improve order processing flow:
    * `mida-processor` now pushes orders with status `UPD_OK` to the queue.
    * `mida-releaser` consumes the queue and moves orders to `REL_OK` only upon successful release.
- **Customer List Retry Mechanism:** Safety layer, ensuring any unauthorized customers are automatically filtered out before or during each API call.

### Fixed

- **`DhlEventCode` Relocation:** Corrected the data model where `DhlEventCode` was directly set on the `Order` object. It is now correctly nested under `Order` -> `extended` object.
- **API Response Safety:** Improved safe evaluation and handling of responses received from the Manhattan API. This prevents errors from unexpected or malformed response formats.

### Changed

- **Repository and Service Refactoring:** Refactored the following classes into support functions (module-level functions) to eliminate the need for instantiation, favoring a cleaner, functional pattern:
    - `customer_repository`
    - `facility_repository`
    - `order_repository`
    - `rule_repository`
    - `scenario_repository`
    - `sla_repository`
    - `order_service`
    - `manhattan_api_client`
- **`main` Execution Logic:**
    - Facility and customer specific logic has been successfully moved to the new `customer_service` and `facility_service` modules.
    - Added process logic and a **while-true schedule loop**. The main execution loop is now run and automatically terminates after exceeding a predefined maximum execution time threshold.
- **MIDA Logic Change (Focus Shift):** The application now **only evaluates non-`PREPROCESSED` orders** and runs a single **`RELEASE` action** for them. The **`RE-RELEASE`** action (in case of a pipeline failure) has been delegated to an external tool: [mida-release](https://git.dhl.com/EU-FFN/mida-releaser).
- **Logging Refactor:** General text improvements across log messages for better clarity and context.
- To avoid propragation errors: Switch OriginalOrder and Order update. Remove PREPROCESSED from Order update. Add more safety check to responses.
- **Row Order Update:** Rows update has been shifted from the `Order` field to the `OriginalOrder` field. This change addresses an issue where modifications made to the `Order` field were inadvertently overwritten by the automatic setting of the `OriginalOrder`.

### Removed

- **Prometheus Metrics:** All Prometheus metrics have been removed from this project (metrics collection is now handled by Grafana).

### Jira

- [**FNPD-7157** - Order instructions visibility](https://dhl.atlassian.net/browse/FNPD-7157)
- [**FNPD-7158** - Add function to MIDA to check all articles in the orders respect given constraints (max dim, max vol, max weight, max girth)](https://dhl.atlassian.net/browse/FNPD-7158)
- [**FNPD-7799** - MIDA Function -ANY_ITEM_FIT_INTO()](https://dhl.atlassian.net/browse/FNPD-7799)

---

## [2510.1.0] - UAT: 2025-10-20 - PROD: aborted

### ⚠️ Status Summary

- **PROD Deployment Aborted** due to critical integration flaw identified during UAT.
- **Root Cause (False Negative):** The Manhattan release pipeline call is asynchronous. The client frequently receives a **HTTP 500 error** (Manhattan timeout, 20s) even though the pipeline is successfully initiated. This leads to inaccurate status reporting.

### Added

- **Centralized Configuration Module (`context.py`)**: Introduced a dedicated module for robust application configuration. It validates the environment (`DEV`, `TEST`, etc.), loads settings from environment-specific **YAML files**, and automatically resolves **environment variables** (`${VAR_NAME}`).
- **Centralized Constants**: Introduced **`config.py`** and **`constants.yaml`** to centralize all non-variable settings and constants, improving code clarity.
- **`EventTracker`**: Implemented a new utility to manage persistent logs. Currently relies on `LegacySQLLogger` but is designed for future extension with a dedicated tracking system.
- **Added Test Overrides for Order Processing.** Introduced `FORCE_FACILITY`, `FORCE_CUSTOMER`, and `FORCE_ORDER` environment variables. These variables allow developers to **force specific processing contexts during local testing** (e.g., simulating a specific facility's configuration or a unique order ID for debugging).
- **SQL Logging for patching calls:** Introduced two new `LegacyActionType` codes to specifically log errors occurring during the **batch execution** (patch) of order updates and releases, as opposed to single-order errors:
    - `UPDATE_ERR`: Logged when the *set* of order updates (patch call) fails.
    - `RELEASE_ERR`: Logged when the *set* of order releases (patch call) fails.
    - `GENERIC_ERR`: For future use.

### Changed

- **General Code Refactoring**: Extensive internal code refactoring performed to improve maintainability, reduce complexity, and standardize coding practices across the codebase.
- **Configuration System Overhaul**: Migrated the configuration framework:
  - Switched configuration file format from **JSON to YAML**.
  - Configuration access is now managed dynamically via the **`ENV`** and **`CONFIG_DIR`** environment variables.
- **Tooling & Dependencies**:
  - Project dependency management and deployment pipeline migrated to **[UV](https://github.com/astral-sh/uv)**, defined via **`pyproject.toml`**.
  - Migrated logging library from `dhl_logger` to **`loguru`** for enhanced performance and simplified log management.
  - Debug verbosity setting moved from configuration files to the **`DEBUG_LEVEL`** Environment Variable.
- **API Integration Logic Refinement**:
  - API calls to retrieve orders are now made **per facility** only (previously per facility + customer).
  - Consolidated order update/release into a **single unique API call** per customer.
- **Proxy Settings**: Proxy configurations were moved from `configurations` to Environment Variables (`PROXY_HTTP`, `PROXY_HTTPS`) for increased security and flexibility (both optional).
- **Prometheus Base URL is now configurable.** The base URL for Prometheus API calls has been moved from being hardcoded within the `prometheus_client.py` module to the application's configuration files. This significantly improves deployment flexibility and maintainability.
- **Prometheus Collector Refactoring:** The logic for exposing custom application metrics has been refactored into a dedicated collector class to ensure thread-safe metric registration and adherence to Prometheus's best practices.
- Refactor **Prometheus metrics**:
  - `last_exec_time`: Track the time since the last execution.
  - `job_total_exec_time_seconds`: Gauge reporting the total time elapsed (in seconds) for the job to complete its run.
  - `job_last_exec_time`: Gauge reporting the **timestamp** (Unix epoch) of when the job successfully finished execution.
  - `number_of_order_processed`: Tracks the job's throughput to monitor performance over the last hour and signal potential processing issues related to a specific customer or facility.
- Refactored *alerting via MS Teams* when the metric indicates potential problems.

### Security

- **Secrets Moved to Environment Variables**: All sensitive credentials (passwords, secrets, tokens) have been removed from configuration files and are now exclusively managed via environment variables:
  - `DB_PWD`
  - `MANHATTAN_PWD`

### Removed

- **Legacy Code Cleanup**: Removed files and folders related to retired projects and components.
- **Global Rules Logic**: Eliminated the legacy functionality that applied "global rules" taken from facility `*` and customer `*` to every order.
- **Unused Functions**: Removed deprecated support functions (`DAYS_BETWEEN` and `GET_ZONE`).

### Jira

- [**FNPD-7063** - Bulk import API for order update and release](https://dhl.atlassian.net/browse/FNPD-7063)
- [**FNPD-7057** - Download orders by warehouse](https://dhl.atlassian.net/browse/FNPD-7057)