# Oracle AI Database Metrics Exporter — Copilot Instructions

## Project Overview

This is the **Oracle AI Database Metrics Exporter** (`oracledb_exporter`), a Prometheus/OTEL-compatible metrics exporter for Oracle AI Database. It scrapes Oracle DB metrics via SQL queries and exposes them at an HTTP endpoint. The project also exports alert logs in JSON format.

- **Module**: `github.com/oracle/oracle-db-appdev-monitoring`
- **Version**: 2.2.2 (set via `LDFLAGS` at build time)
- **Default port**: `:9161`
- **Default metrics path**: `/metrics`
- **License**: Universal Permissive License v1.0 + MIT

## Language & Runtime

- **Language**: Go (currently `go 1.25.7`)
- **Build tags**: `godror` (default) or `goora` to select the Oracle DB driver
- **CGO**: Required when building with the `godror` driver (`CGO_ENABLED=1`)

## Key Dependencies

| Package                                                             | Purpose                                                  |
| ------------------------------------------------------------------- | -------------------------------------------------------- |
| `github.com/godror/godror`                                          | Oracle DB driver (default, CGO, uses Oracle client libs) |
| `github.com/sijms/go-ora/v2`                                        | Pure-Go Oracle DB driver (no CGO, build tag `goora`)     |
| `github.com/prometheus/client_golang`                               | Prometheus metrics exposition                            |
| `github.com/prometheus/exporter-toolkit`                            | TLS/auth web server toolkit                              |
| `github.com/BurntSushi/toml`                                        | TOML parsing for metrics files                           |
| `gopkg.in/yaml.v2`                                                  | YAML parsing for config files                            |
| `github.com/alecthomas/kingpin/v2`                                  | CLI flag parsing                                         |
| `github.com/oracle/oci-go-sdk/v65`                                  | OCI Vault integration                                    |
| `github.com/Azure/azure-sdk-for-go/sdk/security/keyvault/azsecrets` | Azure Vault integration                                  |
| `github.com/hashicorp/vault/api`                                    | HashiCorp Vault integration                              |

## Project Structure

```
main.go                         # Entry point: CLI flags, HTTP server, Prometheus registration
collector/
  collector.go                  # Exporter struct, NewExporter, Describe/Collect implementation
  config.go                     # MetricsConfiguration, DatabaseConfig, VaultConfig YAML structs
  database.go                   # Database struct, connection management, UpMetric
  metrics.go                    # isScrapeMetric, getScrapeInterval, parseFloat helpers
  types.go                      # Core types: Exporter, Database, Metric, Config, MetricsCache
  data_loader.go                # Loads and parses TOML/YAML metric definition files
  default_metrics.go            # Embedded default metrics
  default_metrics.toml          # Default SQL metric definitions (embedded)
  cache.go                      # MetricsCache read/write helpers
  connect_godror.go             # DB connection using godror driver (build tag: godror)
  connect_goora.go              # DB connection using go-ora driver (build tag: goora)
  errors.go                     # Error type helpers
  collector_test.go             # Unit tests for collector
  database_test.go              # Unit tests for database
alertlog/
  alertlog.go                   # Exports Oracle alert log in JSON format
azvault/
  azvault.go                    # Azure Key Vault credential fetching
hashivault/
  hashivault.go                 # HashiCorp Vault credential fetching
ocivault/
  ocivault.go                   # OCI Vault credential fetching
custom-metrics-example/         # Example TOML metric files
docker-compose/                 # Full local stack (Oracle DB, Prometheus, Grafana)
kubernetes/                     # Kubernetes deployment manifests
site/                           # Docusaurus documentation site
```

## Configuration

### Config File (YAML)

Primary configuration is a YAML file passed via `--config.file` or `CONFIG_FILE` env var. Key sections:

