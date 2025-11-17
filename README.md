# Real-Time Change Data Capture (CDC) with PostgreSQL, Debezium, Kafka, and ClickHouse

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](https://opensource.org/licenses/MIT)

This project demonstrates a robust, real-time Change Data Capture (CDC) pipeline. It is designed to capture data modifications (INSERTs, UPDATEs, DELETEs) from a transactional PostgreSQL database, stream them through Apache Kafka using Debezium, and replicate them into a ClickHouse data warehouse for high-performance analytical queries. The entire environment is orchestrated using Docker Compose for easy setup and reproducibility.

This architecture is ideal for scenarios like feeding data to analytical platforms, creating audit trails, or synchronizing microservices without putting any load on the source database.

## Table of Contents

1.  [System Architecture](#system-architecture)
2.  [Technology Stack](#technology-stack)
3.  [In-Depth Component Breakdown](#in-depth-component-breakdown)
4.  [Project Directory Structure](#project-directory-structure)
5.  [Getting Started: Setup and Configuration](#getting-started-setup-and-configuration)
    -   [Prerequisites](#prerequisites)
    -   [Step 1: Clone the Repository & Prepare Configs](#step-1-clone-the-repository--prepare-configs)
    -   [Step 2: Launch the Docker Environment](#step-2-launch-the-docker-environment)
    -   [Step 3: Configure the Debezium PostgreSQL Connector](#step-3-configure-the-debezium-postgresql-connector)
    -   [Step 4: Verify the Debezium Connector Status](#step-4-verify-the-debezium-connector-status)
    -   [Step 5: Set Up the Database Tables](#step-5-set-up-the-database-tables)
6.  [Testing the CDC Pipeline](#testing-the-cdc-pipeline)
    -   [Step 1: Make Changes in PostgreSQL](#step-1-make-changes-in-postgresql)
    -   [Step 2: Observe the Change Events in Kafka](#step-2-observe-the-change-events-in-kafka)
    -   [Step 3: Query the Replicated Data in ClickHouse](#step-3-query-the-replicated-data-in-clickhouse)
7.  [Shutting Down the Environment](#shutting-down-the-environment)

## System Architecture

The data flows through the pipeline in the following sequence:

1.  **Source Database (PostgreSQL):** A user performs a standard `INSERT`, `UPDATE`, or `DELETE` operation on a table (e.g., `public.users`). This change is recorded in PostgreSQL's Write-Ahead Log (WAL).
2.  **Capture Agent (Debezium):** The Debezium PostgreSQL connector is configured to monitor the WAL. It reads the change events in a non-intrusive way, converts them into a structured JSON format, and publishes them to a specific Apache Kafka topic.
3.  **Message Bus (Apache Kafka):** Kafka acts as a durable, distributed, and scalable buffer for the change events. This decouples the source database from the destination systems, ensuring data resilience.
4.  **Analytical Sink (ClickHouse):** ClickHouse subscribes to the Kafka topic using its built-in `Kafka` engine. A `MATERIALIZED VIEW` is used to automatically read messages from the topic and insert the transformed data into a final, query-optimized table.
5.  **Query Federation (Trino):** Trino is configured to connect to ClickHouse, allowing users to run federated SQL queries against the replicated data from a unified endpoint.

 <!-- Replace with your actual architecture diagram image link -->

## Technology Stack

| Component               | Technology                                             | Role in Pipeline                                               |
| ----------------------- | ------------------------------------------------------ | -------------------------------------------------------------- |
| **Source Database**     | PostgreSQL                                             | The OLTP (transactional) database where changes originate.     |
| **CDC Platform**        | Debezium                                               | Captures row-level changes from PostgreSQL's WAL.              |
| **Message Broker**      | Apache Kafka                                           | Transports change events reliably from Debezium to consumers.  |
| **Coordination**        | Zookeeper                                              | Manages the Kafka cluster state.                               |
| **Data Warehouse**      | ClickHouse                                             | The OLAP (analytical) database for storing and querying data.  |
| **Query Engine**        | Trino                                                  | Provides a federated SQL interface to query ClickHouse.        |
| **Orchestration**       | Docker & Docker Compose                                | Manages the multi-container environment and networking.        |

## In-Depth Component Breakdown

-   **PostgreSQL:** Configured with `wal_level=logical` to enable the logical decoding feature required by Debezium.
-   **Debezium:** Acts as a Kafka Connect source. The connector configuration includes powerful transformations (`ExtractNewRecordState`) to flatten the complex Debezium event structure into a simple JSON object, making it easier for downstream consumers like ClickHouse to process.
-   **ClickHouse:** Uses a two-step process for data ingestion from Kafka. A `Kafka` engine table provides a direct, real-time link to the topic, and a `Materialized View` triggers on new messages to transform and insert them into a permanent `MergeTree` table, which is highly optimized for analytical queries.
-   **Trino:** The `clickhouse.properties` file in the `trino/catalog` directory contains the JDBC connection details that allow Trino to discover and query the schemas in the ClickHouse instance.

## Project Directory Structure

For the pipeline to work, your project must follow this directory structure. The config files are essential as they are volume-mounted into their respective containers.

```
project/
├── docker-compose.yml
├── clickhouse/
│   ├── config.xml
│   └── users.xml
└── trino/
    └── catalog/
        ├── postgresql.properties
        └── clickhouse.properties
```

## Getting Started: Setup and Configuration

### Prerequisites

-   **Git:** To clone the repository.
-   **Docker** and **Docker Compose:** Must be installed and running on your local machine.
-   **cURL:** A command-line tool for making HTTP requests (used to configure Debezium).

### Step 1: Clone the Repository & Prepare Configs

Clone this repository and ensure all configuration files (`config.xml`, `postgresql.properties`, etc.) are present and correctly configured as per the project's requirements.

```bash
git clone https://github.com/your-username/your-cdc-project.git
cd your-cdc-project
```

### Step 2: Launch the Docker Environment

From the root directory of the project, run the following command to build and start all services in detached mode.

```bash
docker-compose up -d
```

Use `docker ps` to verify that all containers (`postgresql`, `kafka`, `zookeeper`, `debezium`, `clickhouse`, `trino`) are up and running.

### Step 3: Configure the Debezium PostgreSQL Connector

Once the containers are running, you must register the PostgreSQL connector with the Debezium Connect service.

```bash
curl -X POST http://localhost:8083/connectors -H "Content-Type: application/json" -d '{
  "name": "postgres-connector",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "database.hostname": "postgresql",
    "database.port": "5432",
    "database.user": "postgres",
    "database.password": "postgres123",
    "database.dbname": "testdb",
    "topic.prefix": "dbserver1",
    "table.include.list": "public.users",
    "plugin.name": "pgoutput",
    "transforms": "unwrap",
    "transforms.unwrap.type": "io.debezium.transforms.ExtractNewRecordState",
    "transforms.unwrap.drop.tombstones": "false",
    "transforms.unwrap.delete.handling.mode": "rewrite",
    "transforms.unwrap.add.fields": "op,source.db,source.schema,table,lsn,source.ts_ms,source.connector,source.name",
    "slot.name": "debezium",
    "publication.name": "dbz_publication"
  }
}'
```

### Step 4: Verify the Debezium Connector Status

Check if the connector was created successfully and is running.

```bash
curl -X GET http://localhost:8083/connectors/postgres-connector/status
```

You should receive a JSON response where the `connector` state and `tasks` state are both `"RUNNING"`.

### Step 5: Set Up the Database Tables

You need to create the source table in PostgreSQL and the destination tables/views in ClickHouse.

#### In PostgreSQL:

```bash
docker exec -it postgresql psql -U postgres -d testdb
```

```sql
-- Inside the psql shell, run:
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  name VARCHAR(100),
  email VARCHAR(100),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

#### In ClickHouse:

```bash
docker exec -it clickhouse clickhouse-client
```

```sql
-- Inside the clickhouse-client shell, run these statements one by one:

-- 1. Create a table that maps to the Kafka topic
CREATE TABLE kafka_users (
  id UInt32,
  name String,
  email String,
  created_at UInt64,
  __op String,
  __table String,
  __source_ts_ms UInt64,
  __lsn UInt64
)
ENGINE = Kafka
SETTINGS
  kafka_broker_list = 'kafka:29092',
  kafka_topic_list = 'dbserver1.public.users',
  kafka_group_name = 'clickhouse-consumer-group',
  kafka_format = 'JSONEachRow';

-- 2. Create the final destination table
CREATE TABLE users (
  id UInt32,
  name String,
  email String,
  created_at DateTime64(3),
  op String,
  source_table String,
  source_ts_ms DateTime64(3)
)
ENGINE = MergeTree()
ORDER BY source_ts_ms;

-- 3. Create a Materialized View to automatically move data from Kafka to the final table
CREATE MATERIALIZED VIEW users_mv TO users AS
SELECT
  id,
  name,
  email,
  toDateTime64(CAST(created_at, 'UInt64') / 1000, 3) + INTERVAL 3 HOUR AS created_at,
  __op AS op,
  __table AS source_table,
  toDateTime64(CAST(__source_ts_ms, 'UInt64') / 1000, 3) + INTERVAL 3 HOUR AS source_ts_ms
FROM kafka_users;
```

## Testing the CDC Pipeline

### Step 1: Make Changes in PostgreSQL

Open a `psql` shell into the PostgreSQL container.

```bash
docker exec -it postgresql psql -U postgres -d testdb
```

```sql
-- Run some DML statements:
INSERT INTO users (id, name, email) VALUES (333, 'akif', 'akif@mail.com');
INSERT INTO users (id, name, email) VALUES (334, 'mehmet', 'mehmet@mail.com');
UPDATE users SET name = 'akif selim' WHERE id = 333;
DELETE FROM users WHERE id = 334;
```
<img width="768" height="247" alt="image" src="https://github.com/user-attachments/assets/279b50ab-6bb4-4170-aa2b-43a1288e8a6e" />

### Step 2: Observe the Change Events in Kafka

In a new host terminal, start a Kafka console consumer to see the raw change events published by Debezium.

```bash
docker exec -it kafka kafka-console-consumer \
  --bootstrap-server kafka:29092 \
  --topic dbserver1.public.users \
  --from-beginning
```

You will see detailed JSON messages for each `INSERT` (`c`), `UPDATE` (`u`), and `DELETE` (`d`) operation.
<img width="1516" height="253" alt="image" src="https://github.com/user-attachments/assets/a268e4cc-61fa-4789-8ffc-4bf48dc8390d" />


### Step 3: Query the Replicated Data in ClickHouse

Verify that the changes have been automatically replicated into the `users` table in ClickHouse using the Trino CLI.

```bash
docker exec -it trino trino
```

```sql
-- Inside the Trino CLI, run:
SELECT * FROM clickhouse.default.users ORDER BY source_ts_ms;
```
<img width="945" height="149" alt="image" src="https://github.com/user-attachments/assets/5fd2c17b-75ee-4f46-a433-8e33894c1901" />

The output will show the final state of your data, reflecting all the changes you made in PostgreSQL.

## Shutting Down the Environment

To stop and remove all the containers, networks, and volumes created by this project, run the following command in the project's root directory:

```bash
docker-compose down -v
```
> **Warning:** The `-v` flag will delete the named volumes, which means all your data in PostgreSQL and ClickHouse will be permanently lost. Omit this flag if you wish to preserve the data for the next run.
