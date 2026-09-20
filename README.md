# Mulesoft Library Dashboard

A MuleSoft 4 application that aggregates data from three different backend systems — a REST API (Books), a SOAP Web Service (Members), and a MySQL database (Loans) — into a single unified JSON dashboard response.

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Project Structure](#project-structure)
- [Flows](#flows)
- [Connectors and Dependencies](#connectors-and-dependencies)
- [Configuration and Properties](#configuration-and-properties)
- [Running Locally (Anypoint Studio)](#running-locally-anypoint-studio)
- [Running MUnit Tests](#running-munit-tests)
- [API Endpoint](#api-endpoint)
- [Known Limitations](#known-limitations)

---

## Overview

The **Library Dashboard** exposes a single HTTP endpoint (`GET /library/dashboard`) that concurrently fetches:

| Data | Source | Protocol |
|---|---|---|
| Books / Products | External REST API | HTTP (JSON) |
| Members | External SOAP service | SOAP / WSDL |
| Loans | MySQL database | JDBC (DB Connector) |

All three results are fetched **in parallel** using a Scatter-Gather, then flattened into a single JSON array.

---

## Architecture

```
Client
  |
  v
GET /library/dashboard  (HTTP Listener :8082)
  |
  v
+------------------- Scatter-Gather -------------------+
|                                                       |
|  +- getBooks -------------------------------------- + |
|  |  HTTP Request -> localhost:7071/rest/products   | |
|  +-------------------------------------------------- + |
|                                                       |
|  +- getMembers ------------------------------------ + |
|  |  WSC Consume -> localhost:6060/ws (SOAP)        | |
|  +-------------------------------------------------- + |
|                                                       |
|  +- getLoans -------------------------------------- + |
|  |  DB Select -> MySQL (librarydb.loan table)      | |
|  +-------------------------------------------------- + |
+-------------------------------------------------------+
  |
  v
Transform Message (DataWeave)
  flatten(payload..payload)
  |
  v
Single JSON Array Response
```

---

## Project Structure

```
library/
|-- src/
|   |-- main/
|   |   |-- mule/
|   |   |   `-- library.xml          # Main Mule configuration & all flows
|   |   `-- resources/
|   |       |-- config.properties    # Local DB property values (Studio only)
|   |       |-- products.wsdl        # WSDL for the SOAP member service
|   |       |-- application-types.xml
|   |       `-- log4j2.xml
|   `-- test/
|       `-- munit/
|           `-- library-test.xml     # MUnit test suite (5 tests)
|-- pom.xml                          # Maven dependencies & Mule plugin config
|-- mule-artifact.json               # Mule app metadata
|-- rest.bat                         # Quick helper to call the REST backend
|-- soap.bat                         # Quick helper to call the SOAP backend
`-- README.md
```

---

## Flows

### `mainflow`
- **Trigger:** `GET /library/dashboard` on port `8082`
- **Logic:** Runs `getBooks`, `getMembers`, and `getLoans` in parallel via Scatter-Gather
- **Transform:** DataWeave `flatten(payload..payload)` merges all three route payloads into one flat JSON array
- **Response:** `application/json`

### `getBooks`
- Makes an HTTP GET request to `localhost:7071/rest/products`
- Returns a JSON array of book/product records

### `getMembers`
- Builds a SOAP XML request using DataWeave
- Calls the `getAllProducts` operation on the SOAP service at `localhost:6060/ws`
- Uses the `products.wsdl` descriptor

### `getLoans`
- Executes `SELECT * FROM loan` against the configured MySQL database
- Returns all active loan records

---

## Connectors and Dependencies

| Connector | Version | Purpose |
|---|---|---|
| `mule-http-connector` | 1.11.3 | HTTP Listener (inbound) & HTTP Request (outbound to REST) |
| `mule-wsc-connector` | 2.2.1 | SOAP Web Service Consumer |
| `mule-db-connector` | 1.16.2 | MySQL database queries |
| `mule-sockets-connector` | 1.2.7 | Required by HTTP connector |
| `mysql-connector-java` | 8.0.30 | MySQL JDBC driver (shared library) |

**Mule Runtime:** `4.12.1`  
**Mule Maven Plugin:** `4.10.1`

---

## Configuration and Properties

Database connection values are **externalized as property placeholders** and must never be hardcoded.

| Property | Description |
|---|---|
| `${db.host}` | MySQL host address |
| `${db.port}` | MySQL port (default: `3306`) |
| `${db.user}` | MySQL username |
| `${db.password}` | MySQL password |
| `${db.database}` | Database/schema name |

### For Local Development (Anypoint Studio)

Create or edit `src/main/resources/config.properties`:

```properties
db.host=localhost
db.port=3306
db.user=root
db.password=your_password
db.database=librarydb
```

> **Note:** Do not commit real passwords. `config.properties` should be added to `.gitignore` for production projects.

---

## Running Locally (Anypoint Studio)

### Prerequisites
- Anypoint Studio 7.x installed
- JDK 8 or 11
- MySQL running locally with the `librarydb` database and `loan` table
- REST products service running on port `7071`
- SOAP member service running on port `6060`

### Steps

1. **Clone the repository**
   ```bash
   git clone https://github.com/Riyaz510/Mulesoft-Library-Dashboard.git
   cd Mulesoft-Library-Dashboard
   ```

2. **Import into Anypoint Studio**
   - `File -> Import -> Anypoint Studio -> Anypoint Studio Project from File System`
   - Select the cloned folder

3. **Set up `config.properties`**
   - Create `src/main/resources/config.properties` with your local DB credentials (see above)

4. **Start the backend services**
   - Start your local MySQL and ensure the `librarydb.loan` table exists
   - Start the REST products service on `:7071`
   - Start the SOAP service on `:6060`

5. **Run the application**
   - Right-click the project -> `Run As -> Mule Application`

6. **Test the endpoint**
   ```bash
   curl http://localhost:8082/library/dashboard
   ```

---

## Running MUnit Tests

The test suite at `src/test/munit/library-test.xml` contains **5 tests** that mock all external dependencies — no real database, REST, or SOAP service needs to be running.

### Tests Included

| Test | Description |
|---|---|
| `getBooks-returns-product-list` | Mocks HTTP outbound call, asserts 2-book response |
| `getMembers-returns-member-list` | Mocks SOAP consume, asserts 2-member response |
| `getLoans-returns-loan-rows` | Mocks DB select, asserts 1-loan response |
| `mainflow-flattens-all-three-sources` | Full happy path — asserts Scatter-Gather flattens to 3 total items |
| `mainflow-fails-when-one-route-errors` | Documents failure behavior when one route (SOAP) is down — asserts `MULE:COMPOSITE_ROUTING` error |

### Run via Maven

```bash
mvn clean test
```

### Run a specific test

```bash
mvn clean test -Dmunit.test=library-test
```

### Run via Anypoint Studio

Right-click `library-test.xml` -> **Run MUnit Suite**

---

## API Endpoint

| Method | URL | Description |
|---|---|---|
| `GET` | `http://localhost:8082/library/dashboard` | Returns aggregated books, members, and loans as a flat JSON array |

### Sample Response

```json
[
  { "id": 1, "title": "Clean Code" },
  { "id": 2, "title": "Refactoring" },
  { "memberId": 101, "name": "Riyaz" },
  { "memberId": 102, "name": "Ayesha" },
  { "loanId": 1, "bookId": 1, "memberId": 101 }
]
```

---

## Known Limitations

> **No error handler on Scatter-Gather** — If any one of the three backend services (REST, SOAP, or DB) is unavailable, the **entire** `/library/dashboard` request will fail with a `MULE:COMPOSITE_ROUTING` error. The flow does not support partial/degraded responses. This is documented in Test 5 of the MUnit suite and is a known improvement area.