```yaml
databases:
  <name>:
    username: ...
    password: ...
    url: host:port/service # Oracle connect string
    queryTimeout: 5 # seconds
    maxOpenConns: 10
    maxIdleConns: 10
    tnsAdmin: /path/to/wallet # optional Oracle Wallet
    externalAuth: false
    role: SYSDBA # optional
    labels: # optional const labels added to all metrics from this DB
      env: prod
    vault: # optional, mutually exclusive vault providers
      oci: ...
      azure: ...
      hashicorp: ...
metrics:
  default: default-metrics.toml # path to default metrics TOML
  custom:
    - /path/to/custom.toml
```

### Metric Definition Files (TOML or YAML)

Custom metrics are defined in TOML (preferred) or YAML files:

```toml
[[metric]]
context = "sessions"
labels = ["status", "type"]
metricsdesc = { value = "Gauge metric with current count of sessions by status and type." }
request = "SELECT status, type, COUNT(*) as value FROM v$session GROUP BY status, type"
scrapeInterval = "60s"     # optional per-metric interval
queryTimeout = "10s"       # optional per-metric timeout
databases = ["default"]    # optional: restrict to specific databases
```

Supported metric types: `gauge` (default), `counter`, `histogram`.

## Build Commands

```bash
# Local build (current OS/arch)
make local-build          # outputs to ./dist/

# Cross-compile
make go-build-linux-amd64
make go-build-linux-arm64
make go-build-darwin-arm64
make go-build-windows-amd64

# Docker image (AMD64)
make docker               # builds container-registry.oracle.com/database/observability-exporter:<version>-amd64

# Run local docker-compose stack
make run
```

Build outputs go to `./dist/oracledb_exporter-<version>.<os>-<arch>/`.

## Running Tests

```bash
go test ./...
go test ./collector/...
```

## CLI Flags (key ones)

| Flag                      | Env Var                 | Default                | Description                               |
| ------------------------- | ----------------------- | ---------------------- | ----------------------------------------- |
| `--config.file`           | `CONFIG_FILE`           | ``                     | Path to YAML config file                  |
| `--default.metrics`       | `DEFAULT_METRICS`       | `default-metrics.toml` | Default metrics TOML file                 |
| `--custom.metrics`        | `CUSTOM_METRICS`        | ``                     | Comma-separated custom metrics TOML files |
| `--web.telemetry-path`    | `TELEMETRY_PATH`        | `/metrics`             | Metrics endpoint path                     |
| `--query.timeout`         | `QUERY_TIMEOUT`         | `5`                    | Query timeout in seconds                  |
| `--scrape.interval`       | —                       | `0s`                   | Global scrape interval (0 = on-demand)    |
| `--database.maxIdleConns` | `DATABASE_MAXIDLECONNS` | `10`                   | Connection pool max idle                  |
| `--database.maxOpenConns` | `DATABASE_MAXOPENCONNS` | `10`                   | Connection pool max open                  |

## Key Patterns & Conventions

- **Metric namespace**: `oracledb` (defined in `collector/collector.go`)
- **Database label**: configurable via `databaseLabel` in config; added to all metrics as a const label to distinguish multi-database scrapes
- **Driver selection**: controlled by build tags (`godror` vs `goora`); `godror` requires Oracle Instant Client installed
- **Per-metric scrape intervals**: metrics can override the global scrape interval; cached results are returned between intervals
- **Vault integration**: credentials can be fetched from OCI Vault, Azure Key Vault, or HashiCorp Vault at startup
- **Alert log**: exported separately as JSON at a configurable HTTP path; controlled by `--log.*` flags
- **DSN masking**: passwords are masked in logs via `maskDsn()` before logging connection strings
- **Copyright header**: all source files begin with `// Copyright (c) <years>, Oracle and/or its affiliates.`

## Metric Types

The `Metric` struct (in `collector/types.go`) supports:

- `gauge` — default, point-in-time value
- `counter` — monotonically increasing
- `histogram` — requires `buckets` configuration

## Vault Secret Management

Credential vault is configured per-database under `vault:`. Only one provider per database:

- **OCI**: uses instance principal or API key auth
- **Azure**: uses managed identity or service principal
- **HashiCorp**: connects via proxy socket, supports `kv` and `database` mount types
