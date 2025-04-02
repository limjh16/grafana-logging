# Logging with Grafana, Alloy, and Loki

This repository includes config and `docker-compose.yml` files to set up a remote logging server with Grafana.

## Supported logging sources

New sources can be added as long as Alloy / Loki supports them, but as of now, raw http (like [curtin](https://curtin.readthedocs.io/en/latest/topics/reporting.html#example-http-request) uses during ubuntu-server installation), syslog and docker are built in to the configuration files.

- Point syslog to this machine's IP, port :5140, with TCP protocol.
    - By default, `config.alloy` is set up to injest syslog messages with **RFC 3164** format to accomodate ChirpStackOS and OpenWRT (this is not documented anywhere, but testing shows that it uses the older RFC 3164 format)
- Point webhook JSON log reporting to this machine's IP, port :5555, **with the endpoint `/loki/api/v1/raw`**

## Integration

This repository should ideally be included as a git submodule in a larger Grafana stack, integrated with other metrics / dashboards.

Example integration with a larger `docker-compose.yml`:

```yaml
services:

  # ... other services

  alloy-logging:
    extends:
      file: grafana-logging/docker-compose.yml
      service: alloy-logging
    networks:
      - grafana-network # remember to change this if another network name is defined

  loki:
    extends:
      file: grafana-logging/docker-compose.yml
      service: loki
    networks:
      - grafana-network # remember to change this if another network name is defined

  networks:
    grafana-network:
```

To provision Loki by default without clicking through the webui, mount [`grafana/provisioning/datasources/datasource-loki.yml`](./config/grafana/provisioning/datasources/datasource-loki.yml) in the Grafana container. Example:

```yaml
services:
  grafana:
    # ... other config
    volumes:
      - ./grafana-logging/config/grafana/provisioning/datasource-loki.yml:/etc/grafana/provisioning/datasource-loki.yml:ro
```

## Development

### Quick Start and Testing

Running `docker compose up -d` in this repository should spin up the required containers. Access the webui at port `:13000`, then you can explore logs at the `Explore > Logs` tab in the sidebar

### Data flow

Syslog / JSON over HTTP / Docker / ... --> Alloy (Agent) --> Loki (Server) --> Grafana (Display)

- Alloy is a OTel collector, it manages how data flows into the system, thus, inputs and ports are all defined with the `alloy` docker service or in `config.alloy`
- Loki is a Log Aggregator, it injests and stores logs, and manages the labelling / indexing / querying / storage of all the logs
    - Loki is currently set up in a Monolithic mode, however if there are too many logs to manage, refer to [the other possible deployment modes](https://grafana.com/docs/loki/latest/get-started/deployment-modes/)
    - Logs are currently persisted in the `loki-data` docker volume
- Grafana just reads and displays the logs from Loki. All queries keyed into Grafana are passed to Loki, and processed in Loki.

### Naming

- To allow for other deployment topologies, and potentially segmenting different Alloy instances, the Alloy service is deliberately named `alloy-logging`.
    - If scaling up in the future, refer to <https://grafana.com/docs/alloy/latest/set-up/deploy/>
- The Grafana service in this repository is named `grafana-logging` for separate testing in machines with multiple Grafana instances
    - NOTE! `grafana-logging` is intentionally set up with port `:13000` instead of the normal `:3000` for this purpose.

### Provisioning

To ensure that the Grafana setup is repeatable, we should always provision data sources and dashboards with an appropriate configuration file, instead of using the webui.

- Datasource YAMLs should be in `provisioning/datasources` directory, while Dashboard JSONs should be in `provisioning/dashboards` directory.
    - A [Loki datasource is included in this repository](./config/grafana/provisioning/datasources/datasource-loki.yml), however no dashboard JSONs have been created yet.
- Reference: <https://grafana.com/docs/grafana/latest/administration/provisioning>

### Labels

TODO: Loki has a [minimal indexing approach](https://grafana.com/oss/loki/), and only indexes the labels + timestamp. However, the webhook JSONs and RFC3164 syslog messages might not be processed ideally by default.

Reference:

- <https://grafana.com/docs/loki/latest/get-started/labels/> labels best practices
- <https://grafana.com/docs/alloy/latest/reference/components/loki/loki.relabel/> use this to relabel
