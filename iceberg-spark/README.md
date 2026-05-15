# Iceberg Sink — Local Test Environment

End-to-end setup for testing the Iceberg sink connector with dynamic routing against a local Iceberg REST catalog backed by MinIO.

## Prerequisites

- Docker and Docker Compose
- A running Iggy server on `localhost:8090` (default credentials `iggy`/`iggy`)
- Rust toolchain (for building the connector)

## 1. Start the local Iceberg stack

From this directory:

```bash
docker compose up -d
```

This starts:

- **MinIO** on `localhost:9000` (S3-compatible store)
- **Iceberg REST catalog** on `localhost:8181`
- **Spark** on `localhost:10000`
- A one-shot `table-init` container that creates the `nyc.events` table automatically

Wait a few seconds for `table-init` to finish before sending messages.

## 2. Create the Iggy stream and topic

```bash
iggy stream create qw
iggy topic create qw example 1
```

## 3. Build the Iceberg sink plugin

From the repository root:

```bash
cargo build --release
cargo build --release -p iggy_connector_iceberg_sink

```

## 4. Run the connector

From the repository root:

```bash
export IGGY_CONNECTORS_CONFIG_PATH=core/connectors/runtime/example_config/config.toml
export IGGY_CONNECTORS_IGGY_PASSWORD=ukFMgFm3P6wXSsZR7Lw1CIUM
cargo run --bin iggy-connectors
```

The connector loads its configuration from `core/connectors/runtime/example_config/config.toml` and the sink config from `core/connectors/runtime/example_config/con/iceberg_sink.toml`.

It will listen on stream `qw`, topic `example`, and route each message to the Iceberg table named in the `db_table` field of the JSON payload.

This is how the iceberg_config looks like:

```toml
[[streams]]
stream = "qw"
topics = ["example"]
schema = "json"
batch_length = 100
poll_interval = "5ms"
consumer_group = "iceberg_sink_connector"

# Local S3 example
[plugin_config]
tables = []
catalog_type = "rest"
warehouse = "warehouse"
uri = "http://localhost:8181"
dynamic_routing = true
dynamic_route_field = "db_table"
store_url = "http://localhost:9000"
store_access_key_id = "admin"
store_secret_access_key = "password"
store_region = "us-east-1"
store_class = "s3"

[transforms.add_fields]
enabled = true

# [[transforms.add_fields.fields]]
# key = "db_table"
# value.static = "nyc.events"
```

Which will set dynamic routing to true and expect the table name from "db_table"

## 5. Send a test message

```bash
iggy message send qw example \
  '{"user_id":"u-123","user_type":1,"email":"john@example.com","source":"web","state":"NY","created_at":"2026-05-15T10:00:00Z","message":"User signed up","db_table":"nyc.events"}'
```

The `db_table` field drives dynamic routing — its value must match an existing Iceberg table in the format `namespace.table`.

## 6. Verify with Spark SQL

```bash
docker exec -it spark-iceberg spark-sql
```

```sql
REFRESH TABLE nyc.events;
SELECT * FROM nyc.events;
```
